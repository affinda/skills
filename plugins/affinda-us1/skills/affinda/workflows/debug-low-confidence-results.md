# Workflow: Debug wrong or low-confidence extraction

The user is unhappy with an extractor's accuracy. This workflow walks
through a structured diagnosis instead of guessing. Work the levers
in order — each step either fixes the issue or rules a cause out
before moving to the next.

## When to use this workflow

Trigger phrases:
- "Why are extractions wrong?"
- "Confidence scores are low"
- "Field X keeps coming out wrong"
- "It used to extract this correctly and now it doesn't"
- "Can we improve accuracy?"

## Diagnostic order

The levers, from most to least likely to resolve the issue:

1. **OCR** — rule it in or out first; if the text going into the
   model is wrong, nothing downstream can recover.
2. **Model memory reference** — the highest-leverage check when the
   text is fine but predictions are wrong.
3. **Workspace configuration** — OCR mode, classification, model
   memory strategy.
4. **Field names and descriptions** — the secondary lever; reach for
   it only after the reference has been checked.

A common failure mode (for users and agents alike) is jumping
straight to field descriptions or adding more documents. Those are
rarely the cause; check the reference document first.

## Plan

### 1. Identify the document type

If the user hasn't named one, list candidates:

```
list_document_types(organization_id=<...>)
```

Confirm which one is causing trouble.

### 2. Sample real extractions and characterise the failure

```
list_documents(workspace_id=<...>, limit=20)
get_document_extraction(document_id=<one_of_the_above>)
```

Pull 3–5 recent documents. Look at per-field `parsed`, `raw`, and
`confidence`. The pattern matters:

- **Extracted text doesn't match what's on the page** — characters
  missing or garbled, values empty despite being plainly visible
  (often inside a logo or scanned region) → suspect OCR, step 3.
- **Text is right but values land in the wrong place** — wrong
  instance picked, fields shifted, rows missing, a
  previously-good format gone bad → suspect the model memory
  reference, step 4.
- **All fields drop on the same documents** → those documents likely
  mismatch the OCR mode (scans in a workspace not set to
  `always-full`), or were classified into the wrong type.
- **One field consistently slightly off** — extra trailing text, the
  wrong one of several candidates → field name / description,
  step 6.

### 3. Rule OCR in or out

Compare each suspect field's `raw` value (and `raw_text`) against
what's actually on the page. If they diverge, the problem is
upstream of the model and no model-side change will help.

The conclusive test is to reparse the document with full OCR — in
the app this is **⋯ → Apply Full OCR** on the document. If that
fixes the prediction, OCR wasn't being applied where it was needed;
change the workspace `ocr_mode` for ongoing protection
(`update_workspace`). If the option is unavailable or doesn't help,
OCR ran but misread.

Two distinct root causes:

- **OCR wasn't applied to the area.** The workspace is on `skip`; or
  on `auto-detect` and the document had a text layer the detector
  trusted (text layers can pass the heuristics while still being
  corrupted — this produces wrong values that *look* like misreads);
  or the text sits inside a logo/image and the mode doesn't OCR
  images. Fix: more OCR — `always-partial` for mostly-good text
  layers with problem regions, `always-full` for scans and
  unreliable text layers. When OCR issues recur, the move is almost
  always toward more OCR, and OCR cost is negligible next to the
  cost of silently wrong predictions.
- **OCR ran but misread.** OCR is very good but not perfect —
  handwriting and poor scans produce some misreads that can't be
  fixed at the OCR layer. Protect downstream instead: validation
  rules to flag suspicious values, data source matching to correct
  against known reference data, human review for low-confidence
  fields. The immediate fix for one document is correcting the value
  in the validation tool.

If full OCR has been applied and OCR still returns nothing for an
area where text clearly exists, that's a bug — flag it to the user
rather than working around it.

### 4. Check the model memory reference

When the text is fine but predictions are wrong, the reference
document is the first thing to check — its annotations directly
drive the prediction, so a problem there is usually the root cause.

Fetch the suspect document's extraction and follow
`reference_document_id` (null means no reference was used). Fetch
the reference document's own extraction and compare the suspect
fields. What you find falls into two cases:

