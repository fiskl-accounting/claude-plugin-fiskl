# How Fiskl's ledger records a transaction

Read this when a user asks *why* — why there is no journal entry, why the amount has no
sign, why the tax has to be a rate, why an entry cannot be changed. It explains the model
the `record-transactions` rules come from. It is not accounting advice.

## Every transaction is a small balanced journal

Fiskl stores one `AccountTransaction` per event, made of lines that must sum to zero:
the first line always sits on the **holder account** (the account whose register the
entry was made in), the other lines are the **categories** it was posted against. A
receipt for 120.00 paid from the bank is two lines: bank −120.00, Office Supplies
+120.00. A split receipt is one bank line and several category lines.

There is **no transaction type stored anywhere**. Fiskl derives what an entry *is* from
which account holds it and what its lines point at:

| Holder account | Counterpart | What Fiskl calls it |
|---|---|---|
| Bank, cash or credit card (a "money account") | expense / income / asset categories | an expense or income (money out or in) |
| Money account | another money account | a transfer |
| Accounts Payable | purchase categories | a supplier bill you owe |
| Accounts Receivable | income categories | a customer receivable |
| The hidden Manual Journal account | anything | a manual journal — **not available through the API** |

The API's `type` field is a name for these shapes so that callers never have to know the
system account names. It also decides where the API puts things: `BILL` and `RECEIVABLE`
do not take an `accountId` because the holder is always the Payable or Receivable
account.

## Why amounts are unsigned

Internally, a positive amount means the holder account's **natural balance went up**.
Money into a bank account is positive, but a purchase on a credit card is *also*
positive — the card is a liability and a purchase increases it. The same 120.00 spend is
therefore +12000 on a card and −12000 on a bank account. That is correct double-entry, and
it is exactly the kind of convention an assistant gets wrong half the time. The API takes
an unsigned amount and lets `type` supply the direction; the server applies the sign for
the account's nature. Never try to encode a refund as a negative amount — record an
`INCOME` on the same account instead.

## Why tax is a rate on a gross line

Tax reaches a VAT/GST return through exactly one mechanism: when a line carries a tax
**rate**, the server derives the net and mints a tax line against the rate's
registration. There is no tax account to choose and no tax amount to send:

- **Amounts are gross** (tax inclusive). Post 120.00 with a 20% rate and Fiskl derives
  100.00 net and 20.00 tax. Post 100.00 with the rate attached and the books still
  balance, but the return is short by a sixth — silently.
- **A tax account is never a category.** Posting to "VAT payable" directly balances the
  ledger and bypasses the return entirely.
- **One registration holds both directions.** Sales and purchase tax are separated at
  report time from the category's side, not by choosing an account. A tax account in
  debit just means input tax exceeded output tax.
- Some categories are **tax-disabled** (cash, cards, payables, loans, equity, tax
  payable, uncategorised): tax on them is refused because it could never reach a return.

So `preview_transaction` returning `netAmount` and `taxAmount` is not a calculation the
API did for display — it is what the books will actually carry.

## Clients and vendors are structural, not decorative

A vendor on a line is what makes the entry appear in supplier reports and Accounts
Payable ageing; a client on a line is what makes it billable or part of a customer
balance. That is why Fiskl **requires** a vendor when posting to Accounts Payable and a
client when posting to Accounts Receivable, and **forbids** a client on cash, card,
payable, tax and liability categories — a client there would be counted where no customer
balance exists. "Uncategorised Expenses" cannot carry a client for the same reason: it is
a holding pen, not a cost the client owes.

## Currency

Each money account has its own currency and posts in it; a bill or receivable names the
**document** currency. Fiskl also carries every entry in the company's base currency
(`baseCurrencyAmount`) at the rate for the date, and a single line may be in a third
currency when the cost was incurred abroad — only on income and expense categories, which
is why `currencyCode` on a line is refused elsewhere. Rates you pass in `exchangeRates`
are "units of that currency per one unit of the transaction currency"; anything you omit
is looked up.

## Transfers and the bank-fee shortfall

A transfer is money leaving one money account and arriving in another. Banks sometimes
deliver less than was sent. Fiskl allows that in the same currency and books the
difference to **Bank Charges** automatically — you will see it as a second line in the
preview. More arriving than was sent is refused: money cannot appear in transit, so it
means the wrong counterpart was chosen. Across currencies, the arrived amount is required
because it *is* the exchange rate.

## Mirror rows

Every transaction is stored once, on its holder account, but it is *visible* from both
sides. Listing the receiving account of a transfer shows the same entry with `mirror:
true` and the amount from that account's point of view. The `id` is the one record; use it
to read or update. Never treat a mirror row as a separate entry to be "matched" or
duplicated.

## Why some entries cannot be changed through the API

`editable: false` on a transaction means one of:

- **`managedBy` is set** — a bill, expense claim or goods receipt owns this posting and
  keeps it consistent with its own document. Change the document, not the posting.
- **Matched or linked lines** — a payment applied to an invoice, a settlement against a
  bill, a matched expense. Replacing the lines would orphan the match; Fiskl protects it.
  The match is undone in the app first.
- **Locked period** — the company closed its books to that date. Moving the date to get
  past the lock is falsifying the period; refuse and explain.
- **A shape the API does not model** — a manual journal, or a posting held on a
  non-money, non-AP/AR account.

There is deliberately **no delete**. A wrong entry is corrected by replacing it (if
editable) or removed by the user in the app, where the audit trail records who did it.

## Fixed assets are two records

The **purchase** is an ordinary transaction whose category is an asset cost account
(Property, Plant and Equipment). The **register entry** is a depreciation plan: cost,
residual value, the accounts to debit (depreciation expense) and credit (accumulated
depreciation), method and life. Fiskl posts each period's depreciation from the plan
automatically. `sourceTransactionId` links the two so the register can show what was paid
and when. A plan created without the purchase depreciates an asset the balance sheet never
received; a purchase without a plan sits at full cost forever — the two failure modes
`diagnose-the-books` finds most often.

Migrated assets already partly depreciated elsewhere use `postingStartDate` and
`openingAccumulatedDepreciation`, so the periods before Fiskl took over are recorded as
brought forward instead of posted again.

## Idempotency: why `externalId` matters more than it looks

The write commits **before** the response is built. A timeout or a mapping error can
therefore come back after the entry is already in the books. With an `externalId`, a
retry returns the existing entry; without one, a retry creates a duplicate that balances
perfectly and is invisible until someone reconciles. Always pass one, always derive it
from something stable about the event (statement line, receipt number and date), and
never from the current time.
