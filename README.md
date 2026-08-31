# Podigee for agents and the command line

`pdg` brings Podigee podcast workflows to your terminal, scripts, and AI agents. Use it to inspect
your account, manage podcasts and episodes, check analytics and distribution status, upload media,
and publish episodes without leaving your existing workflow.

It is designed for creators, production teams, developers, and technical operators who want a
fast, script-friendly way to work with Podigee.

This repository is the public home of:

- signed `pdg` downloads and release notes
- [Podigee MCP server documentation](docs/mcp-server.md)
- the [public lowercase `podigee` agent skill](skills/podigee/SKILL.md)

Public release artifacts and release notes are published from this repository.

## What you can do

- Sign in securely with your Podigee account
- List, create, and update podcasts
- View, update, and delete episodes
- Read episode transcripts and search what was said across a podcast
- Upload media and publish new or existing episodes
- Monitor production and encoding progress
- Review podcast and episode analytics
- Check distribution status and recommended next steps
- Use structured JSON output in scripts and agent workflows
- Run preflight checks before publishing

## Supported platforms

- macOS on Apple Silicon and Intel
- Linux on arm64 and amd64
- Windows on amd64

## Install

### Homebrew on macOS or Linux

```sh
brew tap podigee/tap
brew install --cask pdg
```

### Windows with WinGet

```powershell
winget install --id Podigee.pdg --exact
```

### Direct download

