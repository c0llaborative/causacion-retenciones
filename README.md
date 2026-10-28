# Causación de retenciones

A monthly close for Colombian withholding tax, rebuilt from the source
invoices instead of typed by hand.

Status: in use at an accounting practice since June 2026.
**The source is private.** It runs against a client's purchase invoices and
their accounting ledger, and it holds credentials for the DIAN tax portal.
This page is the architecture and the measured result.

## The problem

Every month a Colombian accounting practice prepares a working paper called
*pruebas de causación*. For each purchase invoice it records the supplier, the
tax base, the payment terms, the supplier's tax standing, and then the three
withholdings that follow from all of it: retefuente (income tax withheld at
source), ReteIVA (VAT withheld) and ICA (a municipal turnover tax whose rate
depends on the activity code and the municipality).

It is done by reading each PDF invoice and typing into a spreadsheet. It takes
about two hours, the rates come from a table nobody re-derives, and the errors are quiet:
a wrong rate produces a plausible number.

## What it does

Given a month of purchase invoices as PDFs and the accounting ledger export,
it produces three files:

1. The client's working paper, in the format the practice already uses.
2. An internal findings file listing every invoice where the computed result
   disagrees with what was recorded, with the reason.
3. An audit log recording what was read, what rule fired, and why.

Every withholding is recomputed from the 2026 rate tables and the invoice
itself. The supplier master is never consulted for rates, which matters for
the reason in the next section.

## How it works

```
purchase invoices (PDF)  ──┐
                           ├──►  extract  ──►  classify  ──►  apply rules  ──►  working paper
accounting ledger (xlsx) ──┘         │              │              │                findings
                                     │              │              │                audit log
                              vision OCR for   tax standing    2026 retefuente,
                              scanned invoices  from invoice    ReteIVA and ICA
                                                 text          rate tables
                                                                    │
                                              DIAN portal  ◄────────┘
                                              CUFE verification
```

Four decisions carried most of the weight:

**Rates come from the rules, never from the answer sheet.** The verification
run used an empty supplier master, so a rate could only come from the 2026
tables and the invoice. Had it read rates from the existing supplier file, the
test would have been reading back its own inputs and would have proven nothing.

**Every invoice is checked against the DIAN portal by its CUFE**, the unique
electronic-invoice identifier. This catches an invoice that exists on paper
but was never issued electronically. The module reproduced 10 out of 10 of
June's portal results.

**Scanned invoices go through a vision model, not classical OCR.** The
original pipeline used Tesseract. It had been failing silently: returning
empty text on scanned documents rather than an error, which meant those
invoices quietly fell through with no extracted data. Replacing it with a
vision model, and making an empty extraction a hard failure, closed that hole.

**Findings are separated from output.** The tool never silently corrects the
client's file. A disagreement becomes a line in the findings file for a human
to rule on, because the tool is frequently right and occasionally wrong, and
the accountant is accountable either way.

## The result

The acceptance criterion was reproducing a month that had already been done by
hand, not producing a July file that looked plausible. The tool ran over June's
inputs and was compared cell by cell against the working paper a person had
prepared, across 28 invoices and 14 fields.

| Field | Same | Different |
|---|---:|---:|
| NIT (tax ID) | 28 | 0 |
| Invoice date | 28 | 0 |
| Posting date | 28 | 0 |
| Retefuente withheld | 28 | 0 |
| ReteIVA withheld | 28 | 0 |
| ICA withheld | 28 | 0 |
| ReteIVA rate | 28 | 0 |
| Supplier | 27 | 1 |
| Retefuente rate | 27 | 1 |
| ICA rate | 27 | 1 |
| Tax base | 26 | 2 |
| Subtotal | 24 | 4 |
| Payment terms | 21 | 7 |
| Tax standing | 19 | 9 |

**93.6% agreement across 392 comparisons.** All three withholding amounts, the
figures the whole paper exists to produce, matched on every invoice.

The disagreements are the interesting part. Each was opened against the
original PDF, and most of them are mistakes in the hand-made paper:

- **Payment terms, 7 differences.** The paper recorded "Crédito" where the
  invoice prints "Contado", and the reverse. Transcription errors, confirmed
  against three suppliers' invoices.
- **Tax standing, 9 differences.** In every case reviewed, the tool marked
  *more* than the person had, correctly. One supplier's invoice prints that it
  is a large ICA taxpayer in Bogotá; the paper had not recorded it, and the
  withholding charged to it was wrong as a result.

That inversion is the finding worth stating plainly: measured against a human
baseline, most of the gap was the baseline being wrong. A number like 93.6%
read as a grade would be misleading. What it actually measures is how far the
two documents differ, and the tool is on the right side of most of that
difference.

## Why the source is private

The repository contains a client's purchase invoices, their accounting ledger,
their supplier list, and stored credentials for the DIAN portal. None of that
is mine to publish, and stripping it would leave a pipeline with nothing to
run against.

## Related

- [`accountant-releases`](https://github.com/c0llaborative/accountant-releases),
  the public release channel for the desktop tools used alongside this one.
