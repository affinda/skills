# Document Types and Fields — operational reference

Operational rules for creating and maintaining document types,
fields, field groups, and validation rules. Pairs with
`reference/concepts.md` (which defines the objects) and
`reference/field-schemas.md` (which has worked-example schemas plus
the field-types reference + slug conventions + parent-field rules).

## Document type settings

| Setting | Description | Default |
|---|---|---|
| `name` | Display name (e.g. "Invoice", "Resume") | required |
| `disable_confirmation_if_validation_rules_fail` | Block confirmation if validation fails | `false` |
| `show_redact_button` | Show redaction button in UI | `false` |
| `auto_refresh_validation_results` | Auto-refresh validation when document changes | `false` |
| `date_format_preference` | DMY / MDY / YMD — input parsing hint, see below | unset |

## Field settings — Core

| Setting | Description | Default |
|---|---|---|
| `slug` | API identifier — camelCase, alphanumeric only, lowercase first letter (see `reference/field-schemas.md` for the convention). | required |
| `label` | Human-readable name shown in UI | required |
| `field_type` | Data type (see `reference/field-schemas.md` for the full reference table) | required |
| `description` | Extraction prompt for the model. Optional — see "Writing field descriptions" below before adding one. | none |
| `enabled` | Whether the field is active | `true` |

## Field settings — Extraction behaviour

| Setting | Description | Default |
|---|---|---|
| `multiple` | Field can have multiple **distinct** values in the document (e.g. several work-experience entries on a resume, multiple line items, multiple signatories). **Not** for the same single value appearing multiple times — that's still `multiple=false`. Tables and table rows almost always have `multiple=true`. | `false` |
| `no_rect` | Field doesn't require a location rectangle (calculated / manual fields); use this for generating document summary text fields | `false` |
| `return_first_instance_only` | Return only the first value when `multiple=true` | `false` |
| `manual_entry` | Field requires manual input, not extracted | `false` |

## Field settings — Display & formatting

| Setting | Description | Default |
|---|---|---|
| `display_enum_value` | Show enum value instead of label | `false` |
| `hide_enum_detail` | Hide additional enum information | `false` |
| `drop_null` | Omit field from results if null | `false` |
| `display_raw_text` | Show the original extracted text | `false` |

## Field settings — Organisation

| Setting | Description |
|---|---|
| `parent_id` | Parent field id for nested / hierarchical fields. Set at create time only — **never** reassign on an existing field, and `update_field` has no `parent_id` parameter for this reason. To restructure nesting, delete and recreate. |
| `field_group_id` | Field group (heading) for visual organisation in the UI. Set on create or change via update. |
| `position` | Display order within a field group (0-based). Set via `update_field`. |

## Default behaviour when creating fields

Create fields immediately with sensible defaults. Don't ask follow-up
questions unless information is genuinely ambiguous.

| Setting | Default | Deviate when |
|---|---|---|
| `field_type` | infer from name (see below); fall back to `text` | Only if genuinely ambiguous |
| `enabled` | `true` | Never ask |
| `multiple` | `false` | User says "multiple", "list", "table", "repeating" |
| `no_rect` | `false` | Never ask |
| `manual_entry` | `false` | User says "manual entry" or "not extracted" |
| `description` | omit | Only if user provides extraction guidance, or name is ambiguous |

### Field-type inference from the field name

| Name contains | Inferred type |
|---|---|
| "date", "time", "expiry", "due", "issued" | `date` |
| "amount", "total", "price", "cost", "quantity", "rate", "percentage", "subtotal", "tax" | `float` |
| "phone", "mobile", "fax" | `phonenumber` |
| "url", "website", "link" | `url` |
| "address", "location" | `location` |
| Starts with "is_" / "has_", or yes/no checkbox | `checkedboxboolean` |
| "section", "group", "line items", or will have children | `group` |
| "table" with rows / columns | `table` |
| Everything else (including "email") | `text` |

## Writing field descriptions

