# Affinda Concepts

A short reference for the domain objects exposed by this MCP server.
Most tool names and parameters reference these terms directly.

## Organisation

Top-level tenant. A user belongs to one or more organisations; every
other object lives inside exactly one organisation. Listed via
`list_organizations`. The only mutator is `update_organization`
(rename, admin only).

## Workspace

A container for documents that share processing rules. A workspace has
its own settings:

- **OCR mode** — `auto-detect`, `always-full`, `always-partial`, or
  `skip`. Tells the extractor whether and how aggressively to OCR
  uploads.
- **Document splitting** — when enabled, multi-document files (e.g. a
  PDF "loan pack") are split into per-document units before
  classification and extraction.
- **Document classification** — when enabled, uploads with no document
  type are auto-classified into one of the workspace's document types.
- **Model-memory strategy** — how validated documents become training
  examples (`auto`, `manual`, `always`).
- **Validation flags** — `enable_validation`, `auto_validation`.

Workspaces hold zero or more **document types** (assigned via
`assign_document_type_to_workspace`).

## Document Type

A schema describing what to extract from one kind of document — e.g.
"Invoice", "Resume", "Loan Application". A document type owns:

- A list of **fields** (the extraction schema).
- Zero or more **field groups** (grouping headers in the UI).
- Zero or more **validation rules** (LLM-evaluated checks like
  "subtotal + tax must equal total").

Created via `create_document_type`. Once created, it can be assigned to
multiple workspaces.

## Field

A single extracted value. Each field has:

- A **slug** (machine name, e.g. `invoiceNumber`).
- A **label** (human-readable).
- A **type** — text, integer, decimal, date, datetime, boolean, enum,
  table, group, or one of the structured Affinda types (location,
  passport, etc.).
- Optional **transformation_prompt** — natural-language guidance for the
  LLM extractor on how to read this field.
- Optional **mapping_data_source_id** — links the extracted value to a
  reference list (see Data Source).

`create_field` adds one; `bulk_create_fields` adds many in one call.

## Field Group

A heading-style organiser that groups related fields in the UI. Distinct
from a `field_type="group"` field, which is a *parent* field for nested
extraction. Field groups are visual; group-typed fields are structural.

## Validation Rule

An LLM-evaluated rule on a document type. Runs against extracted values
post-extraction. Rules use the `@<field_id>` reference syntax to point
at fields. Created via `create_validation_rule`; their results are
visible per document. `create_validation_run` re-runs all rules
explicitly.

## Data Source

An external reference list (vendors, account codes, products) that
fields can be matched against. A data source has:

- A **schema_definition** — the columns of the reference list.
- **Values** — the rows (created in bulk via
  `bulk_create_data_source_values`).
- **Matching criteria** — how the lookup behaves at extraction time
  (`create_matching_criterion`, `update_matching_criterion`).

A field is connected to a data source via `update_field` with
`mapping_data_source_id`.

## Integration

User-authored Python that runs on every document matching its scope
(workspace and/or document type). Integrations are **versioned**:
editing the code creates a draft; the running version doesn't change
until `deploy_integration_version` is called. `run_integration`
executes the deployed version against a single document for
verification. External APIs are reached via Pipedream **connections**
(`add_connection_to_integration`) when the service exists in the
directory. Custom/private APIs can also be reached with direct HTTP from
the Lambda, with credentials stored as integration secrets and read from
environment variables.

## Document

An uploaded file, classified into a document type, with extracted field
values. A document moves through states: `pending` → `processing` →
`review` → `validated`. `confirm_documents`, `reject_documents`, and
`archive_documents` move it forward; `reassign_document_type` corrects
misclassification (and triggers a reparse by default).

## Model Memory

Validated documents (subject to the workspace's model-memory strategy)
that the extractor uses as in-context examples for future documents of
the same type. Model memory is shared across workspaces — confirming a
document of type X in workspace A makes it eligible as a training
example anywhere else type X is used. Splitting and integrations don't
use model memory; only extraction and classification do.
