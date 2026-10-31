---
name: fiskl-record-transactions
description: Record what is missing from the books in Fiskl, and fix what diagnose-the-books found, through preview-then-confirm writes. Use when the user wants an entry recorded — "record this expense", "this card payment isn't in Fiskl", "add the bill from Staples", "log the money that came in from Acme", "move money between accounts", "I paid the supplier from the wrong account, fix it", "record the laptop I bought", "register this asset and depreciate it over three years", "the depreciation is posting but the asset was never registered" — or when they have the receipt or supplier bill itself as a file, "scan this receipt", "here's a photo of the receipt", "I have the bill as a PDF", "send this to AI Expenses" — or after a check-up, "fix what you found", "correct that category", "record the missing transfer". Also use when someone asks what Fiskl needs to record a purchase, bill or asset. Writes only after the user has seen a preview and said yes; never posts a manual journal, never deletes, never guesses an account or a tax rate.
version: 0.4.0
---

# Record transactions

The user wants something recorded in Fiskl that is not there, or wants an entry
corrected. Your job is to get it into the books **once**, on the **right account**, with
the **right tax rate**, after the user has seen exactly what will be posted.

This is the write-side companion of `fiskl-diagnose-the-books`. That skill finds the
problem; this one records the fix. Everything here goes through the same rules Fiskl's
own transaction editor enforces, so a request that would be refused in the app is
refused here too — the difference is that the error names the field.

## Always do this first

1. **`me_permissions`** — you need `transaction.write` (scope `transactions:write`), or
   `bill.write` (scope `bills:write`) to scan a receipt or bill from a file. Without it,
   say so and stop; a missing scope means reconnecting with more permissions, not a
   missing feature.
2. **`company_info`** — the base currency and the accounting basis.
3. **`date_periods`** — the `Today` period gives you the current date; the response also
   tells you the accounting lock date. **Never compute a date yourself** unless the user
   gave an explicit one.
4. **`list_accounts`** — resolve the money account the funds moved through *and* the
   category. It returns `id`, `name`, `accountNumber` and a broad `type` only; companies
   rename accounts. **If more than one account could fit, show the candidates and ask.**
5. **`list_taxes`** — resolve the tax rate id when the entry carries tax. Never pick a
   rate from its percentage alone if two codes share it; ask which one. **If it returns
   no rates at all**, the company has not set up tax in Fiskl: record the entry without
   tax, say so, and do not invent one.
6. **`list_transactions`** on that account around the date — the entry may already be
   there under a different name. Recording it twice is worse than not recording it.

**All monetary amounts are integer minor units — 15000 means 150.00.** Amounts you pass
are **unsigned**; the transaction type gives the direction.

**Dates in tool results arrive as arrays — `[2026,9,4]` is 4 September 2026** (year,
month, day). Read them that way and write them back as ISO `YYYY-MM-DD`. This applies to
every Fiskl tool, not just these.

**Missing parties.** A `BILL` needs a vendor and a `RECEIVABLE` a client. If `list_vendors`
or `list_clients` has no match, offer to create one with `create_vendor` / `create_client`
first — with the user's agreement, and with the name they give you — then record.

For the reasoning behind these rules (why there is no transaction type, why amounts are
unsigned, why tax is a rate and never an amount, what a mirror row is), read
[references/ledger-model.md](references/ledger-model.md) when a user asks *why*.

## A receipt or supplier bill as a file

When the user has the document itself — a photo, a scan, a PDF — scanning it is the first
choice, not the manual procedure below. Fiskl reads the vendor, date, lines, totals and tax
from the file, exactly as its AI Expenses screen does, so the user does not have to dictate
them and you do not have to resolve them. It needs `bills:write`, a plan with accounting,
and each page counts against the plan's receipt-scanning allowance.

