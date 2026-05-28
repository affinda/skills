# Field Schema Examples

Starting points for new document types. Adapt to the user's actual
document format before applying.

These are the same examples the MCP server hosts at
`affinda://schemas/{invoice,resume,contract}` — kept in sync so the
skill is usable even when the agent's MCP client doesn't surface
resources.

**The nested `fields: [...]` inside a `table` below is a schema diagram,
not a payload.** `bulk_create_fields` takes a flat list and has no
`parent_id` per item, so you cannot create columns under a table in
one bulk call. See **Creating tables — call sequence** below for the
two-step flow (`bulk_create_fields` for top-level + table, then
`create_field` per column under the table's auto-created `rows`
field).

---

## Invoice

```json
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
      {"slug": "amount", "label": "Amount", "field_type": "float"}
    ]
  }
]
```

**Variants**

- Tax-inclusive jurisdictions: drop `taxAmount` and `subtotal`,
  keep `totalAmount`.
- Multi-currency: connect `currency` to a data source of allowed ISO
  codes.
- PO-driven workflows: add `purchaseOrderNumber` (text) before
  `invoiceNumber`.

**Recommended validation rules** (post-create):

1. `subtotal + taxAmount = totalAmount` (within rounding).
2. `sum(lineItems[*].amount) = subtotal`.

---

## Resume

For most resume / CV use cases, prefer `create_recruit_workspace`,
which spins up a pre-built schema in one call. Use this only if the
user explicitly wants a custom resume schema or needs to extend the
default.

```json
[
  {"slug": "candidateName", "label": "Name", "field_type": "text"},
  {"slug": "email", "label": "Email", "field_type": "text"},
  {"slug": "phone", "label": "Phone", "field_type": "text"},
  {"slug": "location", "label": "Location", "field_type": "text"},
  {"slug": "summary", "label": "Summary", "field_type": "text"},
  {
    "slug": "workExperience",
    "label": "Work Experience",
    "field_type": "table",
    "fields": [
      {"slug": "title", "label": "Title", "field_type": "text"},
      {"slug": "company", "label": "Company", "field_type": "text"},
      {"slug": "startDate", "label": "Start", "field_type": "date"},
      {"slug": "endDate", "label": "End", "field_type": "date"},
      {"slug": "description", "label": "Description", "field_type": "text"}
    ]
  },
  {
    "slug": "education",
    "label": "Education",
    "field_type": "table",
    "fields": [
      {"slug": "institution", "label": "Institution", "field_type": "text"},
      {"slug": "degree", "label": "Degree", "field_type": "text"},
      {"slug": "graduationDate", "label": "Graduation", "field_type": "date"}
    ]
  },
  {"slug": "skills", "label": "Skills", "field_type": "text", "multiple": true}
]
```

**Common extensions**

- Internal tracking ID: add `candidateId` (text).
- Compliance flags: booleans for `rightToWork`, `referencesProvided`.
- Salary history: only where the user's jurisdiction permits asking;
  add as a table mirroring `workExperience`.

---

## Contract

Generic contract starting point. Real-world contracts vary widely
(SaaS, employment, NDA, real estate); treat this as a baseline.

```json
[
  {"slug": "contractTitle", "label": "Title", "field_type": "text"},
  {"slug": "effectiveDate", "label": "Effective Date", "field_type": "date"},
  {"slug": "expirationDate", "label": "Expiration Date", "field_type": "date"},
  {"slug": "termLengthMonths", "label": "Term (months)", "field_type": "integer"},
  {
    "slug": "parties",
    "label": "Parties",
    "field_type": "table",
    "fields": [
      {"slug": "name", "label": "Name", "field_type": "text"},
      {"slug": "role", "label": "Role", "field_type": "text"},
      {"slug": "address", "label": "Address", "field_type": "text"}
    ]
  },
  {"slug": "governingLaw", "label": "Governing Law", "field_type": "text"},
  {"slug": "totalValue", "label": "Total Value", "field_type": "float"},
  {"slug": "currency", "label": "Currency", "field_type": "text"},
  {"slug": "autoRenewal", "label": "Auto Renewal", "field_type": "boolean"},
  {
    "slug": "keyClauses",
    "label": "Key Clauses",
    "field_type": "text",
    "multiple": true,
    "transformation_prompt": "Extract titles of clauses such as Termination, Confidentiality, Indemnification, Assignment, etc."
  }
]
```

