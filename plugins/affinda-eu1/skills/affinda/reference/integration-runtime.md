# Integration Runtime — operational reference

What integration code looks like, the runtime utilities it can call,
and known service-specific recipes. Pairs with
`workflows/setup-integration-endpoint.md` (which walks the create →
author → deploy → test cycle) and `reference/concepts.md` (which
defines what an integration *is*).

Integrations are Python AWS Lambda functions triggered by document
events that connect Affinda to external services.

## Critical rules

Two rules are violated most often, and break integrations every
time. Follow them without exception.

1. **Every code change MUST be followed by `deploy_integration_version`.**
   Code written via `replace_integration_code` is NOT live until
   deployed. After any successful `replace_integration_code`, your
   very next tool call must be `deploy_integration_version` — do not
   stop to explain, do not wait for the user, do not test first
   (tests run against the deployed version). A change that isn't
   deployed is a change that didn't happen.

2. **`replace_integration_code` replaces the ENTIRE file — there is
   NO partial-edit tool.** You cannot edit individual lines, apply a
   diff, or "just change the broken function". Send the complete
   file every time, including every line you want to keep. Omit a
   function, import, or helper and it is deleted. Always call
   `get_integration` first to read the current code before composing
   the replacement.

If you ever find yourself about to end a turn after calling
`replace_integration_code` without having called
`deploy_integration_version`, stop and call
`deploy_integration_version` now.

## Sequencing rules

- **Don't block on missing fields.** If the user has named the fields
  they want included (e.g. "vendor name and total amount"), or has
  said "just write the code" / "don't ask questions", proceed
  immediately. Use the named fields converted to camelCase
  (`vendorName`, `totalAmount`). The integration code accesses by
  key — `doc.get("vendorName")` returning `None` at runtime is fine.
  Note in your follow-up that those fields aren't configured yet and
  offer to add them later, but don't stop to ask first.
- **Use `list_fields`.** Use `list_fields` with the document type
  ID to get all fields in a single call. Do NOT use
  `get_field_group` or `get_field` individually — `list_fields`
  returns everything organized by group already.

> IMPORTANT: There should be no need to ever call `get_field_group`
> when configuring integrations. Rely on `list_fields`.
- **Pass `event`, `workspace_id`, `document_type_id` atomically** to
  `create_integration` when you know them. They set the integration
  up fully in one shot.
- **`add_connection_to_integration` after `create_integration`.**
  Without it, Pipedream-backed integration code can't reach the
  external service. Skip this for manual custom APIs that use Secrets
  and direct HTTP instead of Pipedream.
- **No directory app is not a blocker.** If `list_pipedream_apps`
  fails or returns no suitable service, continue with a manual custom
  integration when the user has provided enough API details. Create the
  integration, use Secrets for credentials, write direct HTTP code, and
  deploy. Do not describe the missing directory app as a platform flaw.
- **Secret checks need the integration ID.** Call
  `list_integration_secrets` only after `create_integration` has
  returned the integration ID. Do not put both calls in the same tool
  batch.

## Triggers

Integrations can fire on:

| Event | Description |
|---|---|
| `document.parse.completed` | Document finishes parsing. |
| `document.validate.completed` | Document finishes validation. |

Configure via `configure_integration_settings` with the `event`
parameter.

## Code architecture

Integrations are Python Lambda functions. Skeleton:

```python
from utils import get_document, execute_action, get_validation_results, get_document_portal_url

def lambda_handler(event, context):
    response = get_document(event['document_identifier'])
    doc_fields = response["data"]                    # camelCase keys

    invoice_number = doc_fields.get("invoiceNumber")
    supplier_name = doc_fields.get("supplierName")
    total_amount = doc_fields.get("totalAmount")

    result = execute_action(
        account_id="<connection_account_id>",
        tool_name="<pipedream_action_key>",
        tool_params={"key": "value"},                # dict, NOT a JSON string
    )

    if not result.get("success"):
        raise Exception(f"Failed: {result}")

    return {"success": True, "message": "Data sent successfully"}
```

Manual custom API skeleton (no Pipedream connection):