Field names and descriptions are how a field is communicated to the
extraction model — what to look for, where, and what to avoid. They
are the **secondary** lever for accuracy, after model memory: when a
prediction is wrong, check the model memory reference before
reaching for a description (see
`workflows/debug-low-confidence-results.md`).

**The field name is the first signal — tighten it before adding a
description.** The model reads the name as its primary hint. A vague
name like "Date" or "Amount" forces a guess between candidates when
a document contains several; "Invoice due date" or "Total amount
payable" usually resolves the ambiguity on its own, with no
description needed.

**Descriptions are reactive, not preemptive.** Write one to address
a specific observed failure, not as default field configuration. If
you can write the description without having seen the document or a
concrete failure, it's generic knowledge the model already has
("the date of the transaction" on a field called "Transaction date")
and it won't help. Good uses:

- Disambiguating between plausible candidates the name can't
  separate: *"The name of the borrower as shown in the application
  section, not the name of the broker or the lender."*
- Anti-examples — what NOT to extract: *"The date the invoice is
  due. Do not include the timestamp."*
- Naming layout context the model may lose in conversion: *"the
  column immediately to the right of the description column, even
  when the header is missing."*
- Long-tail rules impractical to teach through a model memory
  example.

**Keep it tight.** One sentence addressing one failure mode beats a
paragraph. Long descriptions cost tokens, dilute the signal, and
accumulate contradictions over time.

**Enum / options fields** are an exception to "reactive only": when
option labels are short, domain-specific, or ambiguous, the
description should say what each value means and when to pick it —
otherwise the model is guessing what the labels represent.
Self-explanatory labels ("Yes" / "No") need nothing.

**Group parents** are the other exception. Group structure can be
abstract to the model, so a description on the parent explaining
what the group represents (and what makes one instance distinct
from another) helps. For groups with `multiple=true`, the model
tends to over-predict instances when there's no model memory
example to anchor against — cardinality limits ("expect no more
than three") and uniqueness constraints ("do not return duplicates")
are worth adding up front.

**What descriptions don't do.** They guide prediction; they don't
transform output, enforce constraints, or override model memory:

- Reformatting ("return as YYYY-MM-DD") → use a text transformation
  (`transformation_prompt`), not the description.
- Constraints (must match a list, fall in a range) → use validation
  rules or data sources.
- They compose with the model memory reference rather than
  overriding it — a description can't fix a bad reference.

## Field groups (UI headings)

Field groups are visual sections; every top-level field belongs to
one. They're distinct from `field_type="group"` (a *parent* field for
nested extraction) — don't conflate the two.

- **Move a field into a different group**: `update_field` with
  `field_group_id`.
- **Reorder fields within a group**: `update_field` with `position`
  (0-based). Call `list_fields` first to see current positions.
- **Delete an empty group**: `delete_field_group` (move fields out
  first via `update_field` if needed).

`list_fields` returns fields organised by group already — there's no
need to call `get_field_group` or `get_field` individually.

## Tables that span multiple pages

A single table annotation cannot cover multiple pages. When a user
asks "the table continues on page 2", the answer is to set
`multiple=true` on the table field. Each page's portion becomes its
own annotation. The same pattern applies to any field that may
repeat across pages.

Never tell the user "a table can span multiple pages" — that's wrong;
guide them to `multiple=true`.

## Tables vs `group` parents — when to use which

For structured / hierarchical data, create a parent `group` field
with `multiple=true`, then create child fields with `parent_id` set
to the parent's id:

```
lineItems (group, multiple=true)
  ├─ description (text)
  ├─ quantity (float)
  ├─ unitPrice (float)
  └─ amount (float)
```

### When to use a `group` parent field

- When there are **multiple instances of a set of fields** that
  should be grouped together (e.g. work-experience entries on a
  resume, signatory parties on a contract, addresses on a form).
  The parent has `multiple=true`; each child has `multiple=false`
  (since each child only occurs once *per group instance*).
- Example: to extract several names with first / last parts, create
  a `name` group with `multiple=true`, and two children `firstName`
  and `lastName` with `multiple=false`.