Download the latest package for your platform from
[GitHub Releases](https://github.com/podigee/pdg-agents/releases), verify its checksum, and place
`pdg` or `pdg.exe` in a directory on your `PATH`.

Every release includes SHA-256 checksums. macOS and Windows binaries are signed by Podigee.

## Get started

Sign in through your browser:

```sh
pdg auth login
```

Confirm which account and profile are active:

```sh
pdg auth status
pdg me
```

Explore your podcasts and episodes:

```sh
pdg podcasts list
pdg episodes list --podcast 123
```

Read one episode's transcript in time order, or search for relevant passages across a podcast:

```sh
pdg transcripts get --episode 456 --limit 25 --offset 0 --json
pdg transcripts search --podcast 123 --query "value-based pricing" --json
```

Retrieval segments use the JSON timestamp fields `start` and `end`. Search snippets use
`start_seconds` and `end_seconds`, and identify the matching episode.
A successful retrieval with zero segments means the episode exists but has no stored transcript;
report that result without inventing content.

Use `transcripts get` when you know the episode and need sequential transcript segments. Use
`transcripts search` when you need matching passages and timestamps across a podcast, optionally
restricted with `--episode ID`. Both commands accept `--limit` and `--offset`. Retrieval reports
`pagination.next_offset` when another page exists; search currently accepts manual offsets but does
not report reliable continuation metadata. Both commands require the `mcp:content:read` scope and
are read-only, so they do not need `--write`. Search is powered by Content Hub and may require
Content Hub plan access or organization permission.

Transcript search is keyword-based rather than semantic and does not translate queries. Use short
keywords in the podcast's language. If the user asks in another language or the first query returns
nothing, retry with a concise translated keyword or synonym before reporting no match.

Check that an episode is ready to publish:

```sh
pdg doctor --podcast 123
```

Publish an episode and wait for processing to finish:

```sh
pdg --write publish episode.mp3 --podcast 123 --title "A new episode" --now
```

Run `pdg --help` or `pdg <command> --help` to see all available commands and options.

## Use `pdg` with scripts and AI tools

Pass `--json` to receive machine-readable output:

```sh
pdg --json podcasts list
pdg --json analytics summary --podcast 123 --days 30
```

`pdg` can be used from shell scripts and from AI tools that can run local commands. Authentication
and account permissions remain controlled by Podigee, so automated workflows receive the same
access and observe the same usage limits as the signed-in user.

For changes to your Podigee account, use the global `--write` flag. This makes write intent
explicit in automated environments:

```sh
pdg --write podcasts update 123 --title "Updated title"
```

Publishing and destructive actions include confirmation and preflight safeguards where needed.

## Connect an MCP client

Podigee also provides a hosted MCP connector at `https://mcp.podigee.com/mcp`. ChatGPT, Claude,
and other compatible clients discover browser OAuth automatically. It gives agents structured
access to podcast management, publishing, imports, analytics, listener access, production status,
and distribution guidance.

The endpoint serves stateless MCP `2026-07-28` by default and accepts `2025-11-25` for legacy client
compatibility. Modern clients use per-request protocol metadata and `server/discover`, with no
initialization handshake, session ID, or server affinity.

See the [MCP server guide](docs/mcp-server.md) for connection details, authentication, tools,
examples, and safe write workflows.

## Give an agent the `podigee` skill

The [`podigee` skill](skills/podigee/SKILL.md) teaches compatible AI agents how to choose the right
command, use JSON output, publish local media safely, preserve operation IDs, and recover from
interrupted work. Its installed name is lowercase `podigee`, so supporting clients invoke it as
`$podigee` or `/podigee`.

Install the `pdg` binary first. Then install its bundled, release-matched skill for one or more
supported agents:

```sh
pdg skill install --target codex
pdg skill install --target claude
pdg skill install --target hermes
```

Use `pdg skill status --all` to inspect installations and `pdg skill update --target NAME` after a
CLI update. The installer refuses to replace modified or unmanaged files unless you pass
`--force`; forced changes create a recoverable backup first.

## Profiles and configuration

Use profiles to work with more than one Podigee account or environment:

```sh
pdg auth login --profile work
pdg auth use work
pdg auth status
```

Configuration is stored in `~/.pdg/config.yml`. Keep this file private because it can contain
authentication credentials. `pdg auth logout` removes credentials for the selected profile.

## Update or uninstall

With Homebrew:

```sh
brew upgrade --cask pdg
brew uninstall --cask pdg
```

With WinGet:

```powershell
winget upgrade --id Podigee.pdg --exact
winget uninstall --id Podigee.pdg --exact
```

For a direct installation, replace the existing binary with a newer verified release. To
uninstall, delete `pdg` or `pdg.exe` from its installation directory. Remove that directory from
your `PATH` only when it was created specifically for `pdg` and contains no other tools.

## Verify a download

Download `SHA256SUMS`, `SHA256SUMS.asc`, and `PDG_RELEASE_KEY.asc` with your release package. Each
release links to the official Podigee page containing the expected signing-key fingerprint. If
that independent fingerprint link is missing, do not use the direct download.

Inspect the downloaded key and compare its full fingerprint with the value on the linked Podigee
page. After they match, import the key and authenticate the checksum manifest:

```sh
gpg --show-keys --fingerprint PDG_RELEASE_KEY.asc
gpg --import PDG_RELEASE_KEY.asc
gpg --verify SHA256SUMS.asc SHA256SUMS
```

Only after the signature is valid, verify the archive before extracting it.

On macOS:

```sh
ARCHIVE=pdg_<version>_darwin_<arm64-or-amd64>.zip
grep " $ARCHIVE$" SHA256SUMS | shasum -a 256 -c -
```

On Linux:

```sh
ARCHIVE=pdg_<version>_linux_<arm64-or-amd64>.tar.gz
grep " $ARCHIVE$" SHA256SUMS | sha256sum -c -
```

On Windows:

```powershell
Get-FileHash .\pdg_<version>_windows_amd64.zip -Algorithm SHA256
```

Compare the Windows result with the matching entry in `SHA256SUMS`. You can also inspect the
extracted executable with `Get-AuthenticodeSignature .\pdg.exe` and confirm that the signature is
valid and issued to Podigee.

## Security and support

Never paste an access token, configuration file, or other credential into an issue. If you find a
security issue, report it privately through the security contact listed on
[podigee.com](https://www.podigee.com/).

For product help or a reproducible CLI problem, visit the
[Podigee Help Center](https://help.podigee.com/). Include your operating system, `pdg version`
output, the command you ran, and the sanitized error message. Do not include private account data.
