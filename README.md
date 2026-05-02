<p align="center">
  <a href="https://stallari.ai">
    <img src="assets/stallari-icon.png" width="96" alt="Stallari">
  </a>
</p>

<h1 align="center">stallari-managed-binaries</h1>

<p align="center">
  <a href="https://github.com/Groupthink-dev/stallari-managed-binaries/actions/workflows/build.yml"><img src="https://github.com/Groupthink-dev/stallari-managed-binaries/actions/workflows/build.yml/badge.svg" alt="build"></a>
  <img src="https://img.shields.io/badge/macOS-arm64-blue" alt="macOS arm64">
  <img src="https://img.shields.io/badge/runner-macos--14-lightgrey" alt="Hosted GitHub runner">
  <img src="https://img.shields.io/badge/license-MIT-green" alt="MIT (code)">
  <img src="https://img.shields.io/badge/binaries-upstream--license-yellow" alt="Binaries: upstream license">
</p>

<p align="center">
  Reproducible builds of third-party binaries supervised by the
  <a href="https://stallari.ai">Stallari</a> daemon — published as GitHub releases with SHA256 sidecars.
</p>

---

The Stallari macOS daemon supervises a set of long-running auxiliary processes — uptime monitoring, tunnel clients, ACME — and downloads each binary on demand when the user enables the corresponding add-on. Several upstream projects no longer publish prebuilt macOS binaries (TwiN/gatus, for one, dropped binary release assets ~50 versions back), and even where binaries exist, the harness benefits from pinning to a build whose provenance is auditable in this repository's CI logs.

This repo runs a single tag-triggered workflow on a hosted `macos-14` GitHub Actions runner: it builds the Go binary from a versioned upstream module with deterministic flags, computes a SHA256, and attaches both to a public GitHub release. The Stallari harness pins the published SHA256 in source — a constant compiled into its signed `.app` — and verifies the download before exec.

---

## Supported binaries