### When NOT to use a `group` parent

- **Don't create a group with `multiple=false`.** If there's only
  one instance, just use orphaned fields under a field group
  (heading). Wrapping in a `multiple=false` parent adds no value.
- **Don't use `group` when a `table` is more appropriate.** Grid-
  shaped data with column headings → `table`.
- It is a common mistake to over-use parent fields. Prefer the
  simpler option (orphaned fields or a table) unless the
  parent-with-children structure is genuinely needed.

### When to use a `table` field

- The data is laid out as a **grid with column headings and rows**
  (line items, transactions, schedule of charges, list of products).
- **Structured financial information** — line items with currency
  amounts, transactions, invoice rows, payment schedules — is
  almost always a good `table` candidate.
- The table itself must have `multiple=true` so it can occur more
  than once per document (e.g. continuing onto page 2).
- When the layout is grid-shaped, **prioritise `table` over `group`**.
  Tables are the right primitive for repeating-row data; reserve
  `group` for genuinely hierarchical (non-tabular) structure.

**Creating a `table` is a two-step flow, not a single bulk call.**
Columns are NOT direct children of the table: the backend
auto-creates a `rows` group field (slug `rows`, `field_type=group`,
`multiple=true`) between the table and its columns. Columns are
children of `rows`. See **Creating tables — call sequence** in
`reference/field-schemas.md` for the exact `bulk_create_fields` →
`list_fields` → `create_field` sequence. Setting `parent_id` to the
table id on a column is a hard error — extraction breaks.

### When NOT to use a `table` field

- If the example document shows a grid layout, but **other documents
  of the same type might present the same information in a non-grid
  layout** (e.g. plain prose, separate sections), use a `group`
  parent field instead. Tables are stricter and rely on a row /
  column structure being present.
- For one-off structured data with a fixed shape (no rows), use
  orphaned fields or a `group`.

### Never retype into or out of `group` / `table`

`update_field` will refuse `field_type="group"` or `field_type="table"`,
and will also refuse to change a field that is *currently* a `group` or
`table` to any other type. The structural relationships those types
carry (child fields, row/column annotations) cannot be migrated by a
retype — flipping the type would orphan children and corrupt existing
annotations. If a structural change is needed, `delete_field` the
existing field and `create_field` a new one with the desired type and
`parent_id`. For sweeping restructures, point the user at the manual
document-type editor in the app.

## Validation rules

Rules check extracted data for correctness — cross-field, format,
range, required, business logic.

**Before creating a validation rule, always call `list_validation_rules`
first.** Document types often have existing rules (including disabled
ones) that match the intended behavior. If a matching rule exists,
use `update_validation_rule` to enable it (and adjust settings if
needed) rather than creating a duplicate. Only create a new rule if
no existing rule covers the requirement.

IMPORTANT: Even if the user says something like "create a new rule",
list existing rules and assess whether any of those ones can be
utilized.

IMPORTANT: When a user explicitly asks to **update** or **modify** an
existing rule (e.g., "update the X rule to also check Y"), always
use `update_validation_rule` to modify that rule's prompt. Combine
the existing prompt with the new requirement into a single updated
prompt. Do NOT create a separate new rule when the user has asked to
update an existing one.

When creating **new** rules (not updating existing ones), try to
split separate logical checks (e.g., required and of a certain
length) into two different validation rules, even if they use the
same field.

| Setting | Description | Default |
|---|---|---|
| `prompt` | Validation prompt using `@<field_id>` references (see below) | required |
| `enabled` | Whether the rule is active | `true` |
| `field_ids` | Fields this rule validates. Pass as a native list, not a JSON string. If you get type errors, omit it and reference fields by label in the prompt instead. | none |
| `missing_data_option` | `PASS`, `FAIL`, `SKIP`, or `IGNORE` (UPPERCASE). Use `FAIL` for required-field rules so missing data is flagged as a failure, not silently skipped. | none |
| `generate_code` | Set `true` to auto-generate Python validation code from the prompt | `false` |

