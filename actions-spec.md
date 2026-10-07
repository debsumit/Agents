# Actions — Product and Engineering Specification

**Status:** Proposed implementation baseline  
**Version:** 1.1  
**Date:** October 7, 2026  
**Parent specification:** [Configurable GPT Builder](spec.md)

**Deployment profile:** [Standalone Chat Agent on Posit Connect](posit-connect-spec.md). Its identity, application-prefix, export, persistence, and process-lifecycle requirements apply to deployed Actions.

## 1. Objective and relationship to the parent specification

Implement the Actions configuration and execution capability for the custom GPT builder using the OpenAI Agents SDK and Responses API. A builder describes an external API, chooses operations, configures authentication, tests requests, and attaches the resulting tools to an assistant. A chat user invokes those operations through natural language.

The supplied screenshot establishes the UI reference: Add actions, Authentication with None/API Key/OAuth choices, a schema editor, Import from URL, Examples, and a Privacy policy field. It is reference content, not an instruction to import, authenticate to, or call any particular service.

This document expands the parent's Actions requirements. OAuth authorization-code connections and schema import from a URL are required release features. The builder publishes/exports definitions, while independently deployed Connect applications execute them for authorized viewers. Application login, installation isolation, callback prefixes, and durable execution follow the deployment companion.

This is application-owned functionality. OpenAI supplies model tool calling and the SDK agent loop; this application supplies the schema compiler, API executor, authentication, connection storage, approvals, and configuration UI. GPT Actions documentation is a behavior reference, not a promise that the Responses API hosts the ChatGPT Actions editor or its authentication infrastructure. [A1], [A2], [A3]

**Completion outcome:** A builder can import a supported API, configure each supported authentication mode, select and test operations, publish an assistant, and execute those operations with the correct user identity, approval behavior, and reliable result reporting.

## 2. Scope and vocabulary

### 2.1 Required features

- Create, edit, duplicate, remove, and disable action sets.
- Paste or upload JSON/YAML OpenAPI documents, or import one from an approved HTTPS URL.
- Validate and compile a bounded OpenAPI 3.0.x subset into function tools.
- Discover, select, classify, and test individual operations.
- Configure None, API Key, or OAuth authentication.
- Support bearer tokens, HTTP Basic credentials, and custom API-key headers under API Key configuration.
- Support OAuth authorization code with PKCE and per-user connections, refresh, disconnect, and reconnect.
- Record an optional privacy-policy link and describe the destination/data transfer at connection and approval time.
- Use the same authorized executor for explicit live tests and model-requested actions.
- Pause for missing connections or required approvals, then resume the existing run safely.
- Preserve action definitions in immutable assistant versions and audit executions.

### 2.2 Deferred features

OpenAPI 3.1, arbitrary remote `$ref`, Swagger 2, arbitrary MCP servers, cookies/query-string credentials, multiple simultaneous authentication schemes per operation, OAuth implicit/password/client-credentials grants, automatic provider discovery, arbitrary custom token exchange code, mTLS, SOAP, GraphQL-specific tooling, multipart uploads, streaming/binary API responses, callback/webhook execution, and automatic unbounded pagination are deferred.

Imported OpenAPI 3.1 documents must produce an explicit version error with remediation guidance; never silently relabel them as 3.0. Supporting the screenshot's workflow does not imply universal compatibility with every schema used in ChatGPT.

### 2.3 Vocabulary

| Term | Meaning |
| --- | --- |
| Action set | One builder configuration: schema, selected API server, authentication settings, selected operations, and privacy metadata |
| Operation | One method/path pair that may become a function tool |
| Action revision | Immutable executable definition of one selected operation |
| Action-set revision | Immutable snapshot linking schema, authentication configuration, and operation revisions |
| Connection | Server-managed API credential or an individual user's OAuth grant |
| OAuth client configuration | Application registration details; distinct from a user's access/refresh tokens |
| Live test | A real request with real possible effects, entered through the builder |
| Validation test | Compilation and argument validation with no external API execution |

## 3. Configuration UI

### 3.1 Entry and layout

The assistant builder's Actions section lists configured action sets, selected operation count, authentication type, validation status, and Edit/Remove controls. Add action opens the configuration screen. Back returns to the assistant builder and prompts only if local unsaved edits would be lost.

Use the screenshot's layout: heading and explanation, authentication summary/settings button, schema section with Import from URL and Examples, available operations, test output, and privacy-policy field. Keep Save and Cancel visible. Do not add a conversational schema-writing agent to the required release.

### 3.2 Authentication dialog

**ACT-01:** Provide None, API Key, and OAuth radio options. Switching the selected mode changes visible fields but does not discard saved credentials until the change is explicitly saved. Cancel closes the dialog without mutation.

| Choice | Editable fields | Saved summary |
| --- | --- | --- |
| None | No secret inputs | No credentials attached |
| API Key — Bearer | Secret token; optional connection label | Bearer, configured/missing |
| API Key — Basic | Username and password | Basic, configured/missing |
| API Key — Custom header | Header name and secret value | Header name, configured/missing |
| OAuth | Client ID, client secret, authorization URL, token URL, optional revocation URL, scopes, token endpoint authentication method | Provider domain, configured state, Connect/Test connection |

API Key is a UI category. Basic mode encodes `username:password` as required by HTTP Basic; do not label an arbitrary API key as Basic without a defined username/password mapping. Supported OAuth token endpoint methods are `client_secret_basic` and `client_secret_post`, using a confidential backend client.