```python
import json
import os

import urllib3
from utils import get_document

http = urllib3.PoolManager()

def lambda_handler(event, context):
    response = get_document(event["document_identifier"])
    doc_fields = response["data"]

    api_key = os.environ["MYOB_API_KEY"]
    payload = {
        "vendorName": doc_fields.get("vendorName"),
        "totalAmount": doc_fields.get("totalAmount"),
    }

    resp = http.request(
        "POST",
        "https://api.example.com/invoices",
        body=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Accept": "application/json",
        },
        timeout=urllib3.Timeout(connect=5.0, read=30.0),
    )

    if not 200 <= resp.status < 300:
        raise Exception(f"External API failed: HTTP {resp.status} - {resp.data.decode('utf-8', errors='replace')}")

    return {"success": True, "message": "Data sent successfully"}
```

> **Code is agent-edited only.** Users can read the integration code
> through the Affinda UI but cannot edit it directly — all
> modifications happen through the MCP tools (`get_integration` →
> `replace_integration_code` → `deploy_integration_version`).

### Optional: Pydantic models

A Pydantic model is auto-generated from each document type's schema
and lives in `pydantic_models.py`. The class name is the document
type name with non-alphabetic characters removed and title-cased:

| Document type name | Model class |
|---|---|
| Invoice | `Invoice` |
| Job Description | `Jobdescription` |
| Tax Form | `Taxform` |

The model accepts both camelCase and snake_case input, so you can
pass the raw API response directly:

```python
from pydantic_models import Invoice

response = get_document(event['document_identifier'])
doc = Invoice(**response["data"])
# Then access as snake_case attributes: doc.invoice_number, doc.supplier_name
```

If you're unsure of the model class name, **stick with the raw dict
approach** — it always works.

### `get_document()` response shape

Returns a flat (NOT JSON:API) response with **camelCase** keys:

```json
{
    "data": {
        "invoiceNumber": "INV-001",
        "invoiceDate": "2024-01-15",
        "supplierName": "Acme Corp",
        "totalAmount": 1500.00,
        "lineItems": [
            {"description": "Consulting", "quantity": 10, "unitPrice": 150.00, "total": 1500.00}
        ],
        "rawText": "INVOICE..."
    },
    "meta": {
        "identifier": "vAbC123XYZ",
        "ready": true,
        "readyDt": "2024-01-16T10:30:00Z"
    },
    "error": { "errorCode": null, "errorDetail": null }
}
```

- All field names are camelCase (`invoiceNumber`, `supplierName`,
  `totalAmount`).
- Document fields are under `response["data"]`.
- Section / group fields (line items, etc.) are nested lists of
  dicts with camelCase keys.
- Document metadata (identifier, dates) is under `response["meta"]`.

### Utility functions in `utils.py`

- **`get_document(document_identifier: str) -> dict`** — fetches the
  response above. `document_identifier` is a string like
  `"vAbC123XYZ"` from `event['document_identifier']`.
- **`execute_action(account_id, tool_name, tool_params=None) -> dict`**
  — runs a Pipedream action. `account_id` from the connection,
  `tool_name` is the Pipedream action key. **`tool_params` must be a
  `dict`, not a JSON string** — serialised internally.
- **`get_validation_results(document_identifier) -> list[ValidationResult]`**
  — validation rule outcomes for a document.
- **`get_document_portal_url(document_identifier) -> str`** — portal
  URL for viewing a document. **Use this** when the user wants a
  link back to Affinda; do NOT construct URLs manually
  (`https://app.affinda.com/documents/{identifier}` is wrong).
- **`api_request(account_id, target_url, method="POST", body=None, headers=None, params=None, files=None, data=None) -> dict`**
  — HTTP through Pipedream auth proxy. Requires a Pipedream
  `account_id`. **`body` must be a `dict`, not a JSON string** —
  serialised internally. Use when a Pipedream connection exists but no
  specific Pipedream action exists or the action is unreliable. To
  upload a file, pass `files={"file":
  ApiRequestFile(content_bytes, "name.pdf", "application/pdf")}` (import
  `ApiRequestFile` from `utils`) for a multipart request, or
  `data=<bytes>` for a raw byte-stream body. Do NOT hand-roll multipart
  with `requests` / `httpx` when using a Pipedream connection.

  **Response shape.** `api_request()` always returns a wrapper — the
  third-party payload is **nested under `["data"]`**, never at the top
  level:

  ```python
  {
      "success": True,           # bool
      "data": { ... },           # ← parsed third-party JSON lives HERE
      "status_code": 200,        # third-party HTTP status (note: status_code, not "status")
      "headers": { ... },
  }
  ```

  There is **no `["body"]` and no `["status"]` key.** Always read the
  payload via `["data"]`, e.g. `(resp.get("data") or {}).get("value", [])`.
  `api_request()` already **raises on any non-2xx response**, so you do
  not need to inspect `status_code` to detect failures.