1. **`create_expense_upload(fileName, contentType, sizeBytes)`** registers the upload and
   returns two ways to send the file, both into the same upload. Use exactly one:
   - **`uploadUrl`** with the ready `curl` command, where you can run commands and reach
     S3 (Claude Code). `sizeBytes` must then be the file's exact size, or S3 refuses it.
   - **`browserUploadUrl`** everywhere else — a plain chat, a phone, a sandbox that cannot
     reach S3. Pass `sizeBytes` 0 and give the user the link to open: it needs no login,
     works once, and expires after 30 minutes. Do not fetch or open the link yourself.
2. **`get_expense_upload(id)`** until `status` is `PROCESSED`, `DUPLICATE` or `FAILED`.
   `UPLOADED` means the file has not arrived or has not been read yet; `INGESTING` means it
   is being read. Scanning usually takes under a minute — wait and poll, and record nothing
   for this purchase meanwhile.
3. **Tell the user what the scan produced.** Depending on the company's settings it is a
   supplier bill draft waiting for their review in Fiskl (`needsReview`), or an Accounts
   Payable transaction already posted (`apTransactionId`, which `get_transaction` shows).
   `DUPLICATE` means Fiskl already had this document and created nothing. `FAILED` means
   it could not be read or posted (a date in a locked period sets `failDate`); fall back to
   the manual procedure with what the user can tell you, or create a new upload to retry.

**The scan is the record.** Never also `create_transaction` for the same purchase, and never
hand-record a purchase whose upload is still `UPLOADED` or `INGESTING` — that duplicate is
the one step 6 above cannot catch, because the entry is not in the books yet. To record a
bill from figures the user already has and no file, use the manual procedure with type
`BILL`.

## Which type is it

Fiskl's ledger has no transaction type of its own — it derives the posting from which
account holds the entry. The API names the type explicitly so you don't have to know
that. Decide it before you touch a tool:

| What happened | `type` | Holder | Must have |
|---|---|---|---|
| Money left a bank, cash or card account | `EXPENSE` | `accountId` = that account | ≥1 category line |
| Money arrived in a bank, cash or card account | `INCOME` | `accountId` = that account | ≥1 category line |
| Money moved between two money accounts | `TRANSFER` | `accountId` = where it left | `transferToAccountId`; no lines |
| A supplier invoiced us and we have not paid yet | `BILL` | Accounts Payable (automatic) | `vendorId`, `currencyCode`; no `accountId` |
| A customer owes us but there is no invoice | `RECEIVABLE` | Accounts Receivable (automatic) | `clientId`, `currencyCode`; no `accountId` |
| We bought something to keep and depreciate | `EXPENSE` or `BILL` with a Property, Plant and Equipment category, **then** register it (below) | | |
| Anything needing arbitrary debits and credits | **Not available.** Manual journals are deliberately outside this connection. Say so, and offer `request_fiskl_feature` or the Fiskl app. | | |

"Paid the supplier" is an `EXPENSE` from the bank; "received the supplier's invoice, will
pay later" is a `BILL`. On a cash basis the user will mostly want the first; on accrual
ask which one they mean if it is not obvious from the story.

## The rules Fiskl will enforce

Learn these so you get the request right first time; the server reports each one against
the field that broke it.

- **Categories.** A line's `categoryId` is a normal ledger account — never a bank, cash
  or card account (that is a transfer), never the hidden system accounts. A `BILL` can
  only post to purchase-side categories (expenses, cost of sales, assets, prepaid); a
  `RECEIVABLE` only to income categories.
- **Parties.** Posting to Accounts Receivable needs a `clientId`; to Accounts Payable a
  `vendorId`. A client cannot be attached to cash, card, payable, tax or liability
  categories. A vendor can only go on expense, asset or payable categories. Never put a
  client on "Uncategorised Expenses".
- **Tax.** Attach the **rate** (`taxId` or `taxGroupId`, never both) to the line whose
  value the tax was charged on, and give the **gross** amount — Fiskl derives net and
  tax. Never send a tax amount, never choose a tax account as a category, never post a
  net amount with a rate attached (it balances, and the return is silently short). Tax
  is forbidden on tax-disabled categories: cash, cards, payables, loans, equity, tax
  payable, uncategorised.