**ACT-02:** Display the application-generated OAuth callback URL with Copy and provider-registration instructions. It uses this application's public HTTPS origin, not ChatGPT's callback URL. Public client/PKCE-only configurations are deferred; the backend keeps the confidential client's secret.

**ACT-03:** Never return stored secrets to the browser. Show a configured indicator with Replace and Remove actions. An unchanged secret field means keep the saved secret; an explicit remove operation means delete it. Do not submit a masked string as a replacement credential. Secrets use explicit saves, not general draft autosave or browser local storage.

### 3.3 Schema editor, URL import, and examples

**ACT-04:** Accept pasted text, `.json`, `.yaml`, and `.yml` uploads. Preserve the original text and detected format. Display parse location, JSON pointer, affected operation, severity, and a practical fix for each validation diagnostic.

**ACT-05:** Import from URL opens a URL input and explicit Fetch action. Fetching is read-only schema retrieval, not an API operation test. Show source URL, fetched time, content hash, and a preview before replacing an existing schema. An import failure must not overwrite the current editor.

Only fetch approved HTTPS URLs without embedded credentials or token-bearing query strings. Imports carry no API action credentials, browser cookies, or application authorization headers. Private schemas can be pasted or uploaded. Reject redirects; enforce the network controls in Section 11 at connection time.

**ACT-06:** Import stores a snapshot. Later executions do not fetch the schema again. Refresh from URL is a manual action that produces a diff; it cannot silently change a published action.

**ACT-07:** Examples offers maintained templates: a public read, an API-key read, an OAuth read, and a confirmed write. Templates use reserved example domains and are not live services. Loading an example confirms replacement when the editor contains unsaved content; it never inserts credentials or starts an API request.

### 3.4 Available operations and tests

**ACT-08:** After validation, list operation ID, method, path, summary, required authentication/scopes, read/write classification, enabled checkbox, validation state, and Test. Unsupported operations remain visible with errors and cannot be enabled. Document-level errors block all operations; operation-level errors block only affected operations.

**ACT-09:** Test opens a form from the original operation input schema. Required fields, enums, nested JSON, and secret-managed headers are distinguished. Provide Validate and Run live controls. Validation does not call the external API. Run live requires explicit confirmation showing destination and request content for both reads and writes.

The result panel shows test ID, status, elapsed time, sanitized request, safe response headers, redacted/bounded result, and outcome classification. It must label fixtures as simulated and real calls as live. A schema passing validation does not imply a working connection or successful operation.

### 3.5 Privacy and save state

**ACT-10:** Accept an optional HTTPS Privacy policy URL for private assistants. Display it near connection and approval UI when supplied; otherwise show Not provided. Validate syntax without automatically fetching the page. It must not be used as an API destination, proof of consent, or evidence of legal compliance.

Saving an action set does not publish the assistant. Surface independent states: Draft saved, Schema valid/invalid, Credentials configured/missing, User connected/not connected, and Published version current/outdated. A builder can save an incomplete draft. Publishing requires all enabled operations to be valid and authentication declarations complete. Export carries declared deployment bindings without secrets; target readiness validates actual configuration. OAuth does not require every future chat user to have connected already.

## 4. OpenAPI validation and compilation

### 4.1 Supported subset

**ACT-11:** Validate the OpenAPI structure before tool compilation. Apply the explicit support matrix below and fail unsupported enabled operations rather than dropping semantics.

| Feature | Initial behavior |
| --- | --- |
| Versions | OpenAPI 3.0.x only |
| Documents | One JSON or YAML document; safe parser; duplicate keys rejected |
| Servers | One explicitly selected absolute HTTPS server URL with optional fixed base path |
| Server overrides | Reject operation/path overrides and unresolved variables |
| Methods | GET, POST, PUT, PATCH, DELETE, HEAD; method is fixed by the saved mapping |
| Parameters | Required scalar path parameters; scalar query and non-sensitive header parameters |
| Serialization | Path/header `simple` scalars; query `form` scalars; reject other styles and arrays/objects in parameters |
| Body | JSON object body; bounded nesting and arrays of supported schema types |
| Primitive schemas | String, number, integer, boolean; enums; OpenAPI `nullable` |
| Validation constraints | Type, enum, required, length, numeric bounds, array size; reject unsupported constraints on enabled inputs |
| Formats | Explicitly supported date/date-time/UUID formats; other asserted formats require a diagnostic |
| Objects | Declared properties only; reject open-ended additional-property maps in tool inputs |
| References | Bounded acyclic local component references only |
| Composition | Reject `allOf`, `oneOf`, `anyOf`, `not`, discriminators, and recursive schemas initially |
| Security | None; one selected supported scheme per operation; authorization-code OAuth only |
| Responses | Bounded JSON or plain text; `204`/HEAD may have no body; reject binary/streaming content at runtime |
| Extensions | Retain as metadata; supported consequence hint can strengthen approval; unknown extensions never execute code |

Do not change a schema merely to make a tool accepted by the model. Record normalized input schema, compiled tool schema, compiler version, and fixed execution mapping separately. Provider schema-size/depth limits must also be checked against the pinned model integration.

### 4.2 Compiler pipeline

**ACT-12:** Compile through deterministic stages:

