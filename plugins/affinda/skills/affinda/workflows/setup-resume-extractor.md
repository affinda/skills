# Workflow: Set up a resume / CV extractor

For recruitment, hiring, candidate search, resume parsing, CV
summarisation, or job-description analysis.

## When to use this workflow

Trigger phrases:
- "Set up a resume extractor"
- "I want to parse CVs"
- "Build a recruitment workspace"
- "Read job descriptions"

Recruitment has a purpose-built one-call setup
(`create_recruit_workspace`) that provisions the workspace and the
four standard document types in a single call. Use it instead of
hand-rolling a workspace + document types for resume use cases.

## Plan

### 1. Locate the organisation

```
list_organizations
```

### 2. One-call setup

```
create_recruit_workspace(organization_id=<from step 1>)
```

That's it. This creates a workspace named "Recruitment" pre-loaded
with four document types from Affinda's templates:

- **Resume** — full structured candidate data (work history,
  education, skills, contacts).
- **Job Description** — title, requirements, location, salary
  signals.
- **Redacted Resume** — same fields as Resume with PII removed.
- **Resume Summary** — auto-summarised condensed view.

The workspace's OCR / splitting / classification / model-memory
settings are tuned for the recruitment domain.

`create_recruit_workspace` is **idempotent** at the workspace level:
calling it again when a "Recruitment" workspace already exists in the
organisation updates the existing one rather than duplicating.

### 3. Tell the user

- They can immediately upload resume / CV / job description files.
- Classification will route uploads into the right document type
  automatically.
- No further field configuration is needed for the standard pipeline.

## When NOT to use this workflow

- The user wants a custom resume schema (e.g. only candidate name +
  email, or a domain-specific pre-screening template). Use
  `create_workspace` + `create_document_type` + `bulk_create_fields`
  instead. See `reference/field-schemas.md` for a starting resume
  schema you can edit.
- The user is recruiting for a specific role and wants an extractor
  scoped tightly to that — same answer, custom path.

## Extending after setup

Common follow-ups the user may ask after the bootstrap:

- **Add an internal candidate-tracking ID**: `create_field` against
  the Resume document type with a `candidateId` text field.
- **Match against a known job database**: connect the Resume's "skills"
  field to a data source of allowed skill values (see
  `workflows/connect-validation-data.md`).
- **Auto-route resumes to a hiring system**: set up an integration
  (see `workflows/setup-integration-endpoint.md`).

## Common mistakes to avoid

- Creating a "Resume" document type by hand and missing the redaction
  / summary variants. Use `create_recruit_workspace`.
- Using `create_recruit_workspace` for non-recruitment use cases (it
  creates a domain-named workspace and templates that won't match).