- **Balance.** Lines must sum to `amount`, in the transaction currency. Split a receipt
  into as many lines as it has categories, not more.
- **Billable** only on money going out, to a client-attachable category.
- **Currency.** Money types post in the account's currency. `BILL` and `RECEIVABLE` need
  `currencyCode`. A line in another currency (foreign expense) needs `currencyCode` on
  that line and works only on income and expense categories; give `exchangeRates` if the
  user knows the rate, otherwise Fiskl looks it up for the date.
- **Transfers.** `transferToAmount` defaults to `amount`. It may be *less* in the same
  currency — the shortfall is booked to Bank Charges and shown in the preview — never
  more. Across currencies it is required: what actually arrived.
- **Lock date.** Nothing can be recorded on or before the accounting lock date. Do not
  move the date to get past it; tell the user.

## Recording — the procedure

1. Resolve every id (accounts, tax, client or vendor). Do not proceed with a guess.
2. **`preview_transaction(...)`** with the full request. It runs the same validation and
   arithmetic as the write and saves nothing.
3. **Show the user** what will be posted, in their currency with the decimal point in the
   right place: the type, the account, each line with its category, net, tax and rate,
   the total, and the base-currency figure if different. Mention any warning the preview
   returned (a bank fee, a locked date).
4. **Wait for a yes.** A description of what you *would* do is not consent.
5. **`create_transaction(...)`** with the same request plus a **stable `externalId`** —
   the bank statement line id, "receipt-INV-88213-2026-03-14", the bill number and date.
   Never a random value: the key is what makes a retry safe.
6. **`get_transaction(id)`** and confirm to the user with the id and the figures Fiskl
   stored.

**An error does not mean nothing was recorded.** The write commits before the response is
built, so a failure or timeout can come back *after* the entry is in the books. If a call
fails, or you are not sure whether it went through, **`list_transactions`** on that
account for that date before doing anything else. Never call `create_transaction` twice
for the same intent without an `externalId` — with one, a retry is safe: it returns the
existing entry instead of creating another.

Errors name the field. `lines[0].taxId … Tax cannot be applied to category X` means
change the category or drop the tax, not try another rate. A subscription error means the
plan has no accounting; say so.

## Correcting an existing entry

`update_transaction` **replaces** the whole entry: `get_transaction` first and resend
every field and line you want to keep. The type cannot change — an expense cannot become
a bill; that is a different holder account and needs a new entry.

Check `editable` on `get_transaction` before promising anything. It is false, and the
update will be refused, when the entry is:

- **owned by another document** (`managedBy`: bill, expense, goods receipt) — change the
  bill or expense instead;
- **matched or linked** — a payment applied to an invoice, a settlement against a bill, a
  matched expense; Fiskl protects the match, unmatch in the app first;
- **in a locked period**;
- **a mirror row** from `list_transactions` — use its `id`, which is the primary record;
- **a journal or other shape** the API does not model.

Explain the reason in plain words. Do not try to work around it by recording an offsetting
entry unless the user explicitly asks for that and understands both entries will remain.

There is **no delete**. A duplicate the user wants gone is removed in the Fiskl app; tell
them which one, by date, amount and id.

## Registering a fixed asset

An asset lives in the register as a depreciation plan. The purchase and the register are
two records, linked by `sourceTransactionId`.

1. **`list_fixed_assets`** — is it already registered? Check by name and cost.
2. **`list_transactions`** on the account it was paid from — find the purchase. If it is
   not recorded, record it first: an `EXPENSE` (or `BILL`) whose category is the asset
   cost account, following the procedure above.
3. **`fixed_asset_default_accounts`** — the cost accounts, the accumulated-depreciation
   contra accounts and the depreciation expense account. When several cost or contra
   accounts exist (vehicles, equipment, computers), **ask which class this asset is**.
