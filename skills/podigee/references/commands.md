# `pdg` command reference

## Global flags

Global flags may appear before or after a subcommand.

| Flag | Purpose |
| --- | --- |
| `--profile NAME` | Select a saved profile |
| `--locale LOCALE` | Select `en`, `de`, `es`, `fr`, or `pl` |
| `--json` | Emit one machine-readable JSON document |
| `--write` | Enable an explicitly requested write workflow |
| `--insecure` | Skip TLS verification for trusted local development only |
| `--version` | Show the installed version |

## Authentication

```sh
pdg auth login
pdg auth login --profile work
pdg auth login --api-key --profile agent
pdg auth status --json
pdg auth use work
pdg auth logout
```

Use browser login for interactive work. Use `--api-key` for hidden token paste. Avoid
`pdg auth token TOKEN` in agent workflows because command arguments can enter shell history.

Browser login discovers the Podigee authorization server from the canonical
`https://mcp.podigee.com/mcp` resource and uses S256 PKCE. A manually supplied token must carry
explicit MCP scopes and must not be a legacy Podigee API key.

Profiles are stored in `~/.pdg/config.yml`. Do not read or expose token values from that file.

MCP personal access tokens expire after 90 days. Browser OAuth profiles use refresh tokens and are
preferred for long-running connected agents. Rotate a PAT with an overlap:

1. Create a replacement MCP token in Podigee under **My settings > API** with the same or narrower
   tenant and scopes.
2. Update the protected secret store. For a `pdg` profile, run
   `pdg auth login --api-key --profile PROFILE` and paste the token only into the hidden prompt.
3. Verify `pdg auth status --json` and `pdg me --json`. Run `pdg doctor --write --json` before a
   publishing automation depends on the replacement.
4. Remove the old token from Podigee only after the replacement works in the real automation.

Do not rotate a legacy REST API key into an MCP profile. Legacy REST tokens follow a separate,
non-expiring compatibility contract and cannot authorize MCP.

## Account and content

```sh
pdg me --json
pdg podcasts list --limit 25 --json
pdg --write podcasts create --title TITLE [--subdomain NAME] --json
pdg --write podcasts update ID [--title TITLE] [--subdomain NAME] --json
pdg episodes list --podcast ID --limit 25 --json
pdg episodes get ID --json
pdg --write episodes update ID [--title TITLE] [--number N] \
  [--published-at ISO8601] --json
pdg --write episodes delete ID [--yes] --json
```

Episode deletion always uses server preview and confirmation. `--yes` skips only the local prompt.

## Transcript retrieval and search

```sh
pdg transcripts get --episode ID [--limit N] [--offset N] --json
pdg transcripts search --podcast ID --query TEXT \
  [--episode ID] [--limit N] [--offset N] --json
```

Use `transcripts get` to read one known episode as sequential, time-ordered transcript segments.
Use `transcripts search` to find relevant passages and timestamps across a podcast. The optional
`--episode ID` flag restricts search to one episode.

The JSON response shapes use different timestamp field names:

- Retrieval returns `episode_id`, `segments[]` with `start`, `end`, `text`, and optional
  `speaker_id`, plus `pagination`.
- Search returns `podcast_id`, `query`, `snippets[]` with `episode_id`, `episode_title`,
  `start_seconds`, `end_seconds`, `text`, and optional `speaker`, plus `total`.

Parse the fields for the command that produced the response. In particular, do not look for
`start_seconds` or `end_seconds` in retrieval segments.

A successful retrieval with `segments: []` and `pagination.total: 0` means the episode exists but
has no stored transcript segments. Report the empty transcript as the result. Do not convert it
into `not_found`, invent transcript text, or make a second episode lookup only to confirm existence.

Search is keyword-based rather than semantic and does not translate queries. Use short keywords in
the podcast's language. When the request is in another language or returns no snippets, retry with
a concise translation, synonym, or inflected form in the podcast's language before concluding that
there is no match.

Both commands accept a positive `--limit` and a non-negative `--offset`. For retrieval, follow
`pagination.next_offset` when present. Search currently returns no `next_offset` or authoritative
all-match count: treat `total` as the number of snippets in the current response and do not loop
until it. Both commands require the `mcp:content:read` scope and are read-only, so they do not need
`--write`.

Transcript search is powered by Content Hub. It may be unavailable when the account's plan does
not include Content Hub or when an organization admin has not granted the signed-in user access.
Direct transcript retrieval does not require Content Hub availability.

## Analytics and distribution

