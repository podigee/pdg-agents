# Podigee MCP connector

The hosted Podigee MCP connector gives ChatGPT, Claude, and other compatible agents structured
access to podcast management, publishing, imports, analytics, private-feed listeners, production,
distribution guidance, episode transcripts, and podcast content search.

Use `pdg` when a workflow starts with an audio or video file on the local machine. The CLI handles
resumable direct-to-storage upload, publishing confirmation, and operation polling. Use MCP
directly for hosted connector workflows and tools not exposed by the CLI.

## Public identity

These URLs are stable public contracts:

| Contract | URL |
| --- | --- |
| MCP resource and Streamable HTTP endpoint | `https://mcp.podigee.com/mcp` |
| OAuth protected-resource metadata | `https://mcp.podigee.com/.well-known/oauth-protected-resource/mcp` |
| Protected-resource compatibility metadata | `https://mcp.podigee.com/.well-known/oauth-protected-resource` |
| OAuth authorization server | `https://app.podigee.com` |
| OAuth authorization metadata | `https://app.podigee.com/.well-known/oauth-authorization-server` |
| Upload control origin | `https://mcp.podigee.com` |

Do not replace the MCP resource with the authorization-server URL. OAuth access tokens are bound
to the canonical MCP resource.

## Connect ChatGPT or Claude

Add a custom MCP connector using:

```text
https://mcp.podigee.com/mcp
```

The client discovers Podigee OAuth, opens browser consent, and uses S256 PKCE. Podigee supports
Client ID Metadata Documents for public hosted clients and does not require a client secret.
Authorization metadata advertises authorization-response issuer identification. Compatible clients
must reject duplicate or mismatched `iss` values, require `iss` when advertised, and validate any
issuer value that is present before exchanging an authorization code.

Consent is always bound to exactly one Podigee billing account or organization. If the signed-in
user can choose among eligible accounts, Podigee requires an explicit selection. Authorize a
second account through a separate connection.

The consent screen shows the connector, account, requested permissions, and redirect host. Review
all four before approving. Disconnect or revoke a connector from Podigee API settings when it is
no longer needed; revocation invalidates its complete access and refresh-token family.

## OAuth scopes

Every MCP tool declares its required OAuth scopes in `securitySchemes` and `_meta.securitySchemes`.
The public v1 scopes are:

| Scope | Permission |
| --- | --- |
| `mcp:podcasts:read` | Read podcasts, episodes, distribution, and operation status |
| `mcp:analytics:read` | Read podcast and episode analytics |
| `mcp:content:read` | Read episode transcripts and search indexed podcast content |
| `mcp:listeners:read` | Read private-feed listeners |
| `mcp:podcasts:write` | Create and update podcasts, episodes, and publishing workflows |
| `mcp:listeners:write` | Create and manage private-feed listeners |
| `mcp:media:write` | Create upload sessions and process media |

Scopes are not implied by legacy `admin`, `read`, or `read_write` labels. A legacy API key, generic
OAuth token, or legacy personal token cannot access MCP.

A valid but under-scoped credential receives `insufficient_scope`. MCP tool errors include an
`mcp/www_authenticate` challenge so a capable client can request the missing permission. After
reauthorization, obtain a new preview before confirming a consequential action.

## Personal access tokens

Browser OAuth is preferred. For an agent or script that cannot open a browser, create an MCP
personal access token in Podigee under **My settings > API** and select the exact scopes it needs.

The token is shown once. Store it only in protected client secret storage, a protected profile, or
the `PODIGEE_TOKEN` environment variable. Never put it in source code, prompts, logs, screenshots,
or shell arguments. MCP personal access tokens cannot be used with the legacy REST API.

MCP personal access tokens expire after 90 days. Podigee sends reminders 14 and 3 days before
expiry. Rotate with an overlap: create a replacement with the same or narrower tenant and scopes,
update the protected secret used by the agent, verify the real automation, and only then remove the
old token. Browser OAuth is preferred for persistent connections because it renews through refresh
tokens. Legacy REST API tokens retain their separate non-expiring compatibility behavior and cannot
be used with MCP.

