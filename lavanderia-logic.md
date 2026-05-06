# Lavanderia — System Logic for Developer

## What This System Does

The business has a laundry service. Payments from customers were recorded by hand in physical ledger books over the years. The system database has all the invoices, but the payment dates recorded there are wrong — they were set to the same date as the invoice, not the actual date the customer paid.

The goal of this system is to:
1. Read those handwritten ledger pages
2. Match each entry to the correct invoice in the database
3. Correct the payment date on each invoice
4. Validate that daily sales totals match what's in the database

---

## The Data Sources

### Source 1 — The Scanned Ledger Pages
Physical handwritten books, photographed as JPG/PNG images.

Each page represents **one day**. The date is written at the top.

Each line on the page represents **one payment**, written like this:
```
L-33115 M  21
L-32965 L  17.5
```

How to read each line:
- `L-33115` — the invoice ID
- `M` — payment method (M = Card)
- `L` — payment method (L = Link)
- `21` / `17.5` — the amount paid

Cash entries appear differently — they are standalone amounts without a suffix letter.

At the bottom of the page, the total SALE number for that day is written.

---

### Source 2 — The Orders Database (Google Sheet: PL - Orders DB)

This is the business database exported to Google Sheets. It has ~40,000 rows.

**Important: one row = one payment.** If an invoice was paid in two installments, it appears as two rows with the same Order ID but different Payment # (1 and 2).

| Column | Description | Used for |
|--------|-------------|----------|
| Order ID | Internal system ID (e.g. 8576) | Reference only |
| Legacy Invoice # | The invoice ID shown on the ledger pages (e.g. L-33115) | **Primary match key** |
| Invoice Date | When the invoice was created | Date window filter |
| Original Amount (AED) | Amount before discount | Reference |
| Discount Type | percentage or amount | Reference |
| Discount Value | Discount % or value | Reference |
| Discount Amount (AED) | How much was discounted | Reference |
| Total After Discount (AED) | Final invoice amount — **source of truth** | Amount validation |
| Remaining Balance (AED) | How much is still unpaid | Partial payment check |
| Payment # | Which installment (1, 2, 3...) | Partial payment handling |
| Payment Method | cash / card / link / bank_transfer | **Controller 2 match** |
| Payment Amount (AED) | Amount paid in this installment | **Controller 1 match** |
| Payment Date | Currently wrong — this is what we need to fix | **Target field to update** |

**Payment methods mapping (ledger → database):**
| On the page | In the database |
|-------------|-----------------|
| M (suffix) | card |
| L (suffix) | link |
| standalone | cash |

---

## The Four Output Sheets

All results go into Google Sheets. There are four sheets:

### 1. PL - Parsed Entries
What was extracted from the scanned image.

| Column | Description |
|--------|-------------|
| Date | The date written on the ledger page |
| Order ID | The invoice ID as read from the page (e.g. L-33115) |
| Payment Method | card / link / cash |
| Amount Parsed | The amount as read from the page |
| Confidence Score | 0–100, how clearly the handwriting was read |

### 2. PL - Match Results
The result of matching each parsed entry against the database.

| Column | Description |
|--------|-------------|
| Page Date | Date from the ledger page |
| Parsed Invoice ID | Invoice ID as read from page |
| DB Order ID | Matched Order ID from database |
| DB Legacy ID | Matched Legacy Invoice # from database |
| Payment Method (Page) | Method as read from page |
| Payment Method (DB) | Method found in database |
| Method Match | YES / NO |
| Amount (Page) | Amount as read from page |
| Amount (DB) | Amount found in database |
| Amount Match | YES / NO |
| Total After Discount (DB) | Full invoice amount from database |
| Remaining Balance (DB) | Remaining unpaid amount |
| Payment Date (Current) | The wrong date currently in the database |
| Payment Date (To Update) | The correct date from the ledger page |
| Match Status | AUTO / FLAG / NO_MATCH |
| Flag Reason | Why it was flagged, if applicable |

### 3. PL - Sales Validation
One row per day scanned.

| Column | Description |
|--------|-------------|
| Date | The date of the ledger page |
| Parsed Sale Total | The SALE number written on the page |
| DB Sale Total | Sum of Total After Discount for all invoices on that date |
| Match | YES / NO |
| Diff | The difference amount if not matching |

### 4. PL - Orders DB
The raw database export. Read-only input. Do not modify.

