---
name: podigee
description: Operate Podigee through the pdg CLI and hosted MCP connector. Use when an agent needs to inspect a Podigee account, manage podcasts or episodes, retrieve or search transcripts, review analytics or distribution, upload local audio or video, publish or schedule an episode, manage private-feed listeners, import a podcast, monitor encoding, or resume interrupted Podigee work.
---

# Podigee

Use `pdg` to perform Podigee work from the command line. Prefer it when the task involves a local
audio or video file because it handles upload, publishing confirmation, and operation polling as
one workflow.

## Start safely

1. Run `pdg --help` to confirm that the application is installed.
2. Run `pdg auth status --json` to inspect the selected profile without making a network request.
3. Run `pdg me --json` to verify authentication and account access.
4. Add `--json` whenever consuming output programmatically.
5. Add `--write` only when the user has asked for a change.

If `pdg` prints a recommended-version notice on stderr, finish the successful task and relay the
recommendation and official update URL to the user. If it returns
`code: "pdg_cli_update_required"`, stop immediately and tell the user to install the stated latest
version. Never retry that command, downgrade the protocol, use a legacy REST fallback, alter the
User-Agent, or continue an upload after a mandatory update response. Run the original task again
only after the replacement `pdg` version is installed.

Stopping means no additional recovery work unless the user explicitly asks for it. Do not inspect
other binaries, search package managers or the web, test alternate installation channels, or try
to install the replacement yourself. Relay the exact official update URL from the structured error.
For tracked work, include both recommended and mandatory update notices in the completion or block
summary so the caller receives the same lifecycle guidance as the human-facing response.

Never print, echo, log, or place a Podigee token in a command argument. Use browser login, the
hidden token-paste login, a protected profile, or the `PODIGEE_TOKEN` environment variable. A
manually supplied token must be an MCP personal access token for the canonical Podigee MCP
resource, not a legacy REST API key.

Prefer browser OAuth for connected agents because refresh tokens keep the connection current. MCP
personal access tokens expire after 90 days. To rotate one, create a replacement in **My settings >
API**, update the protected profile or secret store, verify `pdg me --json` succeeds, and only then
remove the old token. Never delete the working token before the replacement has been verified.

## Choose the right command

| User need | Command |
| --- | --- |
| Inspect the current account | `pdg me --json` |
| List podcasts | `pdg podcasts list --json` |
| List a podcast's episodes | `pdg episodes list --podcast ID --json` |
| Read episode details | `pdg episodes get ID --json` |
| Read an episode transcript | `pdg transcripts get --episode ID --json` |
| Search what was said | `pdg transcripts search --podcast ID --query TEXT --json` |
| Review analytics | `pdg analytics summary --podcast ID --days N --json` |
| Check distribution | `pdg distribution status --podcast ID --json` |
| Check publishing readiness | `pdg doctor --podcast ID --json` |
| Upload without publishing | `pdg --write upload FILE --json` |
| Publish a local file | `pdg --write publish FILE --podcast ID --title TITLE --json` |
| Resume a publish | `pdg --write publish --operation-id UUID --json` |
| Wait for production | `pdg production wait ID --json` |

Read [references/commands.md](references/commands.md) when exact flags, authentication, exit codes,
or recovery behavior are needed.

Use `pdg doctor --write` only as preflight for a user-requested mutation when the result must also
check write access. The doctor command itself does not mutate account data.

## Read and search transcripts

Use retrieval when the episode is known and the task needs sequential, time-ordered segments:

```sh
pdg transcripts get --episode 456 --limit 25 --offset 0 --json
```

Use search to find relevant passages across a podcast. Add `--episode ID` to restrict the search
to one episode:

```sh
pdg transcripts search --podcast 123 --query "value-based pricing" \
  --episode 456 --limit 25 --offset 0 --json
```

The JSON field names differ by command. Retrieval returns
`segments[].start`, `segments[].end`, and `segments[].text`. Search returns
`snippets[].start_seconds`, `snippets[].end_seconds`, and `snippets[].text`, together with the
matching `episode_id` and `episode_title`. Parse the fields for the command that produced them.
A successful retrieval with `segments: []` and `pagination.total: 0` means the episode exists but
has no stored transcript segments. Report that directly without inventing content or making a
second lookup merely to confirm the episode.

Search is keyword-based rather than semantic and does not translate the query. Prefer short
keywords in the podcast's language. If a natural-language request uses another language or the
first query returns no snippets, retry with a concise translation, synonym, or inflected form in
the podcast's language before reporting no match. When a fallback query is what produced the
snippets, tell the user which term matched and that it is a fallback rather than their original
wording.