### Field references in rule prompts: `@<field_id>` syntax

Always reference fields in a rule's `prompt` as `@<numeric_field_id>`,
never as plain text. The UI renders `@<id>` as a styled chip; plain
text shows as raw characters.

- ✅ `"@4821 must be present and @4825 must be greater than zero"`
- ❌ `"Invoice Number must be present and Total Amount must be greater than zero"`

Call `list_fields` to get ids before composing the prompt.

#### Cells inside tables / groups: reference the child IDs, never the parent

When the data being validated lives **inside a `table` or `group` parent
field**, reference the child (cell / column) IDs in the prompt — never the
parent ID. The parent is the row / container; only the children hold the
actual cell values. A rule written against a parent ID checks the container,
not the data, and almost always produces nonsensical results.

For a table like:

```
lineItems        (table,  id=4820)  ← do NOT reference in rule prompts
  ├─ description (text,   id=4821)
  ├─ quantity    (float,  id=4822)
  ├─ unitPrice   (float,  id=4823)
  └─ amount      (float,  id=4824)
```

- ✅ `"@4822 must be greater than 0"` — validates each row's `quantity` cell
- ✅ `"@4823 * @4822 must equal @4824"` — per-row arithmetic across cells
- ✅ `field_ids=[4822, 4823, 4824]` — list only the cell IDs the rule touches
- ❌ `"@4820 must not be empty"` — references the table container, not data
- ❌ `field_ids=[4820]` — passing the parent ID instead of the relevant cells

The same rule applies to `group` parents: reference the children
(`firstName`, `lastName`), not the parent (`name`).

#### How to tell parent from child in `list_fields` output

A field is a **parent** (table / group / table_deprecated) when:

- its `field_type` is `table`, `table_deprecated`, or `group`, AND
- its `children` list is non-empty.

A field is a **child** (cell / column / nested field) when its `parent_id`
is set. Always prefer these child IDs in validation rule prompts and in
`field_ids` whenever the rule concerns data found inside a row / group
instance.

If the user describes the check in row-level language ("each line item must
have a quantity", "every row's total must be positive"), translate that into
references to the relevant **cells**, not the parent table.

## Date parsing hints (DMY / MDY / YMD)

Controlled at two levels:

- **Document type:** `date_format_preference`.
- **Date / datetime / daterange fields:** `dateOrder` inside
  `formatterConfig`.

**Critical:** this is an INPUT parsing hint for ambiguous strings
like `01/02/03` — **not** an output format. The extracted value is
always returned as a canonical ISO date regardless of this setting.

Users frequently confuse this with output formatting. When proposing
a change, be explicit about all three of:

1. **What it affects:** how the model *reads* ambiguous dates off
   the page (DMY vs MDY vs YMD).
2. **What it does NOT affect:** how the extracted date is returned,
   displayed, or stored — that is always canonical.
3. **How to choose:** based on where the documents come from (US →
   MDY, UK/AU/EU → DMY, ISO strings → YMD).

If a user responds with a preference like "I prefer DMY" to a
proposed MDY change, they may be thinking about output formatting.
Clarify that this setting controls interpretation, not presentation,
before applying.

### Example framing

> "These receipts are US-formatted, so `01/02/2025` is being read as
> 1 February instead of 2 January. Setting the Invoice Date field's
> date parsing hint to **MDY** tells the model to treat ambiguous
> dates as month-first. This only changes how the model interprets
> the page — the extracted value is still returned as a standard ISO
> date, so downstream output and display are unaffected."

## Workflow: after creating a document type

1. Create the document type with `create_document_type`.
2. **Immediately** call `list_fields` to check for pre-populated
   template fields.
3. **If fields already exist:** present them to the user. Do not
   create duplicates.
4. **If no fields exist and documents are available:** use the
   field suggester (see "Auto-populating fields" below).
5. **If no fields and no documents:** create fields manually based
   on the user's description.
6. Review the fields with the user; adjust as asked.
7. Optionally add validation rules.

## Auto-populating fields with `populate_document_type_fields`

When a new document type has no pre-populated template fields **and**
the user has uploaded documents, use `populate_document_type_fields`.
This is **strongly preferred** over creating fields manually — it
analyses actual document content, detects appropriate types and
structures (including tables and nested groups), and is much faster.

Step-by-step:

1. **Upload documents** with `upload_document_to_workspace`. If you
   know the document type, pass `document_type_id` to bypass
   classification; otherwise let the workspace classify.
2. **Wait for processing** with `wait_for_document_processing` —
   not manual polling of `list_documents` (which wastes turns and
   risks timeouts). Returns the final list including any children
   created by splitting.
3. **Find matching documents** by filtering the returned list to
   those whose `document_type_id` matches the target type
   (including split children).
4. **Call `populate_document_type_fields`** with the document type id
   and ALL matching document ids. Pass the user's description (if
   any) as `instructions` — keep it ≤15 words.