## Read and write mode

Read tools are available when the credential has access to the requested account and object. Write
tools require all of the following:

1. A credential created with write access.
2. Write access enabled for the account on the server.
3. Permission for the requested podcast, episode, listener, or production.

Call `get_me` to see the first two: `write_enabled` reports what this credential can actually do,
and `server_write_enabled` reports the account-level switch. `doctor` reports the same in context.

There is no request header that turns writing on. Earlier versions of this guide described an
`X-Mcp-Write-Enabled: true` header; the server no longer reads it, and `pdg --write` no longer
sends it. Remove it from any client configuration that still sets it.

Clients should call a write tool only for a user-requested change, not during exploratory reads.

## Streamable HTTP

The endpoint uses stateless Streamable HTTP and supports MCP `2026-07-28` plus legacy
`2025-11-25`. Clients accept both JSON and event-stream responses. A modern request does not
require initialization, an MCP session ID, or server affinity.

The two eras differ as follows. A client picks one and stays with it for the whole connection; do
not mix a legacy `initialize` handshake with modern per-request metadata.

| | `2026-07-28` (current) | `2025-11-25` (legacy) |
| --- | --- | --- |
| Bootstrap | None; `server/discover` is optional | `initialize`, then `notifications/initialized` |
| Version sent in | Header **and** request `_meta`, every request | Header, after initialization |
| Routing headers | `Mcp-Method`, plus `Mcp-Name` when named | None |
| Result shape | Adds `resultType` and server identity | Plain result |
| Unknown method | `-32601` with HTTP 404 | `-32601` with HTTP 200 |
| Client notifications | Rejected with HTTP 400 | Accepted with `202 Accepted` |

A complete modern discovery request is:

```http
POST /mcp HTTP/1.1
Host: mcp.podigee.com
Content-Type: application/json
Accept: application/json, text/event-stream
Authorization: Bearer <MCP token>
MCP-Protocol-Version: 2026-07-28
Mcp-Method: server/discover

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "server/discover",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": {
        "name": "example-client",
        "version": "1.0.0"
      }
    }
  }
}
```

