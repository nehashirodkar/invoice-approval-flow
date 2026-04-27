# Deployment guide

Click-by-click steps to import this flow into a Power Platform environment and get it running end-to-end.

## Prerequisites

- A Power Platform environment with **Dataverse** provisioned and **AI Builder** credits available.
- A Microsoft 365 tenant with:
  - A shared mailbox `ap@contoso.com` (rename per your org).
  - A SharePoint site `https://contoso.sharepoint.com/sites/finance`.
  - A Microsoft Teams team `Finance` with a channel `Invoices`.
- A service account `svc-flow-ap@contoso.com` with the licences listed in [connectors.md](connectors.md).

## Step 1 — Create the SharePoint document library

1. In the **Finance** SharePoint site, click **New → Document library**.
2. Name it `Invoices`.
3. Inside the library, create two folders: `Approved/` and `Rejected/`.

## Step 2 — Create the SharePoint audit list

1. In the same site, click **New → List → Blank list**.
2. Name it `InvoiceAuditLog`.
3. Add columns:

   | Column | Type | Notes |
   |--------|------|-------|
   | `VendorName` | Single line of text | indexed |
   | `InvoiceNumber` | Single line of text | indexed |
   | `Amount` | Currency | 2 decimals |
   | `DueDate` | Date only | |
   | `Status` | Choice | `Approved`, `Rejected`, `Auto-Approved`, `Error` |
   | `Approver` | Person | optional |
   | `ApproverComment` | Multiple lines of text | |
   | `PdfLink` | Hyperlink | |
   | `ProcessedAt` | Date and time | default = today |

## Step 3 — Import the solution

1. Sign in to [https://make.powerautomate.com](https://make.powerautomate.com) **as the service account**.
2. Switch to the target environment (top-right environment picker).
3. Go to **My flows → Import → Import Package (Legacy)**.
4. Upload `solution/definition.json` from this repo.
5. On the import review screen, for each connection reference:
   - **Outlook 365** → select or create a connection authenticated as `svc-flow-ap@contoso.com` (must have Full Access on the shared mailbox).
   - **AI Builder** → select the environment's default AI Builder connection.
   - **SharePoint** → create a connection and pick the `Finance` site.
   - **Teams** → create a connection authorized to post in `Finance / Invoices`.
   - **Approvals** → no configuration; runs as the flow owner.
6. Click **Import**.

## Step 4 — Configure environment variables

After import, open the flow in edit mode and set these values inline (or as environment variables if you wrap the flow in a managed solution):

| Variable | Example value |
|----------|---------------|
| `SharedMailbox` | `ap@contoso.com` |
| `SharePointSite` | `https://contoso.sharepoint.com/sites/finance` |
| `InvoiceLibrary` | `Invoices` |
| `AuditList` | `InvoiceAuditLog` |
| `TeamsChannelId` | (paste from Teams → channel → Get link to channel) |
| `AutoApproveThreshold` | `1000` |
| `DefaultApprover` | `cfo@contoso.com` |

## Step 5 — Smoke test

1. Turn the flow on.
2. From a personal mailbox, send an email to `ap@contoso.com` with subject `Invoice TEST-0001` and a sample PDF invoice attached.
3. Within ~60 seconds the flow should:
   - Extract fields via AI Builder (visible in the run history).
   - Auto-approve (since the test invoice is < $1,000) or send a Teams Adaptive Card if you used a larger amount.
   - File the PDF into `Invoices/Approved/`.
   - Add a row to `InvoiceAuditLog`.
   - Post to the `Finance / Invoices` Teams channel.
4. Verify the audit row, the PDF location, and the Teams message.

## Step 6 — Promote to production

For a real rollout, wrap the flow into a **managed solution** and push through Power Platform Pipelines:

```
Dev environment  ──►  Test environment  ──►  Prod environment
   (unmanaged)        (managed import)      (managed import)
```

Use environment variables (Step 4) so each environment can point at its own SharePoint site and Teams channel without re-editing the flow.

## Rollback

If a deployed version misbehaves, the previous version stays available under **My flows → flow → Versions**. Restore the prior version with one click; no code redeploy is needed.
