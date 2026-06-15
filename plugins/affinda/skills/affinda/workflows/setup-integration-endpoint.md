# Workflow: Set up an integration

Author user-provided Python that runs on every document matching its
scope, deploy it, and verify it works before relying on it.

## When to use this workflow

Trigger phrases:
- "Send extracted data to [external system]"
- "Run code on every document"
- "Integrate Affinda with X"
- "Webhook on extraction complete"

## Concept refresher

Briefly tell the user (only if new to integrations):

Integrations are **versioned**. Editing the code creates a draft;
production keeps running the previously-deployed version until you
explicitly call `deploy_integration_version`. You can also run *any*
version against a single document via `run_integration` without
deploying first — useful for testing.

## Plan

### 1. Get the organisation

```
list_organizations
```

### 2. Create the integration

```
create_integration(
    name="Send to Slack",          # short, descriptive
    organization_id=<...>,
    description="Posts extracted invoice totals to #ap-team Slack",
    event="DOCUMENT_PROCESSED",    # ask user if different
    workspace_id=<...>,             # optional scope
    document_type_id=<...>,         # optional scope (typically with workspace)
)
```

Skim the `description` with the user before sending — it lands in the
integration list and reads like a label.

### 3. Author the code

```
update_integration(
    integration_id=<...>,
    python_code="<full file content>",
)
```

**Critical caveats — relay to the user up-front:**

- `python_code` **replaces the entire file**. There is no partial
  patch. If you're editing existing code, include everything you want
  to keep.
- The deployed version doesn't change here. The code you uploaded is
  still a draft.

### 4. Deploy

```
deploy_integration_version(integration_id=<...>)
```

Makes the latest draft the live version.

If the user wants to test before deploying, skip to step 5 first and
return here once it works.

### 5. Test against a real document

```
run_integration(
    integration_id=<...>,
    document_id=<...>,             # numeric document ID, NOT slug
)
```

Read the result back to the user verbatim — the runtime output is the
source of truth.

### 6. Iterate or hand off

- Wrong output → step 3 again. The cycle is author → deploy → test →
  observe → repeat.
- Right output → tell the user the integration is live for matching
  documents going forward. They can monitor runs via
  `list_integration_runs`.

## External connections and custom APIs

If the integration needs to talk to a third-party service (Slack,
Google Sheets, etc.), first try the Pipedream connection flow. If the
service is not available in the directory, or the directory lookup
fails, continue with a manual custom integration instead of treating the
task as impossible.

### Preferred path: Pipedream service connection

Discover available apps:

```
list_pipedream_apps
```

Pick the slug matching the user's target service.

Initiate the OAuth handshake:

```
create_connect_token(app_name_slug=<...>)
```

This returns a one-time URL the user must visit to authorise. Do
*not* try to perform OAuth from inside Python code — Pipedream
handles it.

### Link the connection to the integration

```
add_connection_to_integration(
    integration_id=<...>,
    connection_id=<from auth flow>,
)
```

The Python code can now use the connection via Pipedream's runtime
helpers.

### Manual path: no matching directory app

When `list_pipedream_apps` returns no suitable match, or the target is a
private/custom API, create the integration without a connection. This is
a normal supported flow.

For authenticated APIs, use the Integration Settings **Secrets** section
for credentials:

1. Choose clear uppercase secret names, e.g. `MYOB_API_KEY`.
2. Call `list_integration_secrets` after creating the integration to see
   whether those names already exist. It requires the integration ID, so
   do not call it in the same tool batch as `create_integration`; wait
   for the create result first.
3. If a secret is missing, tell the user the exact name to add in the
   Secrets section. Never ask them to paste the value into chat.
4. Write code that reads credentials with `os.environ["NAME"]`.
5. Use direct HTTP from the Lambda (`urllib3` + `json`) for the custom
   API. Do not call `execute_action()` or `api_request()` unless there is
   a Pipedream `account_id`.

The absence of a Pipedream app is not a platform failure. Do not say the
integration cannot be built solely because the directory has no match.

## Common mistakes to avoid

- Editing `python_code` and forgetting to deploy. The user uploads,
  expects production to change, and is confused when nothing happens.
- Passing a document slug to `run_integration`. Use the numeric ID.
- Trying to OAuth in Python. Use Pipedream connections when a directory
  app exists; otherwise use the manual secrets + direct HTTP path.
- Replacing the whole file when you only meant to add a function —
  the user loses all other logic. Read first via
  `get_integration` if you're not sure of the current contents.

## Reverting a bad deploy

If a deploy breaks production:

```
list_integration_versions(integration_id=<...>)
revert_integration_version(integration_id=<...>, version_id=<previous>)
```

This re-deploys the previous version and creates an audit trail of
the revert.
