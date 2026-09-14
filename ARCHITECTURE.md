# 🏛️ BillPulse - Technical Architecture & Engineering Decisions

This document details the architectural choices, trade-offs, and design patterns implemented in **BillPulse**.

---

## 1. Client-Side PDF Generation vs. Headless Server-Side Rendering

### The Problem
Traditional document generation SaaS architectures run headless browser instances (e.g., Puppeteer, Playwright, Chrome AWS Lambda) to convert HTML/CSS into PDF files. In a serverless cloud environment (AWS Lambda, Vercel Functions), this introduces major liabilities:
1. **Cold Starts:** Headless Chrome binaries take 2–5 seconds to initialize on a cold Lambda.
2. **High Memory & CPU Usage:** Each rendering job requires 512MB–1GB RAM and high CPU burst limits.
3. **Escalating Cloud Costs:** At scale, server-side PDF generation can account for over 60–80% of serverless computing bills.
4. **Vulnerability Surface:** Maintaining headless browser instances introduces security vulnerabilities and memory leak risks.

### The BillPulse Solution
BillPulse shifts document layout and PDF compilation entirely into the browser via **`@react-pdf/renderer`**:
* **Zero Server Infrastructure:** PDF compilation occurs within the user's web worker/client execution thread.
* **Instant Export:** Eliminates round-trip API latency and network packet drops.
* **Predictable Operational Cost:** Generating 10,000 invoices per month incurs **$0.00** in server compute overhead.

---

## 2. European Peppol BIS Billing 3.0 / UBL 2.1 E-Invoicing Engine

### The Regulatory Landscape
Beginning in 2024–2026, EU member states (France, Germany, Italy, Poland, Belgium, etc.) increasingly mandate standardized B2B/B2G e-invoicing to combat VAT fraud. PDF invoices alone no longer meet legal compliance in many European jurisdictions.

### The Engine Implementation
BillPulse includes an integrated e-invoicing serializer (`lib/einvoice/peppol.ts`) that converts internal invoice entities into structured, validated **UBL 2.1 XML (Universal Business Language)** adhering to **Peppol BIS Billing 3.0**:

* **CustomizationID:** `urn:cen.eu:en16931:2017#compliant#urn:fdc:peppol.eu:2017:poacc:billing:3.0`
* **ProfileID:** `urn:fdc:peppol.eu:2017:poacc:billing:01:1.0`
* **Endpoint Identification:** Supports European standard endpoint scheme IDs (e.g., `0088` GLN, `0190` Dutch Chamber of Commerce, `0208` Belgium Enterprise Number).
* **Tax Category Schemes:** Automatic categorization into `S` (Standard VAT rate), `Z` (Zero rated), or `E` (Exempt).
* **Cryptographic & Syntactic Precision:** Guarantees proper XML formatting, element order, and strict numeric rounding according to CEN/TC 434 specifications.

---

## 3. Multi-Tenancy & Data Isolation

### Row-Level Security (RLS)
Rather than relying solely on application-level filtering (`WHERE user_id = ?`), BillPulse enforces tenant boundary isolation directly at the PostgreSQL database engine:
* Every table (`businesses`, `clients`, `items`, `invoices`, `invoice_items`) contains a `user_id` indexed foreign key.
* Supabase PostgreSQL Row-Level Security policies are applied to all operations (`SELECT`, `INSERT`, `UPDATE`, `DELETE`).
* Even if an application route has a logic vulnerability, PostgreSQL blocks queries that attempt to read another tenant's records.

### Immutable Historical Snapshots
In standard relational models, foreign keys point to `clients` and `businesses`. If a company updates its registered tax address or VAT number two months later, legacy invoices referencing that foreign key would retroactively display the new address, violating accounting auditing standards.

BillPulse solves this with **JSONB Snapshots**:
* When an invoice is created, exact copies of the issuer business profile and recipient client information are stored in `business_snapshot` and `client_snapshot` columns.
* Past invoices remain frozen in time, exactly as they were delivered to the customer, while current contact lists remain editable.

---

## 4. State Management & Dual Persistence Strategy

BillPulse features a hybrid data access layer (`lib/store.ts`):
1. **Zero-Config Local Storage Mode:**
   * Works out-of-the-box in development or offline environments without requiring database keys.
   * Stores invoices, clients, and preferences in `localStorage` with identical schema interfaces.
2. **Cloud PostgreSQL Sync:**
   * When `NEXT_PUBLIC_SUPABASE_URL` and keys are provided, the repository layer automatically routes persistence to Supabase via `@supabase/supabase-js`.