1. Parse bounded source text and validate document structure.
2. Resolve local references and root/path/operation inheritance using OpenAPI rules.
3. Merge path/operation parameters by `(name, in)`; reject conflicting duplicates and unsupported overrides.
4. Derive effective operation security. Missing operation security inherits the root; `security: []` explicitly means no credentials.
5. Resolve one supported security alternative. Multiple schemes in one requirement object require unsupported combined authentication and cannot be enabled. An anonymous alternative must be selected explicitly; never silently downgrade to None.
6. Validate selected server, methods, paths, schemas, and parameter serialization against the support matrix.
7. Generate collision-resistant tool names and strict function schemas.
8. Save immutable operation mappings and diagnostics; enable only valid selected operations.

Authentication UI must agree with effective schema security. Where a document omits security entirely, the builder may explicitly configure an authentication binding as an application override. Store that override visibly. Never attach credentials to an explicitly anonymous operation unless its effective configuration is deliberately revised.

**ACT-13:** Namespace tool names using an opaque action-set identifier and a normalized operation ID, respecting current provider name limits. Detect normalization collisions, truncate only with a stable collision-resistant suffix, and retain the original operation ID for display. Names, descriptions, and schemas are data; none can choose a handler dynamically by import or evaluation.

### 4.3 Strict schemas and optional values

OpenAI strict function schemas require closed objects and required declared fields; optionality therefore needs deliberate representation. [A3]

**ACT-14:** Preserve the difference between an omitted field and an explicit JSON null. Compile each optional field to a required presence wrapper, for example `{"is_set": false, "value": null}` or `{"is_set": true, "value": "urgent"}`. The backend removes fields with `is_set: false`, requires their value to be null, and validates present values against the original schema. Explicit null is accepted only when the source allows it.

Apply this convention recursively to optional JSON body properties. Optional request bodies use a presence wrapper around the body. Required fields retain their actual type, including null only when permitted. The test form exposes natural optional controls; wrappers are compiler/runtime details, not user-facing workflow jargon. Do not insert schema defaults or example values silently into a request.

Model arguments use separated `path`, `query`, `headers`, and `body` groups when those groups exist. Absent parameter groups are omitted from the tool schema. Require declared properties and `additionalProperties: false` throughout generated objects. Revalidate arguments server-side before authorization, approval, and execution.

## 5. Authentication and connections

### 5.1 None and API Key

**ACT-15:** None sends no authentication information. API Key supports exactly one configured method per operation: `Authorization: Bearer <token>`, HTTP Basic, or a validated custom header. API-key header names are fixed configuration, not model inputs. Reject reserved transport headers, CR/LF, and conflicts with schema headers; use Bearer/Basic controls for Authorization.

Connection records are scoped to owner, action-set binding, destination, and authentication configuration revision. API key replacement may rotate the secret on an existing connection; changes to its destination, scheme, or security identity require a new binding. Secret rotation does not copy a secret into historical assistant versions.

The application's OpenAI key is never an action credential. None-auth calls may still send user data, so normal destination, privacy, and approval controls remain applicable.

### 5.2 OAuth setup

**ACT-16:** Store a versioned OAuth client configuration: approved authorization/token/revocation endpoints, client ID, client-secret reference, requested scopes, token-endpoint authentication method, exact callback URL, and API audience/resource binding where explicitly supported. Authentication endpoint origins can differ from the API origin but must be separately approved.

Generate the callback from the operator-configured canonical application base URL, including the Connect content prefix, not untrusted request Host/Forwarded headers. Append `api/v1/oauth/action-clients/{client_id}/callback` without dropping or duplicating that prefix and require exact registration with the provider. Here `client_id` in the path is an opaque application configuration ID, not a secret.

Do not add arbitrary authorization/token parameters from model output. Provider-specific resource/audience or offline-access parameters may be operator-approved configuration with fixed validation. Reject unsupported provider requirements with a clear diagnostic rather than an unreviewed custom script.

### 5.3 Connect flow

**ACT-17:** Implement OAuth authorization code with PKCE S256 and an unpredictable, single-use, expiring state value using a maintained OAuth client library. Configuring OAuth and signing the application user in are distinct operations. This is also distinct from the application's own OIDC login.

1. The authenticated user explicitly clicks Connect for an action set.
2. Backend creates a transaction bound to user/session, action-set revision, OAuth client configuration, exact redirect URI, allowed return path, state, and PKCE verifier.
3. Browser is redirected to the approved authorization endpoint with `response_type=code`, client ID, requested scopes, state, and PKCE challenge.
4. Provider returns a code or error to the application's callback.
5. Backend verifies state, expiry, transaction ownership/session binding, issuer/configuration binding, and exact callback before accepting the code. A missing application session requires safe reauthentication or restart; never attach the grant to an arbitrary logged-in user.
6. Backend atomically claims the transaction and exchanges the code at the token endpoint with the verifier and configured client authentication. The authorization endpoint never receives this token exchange.
7. Validate token response, Bearer token type, lifetime, and granted scopes. Encrypt access/refresh tokens and create the user's connection.
8. Redirect to a fixed same-origin completion page with no codes/tokens in its URL. Mark any pending run resumable.

Prevent callback replay, login CSRF, and authorization-server mix-up by binding every transaction to its configured provider and callback. Avoid logging callback query strings or token request/response bodies. OAuth access tokens may be opaque; do not assume they are JWTs or treat OAuth alone as validated user identity.

### 5.4 Refresh, scopes, and disconnect

**ACT-18:** Resolve OAuth grants for the current user and exact binding. Verify required operation scopes before execution. If the provider omits a scope field, apply documented OAuth/provider semantics and record the assumption; an ambiguous scope response cannot silently authorize broader access.

