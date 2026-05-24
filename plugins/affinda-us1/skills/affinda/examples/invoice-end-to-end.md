# Worked example: invoice extraction, end to end

A realistic agent transcript showing how to apply the
`setup-invoice-extractor.md` workflow to a concrete user request.
This isn't a script the agent should follow verbatim — it's an
illustration of the cadence and the tone.

## User request

> "Hey, I just got hired at a company that processes a lot of supplier
> invoices. We get them mostly as PDFs from email, sometimes scanned.
> I want to extract the standard fields automatically. Can you help me
> set this up in Affinda?"

## Agent response (paraphrased)

"Sure. Let me find your organisation first so I can set this up in the
right place."

```
list_organizations()
```

Returns one organisation: `Acme Procurement` (id: `org_42`).

"I'll set up an Invoices workspace under Acme Procurement. Since you
get a mix of digital PDFs and occasional scans, I'll use the
`always-partial` OCR mode — it trusts embedded text where present and
falls back to OCR for the rest. Sound right?"

(User confirms.)

```
create_workspace(
    name="Invoices",
    organization_id="org_42",
    ocr_mode="always-partial",
    enable_document_classification=True,
    enable_document_splitting=False,
    reject_duplicates=True,
    model_memory_strategy="auto",
)
```

Returns workspace `ws_701`.

"Workspace is up. Now I'll create an Invoice document type and load
it with the standard invoice schema."

```
create_document_type(name="Invoice", organization_id="org_42")
```

Returns document type `dt_812`.

```
bulk_create_fields(
    document_type_id="dt_812",
    fields=[
        {"slug": "invoiceNumber", "label": "Invoice Number", "field_type": "text"},
        {"slug": "invoiceDate", "label": "Invoice Date", "field_type": "date"},
        {"slug": "dueDate", "label": "Due Date", "field_type": "date"},
        {"slug": "vendorName", "label": "Vendor Name", "field_type": "text"},
        {"slug": "vendorAddress", "label": "Vendor Address", "field_type": "text"},
        {"slug": "billTo", "label": "Bill To", "field_type": "text"},
        {"slug": "subtotal", "label": "Subtotal", "field_type": "float"},
        {"slug": "taxAmount", "label": "Tax", "field_type": "float"},
        {"slug": "totalAmount", "label": "Total", "field_type": "float"},
        {"slug": "currency", "label": "Currency", "field_type": "text"},
        {
            "slug": "lineItems",
            "label": "Line Items",
            "field_type": "table",
            "fields": [
                {"slug": "description", "label": "Description", "field_type": "text"},
                {"slug": "quantity", "label": "Quantity", "field_type": "float"},
                {"slug": "unitPrice", "label": "Unit Price", "field_type": "float"},
                {"slug": "amount", "label": "Amount", "field_type": "float"},
            ],
        },
    ],
)
```

```
assign_document_type_to_workspace(
    document_type_id="dt_812",
    workspace_id="ws_701",
)
```

"Done. Want me to add a couple of validation rules — for example,
flagging documents where subtotal + tax doesn't add up to total?"

(User says yes.)

First fetch field IDs so the validation prompts can reference them:

```
list_fields(document_type_id="dt_812")
```

Returns numeric IDs — say `subtotal` is `4821`, `taxAmount` is
`4825`, `totalAmount` is `4827`, and `lineItems` is `4830`.

```
create_validation_rule(
    document_type_id="dt_812",
    prompt="@4821 + @4825 must equal @4827 within rounding tolerance.",
    field_ids=[4821, 4825, 4827],
    enabled=True,
)
create_validation_rule(
    document_type_id="dt_812",
    prompt="The sum of @4830[*].amount must equal @4821.",
    field_ids=[4830, 4821],
    enabled=True,
)
```

"You're set up. To start using it:

1. Upload invoice PDFs via the Affinda UI or API.
2. Documents will land in `review` state once extracted.
3. Open one of them, eyeball the values, then **confirm** to sign it
   off.
4. After ~10 confirmations, you'll see noticeable accuracy
   improvements as the model picks up patterns from your specific
   vendors and formats.

Anything else you'd like me to set up before your first batch?"

## What this example illustrates

- Single confirmation point with the user (OCR mode), then proceed.
- Bulk field creation in one call, not eleven.
- Validation rules proposed as a follow-up question, not silently
  added.
- Concrete next-step instructions with a number for "when does this
  get good?"
- No HITL approval prompts surfaced — tools were called within the
  agent's normal autonomy.

## What didn't happen

- The agent didn't ask the user for a workspace name. "Invoices" was
  picked from context.
- The agent didn't enable splitting — the user said "PDFs from email,
  sometimes scanned", which implies one-document-per-file. If they
  had said "we receive bundles", splitting would have been on.
- The agent didn't ask the user to enumerate every field upfront.
  Proposing the standard schema and offering to extend it after the
  first upload is faster than building a 30-field schema from
  scratch.