**Variants**

- NDAs / non-competes: drop `totalValue`, `currency`,
  `autoRenewal`. Add `restrictedPeriodMonths` (integer) and
  `geographicScope` (text).
- SaaS subscription contracts: add `pricingModel` (enum: per_seat /
  flat / usage), `seatCount` (integer), `paymentFrequency` (enum:
  monthly / annual).
- Employment contracts: add `startDate`, `positionTitle`, `salary`,
  `noticePeriodDays`.

---

## Creating tables — call sequence

**Critical:** Getting the table hierarchy wrong corrupts every annotation
on the table. The agent has historically made the columns direct
children of the `table` field; this is wrong and the documents will
not extract correctly. Follow the sequence below exactly.

### The actual hierarchy

A `table` field is **not** a direct parent of the columns. The real
structure is three levels deep:

```
<table field>          field_type=table,  multiple=true
  └─ rows              field_type=group,  multiple=true   ← auto-created by the backend
       ├─ <column 1>   field_type=text/float/date/...     ← parent_id = <rows id>
       ├─ <column 2>   ...
       └─ <column N>   ...
```

- The table itself MUST be `multiple=true` so each instance is annotated
  independently (a single annotation cannot span pages or multiple
  tables on one page).
- The `rows` field is **created automatically by the backend** when you
  create the table. **Do not create it yourself** — there is no need,
  and a manual attempt will fail or duplicate the auto-created one.
- Columns are children of the `rows` field, **not** of the table.
  Setting `parent_id` to the table id will produce broken extraction.

### The two-step call sequence

`bulk_create_fields` has no `parent_id` per item — every field in a
bulk call lands flat under one `field_group_id`. So tables cannot be
created in a single call. The sequence is:

1. **Create the top-level fields + the table** with `bulk_create_fields`.
   Include the table among the flat list, with `field_type="table"` and
   `multiple=true`. Do not try to nest columns inside it.
2. **Look up the auto-created `rows` field id** with `list_fields` (or
   `get_field` on the table). The rows child has `slug="rows"`,
   `field_type="group"`, `multiple=true`, and `parent_id` set to the
   table.
3. **Create each column with `create_field`** (singular, one call per
   column), passing `parent_id=<rows id>`. There is no bulk path for
   this step — `bulk_create_fields` cannot set `parent_id`, and you must
   not try to work around it by setting it on the table.

### Worked example — invoice line items

Given the invoice schema diagram above, the actual calls are:

```
# 1. Top-level + the table itself (one bulk call)
bulk_create_fields(
  document_type_id=<dt>,
  fields=[
    {"slug": "invoiceNumber", "label": "Invoice Number", "field_type": "text"},
    {"slug": "invoiceDate",   "label": "Invoice Date",   "field_type": "date"},
    ...
    {"slug": "lineItems",     "label": "Line Items",     "field_type": "table", "multiple": true},
  ],
)

# 2. Find the auto-created rows child of lineItems
list_fields(document_type_id=<dt>)
# → lineItems has a child with slug="rows", field_type="group", multiple=true.
#   Capture its id.

# 3. Create each column under rows (one create_field call each)
create_field(document_type_id=<dt>, parent_id=<rows id>, slug="description", label="Description", field_type="text")
create_field(document_type_id=<dt>, parent_id=<rows id>, slug="quantity",    label="Quantity",    field_type="float")
create_field(document_type_id=<dt>, parent_id=<rows id>, slug="unitPrice",   label="Unit Price",  field_type="float")
create_field(document_type_id=<dt>, parent_id=<rows id>, slug="amount",      label="Amount",      field_type="float")
```

### Common mistakes (don't do these)

- **Setting `parent_id` to the table id on the columns.** Columns must
  parent to `rows`, not to the table.
- **Manually creating a `rows` field.** The backend creates it. If you
  don't see one after creating the table, re-`list_fields` — it's there.
- **Setting `multiple=true` on the columns.** Each column's value is one
  cell per row; the repetition is provided by `rows` being multiple.
  Only set `multiple=true` on a column if a single cell can itself hold
  a list.
- **Forgetting `multiple=true` on the table.** Without it, the table
  cannot span pages or have more than one instance per document.

---

