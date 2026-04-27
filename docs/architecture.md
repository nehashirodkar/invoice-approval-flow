# Architecture

## End-to-end data flow

```
+------------------+       1. New email w/ PDF
| Vendor sends     | ----------------------------+
| invoice to       |                             |
| ap@contoso.com   |                             v
+------------------+                  +-------------------------+
                                      |  Outlook 365 trigger    |
                                      |  When a new email       |
                                      |  arrives (V3)           |
                                      +-----------+-------------+
                                                  |
                                                  | 2. Filter: has PDF attachment
                                                  v
                                      +-------------------------+
                                      |  Apply to each          |
                                      |  attachment             |
                                      +-----------+-------------+
                                                  |
                                                  | 3. Get attachment bytes
                                                  v
                                      +-------------------------+
                                      |  AI Builder             |
                                      |  Extract info from      |
                                      |  invoices (prebuilt)    |
                                      +-----------+-------------+
                                                  |
                                                  | 4. Parse fields:
                                                  |    vendor, total,
                                                  |    invoice #, due date
                                                  v
                                      +-------------------------+
                                      |  Condition:             |
                                      |  total <= 1000 ?        |
                                      +-----+-------------+-----+
                                       yes  |             |  no
                                            v             v
                                +-------------+   +-------------------+
                                | Auto-approve|   | Start and wait    |
                                | (skip       |   | for an approval   |
                                |  approval)  |   | (Teams card)      |
                                +------+------+   +---------+---------+
                                       |                    |
                                       +----------+---------+
                                                  |
                                                  | 5. Branch on outcome
                                                  v
                                      +-------------------------+
                                      |  Switch: approved /     |
                                      |  rejected               |
                                      +-----+-------------+-----+
                                       app  |             |  rej
                                            v             v
                              +---------------+   +-----------------+
                              | Move PDF to   |   | Move PDF to     |
                              | Approved lib  |   | Rejected lib    |
                              | + log row     |   | + log row       |
                              | + notify Teams|   | + reply email   |
                              +---------------+   +-----------------+
```

## Components

### 1. Trigger — Outlook 365
- **Connector:** `office365`
- **Action:** `When a new email arrives (V3)`
- **Filter:** mailbox `ap@contoso.com`, importance `Any`, has attachment `Yes`, subject filter `Invoice`
- **Why this trigger:** vendors universally email PDFs to a shared AP mailbox; no portal or SFTP needed.

### 2. Attachment loop
- **Action:** `Apply to each` over `triggerBody()?['attachments']`
- Filters attachments by `contentType` containing `pdf` to skip signature images and embedded logos.

### 3. AI Builder — Invoice extraction
- **Action:** `Predict` against the prebuilt **Invoice Processing** model (`b6e6ed5b-...`).
- **Input:** the PDF attachment bytes.
- **Output:** structured object with `vendorName`, `invoiceTotal`, `invoiceId`, `dueDate`, and a `lineItems` array.
- **Why prebuilt:** the prebuilt model handles 80%+ of invoice formats with no training. A custom model can be swapped in later if vendor-specific accuracy is needed.

### 4. Approval branch
- **Threshold:** $1,000. Below auto-approves to keep low-value invoices out of the manager queue.
- **Approval action:** `Start and wait for an approval` (type: `Approve/Reject — First to respond`).
- **Channel:** Teams Adaptive Card to a configurable approver group, with the vendor / amount / due date and a link to the original PDF.

### 5. Recording
- **SharePoint list `InvoiceAuditLog`** receives one row per invoice with status, approver, timestamps, and a link to the filed PDF.
- **SharePoint document library `Invoices`** has subfolders `Approved/` and `Rejected/` — the original PDF is moved (not copied) so the AP mailbox stays clean.

### 6. Notification
- **Teams channel post** summarizes outcome: vendor, amount, approver, status, link to audit row.
- **On rejection only:** a templated email reply goes back to the vendor citing the rejection reason from the approver's comment.

## Error handling

- All AI Builder and SharePoint actions have `runAfter` paths configured for `Failed` / `TimedOut`, routing to a `Compose` action that posts the failure to a `#flow-errors` Teams channel and writes a row with status `ERROR` to the audit list.
- The flow itself runs under a service account so a single user leaving the org does not break the connection references.

## Out of scope (future iterations)

- **Power App** for finance staff to manually re-queue or reclassify invoices.
- **Dataverse** instead of SharePoint for the audit log if reporting / Power BI integration becomes a requirement.
- **Custom AI Builder model** trained on top-10 vendor invoice formats for higher field accuracy.
- **ALM via Power Platform Pipelines** for promoting the solution Dev → Test → Prod.