In exported Connect applications, current user means trusted Connect identity mapped within the installation namespace. Grants never transfer from the builder. API-key credentials may be per-user or an explicitly provisioned, audience-restricted service identity; the backend records that identity and binds approvals to it rather than substituting the builder owner's account.

Refresh tokens shortly before expiry with a single-flight lock per connection. Persist rotated refresh tokens atomically with a generation check. Refresh only at the configured token endpoint, attach tokens only to the bound API origin, and never send refresh tokens to resource APIs.

An invalid grant or revoked token sets `reconnect_required`. A successful refresh does not justify replaying a possibly completed write. A confirmed authentication rejection on a safe read may be refreshed and retried once. Writes are retried only with verified rejection-before-effect semantics or supported upstream idempotency; otherwise require a new deliberate attempt.

**ACT-19:** Disconnect immediately prevents new execution and invalidates pending approvals for that connection identity. Best-effort revoke at a configured supported endpoint, erase locally held tokens, and report remote revocation failure separately. Already completed or dispatched requests cannot be undone by disconnect.

Changing provider, client ID, scopes, or resource binding creates a new authentication configuration and requires reconnection. Old configurations remain bound to old published versions unless revoked. A user changing external accounts always invalidates pending approvals; routine token refresh for the same grant does not.

## 6. Runtime execution and approvals

### 6.1 Execution path

```mermaid
sequenceDiagram
    participant U as User
    participant R as SDK runner
    participant G as Action gate
    participant C as Connection store
    participant X as External API
    U->>R: Natural-language request
    R->>G: Selected tool and structured arguments
    G->>G: Validate revision, permissions, arguments, destination
    G->>C: Resolve this user's credential binding
    opt Connection missing or insufficient
        G-->>U: Connect account; persist run interruption
        U->>C: Complete OAuth connection
        C-->>G: Authorized grant available
    end
    opt Approval required
        G-->>U: Show destination, identity, inputs, and effect
        U->>G: Approve exact request or deny
    end
    G->>X: Fixed HTTP request with injected credentials
    X-->>G: Response or uncertain transport outcome
    G-->>R: Sanitized structured result
    R-->>U: Answer consistent with the actual outcome
```

**ACT-20:** Integrate actions as SDK function tools using the SDK runner's tool loop. The dynamic tool schema comes from the saved compiler output; execution is bound to the corresponding revision and trusted request context. Do not run a second manual Responses loop around the runner or expose a universal unrestricted `http_request(url, ...)` tool. [A3], [A4]

**ACT-21:** Before network dispatch, validate tool identity, ownership, enabled status, input schema, destination, effective authentication, scopes, approval state, and run limits. Missing required inputs produce an actionable tool error or user clarification, never invented parameter values.

### 6.2 Connection and approval pauses

**ACT-22:** A missing/expired connection pauses only when the selected operation needs it. Emit `connection.required` and set the run to `awaiting_connection`; do not require connecting every configured service before any chat can start. Persist a resumable SDK interruption/checkpoint using the pinned SDK's supported mechanism. Model-visible output never contains authorization codes or tokens.

Connection completion permits the same operation to proceed through authorization and approval checks. It does not grant approval for a write. The run must not be recreated from the original user message. If the SDK cannot safely serialize/resume the required pause, treat that as an implementation blocker for this flow rather than secretly replaying a turn. SDK interruption/resume behavior must be verified in integration tests. [A5]

**ACT-23:** Writes, unknown-effect operations, all preview calls, and all live builder tests require one explicit approval. Published-chat reads may execute automatically under operator policy. GET alone is not proof of no side effects. A consequence annotation can strengthen policy; a `false` annotation cannot override application policy.

Approval binds user, run/test ID, tool-call ID, action revision, method, destination, canonical argument hash, effective connection identity, and policy version. Display the authenticated account label when available and an honest Connected account label when identity cannot be verified. Reject expired, reused, mismatched, revoked, or changed approvals.

In live tests, the final Confirm and run button is the approval; do not ask twice for the same unchanged request. For multiple writes, request a separate approval per operation initially. Refusal returns a structured denied result so the assistant can explain what was not performed.

### 6.3 Dispatch and retries

**ACT-24:** Atomically claim a tool execution before dispatch. Retain execution records across SDK retries and process restarts. Use a stable upstream idempotency key when the selected provider operation supports it; do not invent or send unsupported headers.

A dispatch failure before any bytes leave is different from timeout after dispatch. If a write may have been received, classify it as `outcome_unknown`, preserve any remote request reference, and do not automatically replay. If upstream reconciliation is unavailable, explain the uncertainty and allow a separately approved retry. There is no generic exactly-once guarantee for remote HTTP effects.

**ACT-25:** Serialize external actions per run initially. Allow sequential dependent calls to use prior results, but validate and approve each call separately. Use bounded retries only for validated safe/idempotent cases. Follow neither response-provided URLs nor pagination links automatically; any next request must map to an enabled operation and approved destination.

### 6.4 Result handling

**ACT-26:** Return a structured tool result with `status`, `operation_id`, `http_status`, `data`, `error`, `truncated`, and `request_reference`. Allowed statuses include `succeeded`, `accepted`, `failed`, `denied`, `connection_required`, and `outcome_unknown`.

HTTP `202` means accepted/pending, not completed. HTTP `204` and HEAD may succeed without JSON. Interpret documented business-error envelopes where configured, but do not treat arbitrary prose as definitive completion. Non-2xx, redirects, malformed declared JSON, and unsupported content types produce typed errors. A transport-successful write with an unusable response must not be automatically retried.