---

## The Pipeline — Step by Step

```
Image Upload
     ↓
Step 1: Parse the image (AI Vision)
     ↓
Step 2: Write parsed entries → PL - Parsed Entries
     ↓
Step 3: Read Orders DB from Google Sheets
     ↓
Step 4: Run the matching engine
     ↓
Step 5: Write match results → PL - Match Results
     ↓
Step 6: Write sales check → PL - Sales Validation
```

---

## The Matching Engine — How It Works

This is the most important part. For each parsed entry, the system tries to find the correct row in the database.

### The Two Controllers

**Controller 1 — Invoice ID Match**
Take the invoice ID from the page (e.g. `L-33115`) and look for it in the `Legacy Invoice #` column of the database.

This is a fuzzy match, not exact, because handwriting can be misread:
- The digit `8` can look like `0`
- The digit `1` can look like `7`

Confidence levels:
- Exact match → 100%
- Same number after removing leading zeros → 98%
- One digit looks like a common misread (8↔0 or 1↔7) → 75%
- Two such digits → 55%
- No match → 0%

Only proceed if confidence ≥ 55%.

**Controller 2 — Date Window Filter**
Before comparing amounts, filter the database rows to only those where the `Invoice Date` is within **±6 months** of the page date. This prevents matching against the wrong invoice with the same ID from a different period.

If no match is found within 6 months, expand the search to the full database — but flag the result for review.

**Controller 3 — Payment Match (amount + method)**
Among the candidate rows that passed the ID and date filter, find the best match by comparing:
- Payment Amount (AED) vs the parsed amount — exact match preferred
- Payment Method (card/link/cash) vs parsed method — exact match preferred

Score: 0, 1, or 2 points (one per match).
Pick the row with the highest score.

### Match Status

| Status | Meaning |
|--------|---------|
| AUTO ✅ | ID exact + amount match + method match + no flags → safe to update |
| FLAG ⚠️ | One or more mismatches → needs your review before updating |
| NO_MATCH ❌ | Invoice ID not found in database at all |

### Flag Reasons (examples)
- `Fuzzy ID (75%)` — the ID matched but one digit was a likely misread
- `Method: page=card DB=cash` — payment method doesn't match
- `Amount: page=21 DB=21.5` — amounts don't match exactly
- `Date window >6m` — match found but outside the 6 month window
- `Low OCR conf (65%)` — the handwriting was unclear

### Partial Payments
Some invoices were paid in multiple installments. In the database these appear as multiple rows with the same Order ID but different Payment # values.

When matching a partial payment, the system looks for the specific payment row where both the amount AND the method match — not just the invoice in general.

---

## The Sales Validation — Task 2

This is simpler. For each page scanned:

1. Take the SALE total written at the bottom of the page
2. Query the database: sum all `Total After Discount (AED)` values where `Invoice Date` equals the page date
3. Compare the two numbers
4. Output YES if they match (within 1 AED tolerance), NO if they don't

This is a yes/no check only. No updates are made based on this.

---

## What the Developer Does NOT Touch

- The `Total After Discount (AED)` column — this is the source of truth for amounts
- Any amount columns in the database — amounts are never changed
- The `PL - Orders DB` sheet — read only

**The only field that gets updated in the database is `Payment Date`.**

---

## The Update Flow (Final Step — Not Yet Built)

Once the match results are reviewed and approved:

1. Take all rows in `PL - Match Results` where `Match Status = AUTO`
2. For each row, update the `Payment Date` in the database using:
   - `DB Order ID` to identify the row
   - `Payment Date (To Update)` as the new value
3. Rows with `FLAG` or `NO_MATCH` status are handled manually

This final update step is a batch operation — do all approved updates at once, not one by one.

---

## Summary for the Developer

| What | How |
|------|-----|
| Read ledger images | AI Vision (Claude API) |
| Parse handwriting | Structured JSON output from the AI |
| Store parsed data | Google Sheets (PL - Parsed Entries) |
| Read database | Google Sheets (PL - Orders DB) |
| Match logic | Custom engine: fuzzy ID + date window + amount + method |
| Store results | Google Sheets (PL - Match Results) |
| Sales check | Sum DB totals by date, compare to page total |
| Final DB update | Batch update Payment Date only, after manual approval |
| Human review needed | FLAG and NO_MATCH rows only |
