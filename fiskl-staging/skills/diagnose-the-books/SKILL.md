---
name: fiskl-diagnose-the-books
description: Investigate why a figure in Fiskl looks wrong, and find bookkeeping mistakes nobody has reported yet. Use when the user says a number looks off, wrong, or surprising — "my ratios are terrible but I'm profitable", "why is my current ratio so low", "my gross margin dropped and I don't know why", "this account balance looks wrong", "my Fiskl balance doesn't match my bank", "my P&L looks off", "profit doesn't match my cash", "the trial balance is out", "my VAT return looks short" — or when they ask for a check-up: "check my books", "find mistakes", "spot anomalies", "did I miss any tax", "is anything miscategorised", "review last quarter before I file". Also use when a report figure needs tracing back to the transactions behind it. Read-only: this skill investigates and explains, it never changes data.
version: 0.3.0
---

# Diagnose the books

A figure in Fiskl looks wrong, or the user wants to know whether anything *is* wrong.
Your job is to find the specific transaction or missing entry that explains it — not to
restate the report.

Two distinct jobs live here. Work out which one you're doing before you start.

| Job | Starts from | Approach |
|---|---|---|
| **Explain a wrong number** | A symptom the user noticed | Work backwards from the report to the posting |
| **Find unreported mistakes** | No symptom at all | Compare like against like, look for the odd one out |

The second is where you have the biggest advantage over a person: comparing hundreds of
lines for consistency is fast for you and tedious for them.

## Always do this first

1. **`company_info`** — base currency, and **`accountingBasis` (accrual or cash)**. Never
   assume accrual. On a cash basis, unpaid bills and invoices legitimately don't appear
   in the P&L, which explains a whole class of "my profit looks wrong" reports.
2. **`date_periods`** — take start/end from the period the user named. Never compute a
   quarter or financial-year boundary yourself; Fiskl's fiscal year may not be the
   calendar year.

**All monetary amounts are integer minor units — divide by 100 to display.** A figure
reported 100× out is almost always this and not a real error.

**Dates in tool results arrive as arrays — `[2026,9,4]` is 4 September 2026** (year,
month, day). Read them that way; pass dates back as ISO `YYYY-MM-DD`.

## Explaining a wrong number

Go from the aggregate to the individual posting. Don't stop at the report level — the
answer is nearly always one line.

1. Reproduce the figure: `financial_ratios`, `profit_and_loss`, `balance_sheet` or
   `trial_balance` for the period.
2. Find which account moved. `balance_sheet` and `trial_balance` show composition; a
   balance that is large, negative where it should be positive, or new this period is
   the lead.
3. Resolve that account's id with `list_accounts`, then open it:
   **`transaction_details(start, end, accountId, runningBalance=true)`**. This returns
   the individual postings — this is where you find the answer.
4. Read the lines. Look for the one that doesn't belong: a sign that's inverted relative
   to its neighbours, an amount an order of magnitude out, a date in the wrong period, a
   posting in an account that shouldn't receive that kind of entry.
5. State the specific transaction, what it should have been, and what it did to the
   figure the user asked about.

**A worked case.** "My ratios say I'm deeply in the red but the business is profitable."
Current and quick ratios are assets over liabilities, so a *single* asset booked the
wrong way round moves both hard: the asset is understated and often a liability is
overstated at the same time. `financial_ratios` shows the ratio; `balance_sheet` shows
which asset class looks wrong; `transaction_details` on that account shows the reversed
posting. The P&L can look fine throughout, which is exactly why this hides.

Sign convention, so you read the lines correctly: **Income, Liabilities and Equity are
reverse-sign** in Fiskl; Assets and Expenses are not. The sign is derived from the
account type, never stored on the transaction. When judging whether a line is "backwards",
compare it against the other lines in the same account rather than against an absolute
expectation.

## Finding unreported mistakes

Nothing looks wrong. You are looking for inconsistency, which means comparing a group of
similar transactions and finding the member that differs.

**Start with `books_check(start, end)`.** It runs every check below server-side over every
transaction in the period — uncategorised, missing_tax, inconsistent_tax_rate,
tax_with_zero_amount, category_outlier, duplicate_suspect, missing_vendor, missing_client,
future_dated, unmatched_transfers, round_amounts — and returns findings that name the
transaction (`txId`), each with `confidence` CONFIRMED or SUSPECTED, plus the list of checks
that came back `clean`. Report CONFIRMED as fact and SUSPECTED as "possible — worth
confirming". Only fall back to the manual comparison below when `books_check` is not
available to you or the user asks about a pattern it does not cover. To explain one finding,
call `transaction_details` with its `txId`; for an unmatched transfer, the other leg is on
another money account in the same period.

