# Signing policy

**Binaries published from this repository are not codesigned and not notarised.** This is deliberate. The trust chain runs through the harness, not through the binaries themselves.

## Trust chain

| Layer | Established by |
|---|---|
| Stallari `.app` is genuine | Developer ID 498S93MRXY codesign + Apple notarisation, applied locally on Piers' Mac during `make install-dev`. The `.app` is signed against the same identity used for every other Stallari release. |
| The pin is genuine | The SHA256 pin for each managed binary lives in `<Binary>BinaryDescriptor.swift` (e.g. `Sources/StallariKit/AddOns/Observability/GatusBinaryDescriptor.swift`), compiled into the signed `.app`. Tampering with the pin fails Gatekeeper signature verification on the `.app`. |
| The download is genuine | TLS to `github.com` plus SHA256 match against the pin. A MitM attacker would need to forge a binary that hashes to the pinned value. |
| Gatekeeper allows first exec | `BinaryFetcher` strips `com.apple.quarantine` xattr after SHA verification. The xattr exists only because `URLSession.download` sets it on every download; the daemon's trust posture (signed harness + verified SHA + TLS) is stronger than what the xattr represents. |

## Why no codesign in CI

Putting Apple Developer ID secrets in GitHub Actions secrets is the conventional but wrong default for our threat model:

- **Blast radius.** A leaked Developer ID cert allows an attacker to sign anything as Stallari — not just managed binaries. Revoking forces invalidation of every shipped harness release.
- **GitHub Actions secrets aren't vault-grade.** Encrypted at rest, but any workflow step or compromised dependency action can exfiltrate them. Mitigations exist (environment protections, OIDC, etc.) but layer complexity for marginal gain.
- **The signature is provenance theatre here.** Codesign on a *daemon-spawned subprocess* verified by SHA256 pin adds nothing the pin does not already provide. Gatekeeper UX matters when a user double-clicks a binary in Finder; these are background subprocesses.

## When this policy would change

If a binary in this repo ever ships in a context where the user launches it directly (Finder, Dock, Spotlight), this policy needs revisiting. The right response then is **not** to put the cert in GitHub — it's a local-sign-and-publish skill: CI builds unsigned drafts, a `/publish-managed-binary` skill on Piers' Mac signs + notarises locally, then re-uploads as the published asset. Cert never leaves the Mac.

For the foreseeable future (Gatus, cloudflared, lego — all daemon-spawned subprocesses), the SHA-pin-only model is correct.
