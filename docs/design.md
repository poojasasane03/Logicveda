# PayTrack Phase 1 — Technical Design Document (`v1.0.0`)

**Client:** Saffron Trail Journeys (Simulated Client)  
**Domain:** Multi-Tenant Invoicing & Collections Platform  
**Stack:** Java 21, Spring Boot 3, Spring Security, Spring Data JPA, PostgreSQL (Flyway), Redis, RabbitMQ, React 18, TypeScript, JUnit 5, Testcontainers, Docker  

---

## 1. Architecture

PayTrack Phase 1 is structured as a modular monolith backend (**Spring Boot 3**) paired with a **React 18 + TypeScript** frontend, backed by **PostgreSQL** (system of record + Flyway migrations), **Redis** (caching / rate-limiting / session or token support), and **RabbitMQ** (asynchronous event processing such as recurring invoice triggers and PDF/notification jobs).

### Multi-Tenant Isolation
- Every distributor is an **Organisation** (tenant) isolated by `organisation_id` across all tenant-scoped tables and queries.
- Authentication/authorization extracts the caller's `organisation_id` and role into a request-scoped tenant context; repository and service methods enforce `organisation_id` filtering so a user from one organisation can never read or modify another organisation's invoices or records.
- **Immutability & Credit Notes:** Once an invoice is `ISSUED`, its financial fields and line items cannot be edited in place; corrections or reversals are recorded via a linked **Credit Note**.
- **Document Delivery:** Invoice and credit-note PDFs are generated on the server and downloaded via time-limited signed links.

---

## 2. Data Model

### Core Entities & Relationships
- **`organisations`**: `id`, `name`, `gstin`, `state_code`, `default_currency` (`INR`), `invoice_prefix`, `created_at`.
- **`users`**: `id`, `organisation_id` (FK), `email`, `full_name`, `role` (`OWNER`, `FINANCE_MANAGER`, `ACCOUNTANT`, `COLLECTOR`, `READ_ONLY`), `active`.
- **`customers`**: `id`, `organisation_id` (FK), `name`, `gstin`, `is_overseas`, `currency_code` (e.g., `INR`, `USD`, `EUR`), `place_of_supply_state_code`, `billing_address`, `shipping_address`, `payment_terms_days`, `credit_limit`.
- **`invoice_sequences`**: `id`, `organisation_id` (FK), `financial_year` (e.g., `2026-27`), `prefix`, `next_number`, `padding_length` — unique on `(organisation_id, financial_year)`.
- **`invoices`**: `id`, `organisation_id` (FK), `customer_id` (FK), `invoice_number` (nullable while `DRAFT`, unique per `(organisation_id, invoice_number)` once issued), `status` (`DRAFT`, `ISSUED`, `PARTIALLY_PAID`, `PAID`, `VOID`), `issue_date`, `due_date`, `currency_code`, `exchange_rate` (fixed on issue date for overseas invoices; `1.000000` for INR), `subtotal_amount`, `tax_amount_inr`, `total_amount`, `amount_paid`, `created_at`.
- **`invoice_line_items`**: `id`, `invoice_id` (FK), `organisation_id` (FK), `description`, `quantity`, `unit_price`, `discount_amount`, `tax_rate_percent`, `line_subtotal`, `cgst_amount`, `sgst_amount`, `igst_amount`, `line_total`.
- **`credit_notes`**: `id`, `organisation_id` (FK), `invoice_id` (FK), `credit_note_number`, `reason`, `amount`, `issued_at`.
- **`recurring_invoice_templates`**: `id`, `organisation_id` (FK), `customer_id` (FK), `day_of_month` (`1`–`28`), `active`, `last_generated_date`, `template_payload_json`.

---

## 3. Permission Matrix

Roles from FR-1.01 (`OWNER`, `FINANCE_MANAGER`, `ACCOUNTANT`, `COLLECTOR`, `READ_ONLY`) mapped against core actions:

