# Configuration — Paperless-ngx objects, workflows and ingestion

This covers everything you build **inside** the Paperless web UI after `Installation.md` leaves the app Running: tags, custom fields, document types, correspondents, the five storage paths, workflows, email ingestion and the phone app.

> On a restore from a `document_exporter` backup, **all of this comes back automatically** — don't rebuild it by hand. This document is for a from-scratch rebuild where no export exists, and as the reference for how the pieces fit. See `Disaster-Recovery.md` for which path you're on.

---

## Design principles (why it's built this way)

1. **One storage path per entity (5 total).** Each is a Jinja template that builds `Entity/Category/Sub-folder/Year/file` from the document's **Document Type**. Classify the type correctly → it lands in the right folder.
2. **The entity tag is the single key.** Any channel (drop folder, email, phone, web upload) that sets the entity tag ends up filed correctly, because workflows turn the tag into the storage path. You maintain **5** workflows, not 15.
3. **Numbered categories** (`01 Sales`, `02 Purchases`…) so Windows Explorer always sorts them the same way.
4. **ISO-date filenames** (`2026-11-05 Supplier - Title.pdf`) so files sort by date within each year folder.
5. **Matching = Auto** for document types and correspondents so Paperless learns from your corrections.

---

## 1. Tags

**Entity tags:** `Sibane`, `Ironside Technology`, `Ironside Marketing`, `Riverside Beauty Care`, `Personal`

**Workflow tags:** `Inbox` (set this one as the **inbox tag** in settings), `To Pay`, `Paid`, `Action Required`

**Source tags:** `Email`, `Scan`

**Expense/convenience tags** (as used): Fuel, Groceries, Coffee, Eats, Client Entertainment, etc. — optional, grown as needed.

---

## 2. Custom fields

| Field | Type |
|---|---|
| `Amount` | Monetary (ZAR) |
| `Invoice/Ref No` | String |
| `Due Date` | Date |
| `Expiry/Renewal Date` | Date (insurance, licences, B-BBEE, tax clearance) |

> **Do not** create entity-named custom fields. Three stray string fields — "Ironside Marketing", "Riverside Beauty Care", "Property" — appeared accidentally in production and should be deleted. Entities are **tags**, not fields.

---

## 3. Document types

Names must match the storage-path maps in section 5 **exactly** (case and punctuation). ~63 types total.

**Business:** Client Quote, Client Invoice, Credit Note, Client Purchase Order, Delivery Note, Client Statement, Supplier Quote, Supplier Invoice, Purchase Order, Supplier Statement, Pricelist, Bank Statement, Proof of Payment, Financial Statements, VAT Return, PAYE Return, Income Tax, Tax Clearance, B-BBEE Certificate, COIDA, Client Agreement, Supplier Agreement, NDA, Lease Agreement, Company Registration, Resolution, Company Profile, Policy, Insurance Policy, Insurance Claim, Employment Contract, Payslip, IRP5, Asset Document, Vehicle Document, Marketing Material, Catalogue, Site Survey, Certificate of Compliance, Technical Document, Datasheet, Correspondence

**Personal-only extras:** Identity Document, Certificate, Will & Estate, Legal Document, Investment, Retirement, Loan Agreement, Tax Certificate, Medical Aid, Medical Record, Property Document, Municipal Account, Utility Bill, Subscription, Receipt, Warranty, Manual, Qualification, CV, Travel, Hobby - Homelab (add one `Hobby - X` per hobby)

Shared types (Bank Statement, Insurance Policy, Payslip, Proof of Payment, Income Tax, IRP5…) are disambiguated by the **storage path** — the entity decides which folder they go to.

Set **Matching: Auto** on all of them.

---

## 4. Correspondents

Create your main clients and suppliers; set **Matching: Auto**. Let Paperless create the rest as documents arrive, then merge duplicates.

> **Watch for near-duplicates** (production hit "Copper Mug" vs "The Coppermug"). Pick one, reassign, delete the other, so a correspondent's history stays together.

---

## 5. Storage paths (Documents → Storage Paths)

Create **one per entity**, **Matching: None** (workflows assign them). The four company templates are identical except the first word of the last line; the Personal one has its own map.

### Company template (change only `Sibane/` → entity name)

