# Security policy

## Report a vulnerability

Email **security@webfold.ai** to report a suspected vulnerability. Do not open a
public issue for an unpatched vulnerability. Include the affected version or
commit, impact, and steps to reproduce. The maintainers will acknowledge
reports as soon as practical and coordinate a fix and disclosure with the
reporter.

Once GitHub private vulnerability reporting is enabled in repository settings,
reports can also be submitted through
[GitHub Security Advisories](https://github.com/webfold-ai/youtube-transcript-mcp/security/advisories/new).

## Supported versions

Before the first stable release, security fixes are made against the latest
`main` branch. After a stable release, the latest supported release line will
be listed here.

## Security-relevant behavior

The native HTTP server binds `127.0.0.1` by default — it must be given an
explicit `--host 0.0.0.0` (or equivalent) to be reachable from outside the
local machine. It has no built-in authentication; if you do expose it beyond
localhost, put an authenticated proxy in front of it.

The server fetches attacker-influenced URLs: the `url` parameter (validated
against known YouTube domains/path shapes before any request is made) drives
outbound requests to YouTube's own endpoints. It does not proxy arbitrary
URLs — only requests it can classify as a supported YouTube video/shorts/live
link are fetched.

The optional Whisper fallback (`WHISPER_URL`) sends the video's audio stream to
whatever endpoint you configure. Only point it at a Whisper-compatible service
you trust — the server does not vet or restrict that destination.

The Cloudflare Worker deployment runs the same request-validation logic inside
Cloudflare's sandboxed Workers runtime; it has no filesystem or persistent
state of its own.

## Dependency and release security

Rust dependencies are recorded in `Cargo.lock`; CI builds against it. Pull
requests and scheduled workflows run RustSec advisory checks, cargo-deny
policy checks, SBOM generation, and OpenSSF Scorecard. Release artifacts are
built from tagged commits by the `release.yml` workflow.