| Action | Owner | Finance Manager | Accountant | Collector | Read-Only |
| :--- | :---: | :---: | :---: | :---: | :---: |
| Manage organisation settings & numbering series | ✅ | ✅ | ❌ | ❌ | ❌ |
| Manage users & role assignments | ✅ | ❌ | ❌ | ❌ | ❌ |
| Create & edit customers (GSTIN, addresses, terms, credit limit) | ✅ | ✅ | ✅ | ❌ | ❌ |
| View customers & invoices | ✅ | ✅ | ✅ | ✅ | ✅ |
| Create & update `DRAFT` invoices | ✅ | ✅ | ✅ | ❌ | ❌ |
| Issue `DRAFT` invoice (assigns number & locks immutability) | ✅ | ✅ | ✅ | ❌ | ❌ |
| Void `ISSUED` invoice or create linked Credit Note | ✅ | ✅ | ❌ | ❌ | ❌ |
| Record payment (`PARTIALLY_PAID` / `PAID`) | ✅ | ✅ | ✅ | ✅ | ❌ |
| Configure recurring monthly invoice templates | ✅ | ✅ | ✅ | ❌ | ❌ |
| Download invoice / credit-note PDF via signed link | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## 4. Tax Rule (Hand-Calculable)

### Tax Type Selection
1. **Intra-State (Same State):** Organisation state code equals Customer place-of-supply state code → split the line's GST rate equally into **CGST** (`rate / 2`) and **SGST** (`rate / 2`), with **IGST = 0**.
2. **Inter-State (Different State):** Organisation state code differs from Customer place-of-supply state code → apply full GST rate as **IGST**, with **CGST = 0** and **SGST = 0**.
3. **Client Rule — Overseas Customers:** Invoices for overseas customers are raised in the customer's **foreign currency** with the **exchange rate fixed on the issue date** and stored on the invoice. **Tax is calculated in INR** (using the issue-date exchange rate) and stored with the rate used.

### How to Apply by Hand (Step-by-Step)
1. **Compute Line Base Amount:**  
   $$\text{Base}_{\text{line}} = (\text{Quantity} \times \text{Unit Price}) - \text{Line Discount}$$
2. **If Overseas Customer (Client Rule):**  
   Fix the exchange rate $R$ on the invoice issue date and store $R$ on the invoice. Convert the taxable amount to INR:  
   $$\text{Taxable}_{\text{INR}} = \text{Base}_{\text{line}} \times R$$
   Compute GST in INR on $\text{Taxable}_{\text{INR}}$ using the applicable tax rate, and keep the invoice's billed currency amount in the foreign currency.
3. **Rounding Rule (Round Half Up to 2 Decimal Places):**  
   Round money amounts to **2 decimal places** (`0.01`) using **Half Up**: if the third decimal digit is **5 or greater** (`5, 6, 7, 8, 9`), round the second decimal digit **up by 1**; if it is **4 or less** (`0, 1, 2, 3, 4`), keep the second decimal digit unchanged.
   - Example A: `123.455` → third digit is `5` → **`123.46`**.
   - Example B: `123.454` → third digit is `4` → **`123.45`**.
4. **Invoice Totals:** Sum the rounded line base amounts for the subtotal, sum the rounded tax components for total tax, and add subtotal + tax for the grand total (all calculated on the server).

---

## 5. Numbering Rule (Hand-Calculable)

### Number Format & Financial Year
- Each organisation configures a numbering series per **financial year** (e.g., `2026-27`, running April 1 to March 31) with a prefix and zero-padded sequence width (for example, prefix `STJ-2627-` with 5-digit padding → `STJ-2627-00001`, `STJ-2627-00002`, ...).
- Draft invoices do **not** consume an official invoice number; a number is assigned only when transitioning an invoice to `ISSUED`.

### Zero Gaps & Zero Duplicates Under Concurrency
- When issuing an invoice inside a database transaction (`READ COMMITTED` / transactional service):
  1. Lock the matching `(organisation_id, financial_year)` row in `invoice_sequences` using a pessimistic row lock (`SELECT ... FOR UPDATE`).
  2. Read `next_number` (e.g., `101`), format the invoice number with zero-padding, and save it on the `ISSUED` invoice (enforced by a database unique constraint on `(organisation_id, invoice_number)`).
  3. Increment `next_number` to `102` and commit the transaction.