**ACT-27:** Bound responses while receiving and decompressing, then redact credentials and sensitive configured fields before persisting/model exposure. JSON truncation must produce a valid envelope with a bounded preview, not broken JSON. Do not forward cookies or auth headers. Treat the result as untrusted data; instructions inside it cannot change runtime policy.

**ACT-28:** Cancellation prevents undispatched work and requests best-effort cancellation of active HTTP work. Explain that already dispatched effects may still happen. Record final known/unknown outcomes even if the user closed the stream or cancelled the conversation.

## 7. Concrete schema-to-request example

This example is a non-live fixture. The `.example` host is reserved and is not a deployment destination. Configure a bearer connection separately; no secret belongs in the schema.

```json
{
  "openapi": "3.0.3",
  "info": {"title": "Record Service", "version": "1.0.0"},
  "servers": [{"url": "https://api.records.example/v1"}],
  "security": [{"bearerAuth": []}],
  "paths": {
    "/records/{record_id}": {
      "get": {
        "operationId": "getRecord",
        "summary": "Retrieve one record by its identifier",
        "parameters": [
          {
            "name": "record_id",
            "in": "path",
            "required": true,
            "schema": {"type": "string", "minLength": 1, "maxLength": 100}
          }
        ],
        "responses": {
          "200": {"description": "Record found"},
          "404": {"description": "Record not found"}
        }
      }
    }
  },
  "components": {
    "securitySchemes": {
      "bearerAuth": {"type": "http", "scheme": "bearer"}
    }
  }
}
```

Its illustrative Responses function descriptor is shown below. Actual SDK construction uses the equivalent function-tool interface in the pinned SDK; this JSON is not passed directly as an SDK `Agent` object. [A3]

```json
{
  "type": "function",
  "name": "as_7f2_getRecord",
  "description": "Retrieve one record by its identifier.",
  "strict": true,
  "parameters": {
    "type": "object",
    "properties": {
      "path": {
        "type": "object",
        "properties": {
          "record_id": {"type": "string", "minLength": 1, "maxLength": 100}
        },
        "required": ["record_id"],
        "additionalProperties": false
      }
    },
    "required": ["path"],
    "additionalProperties": false
  }
}
```

The model may produce `{"path":{"record_id":"R-123"}}`. The executor resolves a fixed `GET` request to `/v1/records/R-123` on the configured origin and injects the bearer credential. The model cannot supply a different URL, method, token, or host header.

Compile URL paths using controlled template expansion, preserving the configured server base path. Encode input once, reject traversal/dot-segment ambiguities and values that can alter path structure, and verify the final normalized URL remains in the intended origin/base-path boundary. A generic URL join that discards `/v1` is incorrect.

## 8. Data model and versioning

Extend the parent database instead of introducing a separate service or datastore.

| Entity | Required fields and behavior |
| --- | --- |
| ActionSet | ID, owner, assistant association, name, draft revision, disabled/deleted state |
| ActionSetDraft | Revision/ETag, original schema, source metadata, selected operations/server, auth-config reference, privacy URL, diagnostics |
| ActionImport | Owner, source type, sanitized URL if applicable, fetched time, content hash, bounded source, diagnostics, expiry |
| ActionSetRevision | Immutable schema hash, source snapshot, compiler version, selected server, auth config, policy metadata, operation revision IDs |
| ActionRevision | Existing parent entity; add action-set revision ID, original operation ID, compiled name/schema, source input schema, HTTP mapping, required scopes |
| OAuthClientConfig | Immutable nonsecret settings, client-secret reference, canonical callback, approved endpoints, security-binding version |
| Connection | Existing parent entity; add action binding, OAuth user/grant identity, granted scopes, encrypted token references, expiry, credential generation, status |
| OAuthTransaction | Hashed state, session/user binding, client-config revision, encrypted PKCE verifier, allowed return route, expiry, claimed/consumed state |
| ToolExecution | Existing parent entity; add execution-key uniqueness, approval fingerprint, dispatch stage, remote idempotency reference, HTTP/outcome status |
| ActionTest | Owner, immutable tested revision/snapshot, arguments hash, validation/live mode, execution/approval ID, safe results |

Enforce uniqueness of logical tool execution per run and tool-call identity, including when a paused run is resumed. Connection lookup must include the current user and exact configuration binding. Do not use a global "last connected token".

**ACT-29:** Changes to schema, server, operation selection, authentication configuration, or approval policy create new action revisions. Assistant versions reference immutable operation revisions; they do not dereference mutable drafts. Save and publish use expected revisions to reject stale concurrent changes.

Deleting an action from a draft does not mutate published versions. Disable everywhere revokes all associated execution immediately. Deleting a connection blocks affected operations in old versions. Editing a secret by rotation affects only its deliberate connection binding; changing account identity invalidates prior approvals.

Here, everywhere means within the installation receiving the operation. Independent exports do not receive live revocations from the builder. Operators must revoke or redeploy each affected Connect installation explicitly; the UI must not claim otherwise.

Duplicating an assistant/action set copies nonsecret definitions. API-key bindings, user OAuth grants, and OAuth client-secret bindings require explicit authorized reattachment and are never silently copied or exported.

## 9. Application API contracts

All routes below are under `/api/v1` and use the parent's authentication, error envelopes, pagination, and ownership rules. These are application endpoints, not OpenAI endpoints.