5. **Show results** with `list_fields` and present them.
6. **Refine** as the user requests.

### Precedence: descriptive context isn't a manual-create request

If a document is uploaded to the workspace AND the document type is
new (no fields yet), use `populate_document_type_fields` — **even if
the user has described what's in the document**. A user mentioning
contents (e.g. "it has an email and some text") is context, not a
field list. Pass that description as `instructions`; do not translate
it into a manual `bulk_create_fields` call.

Auto-populate operates on the source document via the ML pipeline;
it works even when the document's current extraction is empty (which
is normal for a doc not yet assigned to any document type). Don't
gate auto-populate on the document's extraction state.

| Scenario | Approach |
|---|---|
| New doc type, no template fields, user uploaded docs | `populate_document_type_fields` (always — see precedence) |
| New doc type that matched a template (fields pre-populated) | Review existing fields; adjust |
| User explicitly asks for a single specific field by name | `create_field` |
| No documents uploaded, user describes specific fields | `bulk_create_fields` (2+) or `create_field` (1) |
| Adding fields to an existing doc type that already has fields | `bulk_create_fields` / `create_field` |

For **multiple document types**: repeat the populate workflow per
type, using only documents classified as that type each time.

## Bulk vs single field creation

Whenever you're adding multiple fields, use `bulk_create_fields`, not
`create_field` in a loop. Single calls trigger separate confirmations
in the UI; bulk shows one. Maximum 50 fields per call.

Use `bulk_create_fields` when:

- The doc type already has fields (auto-populate isn't appropriate).
- No documents are available and the user has named the fields they
  want.
- You're adding extras after an initial auto-populate.

Do **not** use `bulk_create_fields` when a document is uploaded and
the document type is new with no fields — that case demands
`populate_document_type_fields` regardless of any descriptive context.

### Argument-shape pitfalls

- `fields` must be a **native list of dicts**, NOT a JSON-encoded
  string. Pass `[{...}, {...}]`, never `"[{...}, {...}]"`.
- `tool_display` must be a **native dict**, NOT a JSON string. Pass
  `{"base": "...", ...}`.
- `options` (inside a field) must be a **native list of strings**.

If the tool returns `Input should be a valid list ... input_type=str`,
you serialised the argument as a string. **Retry the same call with
the argument as a native list / dict.** Don't fall back to repeated
`create_field` calls — a single bulk request is the request the user
asked for, and splitting it produces many separate confirmations
instead of one.

## Text transformations

Use only when the user **explicitly** asks to reformat extracted text
(e.g. "uppercase", "DD/MM/YYYY", "title case"). Don't add proactively.
Set via `transformation_prompt` on create or update; the system
generates the code automatically.

Don't use transformations for normal extraction guidance (use
`description`), type changes (use `field_type`), or validation (use
rules).

## Re-parsing after schema changes

Field changes (new / edited / deleted fields, edited descriptions /
transformations / data sources / validation rules) do **not** apply
retroactively. Tell the user to reparse documents of that type if
they want the new behaviour reflected in existing extractions.
