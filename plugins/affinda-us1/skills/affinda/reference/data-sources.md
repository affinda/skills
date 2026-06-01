# Data Sources — operational reference

Operational rules for creating and maintaining data sources (lookup
tables of records that fields can be matched against), their values
(rows), and matching criteria. Pairs with `reference/concepts.md`
(which defines what a data source *is*) and
`workflows/connect-validation-data.md` (the end-to-end flow for
wiring a data source into a field).

## Default schema

If no custom `schema_definition` is provided, data sources use a
default three-property schema:

| Property | Type | Required | Description |
|---|---|---|---|
| `value` | string | yes | Unique key (matches `keyProperty`). |
| `label` | string | no | Display text shown in the UI (matches `displayProperty`). |
| `description` | string | no | Optional additional detail. |

- **`keyProperty`** (default `"value"`) uniquely identifies each
  record and is the internal value during extraction.
- **`displayProperty`** (default `"label"`) is what the user sees in
  the UI. It can differ from `keyProperty`.

NOTE: When setting these, you should expect them to be like "email"
not "Email". Check the available keys first.

## File-driven creation: `create_data_source_from_file`

If the user has uploaded a CSV / JSON / Excel (.xlsx) file to the
chat, use `create_data_source_from_file` to create + populate in one
step. It:

- Reads the uploaded file from the conversation.
- Parses it (CSV columns become properties, JSON objects become
  records, Excel uses sheet 1).
- Auto-detects the key column (first column with all unique values)
  and display column.
- Builds the JSON schema from the file's columns.
- Creates the data source and populates it with all rows.

Choose an appropriate name based on the file (e.g. `suppliers.csv` →
"Suppliers"). If no `file_id` is given, the most recent data file in
the conversation is used.

`create_data_source_from_file` only **creates** new data sources.
There's no tool that bulk-loads a file into an existing data source.
If the user wants that, parse rows yourself and feed them through
`bulk_create_data_source_values`, or direct them to the Affinda UI.

## Single-row vs bulk row tools

Conversational, one-row-at-a-time changes use the singular tools:

- **`create_data_source_value`** — add one row.
- **`update_data_source_value`** — partial merge: only fields you
  pass change; everything else on the row is preserved.
- **`delete_data_source_value`** — delete by key.

Multi-row asks use the bulk tool:

- **`bulk_create_data_source_values`** — append many rows at once.

> Every single-row call is a full replace-all under the hood, so
> looping `create_data_source_value` for an N-row request does N
> times the work and produces N separate confirmations in the chat.
> Whenever the user provides more than one row (a pasted list, an
> "add these N suppliers" request, etc.), use the bulk tool.

For multi-row **updates** or **deletes**, there's no bulk shortcut —
call the singular tools per row. Only bulk **append** is supported.

### Finding the row key for update / delete

The `key` is the value of the data source's `key_property` for the
target row. If `key_property == "value"` and the row is `{"value":
"ACME", "label": "Acme Corp"}`, then `key == "ACME"`.

If you don't know `key_property` or the exact row key:

1. Call `get_data_source` to see `key_property`.
2. Call `list_data_source_values` to find the matching row.

### Example flows

**"Add supplier ACME with label Acme Corp"** (data source
`key_property == "value"`):

```python
create_data_source_value(
    data_source_id="...",
    value='{"value": "ACME", "label": "Acme Corp"}',
)
```

**"Add these suppliers: ACME, GLOBEX, INITECH"** (multi-row → use
the bulk tool):

```python
bulk_create_data_source_values(
    data_source_id="...",
    values=[
        {"value": "ACME", "label": "Acme Corp"},
        {"value": "GLOBEX", "label": "Globex Inc"},
        {"value": "INITECH", "label": "Initech"},
    ],
)
```

**"Change Acme's label to 'Acme Corporation'"**:

```python
update_data_source_value(
    data_source_id="...",
    key="ACME",
    value='{"label": "Acme Corporation"}',
)
```

**"Remove supplier ACME"**:

```python
delete_data_source_value(
    data_source_id="...",
    key="ACME",
)
```

### Argument-shape pitfall for `bulk_create_data_source_values`

The single-row tools take `value` as a **JSON string**. The bulk tool
does NOT — it takes `values` as a **native list of dicts**.

❌ Wrong:

```python
bulk_create_data_source_values(
    data_source_id="...",
    values='[{"value": "ACME", "label": "Acme"}]',  # JSON string — fails
)
```

✅ Right:

```python
bulk_create_data_source_values(
    data_source_id="...",
    values=[{"value": "ACME", "label": "Acme"}],   # native list
)
```

If you see `Input should be a valid list ... input_type=str`, you
serialised `values` as a string. Retry the **same** tool with a
native list. Don't fall back to looping single-row calls — preserve
the user's bulk-add intent.

## Schema changes are out of band

These tools don't change a data source's schema or columns. You
cannot add a new column via a row update. To change the schema, the
user has to do it through the Affinda UI.

## Attaching to a field — and the matching-criterion step

A data source only functions on an `enum` field — the server creates
the default matching criterion (and extraction maps values against the
data source) only for enum fields. Both `attach_data_source_to_field`
and passing `mapping_data_source_id` on `create_field` / `update_field`
set the field type to `enum` automatically, so you never need a
separate retype call. A field created with a data source works out of
the box.

After `attach_data_source_to_field` (or passing
`mapping_data_source_id` on `create_field`), the server creates a
default matching criterion automatically. Inspect with
`list_matching_criteria`. The default fuzzy self-match against
`displayProperty` works for most cases, but adjust when:

- **Stricter match wanted**: `update_matching_criterion` with
  `match_type="term"` (exact), `required=True`, or
  `required_strict=True`.
- **Cross-field match**: `create_matching_criterion` with a different
  `source_field_id` (e.g. match a "Supplier Code" field against the
  `code` property in a Suppliers data source).
- **Multiple criteria**: combine fuzzy on `label` with exact on
  `code` for higher confidence.
- **Currency / ISO codes**: `match_type="term"` (exact). Fuzzy
  matching on three-letter codes finds spurious matches.

Skipping the criterion check leaves the field decoratively linked to
the data source but not actually looking it up at extraction time.
After every attach, look at `list_matching_criteria`.

## Match types

Each matching criterion sets a `match_type`:

- `match` — fuzzy matching (default). Tolerates typos and minor
  differences.
- `match_phrase` — partial phrase matching. The extracted text must
  appear as a phrase.
- `term` — exact matching. Only exact string matches are returned.

## Scope and lifecycle notes

- Data sources are scoped to an **organisation** — every workspace
  in the org can use them.
- Deleting a data source removes it from any fields that reference
  it.
- The `id` field is the data-source id used in tool operations
  (update, delete, list values, attach). The `identifier` field is a
  human-readable alphabetic identifier shown in the UI.
- A default matching criterion is auto-created when a data source is
  attached to a field.