- **`upload_file(account_id, file_content: bytes, file_name, content_type=None) -> str`**
  — uploads a file and returns a signed URL. Required when an action
  parameter expects a file path / URL (Google Drive, Xero
  attachments, Gmail attachments).

### Important code rules

- `event['document_identifier']` is always a string (`"vAbC123XYZ"`),
  never numeric.
- Always raise exceptions on failure. Never silently return error
  dicts.
- `pydantic_models.py` is auto-generated. Do NOT modify it. If the
  user provides a Pydantic model, add the imports its type hints
  need (e.g. `from datetime import date`).
- Use `execute_action()` for Pipedream actions, not direct HTTP calls.
  Use `api_request()` when a Pipedream connection exists but you need
  direct HTTP control through that connection. For manual custom APIs
  with no Pipedream app/account, use `urllib3` directly and read
  credentials from `os.environ`.
- All dict parameters (`tool_params`, `body`) are Python dicts, never
  pre-serialised JSON strings.
- The `context` parameter in `lambda_handler` carries no useful data
  — ignore it.
- **Custom identifiers and config values** (spreadsheet IDs, tenant
  IDs, table names, account codes provided by the user) are
  **hard-coded** in the Lambda. The only input is
  `event['document_identifier']`.
- **Credentials are never hard-coded.** For manual custom APIs, tell
  the user the exact secret names to add in the Integration Settings
  Secrets section, then read them from `os.environ["NAME"]`.
- **Prefer asking the user for specifics** (spreadsheet id, channel
  name, etc.) rather than guessing. However, if the user explicitly
  asks you not to ask questions or to "just fix it" / "just write
  the code" / "just build it", do NOT stop to ask clarifying
  questions — proceed and use clearly-labelled placeholder values
  (e.g. `YOUR_SPREADSHEET_ID_HERE`) only for details the user did
  not supply, noting them in your reply so the user knows what to
  replace. When the user has already provided the specifics
  (spreadsheet id, account id, sheet name, field names, etc.), use
  those values directly without further questions.
- Original-file URL is at `response["meta"]["file"]`. Use it when the
  integration needs to forward or attach the source document.

## Service-specific recipes

### Google Sheets

Most users want to append rows. Use `api_request()` with the Google
Sheets API v4 directly — it gives you precise control over column
ordering and append behaviour.

The spreadsheet id is between `/d/` and `/edit` in the URL:
`https://docs.google.com/spreadsheets/d/<SPREADSHEET_ID>/edit`.

**Always read existing headers first**, then push only the matching
columns. If the sheet has no headers, write a header row first using
the document field names, then append data.

The header-write must be a **one-off**: the integration runs on every
matching document, and re-appending the header row each time is a
common bug. Guard against it by reading the existing header row first
and only writing headers when row 1 is empty:

```python
from utils import get_document, api_request

def lambda_handler(event, context):
    response = get_document(event["document_identifier"])
    doc = response["data"]

    spreadsheet_id = "<SPREADSHEET_ID>"
    sheet_name = "Sheet1"
    account_id = "<CONNECTION_ACCOUNT_ID>"

    # Read existing headers from row 1
    header_resp = api_request(
        account_id=account_id,
        target_url=f"https://sheets.googleapis.com/v4/spreadsheets/{spreadsheet_id}/values/{sheet_name}!1:1",
        method="GET",
    )
    existing_headers = ((header_resp.get("data") or {}).get("values") or [[]])[0]

    if not existing_headers:
        # First-run only: write headers from document field names.
        headers = ["invoiceNumber", "invoiceDate", "supplierName", "totalAmount"]
        api_request(
            account_id=account_id,
            target_url=f"https://sheets.googleapis.com/v4/spreadsheets/{spreadsheet_id}/values/{sheet_name}!A1:append",
            method="POST",
            params={"valueInputOption": "USER_ENTERED"},
            body={"values": [headers]},
        )
        existing_headers = headers

    row = [str(doc.get(header, "")) for header in existing_headers]

    api_request(
        account_id=account_id,
        target_url=f"https://sheets.googleapis.com/v4/spreadsheets/{spreadsheet_id}/values/{sheet_name}!A1:append",
        method="POST",
        params={"valueInputOption": "USER_ENTERED"},
        body={"values": [row]},
    )

    return {"success": True, "message": "Row appended to Google Sheets"}
```

