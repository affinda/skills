# Common Errors and Remediation

Tools in this server return failures in two ways:

1. **Structured error in the response model** — most write tools (e.g.
   ``create_workspace``) return a Pydantic response with
   ``success: bool`` plus ``error`` and ``message`` fields. These are
   recoverable; surface ``message`` to the user and follow the
   recommendations below for the matching ``error`` code.
2. **`ToolError` raised** — for transport / API failures the server
   raises a FastMCP `ToolError`. The model receives a string with the
   API's detail. These are surfaced to the user verbatim by most MCP
   clients.

Use this catalog to translate raw errors into actionable next steps.

## `not_found`

The referenced object doesn't exist or was deleted. Often a stale id
held over from an earlier turn.

- **Workspace / document type / field / data source / integration**:
  call the corresponding ``list_*`` tool to refresh; if the user
  intended a different object, ask them to confirm.
- **Document**: a `404` here usually means the document was archived
  or deleted; suggest `list_documents` filtered to the workspace.

## `no_changes`

Returned by ``update_*`` tools when the request body would be a no-op
(no fields actually changing). Read the current state with the matching
``get_*`` tool, diff against the user's request, and only call the
update if there's a real delta.

## `invalid_response`

The Affinda API returned an unexpected payload shape. This is a
server-side issue, not user input. Surface the message to the user and
suggest retrying. If it persists, escalate — do not loop.

## `api_error`

Generic upstream failure. The ``message`` field carries the API's
detail. Common patterns:

- **Validation error on field type**: a field type was rejected. Check
  the type spelling against the schema (`text`, `integer`, `decimal`,
  `date`, `datetime`, `boolean`, `enum`, `table`, `group`, plus
  Affinda's structured types).
- **Permission denied**: the caller's JWT doesn't have rights on the
  target object. Confirm the user belongs to the relevant organisation
  via `list_organizations` and that they have the needed role
  (admin/owner for organisation rename, integration deploy, etc.).
- **Name length / character limits**: workspace names cap at 40
  characters; document type names and field labels have similar
  limits. Don't try to count yourself — submit the user's value and
  let the API reject if too long; surface the rejection verbatim.

## `not_authenticated` / 401 from the API

The Bearer token is missing, expired, or signed by the wrong key.
This is a host-environment issue, not something a tool call can fix.
Surface the authentication-required message and stop.

## Splitting failures

`create_workspace` or `update_workspace` with
`enable_document_splitting=True` requires a `document_splitter_id`.
Always call `list_document_splitters` first; the "General Document
Splitter" entry is the safe default.

## Rate limiting

The API may throttle high-volume bulk operations. If
`bulk_create_fields` or `bulk_create_data_source_values` returns a
rate-limit message, slow down and retry per row rather than retrying
the whole batch — the partial completion is normal.

## Long-running operations

`wait_for_document_processing` polls. Its hard cap is 300 seconds; if
the timeout fires, the document is still processing — call again or
fall back to `get_document` and let the user know.

## Integration runtime errors

Errors inside integration code surface in the result of
`run_integration` and on
`get_integration_run`. They are *not* MCP-level errors. Read the run
output verbatim and guide the user to fix the Python code via
`update_integration` followed by `deploy_integration_version`.

## What never fails

`list_organizations`, `list_workspaces`, `list_documents`, and similar
read tools return empty lists rather than 404 when there is nothing to
return. An empty response is *not* an error — it just means there's
nothing matching the filter.
