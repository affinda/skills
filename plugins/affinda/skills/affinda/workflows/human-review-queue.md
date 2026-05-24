# Workflow: Run a human review queue

Sequence the document state-mutator tools (`confirm_documents`,
`reject_documents`, `archive_documents`, `reassign_document_type`) so
the user can work through extracted documents efficiently.

## When to use this workflow

Trigger phrases:
- "Review my documents"
- "Confirm the invoices I just uploaded"
- "What's in the review queue?"
- "I want to validate extractions"

## Concepts

A document moves through states:

```
pending → processing → review → validated
```

- `pending` / `processing`: server hasn't finished extracting.
  Surface to the user, don't do anything yet.
- `review`: extraction done, waiting for a human. This is the queue
  this workflow operates on.
- `validated`: the user has signed off (`confirm_documents`).

There's also an `archived` state for documents removed from active
work without rejection.

## Plan

### 1. Show the queue

```
list_documents(
    workspace_id=<...>,
    state="review",
    limit=20,
)
```

If the queue is empty, tell the user — there's nothing to review.

### 2. For each document the user wants to inspect

```
get_document(document_id=<...>)             # state, workspace, doc type
get_document_extraction(document_id=<...>)  # extracted field values
```

Surface field values + confidence scores. Highlight low-confidence
fields so the user knows where to look first.

### 3. Apply the user's verdict

| User says | Tool to call |
|---|---|
| "These look right, sign them off" | `confirm_documents(document_ids=[...])` |
| "These are wrong, reject them" | `reject_documents(document_ids=[...])` |
| "I don't need these any more" | `archive_documents(document_ids=[...])` |
| "This is the wrong document type" | `reassign_document_type(document_id=..., document_type_id=...)` |

Always batch — the bulk-document tools accept a list of ids. Don't
loop per document.

### 4. Effect on model memory

`confirm_documents` interacts with the workspace's
`model_memory_strategy`:

- `auto` (default): the system selects which confirmed documents
  become training examples.
- `manual`: only documents explicitly marked become examples.
- `always`: every confirmed document is added.

Tell the user this if they're confirming for the first time —
confirmation isn't just "approve and forget", it's "approve and
teach".

### 5. After review

If the user has been correcting fields, point them at
`debug-low-confidence-results.md` (in this skill) to investigate why
those fields needed correction. Repeated annotations on the same
field are a strong signal the field's prompt or type needs work.

## Document reassignment notes

`reassign_document_type` triggers a **reparse** by default — the
document is re-extracted under the new schema. This is correct for
misclassification but means the document briefly returns to the
`processing` state. Tell the user to expect a short delay.

The target document type must be assigned to the workspace. If it's
not, `assign_document_type_to_workspace` first.

## Common mistakes to avoid

- Calling `confirm_documents` on a single id at a time in a loop.
  Batch them.
- Using `archive_documents` when the user means `reject_documents`.
  Reject is for *bad* extractions; archive is for *no longer
  needed*. Reject participates in model memory ("don't repeat this");
  archive doesn't.
- Calling `reassign_document_type` without checking that the new
  document type is assigned to the workspace. Surface the
  resulting error verbatim if it happens.