The corresponding successful JSON response has this shape:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "complete",
    "supportedVersions": ["2026-07-28", "2025-11-25"],
    "capabilities": {
      "tools": { "listChanged": false },
      "resources": { "subscribe": false, "listChanged": false },
      "prompts": { "listChanged": false }
    },
    "instructions": "Podigee MCP server instructions",
    "ttlMs": 0,
    "cacheScope": "private",
    "_meta": {
      "io.modelcontextprotocol/serverInfo": {
        "name": "podigee-mcp",
        "version": "1.0.0"
      }
    }
  }
}
```

Required `params._meta` keys:

| Key | Required | Notes |
| --- | --- | --- |
| `io.modelcontextprotocol/protocolVersion` | Yes | Must match the `MCP-Protocol-Version` header |
| `io.modelcontextprotocol/clientCapabilities` | Yes | An object; `{}` is valid |
| `io.modelcontextprotocol/clientInfo` | No | Validated when present; informational only |

`clientInfo` never changes authentication, tenant selection, authorization, or audit identity.

Every modern request includes matching protocol metadata and an `Mcp-Method` header. Add `Mcp-Name`
for `tools/call`, `resources/read`, and `prompts/get`, using the tool or prompt name or the resource
URI. A safe plain value is non-empty printable ASCII without leading or trailing whitespace. A
value that starts with the case-sensitive `=?base64?` prefix and ends with `?=` is also unsafe as
plain text because it would be ambiguous. Encode every unsafe value as
`=?base64?<Base64-encoded UTF-8 bytes>?=` using that exact case-sensitive sentinel and canonical
Base64. Recipients recognize the complete sentinel, strictly decode its Base64 payload as UTF-8,
and compare the decoded value with the request body. Podigee applies a stricter boundary and also
rejects decoded control characters.

As a Podigee guarantee rather than a protocol rule, every successful modern Podigee result carries
`resultType: "complete"` and Podigee server identity, because the server sets that value on every
result. The protocol itself also defines an `input_required` result carrying `inputRequests` or
`requestState`, which a client would have to fulfil and retry. Podigee does not emit
`input_required` in this release; the owned clients treat that result type as unsupported and never
retry it automatically.

Modern discovery, tool lists, resource lists and reads, and prompt lists return `ttlMs: 0` with
`cacheScope: "private"`. Refresh these authorization-sensitive catalogs before use. Clients must
honor valid `x-mcp-header` annotations when a server publishes them. Podigee publishes none in the
initial modern catalog.

Legacy clients initialize with `2025-11-25`, send `notifications/initialized`, and include this
header on later requests:

```http
MCP-Protocol-Version: 2025-11-25
```

Podigee always advertises `2026-07-28` first and accepts it by default. The `2025-11-25` flow remains
available only for wire compatibility with legacy clients. A dual-era client starts with
`server/discover` and falls back to legacy initialization only when discovery explicitly supports
`2025-11-25` but not `2026-07-28`, a structured unsupported-version error offers only the legacy
version as the highest mutually supported version, or the response is method-not-found or a
recognized legacy-shaped HTTP 400 discovery rejection. It must not downgrade after modern header
or metadata validation, authentication, authorization, rate-limit, transport, server, network, or
malformed-success failures.

Use `tools/list` as the authoritative catalog. Clients should not hard-code a tool list or infer
tool arguments from names.

## Preview and confirm changes

Consequential changes use a request-bound preview and confirmation flow:

1. Call the tool with `dry_run: true`.
2. Show the normalized plan and consequences to the user.
3. Repeat the same semantic request with `dry_run: false`, a stable `idempotency_key`, and the exact
   `confirm` value returned by the preview.

A confirmation is bound to the grant, actor, tenant, target, action, and request digest. It cannot
be used after reconnecting or reauthorizing. If anything changes, request a fresh preview.

Idempotency is stable across token refresh and reconnects for the same actor, tenant, operation
type, and key. Reusing one key with different arguments returns `idempotency_conflict`. Generate a
new key only for a genuinely new user intent.

### Import preview and recovery

`import_podcast` follows the same safety boundary. Its `dry_run: true` response includes the
canonical `request_digest` and a grant-bound `confirm` value. Starting the import requires the same
semantic request, `dry_run: false`, a stable `idempotency_key`, and that exact confirmation.

If a start was accepted but failed before Podigee created the import record, replaying the same
semantic arguments with the existing idempotency key is a recoverable pre-start action and
requires a fresh preview and confirmation. Changed import arguments represent a new intent and
require a new idempotency key, preview, and confirmation. For a later retryable import, preview
`resume_operation` and submit its exact confirmation. A token from an old or reauthorized grant
is never sufficient.

## Long-running operations

Publishing, imports, and some listener workflows return an `operation_id`. Preserve it and poll
`get_operation` until the operation reaches a terminal state.

If authorization is revoked, new calls and resumptions stop immediately. Running workflows stop
at safe checkpoints and retain completed drafts or imported data. A newly authorized grant for
the same actor and tenant can inspect previous results and resume eligible work with its current
scopes, but it must obtain a fresh preview and confirmation.

## Read transcripts and search podcast content

Both transcript tools require `mcp:content:read` and are read-only:

- `get_episode_transcript` reads one episode as time-ordered `start`, `end`, and `text` segments.
  Follow `pagination.next_offset` when present. An empty segment list is a successful result for an
  episode without stored transcript data.
- `search_podcast_content` searches indexed transcript words across a podcast, optionally scoped
  to one episode, and returns matching passages with `start_seconds` and `end_seconds`. Search is
  powered by Content Hub and may be unavailable because of plan or organization permissions.

Search is keyword-based rather than semantic and does not translate queries. Its current `total`
describes the returned snippets rather than an authoritative all-match count, so use explicit
offsets and do not infer another page from that value.

## Upload a local media file

Prefer the CLI:

```sh
pdg --write upload ./episode.mp3 --json
```

For end-to-end publication:

```sh
pdg --write publish ./episode.mp3 \
  --podcast 123 \
  --title "A new episode" \
  --json