Key points:

- Always read headers first; map data to match column order.
- If headers exist, push only fields with matching header columns.
- Use camelCase for field names (matches both document data and
  produces readable headers).

### Google Drive

When uploading files (e.g. via `google_drive-upload-file`), upload
the file with `upload_file()` first to get a signed URL, then pass
that URL into the action's `filePath` parameter.

### SharePoint / OneDrive / Excel

**Auto-discover first.** Before asking the user for SharePoint /
OneDrive details (site id, drive id, file id, table name, headers),
exhaust available tools and context: the URL the user gave, the
connected account, prior run logs, and Microsoft Graph APIs via
`api_request()`. Only ask the user when discovery genuinely fails or
there's ambiguity (multiple matching files / tables).

**Writing to Excel files**: only support writing rows into named
tables, not arbitrary cells. When no Pipedream action exists for
adding rows, use Microsoft Graph via `api_request()`:

```python
tables_response = api_request(
    account_id=account_id,
    target_url=f"https://graph.microsoft.com/v1.0/sites/{site_id}/drives/{drive_id}/items/{file_id}/workbook/tables",
    method="GET",
)
tables = ((tables_response.get("data") or {}).get("value", []))
if not tables:
    raise Exception("No tables found in workbook — user must create one in Excel (Insert → Table)")

table_name = tables[0]["name"]  # or ask the user if multiple

# api_request() raises on non-2xx, so reaching this line means the row was added.
add_response = api_request(
    account_id=account_id,
    target_url=f"https://graph.microsoft.com/v1.0/sites/{site_id}/drives/{drive_id}/items/{file_id}/workbook/tables/{table_name}/rows/add",
    method="POST",
    body={"values": [[col1_value, col2_value, col3_value]]},
)
created_row = add_response.get("data")
```

Rules:

- Never guess table names like `"Table1"` — discover via API or
  confirm with the user.
- `api_request()` raises automatically on any 4xx / 5xx, so you don't
  need to check the status yourself — a returned response means success.
  Read results from `response["data"]`, never `response["body"]`.
- Never invent OneDrive / SharePoint action names that don't exist.
  Use `api_request()` with Microsoft Graph as the fallback.

### Gmail

For attachments: use `upload_file()` to get signed URLs, then pass
them via `attachmentUrlsOrPaths`. Provide corresponding filenames in
`attachmentFilenames`.

### Slack

Most users want to post messages on document events. Use
`slack-send-message`. Include relevant document data. You need a
channel id or name.

### Salesforce

Use the appropriate Salesforce action for the record type — Lead,
Contact, Account, Opportunity, etc.

### HubSpot

Use HubSpot's create / update actions for the relevant object
(contacts, companies, deals).

### Xero

- **Tenant id**: discover available tenant ids via tools first; then
  hard-code in the Lambda.
- **Scope**: only support creating sales invoices for now. Don't try
  to search for or retrieve contacts.
- **Response shape**: Xero actions return data at
  `response["result"]["ret"]`, NOT `response["data"]`:

```python
response = execute_action(
    account_id=account_id,
    tool_name="xero_accounting_api-xero-create-sales-invoice",
    tool_params=payload,
)

invoices = response.get("result", {}).get("ret", {}).get("Invoices", [])
if not invoices:
    raise Exception("No invoice returned from Xero")
invoice_id = invoices[0].get("InvoiceID")
```

- **Attaching the source file**:

```python
file_url = document_data["meta"].get("file")
if file_url:
    file_name = document_data["meta"].get("fileName", "invoice.pdf")
    execute_action(
        account_id=account_id,
        tool_name="xero_accounting_api-upload-file",
        tool_params={
            "tenantId": tenant_id,
            "filePathOrUrl": file_url,
            "fileName": file_name,
            "documentType": "Invoices",
            "documentId": invoice_id,
            "syncDir": "/tmp",
        },
    )
```

### Airtable

Use `api_request()` (not `execute_action`) to add records:

```python
api_request(
    account_id=account_id,
    target_url=f"https://api.airtable.com/v0/{app_id}/{table_id}",
    method="POST",
    body={"fields": {"Human Field Name": "value", "Another Field": "value"}},
)
```

