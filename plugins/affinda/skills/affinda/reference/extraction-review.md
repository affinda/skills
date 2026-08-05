# Extraction Review — operational reference

How to inspect, explain, and sanity-check what was extracted on a
single document or for a single field across recent documents. Pairs
with `workflows/debug-low-confidence-results.md` (structured
diagnosis when accuracy is low) and `workflows/human-review-queue.md`
(confirm / reject / archive flows).

## Two review axes — one thing at a time

You review along two axes; never both at once, and never in bulk:

1. **One document, all fields** — `get_document_extraction(document_id)`
   gives every field's value on one document. Use when the user is
   looking at a specific document.
2. **One field, recent documents** —
   `list_recent_field_annotations(field_id, filter_by, limit)`
   spot-checks how a single field is being extracted across the
   newest documents of its document type. Use when the user is
   iterating on a field's description / transformation / validation
   and wants a quick read on real behaviour.

> Don't loop `get_document_extraction` over many documents, and don't
> loop `list_recent_field_annotations` over many fields. Each focused
> look produces a better, more reliable judgement; sweeping produces
> shallower answers.

For genuine bulk export of extracted data, point the user at the
**Download extraction** option in the workspace's documents view —
the review tools aren't designed for mass pulls.

## How to read a document

Primary tool: `get_document_extraction(document_id)`. It returns:

- `raw_text` — the OCR / text-layer plain text.
- `fields` — a dict keyed by field slug. Value shape depends on the
  field type:
    - **Scalar fields** (text, number, date, enum, etc.) →
      `{"parsed": ..., "raw": ..., "confidence": ...}`. `parsed` is
      canonical, `raw` is the source text in the document,
      `confidence` is 0–1.
    - **Field groups** (section / group fields) → nested dict of
      child slugs, each in the scalar shape above.
    - **Tables** → list of rows; each row is a dict keyed by column
      slug, each cell in the scalar shape.

If you need the document's type or workspace, use
`get_document(document_id)`.

`list_fields` (with the document type id) gives all fields organised
by group in one call — useful for understanding the configuration's
expectations (descriptions, transformations, data sources,
validation rules). Don't call `get_field_group` or `get_field`
individually.

When presenting findings:

- Lead with a concise summary of the core fields and their values.
- Call out any field where `confidence` is noticeably low or where
  `parsed` looks inconsistent with the surrounding `raw` value.
- For groups, show each child field under its parent label.
- For tables, show rows one-by-one identified by column name and
  value. Flag rows whose values look implausible or contain blanks
  that shouldn't be blank.

Reach for `raw_text` only when the user is questioning a specific
extracted value and you need to check whether the source text
actually supports it. Skip it for routine "look right?" confirmation
— prefer the `fields` summary.

## How to spot-check a field across documents

Use `list_recent_field_annotations(field_id, filter_by, limit)` when
the user wants a quick read on how a single field is being extracted
— typically after tweaking its description, transformation, data
source, or validation rules, or when sanity-checking an existing
field's behaviour. Returns up to 100 recent annotations for the
given field, newest first.

`filter_by` narrows the sample:

- `"any"` — any recent document of this field's document type
  (default).
- `"review"` — documents currently in review.
- `"validated"` — confirmed documents.
- `"in_model_memory"` — documents currently in model memory.

Each annotation includes `parsed`, `raw`, `confidence`,
`is_verified`, and basic document metadata (identifier, file name,
state, created_at).

Rules:

- **One field at a time.** Never call this in parallel for multiple
  fields. Review with the user, then move on if asked. Batching
  undermines the point of a spot-check.
- **Keep the sample small by default** (20–30). Only request the
  full 100 when the user explicitly asks for a broader look.
- **Not a bulk-export tool.** For extracted data across many fields
  or documents, direct the user to **Download extraction**.

When presenting results, lead with the parsed values one per line
with the source document's file name or identifier for reference.
Call out low-confidence values, parsed/raw mismatches, and blanks
that shouldn't be blank — that's what the user is actually asking
about. If the user is iterating on a field's description and the
parsed values don't reflect their recent changes, remind them that
field changes don't apply to previously-parsed documents until those
documents are reparsed.

## Model memory and the reference document

When a document was extracted with the help of a model-memory example,
`get_document_extraction` returns that example's ID as
`reference_document_id` (null when no example was used). This is the
key to answering "why did it extract X as Y?":

1. Fetch the document's extraction and note `reference_document_id`.
2. If set, call `get_document_extraction` with that ID — the reference
   document is a confirmed document, so its values are ground truth.
3. Compare the suspect field on both documents. A surprising value
   often mirrors how the equivalent field was verified on the
   reference document (e.g. a shortened vendor name, a particular
   date format, a column mapped differently than expected).

When you find such a mirror, explain it in plain language: the
extractor followed the confirmed example. If the example itself is
the problem — its verified values don't fit the newer documents —
suggest the user review that reference document, or curate model
memory for the document type.

Two non-obvious mechanics worth surfacing when the reference is the
problem:

- **Fixing a reference requires re-confirming it.** Editing
  annotations on a confirmed document does not update model memory
  by itself — the document must be confirmed again, and affected
  documents reparsed, before predictions change. A user who says
  "I already fixed the reference but it's still wrong" almost
  always skipped the re-confirm.
- **A blank field on a reference can suppress that field.** If the
  reference document has a field left blank, documents matching it
  may stop predicting that field even when the value is present.
  When a format has several confirmed candidates, the best reference
  is the one that exercises the most fields.

See `reference/model-memory.md` for the full picture of how model
memory works and how to curate it.

Use `list_model_memory_documents(document_type_id)` only when the
user asks about model memory as a whole — the full set of confirmed
reference documents the document type uses to help extract new
documents of the same shape. For one specific document, follow
`reference_document_id` instead; don't list the whole memory.

You don't need to fetch model memory or the reference document for a
routine review — reach for them when a value needs explaining.
