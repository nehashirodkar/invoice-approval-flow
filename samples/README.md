# Samples

This folder is intentionally empty of real invoices.

For local testing of the imported flow, generate or download a synthetic
invoice PDF and email it to the trigger mailbox configured in
[../docs/deployment.md](../docs/deployment.md). Suggested sources:

- The [AI Builder invoice processing prebuilt model docs](https://learn.microsoft.com/ai-builder/prebuilt-invoice-processing)
  ship sample invoice PDFs you can use without licensing concerns.
- Generate one with any invoice template tool — the prebuilt model expects
  human-readable text (vendor block, invoice number, line items, total).

**Do not commit real vendor invoices** to this repo. They contain PII and
counterparty data and have no place in a design submission.
