---
name: affinda
description: Process and extract structured data from documents (invoices, resumes, contracts, custom document types) using the Affinda MCP server and Document JSON:API.
---

# Affinda Skill

This skill teaches the agent how to use the Affinda MCP server end-to-end —
beyond what's in individual tool descriptions.

The MCP server itself ships with self-contained tool docstrings, plus a
small set of MCP prompts and resources for the most common flows. This
skill complements those with longer-form workflow guides, worked
examples, and reference material that's too verbose for tool
descriptions.

If the user has only loaded the MCP server, they can still complete the
basic flows. Loading this skill in addition is the difference between
"can call tools" and "can sequence tools fluently for real-world tasks".

## When to consult this skill

The agent should consult this skill (or its individual files via
progressive disclosure) when:

- The user is **setting up something new** (a workspace, document type,
  data source, integration). See `workflows/`.
- A user asks **"why is extraction wrong?"** or wants to **improve
  quality**. See `workflows/debug-low-confidence-results.md` and
  `reference/model-memory.md`.
- The user is **reviewing extracted documents** (confirm / reject /
  archive). See `workflows/human-review-queue.md`.
- The user references **terminology** that needs grounding —
  workspace, document type, field group, validation rule, model
  memory. See `reference/concepts.md`.
- The agent encounters **error responses** it doesn't recognise. See
  `reference/common-errors.md`.

For clean tool-selection guidance the MCP tool descriptions are the
primary source; consult this skill only when the user's intent
requires multi-step reasoning the descriptions don't fully cover.

## Layout

```
plugin root/
└── skills/
    └── affinda/
        ├── SKILL.md                  # this file
        ├── workflows/                # end-to-end flows
        │   ├── setup-invoice-extractor.md
        │   ├── setup-resume-extractor.md
        │   ├── debug-low-confidence-results.md
        │   ├── connect-validation-data.md
        │   ├── setup-integration-endpoint.md
        │   └── human-review-queue.md
        ├── examples/                 # worked examples
        │   ├── invoice-end-to-end.md
        │   └── resume-end-to-end.md
        └── reference/                # focused operational references
            ├── concepts.md                   # domain glossary
            ├── common-errors.md              # error codes + remediation
            ├── field-schemas.md              # starter field schemas + types + slug rules
            ├── workspace-settings.md         # OCR / splitting / classification / model memory / email
            ├── model-memory.md               # how model memory works + curation best practice
            ├── document-types-and-fields.md  # field settings, validation rules, auto-populate
            ├── data-sources.md               # default schema, single vs bulk rows, matching criteria
            ├── extraction-review.md          # how to inspect a document or spot-check a field
            └── integration-runtime.md        # integration code primitives + service-specific recipes
```

## Style notes for the agent

- Tool names below are the **canonical MCP tool names** exposed by
  the Affinda MCP server (e.g. `create_workspace`, `create_field`,
  `delete_validation_rule`). Some hosts may surface aliases — the
  canonical names always work.
- Workflow guides assume the user has at least one Affinda
  organisation already provisioned. If `list_organizations` returns
  empty, surface that — there's nothing this skill can do until an
  org exists upstream.
- Always prefer a single `bulk_create_*` call to a loop of single-row
  creates. The bulk variants are dramatically faster and surface
  per-row errors more usefully.
- When the user describes their input as a "pack", "bundle",
  "packet", or "package", treat it as a single file containing
  multiple documents and enable splitting on the workspace.

## Companion resources

The MCP server hosts five static resources at `affinda://` URIs that
this skill mirrors and extends:

- `affinda://concepts` ↔ `reference/concepts.md`
- `affinda://errors` ↔ `reference/common-errors.md`
- `affinda://schemas/{invoice,resume,contract}` ↔
  `reference/field-schemas.md`

If the agent's MCP client supports resources, fetch the `affinda://`
versions for the canonical (server-validated) text. Otherwise, the
local copies in `reference/` are equivalent.