Paths are relative to the application's Connect base. Builder editing/import/publishing routes are not exposed in exported viewer runtimes. Those runtimes retain authorized connection, OAuth, approval, and execution routes. Production secret/service-identity configuration requires operator authorization.

| Method and route | Input and result |
| --- | --- |
| `POST /assistants/{id}/action-sets` | Create empty action-set draft; return `201` with ETag |
| `GET /assistants/{id}/action-sets` | List authorized action sets and configuration/connection states |
| `GET /action-sets/{id}` | Read draft metadata and original schema; no secrets |
| `PUT /action-sets/{id}/draft` | Save with `If-Match`; invalid schema may be saved as nonexecutable draft with diagnostics |
| `POST /action-imports` | Existing parent route; accept pasted/uploaded schema or `{source_type: "url", url: ...}`; return snapshot and diagnostics |
| `POST /action-sets/{id}/validate` | Validate expected revision without API execution; return per-operation diagnostics |
| `POST /actions` | Existing parent route; compile specified action-set draft revision and selected operations into immutable action revisions; does not publish assistant |
| `POST /actions/{id}/tests` | Existing parent route; validated immutable operation ID, arguments, `mode: validate|live`; live mode returns pending confirmation |
| `GET /action-tests/{id}` | Read current state and sanitized result |
| `POST /approvals/{id}/decision` | Existing parent route; approve/deny a run or live-test execution once |
| `POST /connections` | Store API credential; return nonsecret connection metadata |
| `PATCH /connections/{id}` | Explicit credential replacement/rotation; authorize and audit |
| `DELETE /connections/{id}` | Disable and disconnect; revoke OAuth remotely where configured |
| `PUT /action-sets/{id}/oauth-client` | Save new versioned OAuth client configuration; return canonical callback URL |
| `POST /action-sets/{id}/oauth/start` | Create OAuth transaction; return approved authorization URL; CSRF-protected |
| `GET /oauth/action-clients/{id}/callback` | Provider callback; state/session/provider-bound verification; exchange code server-side |
| `GET /action-sets/{id}/connection` | Current user's connection state, safe account label, scopes, and reconnect state |
| `POST /runs/{id}/resume` | Explicitly resume resolved connection pause; idempotent and authorized |
| `POST /action-sets/{id}/disable` | Revoke execution across versions |
| `DELETE /action-sets/{id}` | Revoke access and queue definition/secret cleanup subject to retention policy |

The OAuth callback is an externally navigated endpoint with its own transaction validation, not a normal JSON mutation endpoint. Authorization codes and state must not leak into frontend analytics. Ordinary missing-cookie handling must not accidentally bypass transaction/session binding.

**ACT-30:** URL import, live-test creation, and resume requests use request-level idempotency where duplicate network work or execution could occur. Publishing the assistant atomically pins the expected action-set revisions; a stale set that changed during publication fails with a revision conflict.

### 9.1 Validation diagnostic example

```json
{
  "status": "invalid",
  "diagnostics": [
    {
      "code": "unsupported_parameter_style",
      "severity": "error",
      "operation_id": "findRecords",
      "pointer": "/paths/~1records/get/parameters/0/style",
      "message": "deepObject query parameters are not supported.",
      "remediation": "Use supported scalar query parameters or a JSON-body operation."
    }
  ]
}
```

### 9.2 Errors and events

Add typed errors for `schema_parse_error`, `unsupported_openapi_version`, `unsupported_schema_feature`, `tool_name_collision`, `destination_not_allowed`, `import_too_large`, `authentication_mismatch`, `connection_required`, `insufficient_scope`, `oauth_state_invalid`, `oauth_exchange_failed`, `reconnect_required`, `approval_expired`, `approval_mismatch`, `external_rate_limited`, `external_timeout`, `external_response_invalid`, and `outcome_unknown`.

A failed OAuth state check returns a generic safe message without exposing which transaction/user exists. An external rate-limit error includes a bounded safe retry hint only when applicable; it does not schedule an automatic write retry.

Extend the parent's SSE stream with `connection.required` and `connection.ready`. Existing `tool.started`, `approval.required`, `tool.completed`, and terminal run events carry safe action/test references. Emit `tool.started` only when execution actually begins; awaiting permission is not an API call. Never include credentials, authorization URLs containing state, callback codes, or raw token responses in persisted SSE events. The UI obtains a fresh sign-in URL through the authenticated start endpoint.

## 10. State transitions and recovery

```text
Action draft:       incomplete -> invalid | valid -> versioned
Connection:         missing -> connecting -> connected -> reconnect_required | revoked
Tool execution:     proposed -> awaiting_connection -> validated
                    validated -> awaiting_approval -> approved -> dispatching
                    dispatching -> succeeded | accepted | failed | outcome_unknown
                    awaiting_approval -> denied | expired
Run connection:     running -> awaiting_connection -> queued -> running
```

These are permitted paths, not mandatory steps: an already connected operation skips connecting, and an authorized published read can skip approval. Persist transitions with transactional compare-and-set guards. A callback completing after a run was cancelled may save the user's connection but cannot resume or execute the cancelled run.

**ACT-31:** After restart, recover unexpired pauses and safe validation/import jobs. A worker lease expiring after `dispatching` does not prove failure before execution. Mark uncertain writes for reconciliation, not requeue. Recovered executions reuse approved canonical arguments and logical execution identity; current policy and access checks still apply.

