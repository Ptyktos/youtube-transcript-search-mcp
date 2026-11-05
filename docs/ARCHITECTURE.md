# Architecture

This page describes what the code does today. Every statement below was
checked against the source in `crates/`.

## Inputs and outputs

One operation: **get the transcript for a YouTube video.**

| Input | Type | Notes |
|---|---|---|
| `url` | string, required | `watch?v=`, `youtu.be/`, `/shorts/`, `/live/`, `/embed/`; scheme optional; a fixed list of YouTube hosts (`crates/core/src/url.rs`). The 11-character ID is validated. |
| `language` | string, optional | `auto` (default, empty also means auto) or a language code, lower-cased. |
| `format` | string, optional | `text` (default), `json`, `srt`, `vtt`, `markdown` (`md` and `webvtt` are accepted aliases). Unknown values silently fall back to `text`. |

Output is one string:

- `text`: cues joined by single spaces.
- `markdown`: one `- [M:SS](watch?v=<id>&t=<secs>s) <text>` line per cue.
- `json`: `{ video_id, language, note, segments: [{ start, dur, text, url }] }`.
- `srt` / `vtt`: standard subtitle files. A language-fallback `note` appears
  in text, markdown, json and vtt, but not srt (no comment syntax).

Entry points (all call the same fetch logic):

| Surface | Native | Worker |
|---|---|---|
| CLI `--url` (prints to stdout, exits) | yes | no |
| MCP over stdio (`--stdio`, built on `rmcp`) | yes | no |
| `GET /transcript` | yes | yes |
| `POST /mcp` (stateless JSON-RPC) | yes | yes |
| `GET` / `POST /sse` (single-shot SSE) | yes | yes |
| `GET /` (info JSON) | yes | yes |

## Main path

```
url ──> extract_video_id ──> POST Innertube /youtubei/v1/player (Android client)
    ──> parse_innertube_response ──> caption tracks (+ audio streams, native only)
    ──> select_track(language)   ──> GET caption XML (track baseUrl)
    ──> parse_transcript_segments ──> format_transcript(format) ──> string
```

1. `extract_video_id` validates the URL and video ID (`InvalidUrl` / `InvalidVideoId`).
2. One `POST` to `INNERTUBE_URL` with a fixed Android client context and the
   Android `User-Agent` (a missing UA once produced empty transcripts; a
   wiremock test guards it).
3. The response is decoded; `playabilityStatus` other than `OK` is a
   `Parse` error. Caption tracks and audio-only streams are collected.
4. `select_track`: `auto` tries a fixed order (`en, es, fr, de, tr, pt, ja,
   ko, zh, it, ru, ar`), then the first track. A specific language that is
   missing falls back to English with a note, otherwise
   `LanguageNotAvailable`.
5. One `GET` of the track's `baseUrl`, parsed with `quick-xml` (both the
   `<text start dur>` and `<p t d>/<s>` shapes), then rendered.

Total: two upstream requests per transcript.

If there are no caption tracks, the native build can try the transcription
fallback (see below); otherwise the result is `NoTranscriptAvailable`.

## Key crates

| Crate | Role |
|---|---|
| `crates/core` (`youtube-transcript-mcp-core`) | No I/O. URL parsing, Innertube body/response decoding, track selection, XML parsing, output formats, `TranscriptError`. Deps: `thiserror`, `serde`, `serde_json`, `quick-xml`, `url`. |
| `crates/native` (`youtube-transcript-mcp`) | Binary. `clap` CLI, `reqwest` (rustls) fetch, `rmcp` stdio server, `axum` + `tower-http` HTTP server, `tokio`, `tracing`. |
| `crates/worker` (`youtube-transcript-mcp-worker`) | Cloudflare Worker (`worker` crate, `cdylib`). Own `[workspace]`, built for `wasm32-unknown-unknown` by `worker-build` via `wrangler`; not part of `cargo build --workspace`. |