```

`pdg` creates or recovers an opaque upload session, uploads up to four parts concurrently, obtains
a fresh signed URL for each retry, skips parts already confirmed after interruption, and asks
Podigee to complete the session. It never sends media bytes inside MCP JSON-RPC.

## Build a custom uploader

Advanced clients can use the same HTTPS control plane. Authenticate every control request with a
token for `https://mcp.podigee.com/mcp` carrying `mcp:media:write`.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `POST` | `/uploads` | Create or replay an upload session |
| `GET` | `/uploads/:upload_id` | Read state and confirmed parts |
| `POST` | `/uploads/:upload_id/parts/:part_number/url` | Obtain one short-lived part URL |
| `POST` | `/uploads/:upload_id/complete` | Validate and complete the upload server-side |
| `POST` | `/uploads/:upload_id/abort` | Idempotently abandon an incomplete upload |

Create a session:

```sh
curl https://mcp.podigee.com/uploads \
  -H "Authorization: Bearer $PODIGEE_TOKEN" \
  -H 'Content-Type: application/json' \
  --data '{
    "filename": "episode.mp3",
    "byte_size": 73400320,
    "content_type": "audio/mpeg",
    "idempotency_key": "upload-episode-2026-07-11"
  }'
```

The response contains an opaque `upload_id`, declared `byte_size`, `single` or `multipart` mode,
server-selected `part_size` and `part_count`, expiry, `preferred_client: "pdg"`, and the upload
documentation URI. Clients must consume these values rather than hard-code thresholds or part
sizes, and must reject a resumed session whose byte size differs from the local file.

The complete public state enum is:

| State | Client behavior |
| --- | --- |
| `initializing` | Wait and query the session again; do not send bytes. |
| `pending` | Resumable; request part URLs and upload missing bytes. |
| `uploading` | Resumable; query confirmed parts and upload only missing bytes. |
| `completing` | Wait while Podigee validates and finalizes storage. |
| `aborting` | Wait while Podigee aborts and cleans up storage. |
| `completed` | Upload is complete and may be attached by `upload_id`. |
| `aborted` | Do not resume; create a new upload for new intent. |
| `expired` | Do not resume; create a new upload. |
| `failed` | Do not send bytes; inspect the error and create or recover work as instructed. |

Only `pending` and `uploading` are resumable byte-transfer states. Treat every other state as
server-owned or non-resumable.

For each part number, request a URL:

```sh
curl -X POST \
  "https://mcp.podigee.com/uploads/$UPLOAD_ID/parts/$PART_NUMBER/url" \
  -H "Authorization: Bearer $PODIGEE_TOKEN"
```

Send exactly that part's bytes to the returned `upload_url` using the returned HTTP `method` and
`headers`, including `Content-Type` and `Content-Length`. The bytes must match `Content-Length`.
Treat this as a separate storage request and send only the returned headers. It MUST NOT include the
Podigee `Authorization` header, cookies, or other control-plane credentials, and it MUST NOT follow
redirects. Use a separate HTTP client with automatic redirects disabled.

The URL is bound to the session, part number, method, and a short expiry. It is one opaque
capability that may contain provider locators, so never parse it into storage keys or provider
identifiers. Request a new URL for every retry. Do not log or persist signed URLs or their query
parameters.

After interruption, call `GET /uploads/:upload_id` and skip the returned confirmed part numbers.
When all bytes are present, complete the session:

```sh
curl -X POST "https://mcp.podigee.com/uploads/$UPLOAD_ID/complete" \
  -H "Authorization: Bearer $PODIGEE_TOKEN"
```

Podigee validates stored parts, their order, and aggregate size, then finalizes the upload.
Completion and abort are idempotent.