The Connect profile guarantees persisted state and explicit recovery, not uninterrupted execution after a worker is terminated. Shared database leases and execution records must work across Connect processes; process-local background tasks cannot be the only record of a pending call.

Never persist arbitrary executable objects or browser-supplied SDK state. Use supported versioned serialization, encrypt sensitive state, and verify pending-run compatibility during dependency upgrades. A deleted secret/configuration may legitimately make an old paused run nonresumable; report the specific safe recovery action.

## 11. Network, data, and security controls

**ACT-32:** Enforce the following controls in deterministic backend code for imports, tests, OAuth backchannel requests, and action calls as applicable:

- Approved HTTPS schemes/origins and ports; TLS certificate verification enabled.
- Reject user-info URLs, fragments where inappropriate, ambiguous host parsing, private/loopback/link-local/metadata IPs, and disallowed IPv4/IPv6 forms.
- Verify all resolved addresses and bind validation to the actual connection, avoiding a validate-then-resolve-again gap. Preserve correct TLS hostname verification. Use an egress layer if the HTTP client cannot enforce this safely.
- Disable redirects for schema fetches, token/revocation requests, and API calls. Browser OAuth authorization navigation is a distinct configured user flow; it does not permit backend redirect following.
- Apply separate allowlists to schema origins, API origins, and OAuth endpoints. Importing a schema never approves its `servers` automatically.
- Prevent model arguments from overriding URL origin/base path, method, cookies, credentials, Host, Content-Length, connection headers, or proxy headers.
- Allow only fixed configured nonsecret headers and validated explicitly supported operation headers; prevent CR/LF injection and credential-header collisions.
- Bound source parsing, reference expansion, request size, header size, decompressed response bytes, and runtime duration.
- Redact secrets before logs, traces, errors, test panels, and model context. Do not depend solely on string masking; credentials are excluded by construction, then sanitized as defense in depth.
- Keep OAuth tokens encrypted at rest; use least-privilege secret access; document key rotation and token erasure.
- Sanitize external descriptions/results in the UI. Never follow instructions found in imported schemas, result bodies, or privacy pages as application commands.
- Maintain access checks on every action-set, import, test, connection, approval, run, and event lookup.

The required release supports approved public API destinations. Access to private corporate networks requires a separately designed egress connector; do not weaken the public URL policy to accommodate it accidentally.

## 12. Limits, observability, and retention

These are proposed application defaults, not claimed OpenAI or ChatGPT Actions limits.

| Control | Initial limit |
| --- | --- |
| Schema source | 1 MB after decompression; one document |
| Selected operations | 20 across an assistant's action sets |
| Schema structure | Maximum 10 object/array nesting levels; 200 resolved component references |
| Schema URL fetch | 10 seconds; no redirects; bounded streaming receive |
| API request body | 256 KB before dispatch |
| API response | 1 MB after decompression; 64 KB model-visible envelope |
| API call | 30-second total timeout; shorter connect/read phase deadlines |
| OAuth token response | 64 KB; 15-second total timeout |
| OAuth transaction | 10-minute expiry; state at least 256 bits of randomness |
| Pending execution approval | 15-minute expiry; connection pause expires after 15 minutes |
| Retries | At most two safe-read retries; no generic automatic write retries |
| Action executions | Parent cap of 20 per run; sequential per run |
| Source refresh | Explicit builder action; no background auto-refresh |

Apply provider limits where lower. Rate-limit imports, sign-in starts/callback failures, tests, and external calls independently per user and destination. Respect supported `Retry-After` within bounded deadlines without holding an execution indefinitely.

Record action-set revision, original operation ID, tool-call ID, actor, auth method, connection ID, approval decision, destination origin, dispatch phase, HTTP status, duration, response size, and outcome. Avoid unnecessary raw parameters/results in audit logs; configured sensitive fields require redaction. Usage and logs should distinguish real calls, retries, denied operations, and validation-only tests.

Retain redacted execution/audit records according to the parent's policy. OAuth transactions lose verifiers and codes after use or expiry. Retain grants until disconnect, revocation, expiry requiring reconnect, or account deletion; do not delete a valid refresh token merely because an access token expired. Cleanup failures and unsuccessful remote revocation must be visible to operators.

## 13. Acceptance scenarios

Use deterministic mock API/OAuth servers for automated tests and explicit opt-in live tests for configured providers. Mock-server results must never be displayed as proof of a real provider integration.