Use `--limit` and `--offset` to keep responses small. Retrieval reports
`pagination.next_offset` when another page exists. Search accepts a manual offset but currently
returns no reliable continuation metadata, so do not infer another page from `total`. Both commands
require `mcp:content:read`. They are read-only, so never add `--write`. Transcript search is powered
by Content Hub and may require Content Hub plan access or permission from an organization admin.
Direct transcript retrieval does not use that Content Hub availability gate.

## Publish local media

For a new draft:

```sh
pdg --write publish ./episode.mp3 \
  --podcast 123 \
  --title "A new episode" \
  --json
```

To publish after encoding, add `--now`. To schedule, add `--at` with an ISO 8601 timestamp. Never
combine `--now` and `--at`.

Let `pdg publish` run its preflight. Do not add `--skip-doctor` merely to bypass a blocker. If the
command exits with code 4, report the blocker to the user and stop unless the user explicitly
chooses how to proceed.

The command creates or resumes an opaque upload session, uploads bytes directly to storage,
previews the publish plan, submits the exact server confirmation, and polls the durable operation.
Preserve the returned `upload_id` and `operation_id` when present.

Only upload sessions in `pending` or `uploading` state are resumable. The complete state enum is
`initializing`, `pending`, `uploading`, `completing`, `aborting`, `completed`, `aborted`, `expired`,
and `failed`. Never send bytes for any state other than `pending` or `uploading`.

Attach uploaded media by opaque `upload_id`. `pdg --write upload FILE --json` returns this
identifier instead of a provider object URL. Attach it to exactly one episode within 24 hours;
only an idempotent retry for that same episode may reuse it. Never embed binary media in an MCP
request.

## Recover interrupted work

If publishing times out, is interrupted, or returns a retryable operation, resume it by operation
ID:

```sh
pdg --write publish --operation-id UUID --json
```

Do not restart a completed upload or create another episode. Repeating
`pdg --write upload FILE --json` queries the opaque upload session and skips parts already confirmed
by Podigee; reuse the same `--idempotency-key` when the original call supplied one, and pass
`--upload-id ID` or `--plan session.json` to resume a specific session. The publish operation resumes from saved
checkpoints and reuses the existing episode, media, and production records.

Use `--no-wait` only when the caller will persist the operation ID and check it later.

## Handle output and failures

- Parse stdout only. Human progress is written to stderr.
- Relay a recommended-version stderr notice after reporting the successful task result.
- Treat `pdg_cli_update_required` as a mandatory stop. Report the installed version, latest
  version, and official update URL from the error. Do not retry or work around enforcement.
- In JSON mode, expect exactly one JSON document on stdout.
- Treat a rate-limit response as exit code 1, not an authentication failure. When
  `retry_after_seconds` is present, wait that long before retrying. Otherwise back off and retry
  later. In either case, retry a read only with unchanged arguments. Retry a mutation only with
  unchanged arguments and the same stable idempotency key. If a mutation outcome is uncertain or
  those conditions cannot be preserved, recover by operation ID when available and do not rerun
  the mutation. Do not rerun OAuth for a rate limit.
- Treat exit code 2 as invalid local input and correct the command.
- Treat exit code 3 as an authentication problem and reauthenticate.
- Treat exit code 4 as a publishing preflight blocker and report it.
- On interruption, preserve any returned operation ID before stopping.
- Never retry a mutation with changed arguments under the same idempotency key.

## Use direct MCP when needed

The CLI intentionally focuses on common command-line workflows. Use the native Podigee connector
or the hosted MCP server directly for RSS imports, private-feed listener management, remote-media
publishing, operation cancellation, or a tool not exposed by `pdg`.

The canonical MCP resource is `https://mcp.podigee.com/mcp`. Let the client discover OAuth and
request any missing scope. Never add a vendor-specific authorization header. If a tool returns an
`insufficient_scope` challenge, authorize the requested scope and obtain a fresh preview before
retrying any confirmed action.

For `import_podcast`, first call with `dry_run: true` under the current grant. Use the returned
`request_digest` only to verify that the execution has the same semantics, and send the exact
one-time, grant-bound `confirm` with `dry_run: false` and a stable `idempotency_key`. For a
recoverable pre-start replay, obtain a fresh `import_podcast` preview under the current grant and
reuse the same semantic arguments and idempotency key. For a later retryable operation, preview
`resume_operation` under the current grant and use that preview's confirmation. Never reuse a
confirmation after reconnecting or reauthorizing, or after it has already been submitted.

Do not invent unsupported CLI commands. In particular, there is no `pdg import` command.

For direct server connection details, upload-control endpoints, and tool safety conventions,
consult the public
[Podigee MCP server guide](https://github.com/podigee/pdg-agents/blob/master/docs/mcp-server.md).