Upload sessions expire after 24 hours, while individual signed URLs expire after 15 minutes. A
completed response contains only the opaque session identity, not a provider object URL. Attach it
to one episode within that 24-hour window. Consumption is exclusive; only an idempotent retry for
the same episode may reuse it. The server currently selects multipart mode at 30 MiB, limits files
to 20 GiB and at most 10,000 parts, but clients must always follow the values returned by
`create_upload`.

The same control plane is exposed through the MCP tools `create_upload`, `get_upload`,
`get_upload_part_url`, `complete_upload`, and `abort_upload`. Upload-session lifecycle operations do
not require preview and confirmation. Attaching media to an episode or publishing still does.

Attach completed media by opaque session identity:

```json
{
  "media": {
    "source": "upload",
    "upload_id": "7d9eb866-2e69-4a6d-a781-260fba53ca98"
  }
}
```

Podigee verifies that the upload is completed and belongs to the authenticated actor and tenant.
Never substitute a provider object URL for `upload_id`, and never place binary media in an MCP tool
call.

Legacy `/api/v1/uploads*` endpoints exist only for existing legacy API clients. MCP credentials are
rejected there, and legacy credentials are rejected by the new `/uploads` endpoints.

## Publish uploaded media

`publish_episode` accepts media only as a completed upload. The server never fetches episode audio
from the web, so the schema defines no remote URL source: it requires `source: "upload"` and an
`upload_id` produced by the upload control plane above. Attach an `upload_id` to exactly one
episode.

First request a preview:

```json
{
  "podcast_id": 123,
  "episode_attributes": {"title": "A new episode"},
  "media": {
    "source": "upload",
    "upload_id": "<upload_id returned by complete_upload>"
  },
  "publication": {
    "mode": "scheduled",
    "published_at": "2026-12-01T09:00:00Z"
  },
  "dry_run": true
}
```

Repeat the same semantic request with `dry_run: false`, a stable `idempotency_key`, and the exact
preview confirmation. Publication modes are `draft`, `now`, and `scheduled`; only `scheduled`
accepts `published_at`.

## Errors and challenges

Missing, expired, revoked, or wrong-resource credentials receive HTTP 401 with a standards-based
`WWW-Authenticate` challenge pointing to protected-resource metadata. Valid credentials without a
required scope receive HTTP 403 `insufficient_scope` on upload-control endpoints and a matching MCP
tool challenge.

Other stable tool errors include `invalid_params`, `forbidden`, `not_found`,
`idempotency_conflict`, and `authorization_revoked`. Treat `forbidden` as a permission failure, not
as evidence that an object does not exist.

Protocol-level failures are rejected before a tool runs, with HTTP 400 and a JSON-RPC error code:

| Code | Meaning |
| --- | --- |
| `-32020` | A required header is missing, or a header does not match the request body |
| `-32022` | The requested protocol version is not supported |
| `-32602` | Modern `_meta` is missing or malformed |

Mirror `MCP-Protocol-Version`, `Mcp-Method`, and `Mcp-Name` from the request body, and send
`params._meta` with the keys listed above. Retry a `-32022` with a version from `data.supported`.

In `2026-07-28`, an unknown method returns HTTP 404 with `-32601`. The JSON-RPC body distinguishes
that from a plain HTTP 404 served by a host that does not run an MCP endpoint at all.

On HTTP 429, respect `Retry-After`. `pdg` returns exit code 1 and includes
`retry_after_seconds` in JSON output when the server provides safe timing guidance. Wait that long
before retrying; do not reauthenticate. Poll operation or upload status at a measured interval
rather than in a tight loop.

## Agent safety checklist

- Use only the scopes needed for the user's request.
- Ask for confirmation before destructive or externally visible actions.
- Use the server's preview and exact confirmation flow where declared.
- Keep each `upload_id` and `operation_id` until work reaches a terminal state.
- Resume eligible work rather than starting it again.
- Never expose credentials, confirmation values, signed URLs, or private listener data.
- Never embed binary media in MCP calls.
- Use the schemas returned by `tools/list`; do not guess argument names.
