<div align="center">
  <img src="assets/logo.png" alt="YouTube Transcript MCP Logo" width="200"/>

  # YouTube Transcript MCP Server

  Retrieve YouTube transcripts through MCP, HTTP or a CLI. Run the Rust server
  locally or deploy the Cloudflare Worker target. Select a language and return
  text, JSON, Markdown, SRT or VTT. The native server can optionally transcribe
  audio when captions are unavailable; that fallback requires an external
  transcription service.
</div>

![YouTube Transcript MCP artwork](assets/social-card.png)

## Features

- **Single tool**: `get_transcript(url, language?, format?)` — works with any
  YouTube URL format.
- **Three ways to call it**:
  - **MCP** over stdio (native), or stateless JSON-RPC over HTTP / SSE
    (native + Worker; `initialize`, `tools/list` and `tools/call` only)
  - **Raw HTTP**: `GET /transcript?url=…&language=…` returns plain text — curl-friendly
  - **CLI one-shot**: `youtube-transcript-mcp --url <URL>` prints to stdout and exits
- **Native binary**: stdio transport for Claude Desktop / local clients, plus
  an HTTP server with `/transcript`, `/mcp`, and `/sse`.
- **WASM Worker**: same Rust code, deployed to Cloudflare Workers. Exposes the
  same `/transcript`, `/mcp`, `/sse` endpoints.