4. Collect what the plan needs: cost (gross of irrecoverable tax, net of recoverable),
   residual value (default 0), the date depreciation starts, the useful life in periods,
   and the method — `StraightLine` unless the user or their accountant said otherwise.
   `DiminishingValue` takes a yearly `ratePercent` and no `periods`.
5. **`preview_fixed_asset(...)`** and show the yearly charges, the end date and the
   closing book value. Wait for a yes.
6. **`create_fixed_asset(...)`** with `sourceTransactionId` set to the purchase. Then
   `get_fixed_asset(id)` to confirm.

`create_fixed_asset` has **no idempotency key**. If the call fails, `list_fixed_assets`
before retrying.

**Migrated assets** — bought and partly depreciated before the company was on Fiskl:
set `postingStartDate` to the first period Fiskl should actually post and
`openingAccumulatedDepreciation` to what was already recognised. Fiskl records the
earlier periods as brought forward instead of posting them again. Without this, a
backdated plan catch-up posts every past period.

## Fixing what diagnose-the-books found

| Finding | Do |
|---|---|
| A bank or card line is missing from Fiskl | `EXPENSE` / `INCOME` on that account, from the statement line |
| Money moved between accounts but only one side is recorded | Do **not** record a second one-sided entry. The existing entry should be changed to a `TRANSFER` in the app, or matched there; the API cannot re-match. Explain. |
| Wrong category on an entry that is `editable` | `get_transaction`, resend with the corrected `categoryId`, same tax rate unless the new category forbids it |
| Tax missing on one of a series of identical bills | `update_transaction` adding the `taxId`, if editable; the amount stays gross |
| Depreciation posting but no asset registered, or asset registered but purchase never recorded | Record the purchase with the asset cost account as category, then register (or ask the user to link) the asset |
| Duplicate entries | Name both; the user deletes one in the app |
| Anything needing a journal (accruals, corrections across equity, opening balances) | Not here — accountant or the app |

One fix at a time, each with its own preview and its own yes. A list of changes the user
approved as a batch still gets recorded one by one, and you stop at the first refusal.

## Worked case

The user says: "My accountant says the books show depreciation on a laptop but the laptop
was never bought in Fiskl."

`list_fixed_assets` shows "MacBook Pro", cost 3,600.00, straight line, 36 months, with no
`sourceTransactionId`. `list_transactions` on the main bank account for the month of the
acquisition date finds a 3,600.00 card payment to "Apple" categorised to "Uncategorised
Expenses". So the purchase *was* paid, just never categorised as an asset — the fix is a
recategorisation, not a new entry, and the asset should then be linked.

`get_transaction` shows `editable: true`. You show the user the current entry and the
proposed one: same date, same 3,600.00, category "Computer Equipment" (the asset's cost
account from `get_fixed_asset`), no tax because the receipt showed none. They agree.
`update_transaction` with that line. Then you tell them the register entry cannot be linked
to the purchase from here — updating an existing asset is done in the app — and give them
both ids.

Had the payment not been there at all, you would have recorded it as an `EXPENSE` from the
card with the asset cost account as category, `externalId` "apple-macbook-2026-01-15",
and *then* asked whether to link it in the app.

## Never do this

- Guess an account, a tax rate, a client or a vendor. Ask, and show the candidates.
- Post a net amount with a rate attached, or invent a tax amount or a tax account.
- Create without a preview the user has seen, or retry a create without an `externalId`.
- Hand-record a purchase the user also sent as a file, or one whose scan is still running.
- Touch a matched, linked, bill-owned or locked entry, or record an offset to get around it.
- Backdate into a locked period, or move a date to make an entry fit.
- Record an entry on the receiving account of a transfer as if it were income.
- Post a manual journal. It is not possible here, by design.

## Staying inside what you know

You record what the user tells you happened, the way Fiskl's rules require. You do not
decide whether an expense is **deductible**, which **tax rate** applies, whether a purchase
should be **capitalised** or expensed, its **useful life**, or how revenue should be
**recognised**. Those depend on jurisdiction and circumstance and belong with their
accountant. When the user has not told you and it changes the entry, ask — do not default.
