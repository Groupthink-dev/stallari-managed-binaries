# stallari-managed-binaries

Build and distribute third-party binaries that the [Stallari](https://stallari.ai) daemon supervises at runtime.

The Stallari harness manages a set of long-running auxiliary processes (Gatus for uptime monitoring, cloudflared for service exposure, lego for ACME, future tsidp/syncthing/etc.). Several upstream projects no longer publish prebuilt macOS binary release assets, and even when they do, the harness benefits from pinning to a build whose provenance is auditable in this repository's CI logs.

This repo runs reproducible Go builds on hosted GitHub Actions runners and publishes the resulting binaries as release assets with SHA256 sidecars. The harness's `BinaryFetcher` downloads on demand when the user enables an add-on, verifies the SHA256 against a pin compiled into the signed harness `.app`, and strips the macOS quarantine xattr before exec.

## Supported binaries

| Binary | Upstream | Purpose | Architectures |
|---|---|---|---|
| [`gatus`](manifests/gatus.yaml) | [TwiN/gatus](https://github.com/TwiN/gatus) | Uptime monitoring + dashboard (DD-191) | `darwin-arm64` |
| `cloudflared` | [cloudflare/cloudflared](https://github.com/cloudflare/cloudflared) | Cloudflare Tunnel client (DD-204 Phase 1) | _planned_ |
| `lego` | [go-acme/lego](https://github.com/go-acme/lego) | ACME client for `LocalAcmeHTTPPresenter` (DD-204 Phase 8) | _planned_ |

## Trust model — SHA256 pin, no Developer ID signing

These binaries are **not codesigned**. The trust chain runs through the harness, not the binaries themselves:

| Layer | Provided by |
|---|---|
| Trust the Stallari `.app` | Developer ID 498S93MRXY + notarised, signed locally during `make install-dev` |
| Trust the SHA256 pin | The pin lives inside `<Binary>BinaryDescriptor.swift`, which is part of the signed `.app` |
| Trust the download | TLS to GitHub Releases + SHA256 match against the pin |
| Bypass Gatekeeper at first exec | `BinaryFetcher` strips `com.apple.quarantine` xattr after SHA verify |
| Provenance audit | "Built from `manifests/<binary>.yaml` v\<version\> at commit \<sha\> via this repo's GitHub Actions logs" |

Codesigning would add nothing the SHA pin does not already provide for *daemon-spawned subprocesses*. If a future binary in this repo were user-launchable (Finder double-click), revisit then with a local-sign-and-publish skill — Apple Developer ID secrets do **not** belong in GitHub Actions.

## Adding a binary

1. Copy `manifests/_template.yaml` to `manifests/<binary>.yaml` and fill in the upstream module, version, build settings, and supported architectures.
2. Open a PR. After merge, push tag `<binary>-v<upstream-version>` (e.g. `gatus-v5.13.0`). Tag must match the `version:` field in the manifest exactly — CI fails otherwise, by design.
3. CI runs on hosted `macos-14`, builds with `go install -trimpath -buildvcs=false`, computes SHA256, attaches binary + `.sha256` sidecar to a GitHub release named after the tag.
4. Update the corresponding `<Binary>BinaryDescriptor.swift` in [`stallari-harness`](https://github.com/Groupthink-dev/stallari-harness) with the release URL and pinned SHA256, version-bump, ship.

### Tag → asset shape

| Tag | Release | Assets |
|---|---|---|
| `gatus-v5.13.0` | `gatus-v5.13.0` | `gatus-5.13.0-darwin-arm64`, `gatus-5.13.0-darwin-arm64.sha256` |

**Apple Silicon only.** Stallari does not run on Intel Macs; managed-binary manifests do not declare `darwin-amd64`. Add `linux-arm64` / `linux-amd64` only when a daemon port to those platforms requires them.

### Verifying an asset locally

```bash
shasum -a 256 -c gatus-5.13.0-darwin-arm64.sha256
```

## Why this repository exists separately

`Groupthink-dev/stallari` (the facade) releases at app cadence — multiple times per week. Mirrored binaries change at upstream cadence — every few weeks. Different release tracks, different build matrices (Swift app vs Go binaries), different signing-scope decisions. Splitting the concerns keeps each repo's CI tight.

## License

Code in this repository (workflows, manifests, scripts) is MIT-licensed. **Binary release assets carry the upstream project's license**, not MIT — see each upstream's `LICENSE` file. Each release's notes link directly to the upstream license at the built version.

## Convention

The mirror, build, and distribution pipeline is codified in **DD-205 — Managed binary mirror convention** (Stallari vault — published documentation forthcoming).
