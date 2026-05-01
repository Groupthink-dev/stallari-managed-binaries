# stallari-managed-binaries

Build and distribute third-party binaries that the Stallari daemon supervises at runtime.

## Architecture

Hosted GitHub Actions runs reproducible Go builds on `macos-14`. Each binary has a YAML manifest in `manifests/<binary>.yaml` declaring upstream module, version, Go toolchain, and supported architectures. Tag push `<binary>-v<version>` triggers the build.

| Path | Purpose |
|---|---|
| `manifests/<binary>.yaml` | Per-binary build spec; `version:` must match the tag |
| `manifests/_template.yaml` | Schema reference for new binaries |
| `.github/workflows/build.yml` | Tag-triggered build + release pipeline |
| `docs/signing-policy.md` | Why no Developer ID signing in CI |

## Trust model

SHA256 pin only — no codesign, no notarisation. The harness `.app` is the trust root (signed locally on Piers' Mac during `make install-dev`); the pinned SHA256 lives inside that signed `.app`. `BinaryFetcher` strips `com.apple.quarantine` after SHA verify so daemon-spawned subprocesses don't trigger Gatekeeper dialogs.

## Convention

DD-205 — Managed binary mirror convention. Vault path: `atlas/utilities/agent-harness/decisions/DD-205.md`.

## Adding a binary

1. Copy `manifests/_template.yaml` → `manifests/<name>.yaml`, fill in
2. PR + merge
3. Push tag `<name>-v<upstream-version>`
4. Pin the published SHA256 in `<Name>BinaryDescriptor.swift` in `stallari-harness`

## Build locally (debug only)

```bash
BINARY=gatus VERSION=5.13.0 \
  go install -trimpath -buildvcs=false github.com/TwiN/gatus/v5@v$VERSION
```

This produces a binary suitable for SHA verification but is **not** what CI publishes — CI builds with `CGO_ENABLED=0` and per-arch cross-compilation. Always trust the CI artifact, not a local one.