```jinja
{%- set f = {
 "Client Quote": "01 Sales/Quotes to Clients",
 "Client Invoice": "01 Sales/Invoices to Clients",
 "Credit Note": "01 Sales/Credit Notes",
 "Client Purchase Order": "01 Sales/Client Purchase Orders",
 "Delivery Note": "01 Sales/Delivery Notes",
 "Client Statement": "01 Sales/Client Statements",
 "Supplier Quote": "02 Purchases/Quotes from Suppliers",
 "Supplier Invoice": "02 Purchases/Supplier Invoices",
 "Purchase Order": "02 Purchases/Purchase Orders to Suppliers",
 "Supplier Statement": "02 Purchases/Supplier Statements",
 "Pricelist": "02 Purchases/Pricelists",
 "Receipt": "02 Purchases/Receipts & Expenses",
 "Bank Statement": "03 Banking & Finance/Bank Statements",
 "Proof of Payment": "03 Banking & Finance/Proof of Payment",
 "Financial Statements": "03 Banking & Finance/Financial Statements",
 "VAT Return": "04 Tax & Compliance/VAT",
 "PAYE Return": "04 Tax & Compliance/PAYE & UIF",
 "Income Tax": "04 Tax & Compliance/Income Tax",
 "Tax Clearance": "04 Tax & Compliance/Tax Clearance",
 "B-BBEE Certificate": "04 Tax & Compliance/B-BBEE",
 "COIDA": "04 Tax & Compliance/COIDA",
 "Client Agreement": "05 Agreements/Client Agreements",
 "Supplier Agreement": "05 Agreements/Supplier & Reseller Agreements",
 "NDA": "05 Agreements/NDAs",
 "Lease Agreement": "05 Agreements/Leases",
 "Company Registration": "06 Company Documents/CIPC & Registration",
 "Resolution": "06 Company Documents/Resolutions & Minutes",
 "Company Profile": "06 Company Documents/Company Profile & Branding",
 "Policy": "06 Company Documents/Policies & Procedures",
 "Insurance Policy": "07 Insurance/Policies & Schedules",
 "Insurance Claim": "07 Insurance/Claims",
 "Employment Contract": "08 HR & Payroll/Employment Contracts",
 "Payslip": "08 HR & Payroll/Payslips",
 "IRP5": "08 HR & Payroll/IRP5 Certificates",
 "Asset Document": "09 Assets & Vehicles/Assets & Warranties",
 "Vehicle Document": "09 Assets & Vehicles/Vehicles",
 "Marketing Material": "10 Marketing/Marketing Material",
 "Catalogue": "10 Marketing/Catalogues & Brochures",
 "Site Survey": "11 Projects & Technical/Site Surveys",
 "Certificate of Compliance": "11 Projects & Technical/COCs & Certificates",
 "Technical Document": "11 Projects & Technical/Technical & Handover",
 "Datasheet": "11 Projects & Technical/Datasheets & Manuals",
 "Correspondence": "99 General/Correspondence"
} -%}
Sibane/{{ f.get(document_type, "99 General/Unsorted") }}/{{ created_year }}/{{ created }} {{ correspondent }} - {{ title }}
```

Create the other three by changing the last line to `Ironside Technology/`, `Ironside Marketing/`, `Riverside Beauty Care/`.

### Personal template

```jinja
{%- set f = {
 "Identity Document": "01 Identity & Legal/Identity Documents",
 "Certificate": "01 Identity & Legal/Certificates",
 "Will & Estate": "01 Identity & Legal/Will & Estate",
 "Legal Document": "01 Identity & Legal/Legal",
 "Bank Statement": "02 Banking & Investments/Bank Statements",
 "Proof of Payment": "02 Banking & Investments/Proof of Payment",
 "Investment": "02 Banking & Investments/Investments",
 "Retirement": "02 Banking & Investments/Retirement",
 "Loan Agreement": "02 Banking & Investments/Loans & Credit",
 "Income Tax": "03 Tax/Income Tax",
 "Tax Certificate": "03 Tax/Tax Certificates",
 "IRP5": "03 Tax/IRP5",
 "Insurance Policy": "04 Insurance/Policies & Schedules",
 "Insurance Claim": "04 Insurance/Claims",
 "Medical Aid": "04 Insurance/Medical Aid",
 "Property Document": "05 Property & Home/Property Documents",
 "Lease Agreement": "05 Property & Home/Leases",
 "Municipal Account": "05 Property & Home/Municipal Accounts",
 "Vehicle Document": "06 Vehicles",
 "Medical Record": "07 Health/Medical Records",
 "Utility Bill": "08 Utilities & Subscriptions/Utility Bills",
 "Subscription": "08 Utilities & Subscriptions/Subscriptions",
 "Receipt": "09 Receipts & Warranties/Receipts",
 "Warranty": "09 Receipts & Warranties/Warranties",
 "Manual": "09 Receipts & Warranties/Manuals",
 "Qualification": "10 Education & Career/Qualifications",
 "CV": "10 Education & Career/CV & Employment",
 "Payslip": "10 Education & Career/CV & Employment",
 "Hobby - Homelab": "11 Interests & Hobbies/Homelab",
 "Travel": "12 Travel",
 "Correspondence": "99 General/Correspondence"
} -%}
Personal/{{ f.get(document_type, "99 General/Unsorted") }}/{{ created_year }}/{{ created }} {{ correspondent }} - {{ title }}
```

