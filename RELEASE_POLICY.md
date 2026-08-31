# Release security

Podigee publishes one immutable set of downloads for each `pdg` version. Users can verify that a
download is authentic before running it.

Every release includes:

- SHA-256 checksums for all platform packages
- a signed checksum manifest
- a software bill of materials
- third-party dependency notices

macOS downloads are signed and notarized by Podigee. Windows downloads are Authenticode-signed by
Podigee. Podigee does not publish unsigned macOS or Windows packages, including beta versions.

Download `pdg` only from
[this repository's releases](https://github.com/podigee/pdg-agents/releases) or through a package
source linked from this repository. Follow the verification steps in the [README](README.md) before
using a direct download.
