# Demo build guide — minimum-viable flow in 20 minutes

This guide walks you through building a **trimmed-down version of the invoice approval flow** in the Power Automate designer, end-to-end, in your existing work/school tenant. Use it as a script while clicking. The full production flow (`solution/definition.json`) is the design artifact; this is the live demo.

## What you're building

```
Personal mailbox ──► Predict (Invoice Processing) ──► Compose (show fields)
                                                           │
                                                  Condition: total <= 1000
                                                  ┌────────┴────────┐
                                              ≤ 1000             > 1000
                                                  │                 │
                                          Teams: auto-approved   Teams: needs approval
                                                  └────────┬────────┘
                                                  Create item in SharePoint list
```

Stripped from the production flow: shared mailbox, document library moves, Approvals action, vendor reply, error-handling scope. Those are documented in `architecture.md` and `definition.json` for the submission write-up.

## Prerequisites

- Your work/school account is signed into [make.powerautomate.com](https://make.powerautomate.com).
- AI Builder Invoice Processing prebuilt model is available (already verified).
- A SharePoint site you can write to. Easiest: your personal **OneDrive for Business** site, or any team site where you have Edit permissions.
- A sample invoice PDF. Microsoft publishes one here:
  https://github.com/Azure-Samples/cognitive-services-sample-data-files/tree/master/Form%20Recognizer
  Pick `Invoice/Train/Invoice_1.pdf` (or any of the invoice samples). Download it locally — you'll email it to yourself in the smoke test.

## Step 0 — Create the SharePoint list (3 min)

1. Open your SharePoint site (or https://www.office.com/launch/sharepoint).
2. **+ New → List → Blank list**.
3. Name it `InvoiceAuditLog`. Click **Create**.
4. Add these columns (**+ Add column** at the top):

   | Column | Type |
   |--------|------|
   | `VendorName` | Single line of text |
   | `InvoiceNumber` | Single line of text |
   | `Amount` | Number (2 decimals) |
   | `Status` | Choice → values: `Auto-Approved`, `Needs Approval` |

   The default `Title` column stays — we'll write the invoice number into it.
5. Keep the URL of this list handy (you'll point the SharePoint action at it).

## Step 1 — Create the flow (1 min)

1. https://make.powerautomate.com → **+ Create**.
2. Below the Copilot box, find **Start from blank → Automated cloud flow**.
3. Flow name: `Invoice Approval Demo`.
4. Trigger: search **"When a new email arrives (V3)"** (Office 365 Outlook). Select it. Click **Create**.

## Step 2 — Configure the email trigger (2 min)

In the trigger card:

- **Folder**: `Inbox` (default).
- Click **Show all** in the parameters panel:
  - **Subject Filter**: `Invoice`  *(only fire when the subject contains "Invoice")*
  - **Include Attachments**: `Yes`
  - **Only with Attachments**: `Yes`
  - Leave everything else default.

## Step 3 — Loop over attachments (1 min)

1. Click **+ New step** → search **"Apply to each"** (Control connector). Select.
2. In **Select an output from previous steps**, click the lightning-bolt **Dynamic content** picker → choose **Attachments** (from the trigger).

Everything below goes **inside** this Apply-to-each loop.

## Step 4 — Predict with AI Builder (2 min)

1. Inside the loop: **Add an action** → search **"Predict"** (AI Builder).
2. **Model**: pick **Invoice Processing Model** (the prebuilt one).
3. **File**: click into the field → Dynamic content picker → choose **Attachments Content** (from the current loop item — should appear as `items('Apply_to_each')?['contentBytes']` if you peek at the code view).

## Step 5 — Compose to show extracted fields (2 min)

This step is purely so the run history visibly shows what AI Builder extracted — it's the screenshot you want for the submission.

1. Inside the loop, after Predict: **Add an action** → search **"Compose"** (Data Operation).
2. Click into **Inputs** → switch to the **Expression** tab and paste:

   ```
   json(concat('{"vendor":"', outputs('Predict')?['body/predictionOutput/fields/VendorName/value'], '","invoiceNumber":"', outputs('Predict')?['body/predictionOutput/fields/InvoiceId/value'], '","total":', coalesce(outputs('Predict')?['body/predictionOutput/fields/InvoiceTotal/value'], 0), '}'))
   ```

   *(If you'd rather not paste an expression, you can simply pick the three Predict outputs as separate Dynamic-content tokens in the Inputs box — both work.)*

3. Rename the action to `Extracted fields` (top-right → **Rename**) so it's obvious in screenshots.

## Step 6 — Condition: total ≤ 1000 (3 min)

1. Inside the loop, after Compose: **Add an action** → **"Condition"** (Control).
2. Left-hand value: Dynamic content → search "InvoiceTotal" → pick **Predict — Invoice total / Value** (the numeric one, not the formatted text). If you only see a string version, wrap it in an expression: `float(outputs('Predict')?['body/predictionOutput/fields/InvoiceTotal/value'])`.
3. Operator: **is less than or equal to**.
4. Right-hand value: `1000`.

### In the **If yes** branch:

1. **Add an action** → **"Post message in a chat or channel"** (Microsoft Teams).
2. **Post as**: `Flow bot`. **Post in**: `Chat with Flow bot`. **Recipient**: pick your own email address.
3. **Message**: paste:

   ```
   ✅ Auto-approved invoice from @{outputs('Predict')?['body/predictionOutput/fields/VendorName/value']} for $@{outputs('Predict')?['body/predictionOutput/fields/InvoiceTotal/value']}
   ```

   *(Type `@` characters as `at` if the editor is fussy, or use Dynamic content tokens for the vendor name and invoice total.)*
4. **Add an action** → **"Set variable"**? No — instead just remember to set Status to `Auto-Approved` in the SharePoint step. We'll handle that with a `Compose` for the status string. Easier: add a **Compose** here with input `Auto-Approved`. Rename it `Status_value`.

### In the **If no** branch:

1. **Add an action** → **"Post message in a chat or channel"** (Teams). Same chat as above.
2. **Message**:

   ```
   ⚠️ Invoice from @{outputs('Predict')?['body/predictionOutput/fields/VendorName/value']} for $@{outputs('Predict')?['body/predictionOutput/fields/InvoiceTotal/value']} needs manager approval.
   ```

3. Add a **Compose** with input `Needs Approval`. Rename it `Status_value`.

> **Why two Compose actions named the same thing?** Because Power Automate scopes actions by name; only the one in the executed branch produces an output, and we reference whichever one ran in the next step.

## Step 7 — Write to SharePoint (3 min)

After the Condition (still inside the loop):

1. **Add an action** → **"Create item"** (SharePoint).
2. **Site Address**: pick your site from the dropdown (or paste the URL).
3. **List Name**: pick `InvoiceAuditLog`.
4. Fill the fields:

   | Field | Value (use Dynamic content) |
   |-------|------|
   | **Title** | Predict → InvoiceId Value |
   | **VendorName** | Predict → VendorName Value |
   | **InvoiceNumber** | Predict → InvoiceId Value |
   | **Amount** | Predict → InvoiceTotal Value |
   | **Status** | Outputs of `Status_value` *(pick whichever Compose is in scope — if the picker shows two, pick either; only the one in the executed branch will have a real value)* |

## Step 8 — Save and test (3 min)

1. Top-right: **Save**.
2. Click **Flow checker** (top-right) → confirm `0 errors, 0 warnings`. **Screenshot this.**
3. From your phone or another mail client, email **yourself** with:
   - Subject: `Invoice TEST-0001`
   - Attach: the sample PDF you downloaded.
4. Wait ~30–60 seconds. Open the flow → **28-day run history** → click the latest run.
5. Verify each action has a green checkmark. Click into **Predict** to see the extracted JSON.
6. Open the SharePoint list — there should be a new row.
7. Open Teams (or the Flow bot chat) — there should be a message.

## Screenshots to capture for the submission

Save these to `docs/screenshots/` in the repo before you push:

1. **`01-flow-designer.png`** — full designer view with all actions visible (zoom out if needed).
2. **`02-flow-checker.png`** — the "0 errors, 0 warnings" confirmation.
3. **`03-run-success.png`** — run history showing all green checkmarks for one run.
4. **`04-predict-output.png`** — expanded **Predict** action's Outputs panel showing the extracted vendor / invoice number / total.
5. **`05-sharepoint-row.png`** — the SharePoint list with the audit row.
6. **`06-teams-message.png`** — the Teams Flow-bot message.

Optional but nice:
7. **`07-test-email.png`** — the email you sent (so reviewer sees the input).
8. **`08-condition-branch.png`** — zoomed view of the Condition card showing both branches.

## After the demo

Add a short note to the top of `README.md`:

> **Status:** Deployed and tested in a Microsoft 365 tenant on YYYY-MM-DD. See `docs/screenshots/` for evidence.

…and commit + push. That turns the design-only repo into a "designed and demonstrated" submission.

## Troubleshooting

- **Predict fails with "model not found"** — the prebuilt model needs to be activated in the environment. Open https://make.powerapps.com → **AI hub** → search "Invoice processing" → **Use prebuilt model**.
- **InvoiceTotal comes back as a string with `$`** — the prebuilt model often returns the value as text. Wrap with `float()` in expressions, or use the action's structured output if your version exposes a numeric `value` field.
- **Apply-to-each iterates over inline images, not just the PDF** — add an extra **Condition** at the top of the loop that checks `contains(toLower(items('Apply_to_each')?['contentType']), 'pdf')` and put everything else in the **If yes** branch.
- **Teams "Post as Flow bot" greyed out** — switch to **User** as posting identity, or use **Send me a mobile notification** (Microsoft Notifications connector) as a fallback.
