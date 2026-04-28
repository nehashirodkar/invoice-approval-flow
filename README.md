# Invoice Approval Automation — Power Platform

A design-only repository describing an end-to-end **invoice approval workflow** built on Microsoft Power Platform. This repo contains the flow definition, architecture documentation, and deployment steps — no code is executed locally; the flow runs in a Power Platform environment after import.

> **Tenant verification (2026-04-28):** AI Builder's prebuilt **Invoice Processing** model was confirmed available in the target Microsoft 365 tenant, the **Office 365 Outlook** trigger configures cleanly, and the **SharePoint** audit list (`InvoiceAuditLog`) was provisioned per `docs/deployment.md`. A full end-to-end run was not captured for this submission.

## Scenario

When a vendor emails an invoice to a shared finance mailbox, the workflow:

1. **Detects** the new email and downloads the PDF attachment.
2. **Extracts** structured fields (vendor, invoice number, total amount, due date, line items) using **AI Builder's prebuilt invoice processing model**.
3. **Branches on amount**:
   - **≤ $1,000** → auto-approved, recorded, and filed.
   - **> $1,000** → sent to the appropriate manager via a Teams Adaptive Card for approval.
4. **Records** the outcome in a SharePoint list and moves the PDF to either an `Approved/` or `Rejected/` folder in SharePoint.
5. **Notifies** the finance Teams channel with a summary card.
6. **Replies** to the vendor automatically when an invoice is rejected.

## Architecture

```
   Outlook 365            AI Builder           Approvals (Teams)
   (shared mailbox)     (Invoice model)      (Adaptive Card)
        |                     |                      |
        v                     v                      v
   +-----------------------------------------------------------+
   |                  Power Automate Cloud Flow                |
   |     trigger -> extract -> condition -> approval -> log    |
   +-----------------------------------------------------------+
        |                     |                      |
        v                     v                      v
   SharePoint List       SharePoint Library      Teams Channel
   (audit trail)         (Approved / Rejected)   (notifications)
```

See [docs/architecture.md](docs/architecture.md) for a component-level breakdown.

## Repository layout

```
invoice-approval-flow/
├── README.md
├── docs/
│   ├── architecture.md       Component breakdown and data flow
│   ├── connectors.md         Connectors used + required permissions
│   ├── deployment.md         Click-by-click import and configuration
│   └── demo-build-guide.md   Step-by-step build of a trimmed demo flow in the designer
├── solution/
│   ├── definition.json       Power Automate flow definition (Workflow Definition Language)
│   └── manifest.xml          Solution manifest stub
└── samples/
    └── README.md             Notes on test data (no real invoices committed)
```

## Power Platform components used

| Component | Purpose |
|-----------|---------|
| **Power Automate** (cloud flow) | Orchestration |
| **AI Builder** | Invoice field extraction (prebuilt model) |
| **Approvals** | Manager sign-off via Teams Adaptive Card |
| **Outlook 365** connector | Email trigger + reply |
| **SharePoint** connector | Audit list + document library |
| **Teams** connector | Channel notifications |

A future iteration could add a **Power App** (canvas) for finance staff to review pending invoices, and **Power BI** for spend dashboards. Both are out of scope for this submission.

## Getting started

This is a design-only repo. To stand the flow up in a real environment, follow [docs/deployment.md](docs/deployment.md). The high-level steps are:

1. Provision a Power Platform environment with Dataverse + AI Builder credits.
2. Create the SharePoint list and document library described in `docs/deployment.md`.
3. Import `solution/definition.json` into Power Automate (My Flows → Import → Package).
4. Edit each connection reference and re-point to your tenant's Outlook / SharePoint / Teams.
5. Turn the flow on and email a test invoice to the trigger mailbox.

## Status

Design-only — flow definition is authored to the Power Automate Workflow Definition Language schema and is ready for import. Live tenant deployment, screenshots, and runtime telemetry are deferred.