- **Transcription fallback** (native only, optional): when a video has no
  caption tracks and `WHISPER_URL` is set, the server downloads the audio
  stream and sends it to an external
  [faster-whisper-server](https://github.com/fedirz/faster-whisper-server)
  compatible endpoint. Not available in the Worker.
- **URL flexibility**: `watch?v=`, `youtu.be/`, `/shorts/`, `/live/`, `/embed/`,
  international domains (`youtube.co.uk`, `youtube.de`, …), with or without scheme.
- **Language selection**: BCP-47 codes (`en`, `es`, `fr`, …) or `auto` (default).
  Falls back to English with an explanatory note when the requested language
  is unavailable.
- **Output formats**: `text` (default), `json`, `srt`, `vtt`, or `markdown`.
  `json` and `markdown` embed a clickable deep link (`watch?v=…&t=<secs>s`) for
  every cue, so you can open the file and jump straight to that moment in the
  video; `srt`/`vtt` are standard subtitle files.

## Repository layout

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the request path, state,
error handling and the native vs Worker differences.

```
crates/
  core/      # pure transcript logic (URL parsing, Innertube/XML decoding, language selection)
  native/    # rmcp-based binary (stdio + HTTP/SSE)
  worker/    # Cloudflare Worker (compiled to wasm32-unknown-unknown via worker-build)
```

`crates/core` is I/O-free and is consumed by both the native binary and the
Worker. The Worker crate declares its own `[workspace]` so wrangler can build
it for `wasm32-unknown-unknown` without affecting the host workspace.

## Benchmarks

![p50 latency, 80 ms simulated RTT](bench/results/chart-latency-prod.svg)

![Latency vs memory](bench/results/chart-latency-vs-memory.svg)

| MCP server | p50, 80 ms simulated RTT | Cold start | Peak RSS |
|---|---:|---:|---:|
| **this repo (Rust)** | **164 ms** | **171 ms** | **7.7 MiB** |
| nabid-pf (Node, direct Innertube) | 169 ms | 430 ms | 114 MiB |
| spinalshock (Go + yt-dlp), rate-limit removed | 503 ms | 511 ms | 14 MiB |
| anaisbetts (Node + yt-dlp) | 506 ms | 638 ms | 81 MiB |
| spinalshock (Go + yt-dlp), as shipped | 2411 ms | 2863 ms | 13 MiB |

Measured against a local YouTube mock (80 KiB caption track, 80 ms added per
response), with the same MCP client for every server and competitors at pinned
versions. The yt-dlp-based servers ran against a `yt-dlp` stub that sleeps for
a measured start-up cost, so their rows are simulated, not real yt-dlp against
real YouTube. Under this setup this repo is about 3× faster than the tested
yt-dlp-based servers, has similar latency to the direct-Innertube Node server
(nabid-pf), and used the least memory of the servers tested. This is not a
real-YouTube latency claim. Setup, loopback numbers, parser numbers, what isn't
claimed, and how to reproduce: [BENCHMARKS.md](BENCHMARKS.md).

## Self-hosting (native binary)

### From source

```bash
git clone https://github.com/webfold-ai/youtube-transcript-mcp
cd youtube-transcript-mcp
cargo build --release -p youtube-transcript-mcp
# binary at target/release/youtube-transcript-mcp
```

Quick check without network access to YouTube: see
[Smoke test](#smoke-test).

### Run

```bash
# One-shot CLI — print transcript to stdout and exit
./target/release/youtube-transcript-mcp --url 'https://youtu.be/dQw4w9WgXcQ'
./target/release/youtube-transcript-mcp --url '…' --language es
# Markdown with clickable timestamp links (also: json, srt, vtt)
./target/release/youtube-transcript-mcp --url '…' --format markdown

# stdio MCP (Claude Desktop, local MCP clients)
./target/release/youtube-transcript-mcp --stdio

# HTTP server (raw API + MCP) — defaults to 127.0.0.1:3000
./target/release/youtube-transcript-mcp
./target/release/youtube-transcript-mcp --host 0.0.0.0 --port 8080
```

HTTP endpoints:

| Method | Path                                     | Purpose                                                   |
|--------|------------------------------------------|-----------------------------------------------------------|
| GET    | `/`                                      | Server info JSON                                          |
| GET    | `/transcript?url=…&language=…&format=…`  | Transcript in the requested `format` (defaults to `text`) |
| POST   | `/mcp`                                   | Stateless MCP JSON-RPC over HTTP (single request/response) |
| GET    | `/sse`                                   | SSE handshake                                             |
| POST   | `/sse`                                   | MCP JSON-RPC over SSE (single-shot)                       |

```bash
curl 'http://127.0.0.1:3000/transcript?url=https://youtu.be/dQw4w9WgXcQ&language=en'
# Markdown with clickable timestamp links
curl 'http://127.0.0.1:3000/transcript?url=https://youtu.be/dQw4w9WgXcQ&format=markdown'
```

### Claude Desktop (stdio)

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "youtube-transcript": {
      "command": "/absolute/path/to/youtube-transcript-mcp",
      "args": ["--stdio"]
    }
  }
}
```

### Transcription fallback (optional, external service required)

When a video has no caption tracks, the native server can fall back to ASR,
but only if `WHISPER_URL` points at a transcription service you run. The
server posts to `<WHISPER_URL>/v1/audio/transcriptions` (OpenAI-style, with
model `small`), for example
[faster-whisper-server](https://github.com/fedirz/faster-whisper-server):

```bash
WHISPER_URL=http://localhost:8000 ./target/release/youtube-transcript-mcp --stdio
```

The transcript is prefixed with `[AI-generated transcript — no captions
available]`. ASR output has no per-cue timestamps, so this path always returns
plain text regardless of the requested `format`.

## Cloudflare Workers (WASM)

The Worker uses the [`worker`](https://crates.io/crates/worker) crate and is
compiled to WebAssembly by [`worker-build`](https://crates.io/crates/worker-build)
during `wrangler deploy`.

### Prerequisites

- Rust toolchain with `wasm32-unknown-unknown`:
  `rustup target add wasm32-unknown-unknown`
- [`wrangler`](https://developers.cloudflare.com/workers/wrangler/install-and-update/)
- A Cloudflare account

### Deploy

```bash
cd crates/worker
wrangler deploy
```

`wrangler dev` runs the Worker locally on `http://127.0.0.1:8787`.

The Worker exposes the same endpoints as the native HTTP server:

- `GET /` — server info JSON
- `GET /transcript?url=…&language=…&format=…` — transcript in the requested `format`
- `POST /mcp` — stateless MCP JSON-RPC over HTTP
- `GET /sse` / `POST /sse` — MCP JSON-RPC over Server-Sent Events

The Worker has no transcription fallback, and it also accepts `lang` and `fmt`
as aliases for `language` and `format`.

