# Workflow: Connect a data source to a field

Wire a reference list (vendors, account codes, product SKUs) into a
document-type field so extracted values are validated and normalised
against the user's canonical record.

## When to use this workflow

Trigger phrases:
- "Validate against our vendor list"
- "Match extracted values to a master list"
- "Map invoice vendor names to our system IDs"
- "Use our account codes"

This is the same flow as the MCP `connect_data_source` prompt; this
file is the long-form expansion.

## Prerequisites

Confirm with the user before starting:

1. **Which document type and field** the lookup is for.
2. **Where the reference data lives** — a CSV / spreadsheet, an
   inline list, or something they want to enter manually.

## Plan

### 1. Get the organisation

```
list_organizations
```

### 2. Create the data source

```
create_data_source(
    name="Vendors",                # short, descriptive
    organization_id=<...>,
    key_property="vendorId",       # uniquely identifies a row
    display_property="vendorName", # column the user will recognise
    schema_definition=[...],       # column schema
)
```

`schema_definition` should mirror the user's source data. Mark
required columns explicitly.

### 3. Populate the values

For any list of more than one row, **always use the bulk tool**:

```
bulk_create_data_source_values(
    data_source_id=<...>,
    values=[
        {"vendorId": "V001", "vendorName": "Acme Corp"},
        {"vendorId": "V002", "vendorName": "Globex"},
        ...
    ],
)
```

The bulk tool tolerates JSON-string and native-list inputs. After
populating, sanity-check:

```
list_data_source_values(data_source_id=<...>)
```

### 4. Attach the data source to the target field

```
update_field(
    field_id=<...>,
    mapping_data_source_id=<...>,
)
```

This connects the field to the lookup. Attaching automatically sets
the field's type to `enum` — a data source only functions on an enum
field, so you don't need a separate `update_field` call to retype it.
**Extraction behaviour does not change yet** — the server creates a
default matching criterion for you (next step).

### 5. Configure matching behaviour

When you attach a data source the server creates a default matching
criterion automatically. Inspect it:

```
list_matching_criteria(field_id=<...>)
```

Decide whether to adjust:

- **Default fuzzy match** is fine when the document field maps
  directly to the data source's `display_property`.
- **Stricter matching**: `update_matching_criterion` with
  `match_type="exact"`, `required=True`, or `required_strict=True`.
- **Multi-field matching** (e.g. match vendor name AND vendor address):
  `create_matching_criterion` for the additional field pairings.

### 6. Validate end-to-end

Tell the user the connection is live. Suggest they upload a test
document. Existing documents in the workspace can be reprocessed via
`reassign_document_type` if they want to re-extract with the new
matching.

## Common pitfalls to avoid

- **Skipping step 5** — attaching a data source without a matching
  criterion gives a decoratively-linked field that doesn't actually
  look anything up at extraction time. Always check
  `list_matching_criteria` after attach.
- **Calling `create_matching_criterion` before `update_field`** — the
  field must be linked to the data source first.
- **Looping `create_data_source_value` for a CSV import** — always
  use `bulk_create_data_source_values`.
- **Adding values incrementally and forgetting `bulk_*`** — the bulk
  tool is append-only, so users sometimes mistakenly fall back to
  single-row creates for additions. Single-row is fine for *adding*
  to an existing source; the bulk-vs-single rule only matters for
  initial population.

## Variants

| Use case | Adjustment |
|---|---|
| Multi-currency invoices | Connect `currency` field to a data source of allowed ISO codes. Use `match_type="exact"`. |
| Account-code validation | Connect `accountCode` field to a chart-of-accounts data source. Use `required=True` to block extractions with unmapped values. |
| Approved-vendor list | Connect `vendorName` to vendor data source. Don't use `required=True` initially — it'll reject unknown vendors before the user has a chance to review them. |