- **Case 1 — right format, wrong annotations.** The reference
  matches the document's layout but its confirmed values are wrong:
  a field annotated in the wrong place, a missing row, a bad value.
  The model is faithfully copying a bad example. Fix: correct the
  annotations on the reference document itself, **re-confirm it**
  (editing without re-confirming leaves the old state in model
  memory — the most common reason a "fixed" reference keeps
  producing wrong predictions), then reparse the affected document.
- **Case 2 — wrong format.** The reference is structurally different
  from the document — different layout, fields, or table structure —
  and the model is applying annotations that don't fit. This usually
  means no closer match exists in model memory. Fix: confirm a clean,
  correctly-annotated version of the new format so future documents
  match it instead. (If a closer match clearly *does* exist in model
  memory and isn't being picked, that's a matching bug — flag it
  rather than working around it.)

**If neither case applies, the model memory diagnostic is
exhausted.** Adding more confirmed documents or re-confirming
existing ones won't help — move on to configuration and field-level
levers. See `reference/model-memory.md` for how model memory works
and how to curate it.

### 5. Read the workspace's processing settings

```
get_workspace_details(workspace_id=<...>)
```

Three settings frequently explain poor results:

- **`ocr_mode`** — wrong mode for the document mix (step 3 will
  usually already have surfaced this).
  - Lots of scans → must be `always-full`.
  - Mixed digital + scan → `always-partial` is the safe default.
  - `skip` for anything that isn't a native text PDF produces
    empty extractions, not just low confidence.
- **`enable_document_classification`** — if `False` and the user has
  multiple document types, mismatched uploads silently route to the
  wrong type and the "extraction" they see is the wrong schema being
  applied to the wrong document.
- **`model_memory_strategy`** — `manual` with no marked examples
  means the model has nothing to learn from.

### 6. Field names and descriptions

Only after the reference has been checked. See "Writing field
descriptions" in `reference/document-types-and-fields.md` for the
full guidance; in short:

- **Tighten the field name first.** The name is itself extraction
  guidance — "Invoice due date" resolves ambiguity that "Date"
  forces the model to guess at. A rename is often the whole fix.
- **Add a description only against an observed failure** — the wrong
  candidate being picked, trailing text being captured, a section
  being confused with another. Don't add generic descriptions
  preemptively.

Also check the field's type — text where a date is expected, or a
missing `multiple=true`, produces systematic errors no description
will fix.

```
list_recent_field_annotations(field_id=<...>)
```

is useful here: if users are manually correcting the same field
repeatedly, the corrections show exactly what the field should have
extracted.

### 7. Propose at most three changes

Order by expected impact, with the supporting evidence ("8 of 10
sampled documents had low confidence on `vendorAddress`"). Typical
shapes:

1. **Fix the model memory reference** (user re-confirms the
   corrected reference in the app, then reparses).
2. **OCR-mode change** — `update_workspace(workspace_id=...,
   ocr_mode=...)`.
3. **Field-level fix** — `update_field` to tighten the name or
   description, or fix the type.
4. **Validation rule** — `create_validation_rule` to flag bad
   extractions for review rather than relying on raw confidence.

### 8. Wait for confirmation before applying

Don't silently call `update_*` or `delete_*` tools — the user owns
the schema.

## What this workflow doesn't fix

- **Genuinely ambiguous documents.** Some inputs really are
  unreadable or contradictory. Confidence is supposed to be low.
- **OCR misreads on handwriting / poor scans.** Extraction is about
  as good as a careful human reading the same page; some inputs
  defeat both. The answer is downstream protection (validation
  rules, data source matching, human review), not config tuning.
- **Persistent model-side failures.** If the issue survives a clean
  reference, correct OCR, and tightened fields — across multiple
  format variants — it needs internal investigation by Affinda.
  Suggest the user contact support rather than iterating further.

## Common mistakes to avoid

- Reaching for field descriptions before checking the model memory
  reference. Descriptions are the secondary lever and won't fix a
  bad reference.
- Adding more documents to model memory for a format that's already
  represented. One good reference per format is usually enough; more
  can hurt.
- Editing a reference document's annotations without re-confirming
  it — model memory keeps serving the old annotations.
- Recommending more uploads before checking configuration.
- Deleting and recreating a whole document type. That throws away
  model memory. Edit individual fields instead.
- Adding lots of validation rules without first asking what the user
  wants flagged. Validation is human-review work; more rules = more
  work for them.
