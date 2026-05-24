# Workflow: Set up an invoice extractor

End-to-end setup: an Affinda workspace + Invoice document type ready
to receive uploads.

The MCP server hosts the same flow as a one-shot prompt
(`setup_invoice_extractor`). This file is the long-form expansion: same
plan, with edge cases, decision points, and what to say to the user at
each step.

> **Slug case:** examples below use **camelCase** slugs
> (`invoiceNumber`, `taxAmount`, `lineItems`) — the convention this
> platform standardises on. The integration code path reads fields by
> key (`doc.get("invoiceNumber")`), so consistency matters.

## When to use this workflow

The user wants to extract structured data from invoice documents. The
trigger phrase can be any of:

- "Set up an invoice extractor"
- "I need to extract invoice data"
- "How do I process invoices in Affinda?"
- "Read invoice fields automatically"

Don't use this for receipts (different schema), purchase orders
(different schema), or generic billing documents. For those, follow
the same general structure with the relevant field set.

## Plan

### 1. Locate the organisation

```
list_organizations
```

If multiple are returned, ask the user which one. If none, the user has
no Affinda account yet and this skill can't proceed.

### 2. Create the workspace

```
create_workspace(
    name="Invoices",                   # or the user's own phrasing
    organization_id=<from step 1>,
    visibility="organization",         # default unless user says private
    ocr_mode="always-partial",         # see decision below
    enable_document_splitting=False,   # see decision below
    enable_document_classification=True,
    reject_duplicates=True,
    model_memory_strategy="auto",
)
```

**OCR mode decision:**
- Mostly digital PDFs from accounting software → `always-partial`.
- User mentions "scans", "phone photos", "physical bills" → `always-full`.
- Pure native PDFs only, never scans → `skip` (rare).

**Splitting decision:**
- User mentions "batch", "pack", "multiple invoices in one file" →
  `enable_document_splitting=True`. Then call
  `list_document_splitters` first and pass `document_splitter_id` for
  the "General Document Splitter" entry.
- User uploads one invoice per file → `False`.
- Unsure → `False` initially; can flip later via `update_workspace`.

### 3. Create the document type

```
create_document_type(
    name="Invoice",
    organization_id=<from step 1>,
    disable_confirmation_if_validation_rules_fail=False,
)
```

### 4. Add the canonical fields in one call

Use `bulk_create_fields`. The schema below is the standard starting
point — adapt to the user's actual format:

```python
[
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
]
```

### 5. Assign the document type to the workspace

If you used `create_document_type` in step 3 with the workspace context,
this is automatic. Otherwise:

```
assign_document_type_to_workspace(
    document_type_id=<from step 3>,
    workspace_id=<from step 2>,
)
```

### 6. (Optional) Add validation rules

For invoices, two are usually worth proposing:

```
create_validation_rule(
    document_type_id=<from step 3>,
    prompt="@<subtotal_id> + @<taxAmount_id> must equal @<totalAmount_id> within rounding tolerance.",
    field_ids=[<subtotal_id>, <taxAmount_id>, <totalAmount_id>],
    enabled=True,
)
create_validation_rule(
    document_type_id=<from step 3>,
    prompt="The sum of @<lineItems_id>[*].amount must equal @<subtotal_id>.",
    field_ids=[<lineItems_id>, <subtotal_id>],
    enabled=True,
)
```

> Validation rule prompts use ``@<numeric_field_id>`` references (the
> integer id from ``list_fields``), not field slugs. The UI renders
> ``@<id>`` as a styled chip; plain text shows as raw characters.

Note the `@<field_id>` syntax — fetch field IDs from the
`bulk_create_fields` response before composing.

### 7. Tell the user what to do next

- The workspace and document type are ready.
- They can upload invoice files via the Affinda UI or API.
- Classification is enabled, so misuploaded docs get flagged rather than
  silently mis-extracted.
- Model memory is on `auto`, so the first ~10 confirmed documents will
  noticeably improve later extraction quality.

## Variants

| User context | Adjustment |
|---|---|
| Tax-inclusive jurisdiction | Drop `subtotal` and `taxAmount`, keep `totalAmount` only. |
| Multi-currency | Connect `currency` to a data source of allowed ISO codes (`workflows/connect-validation-data.md`). |
| PO-driven AP workflow | Add `purchaseOrderNumber` (text) before `invoiceNumber`. |
| Recurring vendors | Connect `vendorName` to a data source of approved vendors. |

## Common mistakes to avoid

- Calling `create_field` in a loop instead of `bulk_create_fields`.
- Adding fields piecemeal *after* uploading documents — early fields
  influence model memory; rebuilding the schema later means earlier
  extractions don't benefit.
- Enabling splitting without first calling `list_document_splitters` —
  the API rejects splitting without a `document_splitter_id`.
- Setting `ocr_mode="skip"` for any workspace that might receive
  scanned input. You'll get empty extractions.
