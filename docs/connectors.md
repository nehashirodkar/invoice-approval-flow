# Connectors and permissions

The flow uses six standard connectors plus AI Builder. All are available on a Power Automate **Premium (Per-User with attended RPA)** or **Per-Flow** licence; AI Builder is metered separately by credits.

| # | Connector | Auth | Actions used | Permissions required on the service account |
|---|-----------|------|--------------|----------------------------------------------|
| 1 | **Office 365 Outlook** | OAuth (delegated) | `When a new email arrives (V3)`, `Reply to email (V3)` | `Mail.Read` and `Mail.Send` on the shared mailbox `ap@contoso.com` |
| 2 | **AI Builder** | Service principal (env-bound) | `Predict` (Invoice Processing prebuilt model) | AI Builder credit allocation on the environment; `Environment Maker` role on the service account during import |
| 3 | **Approvals** | Built-in | `Start and wait for an approval` | None — uses the running user identity |
| 4 | **Microsoft Teams** | OAuth (delegated) | `Post message in a chat or channel` | Member of the target Team; channel `Finance / Invoices` must exist |
| 5 | **SharePoint** | OAuth (delegated) | `Get file content`, `Move file`, `Create item` | `Contribute` on the `Invoices` library and `InvoiceAuditLog` list |
| 6 | **Data Operations** | Built-in | `Compose`, `Parse JSON`, `Select` | None |
| 7 | **Control** | Built-in | `Condition`, `Switch`, `Apply to each`, `Scope` | None |

## Recommended: service-account principal

The flow's connection references should all be owned by a dedicated Microsoft 365 account (e.g., `svc-flow-ap@contoso.com`) so the workflow does not break when an individual user changes role or leaves the organization.

Required licences on that service account:

- Microsoft 365 E3 (or equivalent — for Outlook, SharePoint, Teams)
- Power Automate Premium Per-User
- Access to the AP shared mailbox (Full Access + Send As)

## AI Builder credit estimation

Prebuilt **Invoice Processing** model is metered at **1 credit per page processed**. At ~500 invoices/month and an average of 2 pages each:

- **~1,000 credits/month** consumed
- The default 1M-credit AI Builder add-on covers this comfortably; no per-invoice cost concerns at this volume.

## Network / DLP considerations

- All connectors are first-party Microsoft connectors and live in the **Business** data group, so a default DLP policy that blocks Business ↔ Non-Business connector mixing will not affect this flow.
- No on-premises data gateway is required.
- No custom connector or HTTP action is used, so the flow is portable across environments without code changes.