Response carries `success` (bool) and `data` (created record).
Discover the correct table id via tools and hard-code it.

### AWS S3

The `aws-s3-upload-file-url` action's `filename` parameter does
**not** support full S3 keys with folder prefixes — it only places
files at the top level. The value must not contain `/` characters.
Don't offer the user an option to specify an S3 prefix or folder.

### Webhooks / custom APIs

When a Pipedream connection exists but no packaged Pipedream action
fits, use `api_request()` for full HTTP control through that connection.

When no Pipedream app or connection exists for the target service, use
the manual custom API path instead: direct `urllib3` requests, secrets
stored in the Integration Settings Secrets section, and credentials read
with `os.environ["NAME"]`.

## Pipedream action discovery

- Use `list_pipedream_apps` to find apps. Search is imprecise — try
  short, simple terms (`"sheets"` rather than `"google sheets"`).
- If no matching app is returned, continue with the manual custom API
  path when the user has provided an endpoint and auth details. Do not
  tell the user the task is impossible just because the service is not
  in the directory.
- There's no tool that returns a Pipedream action's parameter schema.
  When `execute_action` fails on bad params, read the error message
  carefully — it usually indicates what was expected.
- If a Pipedream action consistently fails, fall back to
  `api_request()` for full HTTP control.

## Connection management

- `list_integration_connections` returns connections for the
  organisation. Each carries `id`, `app_name`, `app_name_slug`,
  `account_id`, `is_active`, `organization_id`.
- Each connection's `account_id` is what `execute_action()` calls in
  integration code use.
- If no suitable connection exists, find the right `app_name_slug`
  via `list_pipedream_apps`, then call `initiate_service_connection`
  to start the OAuth flow.
- If no suitable app exists in the directory, stop trying to create a
  Pipedream connection and build a manual custom integration with
  Secrets and direct HTTP instead.

## Debugging a failing integration

When a user reports a failing integration, run this workflow
**before** making any code changes:

1. **Read the current code**: `get_integration` — see the current
   `python_code` and configuration.
2. **Check run history**: `list_integration_runs` — recent runs and
   their pass / fail. Logs in list results are **truncated to 10,000
   characters**.
3. **Inspect error logs**: `get_integration_run` on any failed run
   for the **full, untruncated** logs. Always do this before
   diagnosing — truncated logs from the list often miss the actual
   error.
4. **Diagnose** from logs and code.
5. **Fix**: `replace_integration_code` with the corrected full-file
   contents.
6. **Deploy (MANDATORY)**: `deploy_integration_version` immediately
   after the replace succeeds. The fix isn't live until deployed.
7. **Re-test** with `run_integration`.

Don't skip steps 1–3. You must read the existing code and error
logs before attempting a fix. Don't skip step 6 — a fix that isn't
deployed isn't a fix.

IMPORTANT: Always complete the fix by calling
`replace_integration_code` followed by `deploy_integration_version`.
Do not stop at diagnosis — the user expects a working, deployed fix,
not just an explanation of what went wrong. If you are missing some
configuration detail (e.g., a spreadsheet ID), use a
clearly-labelled placeholder and note it in your response.

Common issues:

- Missing or incorrect `account_id` (check connections).
- Wrong Pipedream action key or parameters (read the error message
  for expected params).
- Document field access errors — fields are **camelCase** at
  `response["data"]`, e.g. `invoiceNumber`, not `invoice_number`.
- Passing a JSON string instead of a dict to `tool_params` or
  `body`.
- Missing permissions on the external service (user re-authorises).

## Deploying and testing

After ANY call to `replace_integration_code`:

1. **Deploy (MANDATORY)** — `deploy_integration_version` as your
   next tool call.
2. **Test** with `run_integration` using a **numeric document ID**
   (e.g. `"1000003"`), NOT the alphabetic identifier / slug.
3. **Review** with `get_integration_run` for logs and output.

After a successful run, suggest the user tests with multiple
documents and checks results in the downstream system to catch edge
cases.

If the user wants to re-export documents that were already uploaded
before the integration existed, they enable the integration and then
trigger it by re-parsing or re-confirming those documents (depending
on the configured trigger).

If the user asks to see logs themselves, point them to the **Runs**
tab in the integration editor — run history and logs live there.

## Reverting

If a deployment causes problems, use `revert_integration_version` to
roll back. Use `list_integration_versions` to find available
versions.
