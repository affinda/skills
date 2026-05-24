# Workspace Settings Reference

Operational guidance for setting up and updating workspace processing
rules. Pairs with `reference/concepts.md` (which defines what a
workspace *is*) and the workflow guides under `workflows/`.

## Basic settings

| Setting | Description | Default |
|---|---|---|
| `name` | Display name. Cap is 40 characters — see "Naming" below. | required |
| `visibility` | `organization` (all members) or `private` (explicit members only) | `organization` |

## Choosing an `ocr_mode`

Map the user's wording onto one of the four enum values:

| Mode | Description |
|---|---|
| `auto-detect` | Detect if OCR is needed based on document content. |
| `always-full` | Force full OCR on every document. Use for scanned/poor quality documents. |
| `always-partial` | Retain embedded text but use OCR for additional detection. Assumes embedded text is reliable. **This is the recommended setting when you are creating workspaces.** |
| `skip` | Only extract embedded text. Use only for native digital PDFs with reliable text. |

**Interpreting user requests:**

- "disable/turn off/skip OCR" → `skip`
- "enable/turn on/auto OCR" → `auto-detect`
- "always/force/full OCR" → `always-full`
- "partial OCR" → `always-partial`

Default to `always-full` when the user says "enable OCR" without
specifying a mode.

`skip` on a workspace that ever receives scanned input produces empty
extractions, not just low confidence. Don't suggest it unless the
user has confirmed all uploads are native digital PDFs.

## Document splitting

| Setting | Description |
|---|---|
| `enable_document_splitting` | Enable / disable automatic splitting. |
| `document_splitter_id` | Which splitter model to use — required when splitting is enabled. |

When enabling splitting, **first call `list_document_splitters`** and
pass the "General Document Splitter" id. The API rejects splitting
without a `document_splitter_id`.

**When to enable splitting:**

- Users upload multi-page files that contain multiple logical
  documents (e.g., a batch of invoices in one PDF).
- Users upload document **packs**, **bundles**, **packets**, or
  **packages** — these terms imply a single file containing multiple
  distinct sections or document types. For example, a "loan
  settlement pack" contains separate documents like a payout
  statement, discharge authority, and payment instructions. Each
  section should be its own document type, and the splitter
  separates them.
- When in doubt about whether a file contains multiple documents,
  **enable splitting**. It is safe to enable even if some uploads
  are single documents — the splitter will simply return them as-is.

**When to disable splitting:**

- Users explicitly say they will upload one document per file, and
  the documents are not packs/bundles.

IMPORTANT: Splitting can only separate PAGES, not content WITHIN a
page. So, if you have two types of documents shown on the same page
of a document, you need a SINGLE document type which handles the
extraction from both. You should NOT set up two document types,
because this page can only be assigned to one of them.

IMPORTANT: Splitting does not use model memory (unlike data
extraction and classification). You should not suggest to the user
that performance will improve as more documents are uploaded and
confirmed.

## Document classification

| Setting | Description |
|---|---|
| `enable_document_classification` | Enable / disable auto-classification. |
| `document_classifier` | Which classifier model to use. |

**When to enable classification:**

- You should essentially always enable document classification in a
  workspace.
- Enabling classification in a workspace is best practice. Even if
  the user only has one document type. Classification only gets
  applied if a document is uploaded to a workspace without specifying
  a document type, so it is harmless to enable even if the user only
  has one document type.

**When to disable classification:**

- You should only ever disable classification if the user explicitly
  asks to do this, and even then, they probably do not actually want
  to disable it.

**Important:** When a user describes a document "pack", "bundle", or
similar, you should create **multiple document types** for the
distinct sections within it, and enable both splitting and
classification. Do not create a single document type for the entire
pack.

## Quality control

| Setting | Description |
|---|---|
| `reject_duplicates` | Reject documents matching an existing document in the workspace. |
| `reject_invalid_documents` | Reject documents that fail quality checks (unreadable, corrupt, unsupported format). |

## Validation

| Setting | Description |
|---|---|
| `enable_validation` | Enable validation rules on extracted data. |
| `auto_validation` | Run validation automatically after extraction. |
| `require_passing_validation` | Documents must pass all validation before confirmation / export. |

## Model memory (`model_memory_strategy`)

| Strategy | Description |
|---|---|
| `auto` | System selects documents to add to memory. **Default, recommended.** |
| `manual` | Only explicitly marked documents are added. Use for full control over training data. |
| `always` | All validated documents added. Use for small, consistent document sets. |

Model memory is shared across workspaces: adding document A (for
document type B) in workspace C makes it eligible to be used as an
example when parsing any document of type B in workspace E.

Model memory only affects document extraction and classification. It
does not affect splitting performance.

## Email upload

| Setting | Description |
|---|---|
| `enable_email_upload` | Allow document upload via email. |
| `ingest_email` | Auto-generated upload email address (read-only). |
| `whitelist_ingest_addresses` | Only accept emails from these addresses (e.g. `*@company.com`). |
| `use_email_html_body` | How to handle HTML email bodies. |

## Naming

Workspace names cap at 40 characters. Submit the value the user
provided as-is — if it exceeds the limit, the API returns a clear
length error and you can ask the user to shorten it.
