# Workflow: Debug low-confidence extraction

The user is unhappy with an extractor's accuracy. This workflow walks
through a structured diagnosis instead of guessing.

## When to use this workflow

Trigger phrases:
- "Why are extractions wrong?"
- "Confidence scores are low"
- "Field X keeps coming out wrong"
- "Can we improve accuracy?"

## Plan

### 1. Identify the document type

If the user hasn't named one, list candidates:

```
list_document_types(organization_id=<...>)
```

Confirm which one is causing trouble.

### 2. Read the configuration

```
get_document_type_details(document_type_id=<...>)
```

Note the field list, the `transformation_prompt` on each field, and
which validation rules are attached.

### 3. Read the workspace's processing settings

```
get_workspace_details(workspace_id=<...>)
```

Three settings frequently explain low confidence:

- **`ocr_mode`** — the most common cause. Wrong mode for the document
  mix.
  - Lots of scans → must be `always-full`.
  - Mixed digital + scan → `always-partial` is the safe default.
  - `skip` for anything that isn't a native text PDF will produce
    empty extractions, not just low confidence.
- **`enable_document_classification`** — if `False` and the user has
  multiple document types, mismatched uploads silently route to the
  wrong type and the "extraction" they see is the wrong schema being
  applied to the wrong document.
- **`model_memory_strategy`** — `manual` with no marked examples
  means the model has nothing to learn from. `auto` is the right
  default for most users.

### 4. Sample real extractions

```
list_documents(workspace_id=<...>, limit=20)
get_document_extraction(document_id=<one_of_the_above>)
```

Pull 3–5 recent confirmed documents. Look at per-field confidence.
Pattern matters:

- **One field consistently low** → that field's `transformation_prompt`
  needs work, or its type is wrong (e.g. text where date is expected).
- **All fields drop on the same documents** → those documents likely
  trigger an OCR-mode mismatch. Check whether they're scans vs digital.
- **All fields low across all documents** → upstream issue. Check OCR
  mode first, then classification, then field schema.

### 5. Inspect annotations

```
list_recent_field_annotations(workspace_id=<...>)
```

If users are manually correcting the same field repeatedly, that
field's prompt or type is wrong — the data is telling you what to fix.

### 6. Propose at most three changes

Order by expected impact:

1. **OCR-mode change** — `update_workspace(workspace_id=..., ocr_mode=...)`.
   Largest single lever.
2. **Field-level fix** — `update_field` to refine the
   `transformation_prompt`, or `delete_field` + `create_field` if the
   type is wrong.
3. **Validation rule** — `create_validation_rule` to flag bad
   extractions for review rather than relying on raw confidence.

### 7. Wait for confirmation before applying

Present the recommendations with the supporting evidence ("8 of 10
sampled documents had low confidence on `vendorAddress`"). Don't
silently call `update_*` or `delete_*` tools — the user owns the
schema.

## What this workflow doesn't fix

- **Genuinely ambiguous documents.** Some inputs really are unreadable
  or contradictory. Confidence is supposed to be low.
- **Sparse training signal.** A brand-new document type with three
  uploads will improve over the next ~10 confirmations regardless of
  config tuning. Suggest patience.
- **OCR provider issues.** If even `always-full` OCR is producing
  unusable text, the problem is upstream of the schema; flag it to
  the user rather than chasing field-level fixes.

## Common mistakes to avoid

- Recommending more uploads before checking configuration. OCR mode
  is the most likely cause and changing it costs nothing — investigate
  there first.
- Deleting and recreating a whole document type. That throws away
  model memory. Edit individual fields instead.
- Adding lots of validation rules without first asking what the user
  wants flagged. Validation is human-review work; more rules = more
  work for them.
