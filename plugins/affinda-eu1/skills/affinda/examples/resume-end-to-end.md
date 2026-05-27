# Worked example: resume extraction, end to end

Short example illustrating the recruitment-specific shortcut.

## User request

> "I'm setting up an HR pipeline for a startup. We need to parse
> resumes — both incoming applications and existing CVs we have on
> file. Can Affinda do this?"

## Agent response

"Yes. Affinda has a recruitment-tuned setup that handles resumes, job
descriptions, and redacted/summarised variants in one call. Let me
get your organisation."

```
list_organizations()
```

Returns one org: `Helios Talent` (id `org_99`).

"Setting up the recruitment workspace for Helios Talent."

```
create_recruit_workspace(organization_id="org_99")
```

Returns workspace `ws_1100` with four document types pre-configured:
Resume, Job Description, Redacted Resume, Resume Summary.

"You're ready. The workspace 'Recruitment' is set up under your org
with four document types tuned for the recruitment domain:

- **Resume** — full structured candidate data.
- **Job Description** — postings.
- **Redacted Resume** — same fields with PII removed (useful for
  bias-reduction during initial screening).
- **Resume Summary** — auto-summarised condensed view.

You can upload resume PDFs and DOCX files, or job descriptions, and
classification will route each upload into the right document type.
Want me to:

- Add an internal candidate-tracking ID field?
- Connect the skills field to a known skills taxonomy for
  validation?
- Set up an integration to push extracted data into your ATS?"

(User: "Tracking ID for now.")

```
create_field(
    document_type_id="dt_resume",
    slug="candidateId",
    label="Candidate ID",
    field_type="text",
    description="Internal tracking ID assigned by the recruitment team.",
)
```

"Added. The Resume document type now has a `candidateId` text
field. You can populate it via API on upload, or fill it in manually
during review."

## What this example illustrates

- One MCP call for the bulk of the work (`create_recruit_workspace`)
  — not a hand-rolled workspace + four document types + dozens of
  fields.
- The agent surfaced the four document types so the user knows what
  they got.
- The follow-up suggestions are domain-specific and the agent only
  acts on what the user picks.

## What didn't happen

- The agent didn't reach for `create_workspace` and start authoring a
  custom resume schema. The recruitment shortcut exists for a reason
  — use it whenever the user's intent is clearly recruitment.
- The agent didn't ask the user about OCR mode or splitting. The
  recruitment template tunes those for the domain.

## When to fall back to a custom flow

If the user says any of:

- "I only want a few specific fields, not the full resume."
- "We have a unique pre-screening template I need to match exactly."
- "We don't process job descriptions, only resumes."

…then `create_recruit_workspace` is overkill. Switch to the generic
`create_workspace` + custom `create_document_type` flow with the
resume schema from `reference/field-schemas.md` as a starting point.