**Adding a category later:** create the document type, add one line to the relevant map, save. Paperless re-files affected documents automatically.

### Resulting Explorer view (read-only `A:` share)

```
Paperless-Archive\
├── Sibane\
├── Ironside Technology\
├── Ironside Marketing\
├── Riverside Beauty Care\
├── Personal\
└── _Unsorted\      ← no storage path yet; should stay empty
```

Each company folder contains `01 Sales … 11 Projects & Technical`, `99 General`; each leaf holds year folders. Paperless creates a folder only when the first document lands in it, so you won't see empty skeletons — that's expected.

---

## 6. Workflows (Administration → Workflows)

### 6a. Entity workflows — one per entity (5)

Each entity workflow has **two triggers** and assigns the storage path + tags.

- **Trigger 1 — Consumption Started**, source **Consume folder**, filter path `*/IronsideTechnology/*` (use the no-space dataset folder name). If the glob doesn't match, try `*IronsideTechnology*`.
- **Trigger 2 — Document Added**, condition **has tag = `<Entity>`** (this makes the entity tag the universal key for email, phone and web uploads).
- **Actions:** assign **Storage path = `<Entity>`**, assign tags `<Entity>` + `Inbox`, set owner.

### 6b. "Re-file on tag" workflows — one per entity (5)

Entity workflows only fire when a document is **first added**, so tagging an *existing* document wouldn't move it. These cover that:

- **Trigger — Document Updated**, condition **has tag = `<Entity>`**
- **Action — assign Storage path = `<Entity>`** only (no tags, so it won't re-add `Inbox` after you've cleared it)

This is what lets you fix a mis-filed phone receipt just by adding the right entity tag later.

### 6c. "App upload" workflow (1)

- **Trigger — Consumption Started**, source **API Upload**
- **Actions — assign tags `Inbox` + `Scan`, set document type `Receipt`**

So phone uploads always get reviewed, and you pick the entity tag at upload time (or add it afterwards, which the re-file workflow then acts on).

---

## 7. Email ingestion

**Mailbox:** `accounts@ironsidetech.co.za`, hosted by **Domains.co.za** ("Email Only Hosting"; webmail on `da16.domains.co.za`). IMAP `mail.ironsidetech.co.za:993`, SSL. Use an **app password** if 2FA is on.

**Per-entity forwarders** into `accounts@` (set up in the Domains.co.za / DirectAdmin portal):

| Forwarder | Entity |
|---|---|
| `docs.sibane@` | Sibane |
| `docs.ironsidetech@` | Ironside Technology |
| `docs.marketing@` | Ironside Marketing |
| `docs.rbc@` | Riverside Beauty Care |
| `docs.mervin@` | Personal |

**Paperless → Mail → Mail Accounts:** add the mailbox (IMAP, SSL, 993). Create an `INBOX.Processed` folder in webmail first.

**Paperless → Mail → Mail Rules** (one per entity, orders 10–50):
- **Filter To:** `docs.sibane` (etc.)
- **Max age:** 30 days
- **Consumption scope:** Attachments only
- **Action:** assign entity tag + `Email`; **move mail to `INBOX.Processed`** (prevents re-import)
- Correspondent: *do not assign* — forwarded mail always shows you as sender, so let auto-matching work it out.

**Rule 90 "Accounts direct"** (catch-all, lowest priority, no To filter): assigns `Ironside Technology` + `Email` + `Inbox`, so nothing arriving straight at `accounts@` is missed.

> **Avoid overlapping mail runs.** Two checks running ~simultaneously imported several test docs twice (docs 60/63, 61/64, plus a duplicate receipt). The `*/10` cron plus "move to Processed" normally prevents this; just don't manually trigger a check while one is running.

---

## 8. Phone app

- **Android:** Paperless Mobile 4.5.0. **iPhone:** Swift Paperless is the common choice.
- Sign in as **`mscorgie`** (the day-to-day user), **not** admin.
- Works over mobile data via the FQDN. On home Wi-Fi it may hit the NPM openresty error (see `Troubleshooting.md`); Tailscale is the clean fix.
- The "App upload" workflow (6c) tags uploads `Inbox` + `Scan` + type `Receipt`. Pick the entity tag on the upload screen so it files straight away.

---

## 9. Daily / weekly routine

1. Drop or email documents.
2. Open the **Inbox** tag in Paperless. Confirm type, correspondent, title, date → remove the `Inbox` tag → the file moves to its final folder.
3. Corrections train the matcher. After a few weeks most documents arrive pre-classified.
4. **Weekly:** check that `_Unsorted` (media root) and each `99 General/Unsorted` are empty.
5. The auto-classifier over-guesses **Receipt** early on — just correct the type in the Inbox and it learns.