| ID | Scenario | Pass condition |
| --- | --- | --- |
| A-AC-01 | Authentication dialog Save/Cancel | Save applies the intended mode; Cancel preserves prior settings |
| A-AC-02 | Secret editing | Masked values never round-trip; explicit replacement/removal works |
| A-AC-03 | Paste/upload | Valid JSON/YAML compiles; duplicate keys and malformed syntax have precise diagnostics |
| A-AC-04 | URL import | Approved HTTPS content snapshots correctly; failure preserves existing schema |
| A-AC-05 | URL attack cases | Redirects, private IPs, metadata endpoints, rebinding, oversized/decompression payloads are blocked |
| A-AC-06 | Manual schema refresh | Diff is shown; published action behavior stays unchanged |
| A-AC-07 | Templates | Load without secrets or external execution; replacement preserves user intent |
| A-AC-08 | Schema support matrix | Unsupported versions, composition, references, encodings, and auth combinations fail explicitly |
| A-AC-09 | Operation selection | Only enabled valid operations become model tools |
| A-AC-10 | Strict optionality | Omission, explicit null, false, zero, and empty string retain distinct semantics |
| A-AC-11 | Parameter mapping | Path/query/header/body locations and server base path are preserved |
| A-AC-12 | Tool names | Duplicates/normalization collisions cannot dispatch the wrong handler |
| A-AC-13 | None/Bearer/Basic/custom header | Correct request is produced; credentials are absent from model-visible inputs/results |
| A-AC-14 | Security inheritance | Root inheritance, overrides, anonymous alternatives, and `security: []` work as specified |
| A-AC-15 | OAuth callback setup | Generated callback belongs to this application and matches configured canonical origin |
| A-AC-16 | OAuth happy path | Code/PKCE exchange creates a grant for the initiating user and exact binding |
| A-AC-17 | OAuth attack cases | State/code replay, expired transactions, wrong session/provider, callback tampering, and open redirects fail |
| A-AC-18 | OAuth denied/insufficient scopes | No protected operation executes; UI requests appropriate reconnect/consent |
| A-AC-19 | Token refresh concurrency | Concurrent use refreshes once and preserves rotated token state |
| A-AC-20 | Per-user identity | Two users cannot use each other's tokens; account changes invalidate approvals |
| A-AC-21 | Disconnect/revoke | New calls stop immediately; remote-revocation failure is honestly reported |
| A-AC-22 | Missing connection pause | Same run resumes after connect; no re-created user turn or duplicate action |
| A-AC-23 | Schema-only versus live tests | Validation makes no API call; final live confirmation authorizes exactly one request |
| A-AC-24 | Approval checks | Changed arguments, identity, destination, revision, or policy invalidate approval |
| A-AC-25 | Approval denial/expiry | No request dispatch; user receives a clear outcome |
| A-AC-26 | Retry and recovery | Safe retries are bounded; ambiguous writes are not replayed after timeout/restart |
| A-AC-27 | Result semantics | `202`, `204`, HEAD, errors, invalid JSON, and truncation are represented accurately |
| A-AC-28 | Cancellation | Undispatched work stops; already dispatched outcomes remain recorded |
| A-AC-29 | Prompt injection | Schema/result instructions cannot change authorization, expose secrets, or enable tools |
| A-AC-30 | Version changes | Old chats retain operation mappings; emergency revocation overrides old versions |
| A-AC-31 | Privacy and accessibility | Policy link is safe; dialog, editor, diagnostics, tests, and approvals work by keyboard |
| A-AC-32 | End-to-end natural language | Agent selects a valid tool, requests missing inputs, and answers from an authentic result |
| A-AC-33 | Secret/data leakage | API responses, callbacks, logs, traces, SSE, exports, and errors contain no credentials |
| A-AC-34 | Cross-resource access | Guessed IDs cannot read or mutate another user's imports/tests/connections/executions |
| A-AC-35 | Actual SDK integration | Pinned SDK/Responses path validates tools and resumes both approval and connection pauses |
| A-AC-36 | Publication gate | Invalid selected operations or incomplete auth config block publish; missing future user grants do not |

## 14. Delivery sequence and definition of done

1. **Editor and compiler:** Action sets, schema input/import, operation list, support-matrix diagnostics, strict mapping, and fixture-based request tests.
2. **Executor and API Key:** None/Bearer/Basic/custom-header connections, outbound controls, validation/live test split, approval binding, structured results, idempotency records.
3. **OAuth:** Client setup, callback registration, authorization-code/PKCE flow, per-user grants, refresh, disconnect, and connection interruptions.
4. **Integration and release:** Immutable assistant bindings, privacy UI, run events, recovery, secret redaction, acceptance suite, and operator documentation.

All four slices are required for this Actions specification. OAuth is not satisfied by showing a disabled radio button, and Import from URL is not satisfied by a text field that never fetches a schema.

Deliver source code, database migrations, nonsecret example schemas, a documented support matrix, API/OAuth fixtures, integration tests, opt-in provider smoke instructions, provider registration guidance, and operator procedures for secret rotation, destination allowlists, revocation, and unknown write outcomes.

## 15. Sources and interpretation

Official OpenAI documentation was reviewed on October 7, 2026. These sources establish the GPT Actions feature reference, function-tool model, and SDK runtime behavior. All application endpoints, UI contracts, quotas, OAuth hardening requirements, and compiler restrictions above are proposed requirements for this application; they are not claims of exact ChatGPT implementation parity.

- **A1:** [GPT Actions overview][A1]
- **A2:** [GPT Action authentication][A2]
- **A3:** [Responses function calling and strict schemas][A3]
- **A4:** [Agents SDK overview][A4]
- **A5:** [SDK sessions, streaming, and resumable interruptions][A5]
- **A6:** [Getting started with GPT Actions, including operation testing][A6]
- **A7:** [GPT Actions production considerations][A7]

[A1]: https://developers.openai.com/api/docs/actions/introduction
[A2]: https://developers.openai.com/api/docs/actions/authentication
[A3]: https://developers.openai.com/api/docs/guides/function-calling
[A4]: https://developers.openai.com/api/docs/guides/agents/sdk
[A5]: https://developers.openai.com/api/docs/guides/agents/running-agents
[A6]: https://developers.openai.com/api/docs/actions/getting-started
[A7]: https://developers.openai.com/api/docs/actions/production

Before implementation, verify provider-specific OAuth requirements, OpenAPI parsing behavior, supported strict-schema keywords, SDK interruption serialization, and actual deployment network controls. Integrations that exceed this bounded support matrix need an explicit supported extension rather than an undocumented workaround.