## Field types reference

| `field_type` | Purpose |
|---|---|
| `text` | Free-form strings (names, descriptions, addresses). |
| `integer` | Whole numbers. |
| `float` | Numbers with fractional parts (amounts, quantities, percentages, prices). Default for any numeric extraction. Pin precision via `formatter_config={"formatter": {"slug": "number", "decimal_places": <0|2>}}`. |
| `decimal` | High-precision decimal values where exact representation matters (e.g. tax-engine inputs that round-trip through fixed-point arithmetic). For ordinary money / quantity fields, prefer `float`. |
| `date` | Calendar date. |
| `datetime` | Date + time. |
| `daterange` | Range between two dates (e.g. employment period). |
| `boolean` | Yes/no, true/false. |
| `enum` | Extracted value mapped against a data source — for large option sets. |
| `options` | Model picks from a short pre-defined list — for status / category. |
| `checkedboxtext` | Multi-select checkbox group with text labels. |
| `checkedboxboolean` | Single yes/no checkbox. |
| `phonenumber` | Phone number with country code. |
| `url` | Website URL. |
| `location` | Structured address (street, city, state, zip). |
| `signature` / `headshot` / `seal` | Image-region detection. |
| `table` | Rows and columns (requires bounding rectangle). **Restricted to a single page.** |
| `group` | Parent field for nested children. **Required** for any field with children. |

Plus Affinda's structured types: `passport`, `driver_license`, etc. —
opt-in domain-specific schemas with their own sub-fields.

## Slug conventions

- **camelCase**, alphanumeric only — no underscores, hyphens, or
  spaces. First letter lowercase. Examples: `invoiceNumber`,
  `totalAmount`, `lineItems`. Not `invoice_number`,
  `Invoice Number`, or `invoice-number`.
- Slugs are stable — they become part of API contracts. Don't rename
  casually after extraction starts; the integration code reads them
  by key (`doc.get("invoiceNumber")`).

## Notes on `multiple: true`

For non-table types, `multiple: true` lets the field hold a list of
values (e.g. multiple skills). Use it when the document naturally
contains a list and the items don't have associated structure. If
items have structure (e.g. work-experience rows have title, dates,
description), use `table` instead.

For tables and any field that may repeat across pages, set
`multiple: true` so each instance is annotated independently — a
single annotation cannot span pages.

## Notes on `rectangle: false`

If a user wants an LLM-generated text field (e.g., a summary of a resume),
this can be accomplished by creating a text field with no rectangle. This means
whatever text is returned is completely generated by an LLM and there is no
grounding of this text to a location on the page. Generally, these fields
are not really desired by customers. Most fields should have a rectangle, as this
prevents hallucinations.

## Parent-field rules

- **Critical:** Parent fields MUST use `field_type: "group"` or
  `field_type: "table"`. Any other type will cause child fields to
  break.
- **Tables have an extra level.** Columns are not direct children of
  the table — they are children of an auto-created `rows` group field
  (slug `rows`, `field_type=group`, `multiple=true`) that sits between
  the table and its columns. See **Creating tables — call sequence**
  above for the exact flow.
- Tables themselves must have `multiple: true` so each occurrence is
  annotated independently (single annotations cannot span pages).
- Bias toward `table` over a `group` parent when the data lays out as
  a grid with column headings — `table` is stricter and handles the
  common case better. Use `group` only when other documents of the
  same type might present the data in a non-grid layout.
- Never reassign an existing field to a different `parent_id` — this
  breaks all existing annotations on that field. Delete and recreate
  under the desired parent.

## Currency fields — don't add manually

If the document type contains currency amounts, the system attaches a
currency-code field automatically. **Never create your own**
`currencyCode` / `currency` field on top — you'll get duplicates.

## Number / float precision guidance

| Purpose | `decimal_places` | Examples |
|---|---|---|
| Currency / financial amount | 2 | `totalAmount`, `subtotal`, `unitPrice`, `taxAmount` |
| Whole-number quantity | 0 | `quantity`, `numberOfItems`, `units`, `headcount` |
| Variable / measurement | omit | `weight`, `percentage`, `temperature`, `rate` |

Set via `formatter_config={"formatter": {"slug": "number",
"decimal_places": <n>}}` on `bulk_create_fields` /
`create_field` / `update_field`.