```bash
curl 'https://your-worker.workers.dev/transcript?url=https://youtu.be/dQw4w9WgXcQ'
```

The `[build]` section of `crates/worker/wrangler.toml` runs
`cargo install -q worker-build && worker-build --release` automatically.

## API reference

### `GET /transcript` (raw)

| Param      | In    | Required | Description                                              |
|------------|-------|----------|----------------------------------------------------------|
| `url`      | query | yes      | YouTube video URL in any supported format.               |
| `language` | query | no       | BCP-47 code (`en`, `es`, …). Defaults to `auto`.         |
| `format`   | query | no       | `text` (default), `json`, `srt`, `vtt`, or `markdown`.   |

Responds with the body in the requested `format` (`Content-Type` is set
accordingly: `text/plain`, `application/json`, `text/vtt`, or `text/markdown`).
`json` and `markdown` include a `watch?v=…&t=<secs>s` deep link per cue. Status
codes: `200`, `400` (invalid URL), `404` (no transcript / unavailable language),
`502` (network), `500` (parse).

### MCP tool `get_transcript`

Same parameters (`url`, `language`, `format`), returned as MCP tool content
(`{"content":[{"type":"text", …}]}`).

**Direct MCP call against the Worker / HTTP server:**

```bash
curl -X POST https://your-worker.workers.dev/mcp \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "get_transcript",
      "arguments": {
        "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
        "language": "en"
      }
    }
  }'
```

### Supported URL formats

- `https://www.youtube.com/watch?v=VIDEO_ID`
- `https://youtu.be/VIDEO_ID`
- `https://m.youtube.com/watch?v=VIDEO_ID`
- `https://www.youtube.com/shorts/VIDEO_ID`
- `https://www.youtube.com/live/VIDEO_ID`
- `https://www.youtube.com/embed/VIDEO_ID`
- International TLDs (`youtube.co.uk`, `youtube.de`, …)

Tracking parameters (`si`, `t`, …) are ignored.

## Smoke test

The repo ships a mock YouTube server (`bench/mock/server.py`) and the native
binary can be pointed at it with `YTMCP_INNERTUBE_URL`, so no real YouTube
request is needed:

```bash
python3 bench/mock/server.py 18080 &     # serves the fixtures in bench/canned/
YTMCP_INNERTUBE_URL=http://127.0.0.1:18080/youtubei/v1/player \
  ./target/release/youtube-transcript-mcp \
  --url 'https://youtu.be/dQw4w9WgXcQ' --format markdown
```

Expected output shape: one Markdown list item per cue, each with a clock
label linking to `watch?v=<id>&t=<secs>s` (the text is the fixture's filler
words, not a real video):

```
- [0:00](https://www.youtube.com/watch?v=dQw4w9WgXcQ&t=0s) the dog lazy lazy jumps
- [0:02](https://www.youtube.com/watch?v=dQw4w9WgXcQ&t=2s) brown while forest quick the brown the lazy bright singing the moon
...
```

An invalid URL (for example `--url https://example.com/x`) exits with status 1
and `Invalid YouTube URL` on stderr. Stop the mock with `kill %1`.

## Development

```bash
cargo fmt --check
cargo clippy --workspace -- -D warnings
cargo test --workspace          # core unit tests + native pipeline tests (wiremock)

# Parser microbenchmark (Criterion) — see BENCHMARKS.md
cargo bench -p youtube-transcript-mcp-core --bench parser

# Worker (separate workspace, wasm32 target)
cargo check -p youtube-transcript-mcp-worker \
  --manifest-path crates/worker/Cargo.toml \
  --target wasm32-unknown-unknown
```

`cargo test --workspace` includes wiremock-backed tests in `crates/native` that
exercise the full Innertube → caption-XML → parse path against a local mock
server — including the assertion that both upstream requests carry the
`User-Agent` header (a missing header there once produced silent empty
transcripts).

CI runs `fmt`, `clippy`, and `test` on every push. Pre-built native binaries
for Linux and macOS are attached to GitHub Releases when a `v*` tag is pushed.

## License

[MIT](LICENSE)