| Binary | Upstream | Purpose | Architecture | Latest |
|---|---|---|---|---|
| [`gatus`](manifests/gatus.yaml) | [TwiN/gatus](https://github.com/TwiN/gatus) | Uptime monitoring + dashboard | `darwin-arm64` | [`gatus-v5.13.0`](https://github.com/Groupthink-dev/stallari-managed-binaries/releases/tag/gatus-v5.13.0) |
| [`ntfy`](manifests/ntfy.yaml) | [binwiederhier/ntfy](https://github.com/binwiederhier/ntfy) | Self-hosted pub/sub notifications | `darwin-arm64` | `ntfy-v2.11.0` _(pending tag)_ |
| [`litestream`](manifests/litestream.yaml) | [benbjohnson/litestream](https://github.com/benbjohnson/litestream) | SQLite WAL replication for active/backup failover | `darwin-arm64` | `litestream-v0.5.11` _(pending tag)_ |
| `cloudflared` | [cloudflare/cloudflared](https://github.com/cloudflare/cloudflared) | Cloudflare Tunnel client (planned) | `darwin-arm64` | _planned_ |
| `lego` | [go-acme/lego](https://github.com/go-acme/lego) | ACME client (planned) | `darwin-arm64` | _planned_ |

Stallari runs on Apple Silicon only — manifests do not declare `darwin-amd64`.

---

## Trust model

**Binaries published here are not codesigned.** This is deliberate. The trust chain runs through the Stallari `.app`, not through the binaries themselves.

| Layer | Established by |
|---|---|
| Stallari `.app` is genuine | Developer ID `498S93MRXY` codesign + Apple notarisation, applied by the Stallari maintainer during local release builds |
| The SHA pin is genuine | Each pin lives inside `<Binary>BinaryDescriptor.swift` in [`stallari-harness`](https://github.com/Groupthink-dev/stallari-harness), compiled into the signed `.app`. Tampering with the pin invalidates the `.app` signature. |
| The download is genuine | TLS to GitHub Releases + SHA256 match against the pin. A MitM would need to forge a binary that hashes to the pinned value. |
| Gatekeeper allows first exec | Stallari's `BinaryFetcher` strips `com.apple.quarantine` after SHA verify. The fetcher's posture (signed `.app` + verified SHA + TLS) is stronger than what the xattr represents. |

Codesign on a *daemon-spawned subprocess* verified by SHA256 pin would add nothing the pin does not already provide. Putting Apple Developer ID secrets in GitHub Actions is the conventional but wrong default for this threat model — see [`docs/signing-policy.md`](docs/signing-policy.md) for the full reasoning.

This convention is documented internally as **DD-205 — Managed binary mirror convention** in the Stallari decision ledger.

### Verifying an asset locally

Every release attaches a `<binary>-<version>-<arch>.sha256` sidecar containing the hex digest. To verify a download:

```bash
curl -fLO https://github.com/Groupthink-dev/stallari-managed-binaries/releases/download/gatus-v5.13.0/gatus-5.13.0-darwin-arm64
curl -fLO https://github.com/Groupthink-dev/stallari-managed-binaries/releases/download/gatus-v5.13.0/gatus-5.13.0-darwin-arm64.sha256

shasum -a 256 -c <(echo "$(cat gatus-5.13.0-darwin-arm64.sha256)  gatus-5.13.0-darwin-arm64")
# gatus-5.13.0-darwin-arm64: OK
```

### Reproducing a build

If you want to verify the build is what we say it is, install the same Go toolchain and reproduce:

```bash
go version  # should match build.go_version in manifests/<binary>.yaml
GOOS=darwin GOARCH=arm64 CGO_ENABLED=0 \
  GOFLAGS="-trimpath -buildvcs=false" \
  GOPATH=$(mktemp -d) \
  go install github.com/TwiN/gatus/v5@v5.13.0

shasum -a 256 $GOPATH/bin/gatus
```

Within Go's well-known reproducibility caveats (with `-trimpath` and `CGO_ENABLED=0`, builds are byte-identical across machines using the same toolchain), the digest should match the sidecar.

---

## Adding a binary

1. Copy [`manifests/_template.yaml`](manifests/_template.yaml) → `manifests/<binary>.yaml`. Fill in `binary`, `version`, `upstream`, `purpose`, `build`, `architectures`.
2. Open a PR. After merge, push tag `<binary>-v<upstream-version>` (e.g. `gatus-v5.13.0`). The tag's version must match the manifest's `version:` field exactly — the workflow refuses to build otherwise, by design.
3. CI runs on hosted `macos-14`, builds with `go install -trimpath -buildvcs=false`, computes SHA256, attaches binary + `.sha256` sidecar to a public GitHub release named after the tag.
4. Pin the published SHA256 in the corresponding `<Binary>BinaryDescriptor.swift` in [`stallari-harness`](https://github.com/Groupthink-dev/stallari-harness), version-bump, ship.

### Tag → release shape

| Tag | Release | Assets |
|---|---|---|
| `gatus-v5.13.0` | [`gatus-v5.13.0`](https://github.com/Groupthink-dev/stallari-managed-binaries/releases/tag/gatus-v5.13.0) | `gatus-5.13.0-darwin-arm64` (~47 MB), `gatus-5.13.0-darwin-arm64.sha256` |

### Repository layout

```
stallari-managed-binaries/
├── manifests/
│   ├── _template.yaml      # schema reference
│   ├── gatus.yaml          # one file per supported binary
│   ├── ntfy.yaml
│   └── litestream.yaml
├── .github/
│   └── workflows/
│       └── build.yml       # tag-triggered build + release
├── docs/
│   └── signing-policy.md   # why no codesign in CI
├── README.md
├── CLAUDE.md
└── LICENSE
```

---

## Manifest schema

```yaml
binary: gatus                          # stable id; matches filename and tag prefix
version: "5.13.0"                      # upstream version, no leading 'v'

upstream:
  repository: TwiN/gatus               # for release-notes link
  module: github.com/TwiN/gatus/v5     # passed to `go install <module>@v<version>`
  license: https://github.com/TwiN/gatus/blob/v5.13.0/LICENSE.md
  homepage: https://gatus.io

purpose: |                             # appears in release notes
  Brief description of what this binary does and how Stallari uses it.

build:
  go_version: "1.26"                   # pinned per manifest for reproducibility
  cgo_enabled: false                   # true only when upstream genuinely needs CGO
  flags: "-trimpath -buildvcs=false"   # standard reproducibility flags
  package_subdir: ""                   # optional, for cmd/<name>-style modules

architectures:
  - darwin-arm64                       # Apple Silicon only
```

---

## Why not just point the Stallari harness at upstream?

Two reasons:

1. **Several upstream projects no longer publish native macOS binaries.** TwiN/gatus has shipped only Docker images and Helm charts for ~50 releases.
2. **Even when an upstream binary exists, the trust posture differs.** The harness's `BinaryFetcher` expects one signing root for everything it fetches; mirroring under our build pipeline gives downstream a single chain to verify against — the harness `.app`'s signed pin.

## Why a separate repo from `Groupthink-dev/stallari` (the facade)?

App releases happen at app-version cadence — multiple times per week. Mirrored binaries change at upstream cadence — every few weeks. Different release tracks, different build matrices (Swift app vs Go binaries), different signing-scope decisions. Splitting the concerns keeps each repo's CI tight and tag namespaces clean.

---

## Provenance

Every release's notes include:

- The exact upstream module and version pinned (`github.com/TwiN/gatus/v5@v5.13.0`)
- The CI workflow run ID + commit SHA the build was driven from
- The Go toolchain version and build flags used
- A link to the upstream project's `LICENSE` at the built version

If you ever need to ask "was this binary really built from upstream tag `v5.13.0` on commit `<sha>`?", the answer lives in the release notes plus the public Actions log.

---

## License

Code in this repository — workflows, manifests, scripts, docs — is **MIT-licensed**. See [`LICENSE`](LICENSE).

**Binary release assets carry the upstream project's license, not MIT.** Each release's notes link directly to the upstream license at the built version. We are redistributing builds of others' work; we do not relicense it.

---

## Issues, contributions, security

For bugs in the build pipeline (workflow, manifest schema, signing policy): open a PR or issue here.

For bugs in a *binary* (Gatus crashes, ntfy misbehaves, Litestream replication stalls, cloudflared loses its tunnel, lego fails): those belong upstream — file with `TwiN/gatus`, `binwiederhier/ntfy`, `benbjohnson/litestream`, `cloudflare/cloudflared`, `go-acme/lego`. We don't patch upstream code; we mirror tagged releases verbatim.

For security concerns about Stallari's overall trust model: open a [security advisory on `stallari-harness`](https://github.com/Groupthink-dev/stallari-harness/security/advisories/new).

---

*This repository is part of the [Stallari](https://stallari.ai) platform by [Groupthink](https://github.com/Groupthink-dev).*
