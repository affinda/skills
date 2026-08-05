# Model Memory — reference

How model memory works, why it is the primary lever for extraction
accuracy, and how to manage it well. Pairs with
`reference/workspace-settings.md` (the `model_memory_strategy`
setting), `reference/extraction-review.md` (inspecting a document's
reference document), and
`workflows/debug-low-confidence-results.md` (diagnosing wrong
predictions).

## How it works

Model memory is the mechanism by which Affinda learns from confirmed
documents. When a new document is processed, Affinda looks for the
most similar previously-confirmed document in model memory for that
document type. If a sufficiently similar match is found, the model
leans heavily on that match's annotations when predicting fields on
the new document — in effect, "this looks like that one; extract
things the same way." The matched example is recorded as the new
document's **reference document** (`reference_document_id` on
`get_document_extraction`).

Nothing about the underlying model changes when a document is
confirmed. What changes is the pool of reference examples available
for matching. This is why improvements are fast: one well-annotated
example can lift accuracy on every future document of the same
format, without hundreds of training documents.

Two related but distinct operations:

- **Confirming** a document signals its extracted data is correct
  (part of the review workflow).
- **Adding** a document to model memory designates it as a reference
  for future predictions.

The workspace's `model_memory_strategy` controls how the two are
linked (see "Choosing a strategy" below). A confirmed document is
not necessarily a model memory reference.

Model memory applies to extraction and classification only — never
splitting or integrations. It is shared across workspaces: a
document of type X confirmed in workspace A is eligible as a
reference anywhere type X is used.

## Model memory is the primary accuracy lever

When the goal is "this document type should extract well", model
memory is where most of the improvement comes from. Field names and
descriptions, validation rules, data sources, and OCR settings all
matter, but they are secondary levers that handle edge cases,
formatting, and pipeline issues. Reach for model memory first, both
when improving accuracy and when diagnosing a wrong prediction (see
the debug workflow).

Two consequences of the matching mechanism:

- **Quality beats volume.** A handful of well-annotated,
  format-diverse documents (often 5–10) typically gives strong
  accuracy on new documents that resemble them. Adding more
  documents of an already-covered format does not generally help and
  can hurt — every reference is one more document that must stay
  correct over time.
- **The unit of coverage is the format, not the sender.** One
  invoice from a given supplier usually lifts accuracy on all
  subsequent invoices from that supplier; a different supplier with
  a different layout won't benefit until its format is represented.
  A thousand suppliers is not a thousand references — many share or
  closely resemble formats already covered.

## When to add a document to model memory

Default rule: **add a document when its format produced inaccurate
predictions and a similar document isn't already in model memory.**

- A new format appears and predictions on it are weak → confirm a
  clean version and add it.
- A variation within a known format predicts weakly (the matched
  reference doesn't quite fit) → confirm a clean version of the
  variant and add it.
- A format is already predicting well → don't add more examples of
  it.

Always correct the annotations **before** confirming. A confirmed
document with wrong annotations becomes a reference that propagates
its errors to every future document that matches it. Fewer, cleaner
references beat many mediocre ones.

**Prefer field-rich references.** When choosing which confirmed
document to use as the reference for a format, pick the one that
exercises the most fields. A field left blank on a reference doesn't
just fail to help — it can suppress that field on future documents
that match the reference, even when the value is present on them.
Keep each document's annotations truthful (blank where genuinely
absent), but select the fullest good example as the reference.

**Seeding a new setup:** a useful starting point is one example for
each of the 5–10 most common formats the user expects (e.g. their
most frequent invoice suppliers), plus an example of any format
known to be unusually complex or variable. Beyond that, let
production usage drive additions — formats that predict well need
no attention; formats that don't will reveal themselves through
wrong predictions. Don't bulk-load large batches up front.

## Choosing a strategy (`model_memory_strategy`)

Set per workspace. Governs what happens when a document is
confirmed:

| Strategy | Behaviour | Fits |
|---|---|---|
| `auto` | The system adds a confirmed document only when no sufficiently similar reference already exists. **Default.** | Users who don't want to manage model memory actively. Avoids similar formats crowding the reference pool. |
| `manual` | Confirmed documents are never auto-added; the user adds references deliberately. | Users who will actively curate references. Keeps "data is correct" (confirm) separate from "use this as a reference" (add). |
| `always` | Every confirmed document is added. | Workspaces whose purpose is to feed model memory — e.g. a dedicated training workspace — or small, consistent document sets. |

`auto` is the right default for most users. Recommend `manual` when
someone on the user's side will actively manage the reference set —
it gives the strictest control over what becomes a reference. The
trade-off of `auto` is precision: it may skip a document the user
wanted added, or add one they didn't.

For users with significant format variation, accuracy-critical
workloads, and someone who will actively curate: a dedicated
**training workspace** (on `always` or `manual`) separate from
production workspaces (on `manual` or `auto`) makes "what's in model
memory" answerable by "what's in this workspace". This is a
power-user pattern — for most users it adds complexity they don't
need, and `auto` in a single workspace is fine.

## Keeping model memory healthy

- **Fixing a bad reference requires re-confirming it.** Editing
  annotations on a confirmed document does not update model memory
  by itself; the document must be confirmed again for the corrected
  annotations to take effect. Until then, predictions keep following
  the old, wrong annotations. After re-confirming, reparse the
  affected documents to see the fix.
- **Inspect before adding more.** When predictions on a format are
  wrong, check the reference document that guided them
  (`reference_document_id`) before adding new examples — a bad or
  stale reference is usually the cause, and adding more documents
  won't fix it. The debug workflow covers this diagnosis in detail.
- Use `list_model_memory_documents(document_type_id)` to review the
  full reference set for a document type when curating.