**The highest-value check: tax applied inconsistently.**

A recurring supplier is booked with the correct tax rate eleven times and, once, without
it. Nothing looks broken — the books balance perfectly. It surfaces only when the VAT
return comes in short, which is after it matters.

To find it:

1. `list_accounts` → resolve the expense account (or accounts) to examine.
2. `transaction_details(start, end, accountId)` over a decent window — a full quarter or
   year, so a pattern exists to break.
3. Group the returned lines by **`clientVendor`** and **`categoryId`**.
4. Within each group, compare **`taxId`** and the ratio of **`taxAmount`** to
   **`netAmount`**. Flag:
   - a line with **no `taxId`** where its siblings all have one
   - a line whose effective rate differs from the group (5% among 20%s)
   - a line with a `taxId` but `taxAmount` of zero
5. Report each as: vendor, date, amount, what the rest of the group does, and what this
   line does instead.

Cross-check against **`tax_report(accountId, start, end, details=true)`** for the tax
account — line-level detail showing what actually reached the return. A transaction the
user believes carries tax but which is absent here has not reached the filing.

**Other consistency checks worth running:**

- **Same vendor, different categories.** Group by `clientVendor` and look at the spread
  of `categoryId`. A supplier that is normally one category and occasionally another is
  usually a categorisation slip, not a real change.
- **Anything still uncategorised.** Accounts under Uncategorised Income or Uncategorised
  Expenses are a holding area, not a destination. Balances sitting there mean work is
  unfinished, and they distort the P&L.
- **Duplicate-looking postings.** Same vendor, same amount, same or adjacent date.
  Report as *possible* duplicates for the user to confirm — recurring charges are
  legitimately identical month to month, so do not assert.
- **Round-number estimates.** In a set of precise amounts, a suspiciously round one is
  sometimes a placeholder someone meant to correct.

## Known symptom → cause chains

Start here when the symptom matches; each still needs confirming in the detail.

| Symptom | Look at first |
|---|---|
| Ratios wildly wrong, P&L looks fine | An asset or liability posted the wrong way round — `balance_sheet`, then `transaction_details` on the odd account |
| Trial balance out by exactly the tax amount | Tax never reached a tax line — check `tax_report(details=true)` against the transaction |
| VAT/GST return lower than expected | A transaction missing its tax rate — run the tax-consistency check above |
| Accounts Payable won't come down | Payments not matched to the original AP entries — `transaction_details` on the AP account, look for entries that never clear |
| Fiskl balance ≠ bank balance | Unrecorded internal transfers leaving "ghost" transactions, or transactions in the bank but not Fiskl |
| Profit healthy, cash isn't (or vice versa) | Timing, not error — compare `profit_and_loss` with `cash_flow`; on accrual, unpaid invoices are profit without cash |
| An expense category looks too small | Some of it booked elsewhere — check the same vendor across categories |
| A client's balance looks wrong | `statement_of_account(clientId, start, end, includeAr=true)` — invoices, payments and running balance in one view |

## Reporting what you find

Business owners act on this, so be precise and honest about certainty.

- **Name the transaction**: date, vendor or client, amount, account. Not "an expense in
  March" but the specific line.
- **Say what it should be** and what to do about it. This skill changes nothing; when
  the fix is a missing or miscategorised transaction and the connection has
  `transactions:write`, hand over to `fiskl-record-transactions`, which previews and
  confirms every write. Otherwise refer to the Fiskl UI.
- **Quantify the effect** on the figure they asked about, where you can.
- **Separate confirmed from suspected.** "This posting is reversed" and "these two look
  like duplicates, please confirm" are different claims. Never present a suspicion as a
  finding.
- **Say when you found nothing.** A clean check is a useful result. Say what you examined
  so they know what was covered.

## Staying inside what you know

You are diagnosing **Fiskl's records**, not giving accounting or tax advice.

- Explaining that a posting is inconsistent with its neighbours: yes.
- Whether an expense is **deductible**, which **tax rate** should apply, whether
  something should be **capitalised**, or how revenue should be **recognised**: no.
  These depend on jurisdiction and circumstance. Say it needs their accountant.
- If a rule you're unsure of would change your conclusion, say so rather than picking.
- Never guess an account. `list_accounts` returns `id`, `name`, `accountNumber` and a
  broad `type` only — it does not tell you an account's sub-category, and companies
  rename accounts. If more than one account could be the one you need, **ask** and show
  the candidates.