The `bench/` directory holds the benchmark harness (see `BENCHMARKS.md`); it is
not part of the shipped binary.

## State and caching

There is none to speak of:

- No transcript cache, no database, no files written. Every call performs both
  upstream requests.
- Native: one shared `reqwest::Client` (connection pooling only) is created at
  start-up and cloned into the stdio server or HTTP state.
- Worker: stateless per request; no bindings or storage are used.
- Configuration is read from the environment on each call or at start-up (see
  below).

## Error and resource boundaries

- `TranscriptError` variants map to HTTP status on `GET /transcript` (native
  and Worker): `InvalidUrl`/`InvalidVideoId` 400, `NoTranscriptAvailable` and
  `LanguageNotAvailable` 404, `Network` 502, `Parse` 500 (this includes
  videos whose `playabilityStatus` is not OK). The error text is the body.
- `POST /mcp` and `/sse` return JSON-RPC errors: -32602 for invalid URL/ID,
  -32601 for unknown method or tool, -1 for everything else. Failures are not
  reported as MCP `isError` results on this path.
- Over stdio (`rmcp`), invalid URL/ID is an invalid-params error and the rest
  are internal errors.
- Native client timeouts: 30 s total, 10 s connect. The Worker relies on
  Cloudflare's own limits.
- Not limited by this code: response body sizes, request rate, concurrency.
  The Whisper path reads the whole audio stream into memory.
- Security-relevant: the HTTP server binds `127.0.0.1:3000` by default, has no
  authentication and allows any CORS origin; the Worker also allows `*`. Only
  recognised YouTube hosts are fetched for the video URL. See `SECURITY.md`.
- In `--stdio` mode logs go to stderr (stdout is the protocol channel); the
  CLI mode writes only the transcript to stdout.
- The `POST /mcp` and `/sse` dispatchers are small hand-written JSON-RPC
  handlers (`initialize`, `tools/list`, `tools/call`; protocol version
  `2024-11-05`), duplicated in `crates/native/src/http.rs` and
  `crates/worker/src/lib.rs`. They do not implement notifications, sessions
  or streaming responses. Only stdio uses `rmcp`.

## Dependencies and optional features

There are no Cargo feature flags. Optional behaviour is runtime configuration
(native only):

| Variable / flag | Effect |
|---|---|
| `WHISPER_URL` | If set and a video has no caption tracks, download the best audio stream (mp4 preferred, then highest bitrate) and `POST` it as multipart to `<WHISPER_URL>/v1/audio/transcriptions` (model `small`, `response_format=text`). The result is prefixed with `[AI-generated transcript — no captions available]`, is plain text whatever `format` was asked for, and is reported as language `en`. Needs an external service you operate; audio is sent to it. |
| `HOST`, `PORT` (or `--host`, `--port`) | HTTP bind address; default `127.0.0.1:3000`. |
| `RUST_LOG` | `tracing` filter; default `youtube_transcript_mcp=info,rmcp=warn`. |
| `YTMCP_INNERTUBE_URL` | Redirects the Innertube POST (used by `bench/` and for the smoke test in the README). Leave unset in production. |

Dependency versions are declared in each crate's `Cargo.toml` and locked in
`Cargo.lock` (the Worker has its own `crates/worker/Cargo.lock`).

## Native vs Worker

| | Native | Worker |
|---|---|---|
| Build | `cargo build --release -p youtube-transcript-mcp` | `wrangler deploy` in `crates/worker` (`worker-build`, `wasm32-unknown-unknown`) |
| Transports | CLI, stdio (`rmcp`), HTTP/SSE | HTTP/SSE only |
| HTTP stack | `axum` + `reqwest` | `worker` `Fetch` API |
| Transcription fallback | optional via `WHISPER_URL` | none (audio streams are ignored) |
| Query aliases | `language`, `format` | also `lang`, `fmt` |
| Timeouts | 30 s / 10 s connect | platform-defined |
| Shared code | `crates/core` | `crates/core` |