- Because concurrent requests wait on the row lock for that `(organisation_id, financial_year)` counter, **20 invoices issued in parallel** acquire the lock one after another and receive **20 consecutive numbers** (e.g., `00101` through `00120`) with no gaps from failed/aborted allocations and no duplicates.

### How to Apply by Hand
Suppose `organisation_id = ORG-1`, `financial_year = 2026-27`, `prefix = INV-2627-`, `padding = 4`, and current `next_number = 12`:
1. Take current `next_number` = `12`.
2. Pad `12` to 4 digits → `0012`.
3. Prepend `prefix` → **`INV-2627-0012`**.
4. Update `next_number` for `ORG-1` (`2026-27`) to `12 + 1 = 13` for the next issued invoice.

---

## 6. API Contract

Base path: `/api/v1` (all endpoints require authentication and enforce `organisation_id` scoping).

- **Auth & Users**
  - `POST /api/v1/auth/login` — Authenticate and issue token/session.
  - `GET /api/v1/users` & `POST /api/v1/users` — List or invite users with a role (`OWNER`, `FINANCE_MANAGER`, `ACCOUNTANT`, `COLLECTOR`, `READ_ONLY`).
- **Customers**
  - `GET /api/v1/customers` & `POST /api/v1/customers` — List or create customers (validates GSTIN format, overseas flag/currency, addresses, payment terms, credit limit).
  - `GET /api/v1/customers/{id}` & `PUT /api/v1/customers/{id}` — Retrieve or update customer details.
- **Invoices**
  - `GET /api/v1/invoices` & `POST /api/v1/invoices` — List tenant invoices or create a `DRAFT` invoice with line items (server computes line & invoice totals).
  - `GET /api/v1/invoices/{id}` — Fetch invoice details (returns `403`/`404` if owned by another organisation).
  - `PUT /api/v1/invoices/{id}` — Update a `DRAFT` invoice (rejected once `ISSUED`).
  - `POST /api/v1/invoices/{id}/issue` — Transition `DRAFT` → `ISSUED`, lock issue-date exchange rate (if overseas), compute/store tax & totals, and assign the next sequential invoice number.
  - `POST /api/v1/invoices/{id}/void` — Mark an `ISSUED` invoice as `VOID` (subject to role & lifecycle rules).
  - `POST /api/v1/invoices/{id}/payments` — Record a payment and update status to `PARTIALLY_PAID` or `PAID`.
  - `GET /api/v1/invoices/{id}/pdf-link` — Return a signed download URL for the generated invoice PDF.
- **Credit Notes**
  - `POST /api/v1/invoices/{id}/credit-notes` — Issue a credit note linked to an existing `ISSUED` invoice.
  - `GET /api/v1/credit-notes/{id}/pdf-link` — Return a signed download URL for the credit-note PDF.
- **Recurring Templates**
  - `GET /api/v1/recurring-templates` & `POST /api/v1/recurring-templates` — Manage monthly recurring invoice templates (`day_of_month`).

---

## 7. Risks

| Risk | Impact | Mitigation |
| :--- | :--- | :--- |
| **Cross-tenant data leak** | Critical (security & AC-1.1 failure) | Enforce `organisation_id` in service/repository queries and test cross-tenant access with automated integration tests. |
| **Duplicate or gapped invoice numbers under concurrent issuance** | High (AC-1.2 failure & audit issue) | Assign numbers only on `ISSUE` using row-locked (`FOR UPDATE`) sequence counter per `(organisation_id, financial_year)` + unique DB constraint. |
| **Rounding discrepancies between line items and invoice totals** | High (AC-1.3 failure) | Perform all math server-side with explicit Half-Up rounding to 2 decimal places (`0.01`). |
| **Editing issued invoices directly** | High (lifecycle violation) | Block mutations on `ISSUED` invoices at the service layer and use linked credit notes for adjustments. |
| **Exchange rate drift or missing rate on overseas invoices** | High (client rule failure for Saffron Trail Journeys) | Fix and persist the exchange rate on the issue date, compute tax in INR, and cover with `testOverseasInvoiceCurrencyAndTaxCalculation()`. |
| **Recurring job running twice on the same day** | Medium (duplicate invoices) | Track `last_generated_date` / idempotency key per template and billing period before issuing. |