```sh
pdg analytics summary (--podcast ID | --episode ID) [--days N] --json
pdg distribution status --podcast ID [--platform NAME] --json
```

Analytics requires exactly one of `--podcast ID` or `--episode ID`. Use a strictly positive day
count.

## Upload and publish

```sh
pdg --write upload FILE [--idempotency-key KEY] --json
pdg --write publish FILE --podcast ID \
  (--title TITLE | --episode-id ID) \
  [--number N] [--now | --at ISO8601] \
  [--no-wait] [--timeout SECONDS] [--interval SECONDS] --json
pdg --write publish --operation-id UUID \
  [--no-wait] [--timeout SECONDS] [--interval SECONDS] --json
```

Omitting `--now` and `--at` creates a draft. Never combine them. For a new episode, provide a title.
For an existing episode, provide `--episode-id` and ensure it belongs to the selected podcast.

By default, upload idempotency is stable for the file's base name and content. After interruption,
repeat the same upload command so `pdg` reuses the derived key, queries the server-owned session,
and skips confirmed parts. To intentionally upload the same file again as a new intent, supply a
new opaque value with `--idempotency-key`; every retry of that new intent must reuse that exact
value. Changing the key during a retry creates a second upload session. Publishing computes stable
idempotency from its request and file content. Resume a retryable publish with the returned
operation ID, and do not create a second operation for the same intent.

Upload responses use the opaque `upload_id`. Only `pending` and `uploading` sessions are resumable.
The full state enum is `initializing`, `pending`, `uploading`, `completing`, `aborting`, `completed`,
`aborted`, `expired`, and `failed`. `pdg publish` attaches a completed session with
`{source: "upload", upload_id: "..."}`. `pdg --write upload FILE --json` returns the opaque `upload_id`, not a
provider object URL. Attach it to one episode within 24 hours; it cannot be consumed by a different
production. The CLI never sends binary media to the MCP tool.

The CLI does not expose podcast import. When using the connector's `import_podcast` tool directly,
preview under the current grant and use `request_digest` only to verify that the execution has the
same semantics. Initial execution must use the exact one-time, grant-bound `confirm` from that
preview. A recoverable pre-start replay requires a fresh `import_podcast` preview under the current
grant, the same semantic arguments and idempotency key, and its new confirmation. A later retryable
operation requires a `resume_operation` preview and that preview's confirmation. Never reuse a
confirmation after reconnecting or reauthorizing, or after it has already been submitted.

## Production and preflight

```sh
pdg production status PRODUCTION_ID --json
pdg production wait PRODUCTION_ID [--timeout SECONDS] [--interval SECONDS] --json
pdg doctor [--podcast ID] [--write] --json
```

The default wait timeout is 900 seconds and the default polling interval is 5 seconds.

## Version

```sh
pdg version
pdg version --json
pdg --version
```

JSON version output includes the Go version, operating system, and architecture.

The hosted service may recommend a newer compatible version on stderr while leaving stdout and
exit status unchanged. Relay that recommendation to the user after the command succeeds. A
mandatory update uses exit code 1 and, in JSON mode, returns
`code: "pdg_cli_update_required"` with `installed_version`, `latest_version`,
`minimum_supported_version`, `reason`, and `update_url`. Stop immediately. Never retry, switch to
legacy REST, change the User-Agent, or continue an upload after this response.

## Exit codes

| Code | Meaning | Agent response |
| ---: | --- | --- |
| 0 | Success | Continue |
| 1 | Runtime, API, transport, or mandatory-update failure | Inspect the structured error and stop on `pdg_cli_update_required` |
| 2 | Invalid command or local input | Correct flags or input |
| 3 | Authentication failure | Reauthenticate or provide a valid protected token |
| 4 | Preflight blocked | Report the blocker to the user |
| 130 | Interrupted | For publishing, preserve the operation ID. Otherwise stop and report any relevant resource ID. |

HTTP 429 and temporary server failures use exit code 1, never the authentication exit code. In
JSON mode, a usable rate-limit response includes `retry_after_seconds`; wait that many seconds and
retry later. If the field is absent, use bounded backoff before retrying. In either case, retry a
read only with unchanged arguments. Retry a mutation only with unchanged arguments and the same
stable idempotency key. If a mutation outcome is uncertain or those conditions cannot be
preserved, use operation recovery when an operation ID exists and do not rerun the mutation. Do
not reauthenticate for rate limiting. A
`retry_after_seconds` value is intentionally omitted when the server guidance is missing,
malformed, already elapsed, or longer than one hour.
