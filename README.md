# llmtop (Rust port)

A realtime memory, swap, SoC, and model-residency monitor for **MLX** and other local LLM workloads on Apple Silicon. Written in Rust; a self-contained binary with no Python or `uv` bootstrap.

`top` doesn't tell you whether your 70B model is actually pinned in unified memory or quietly being paged to swap. `llmtop` does.

The Python single-file version lives at [michaeldtimpe/llmtop](https://github.com/michaeldtimpe/llmtop). This port targets the same on-screen experience while reading kernel state directly (no `vm_stat`/`lsof` subprocess fan-out).

## What it shows

- **System memory** — wired / active / inactive / compressed / free, plus total RAM, derived from `host_statistics64` rather than parsing `vm_stat`.
- **Swap activity** — used vs. total, instantaneous in/out rate, and cumulative counters. Swap-out rate is highlighted red because for an LLM workload it usually means you're about to evict weights.
- **Memory pressure** — kernel pressure level (`Normal` / `Warning` / `Critical` from `kern.memorystatus_vm_pressure_level`), compressor and decompressor throughput in MB/s, file-backed pageout rate (red when non-zero — catches mmap'd weight eviction *before* swap moves), and purgeable headroom.
- **SoC** — chip identity, per-cluster CPU / GPU frequency and utilization, package and ANE power draw, CPU/GPU die temperatures (via [`macmon`](https://crates.io/crates/macmon)).
- **`iogpu.wired_limit_mb`** — the GPU wired-memory cap (`auto` or whatever you've sysctl'd).
- **Active models** — unified view of resident models with size on disk vs. resident memory and a residency `%`. See [Detection](#how-model-detection-works).
- **Matching processes** — top-N by RSS, filtered by name/cmdline patterns.

## Install

Requires Rust ≥ 1.85 and macOS on Apple Silicon.

```sh
git clone https://github.com/michaeldtimpe/llmtop -b rust
cd llmtop
cargo build --release
./target/release/llmtop
```

Optional — install to your `PATH`:

```sh
install -m 755 target/release/llmtop /usr/local/bin/llmtop
llmtop
```

## Usage

```sh
llmtop                              # live TUI, 1s refresh
llmtop -i 0.5                       # 0.5s refresh
llmtop -m ollama -m llama           # custom process filters (repeatable)
llmtop --log run.csv                # also append CSV alongside the TUI
llmtop --jsonl run.jsonl            # also append JSONL
llmtop --no-tui --log run.csv       # headless logger (great for benchmarks)
llmtop --once                       # one JSON sample to stdout, then exit
llmtop --pane models                # show only the active-models pane
llmtop --pane usage --pane memory   # show only those two panes
llmtop --theme llmtop               # pick the initial usage-bar theme
```

### Flags

| Flag | Description |
| --- | --- |
| `-i`, `--interval` | Sample interval in seconds (default `1.0`). |
| `-m`, `--match` | Substring to match against process name + cmdline. Repeatable. |
| `--pane NAME` | Show only this pane. Repeatable. Choices: `usage`, `memory`, `models`. |
| `--log PATH` | Append a CSV row per sample. |
| `--jsonl PATH` | Append a JSON object per sample (full process + model breakdown). |
| `--no-tui` | Skip the TUI; print a one-line summary per tick. |
| `--once` | Take one sample, print JSON to stdout, exit. Handy for scripting. |
| `--proc-scan-interval N` | Seconds between full process rescans (default `5.0`). Between rescans only RSS is re-polled for cached PIDs. |
| `--alert-swap-mb N` | macOS notification when swap used exceeds N MB. |
| `--alert-swap-rate N` | Notify on sustained swapout rate (pages/s). |
| `--alert-pressure` | Notify on transitions into Warning/Critical pressure. |
| `--ollama-port` | Override the ollama API port (default `11434`). |
| `--lmstudio-port` | Override the LM Studio API port (default `1234`; env `LMSTUDIO_PORT`). |
| `--omlx-port` | Override the omlx API port (default: probe `8000`, then `5741`). |
| `--theme NAME` | Initial usage-bar theme. Cycle through the rest with `c` at runtime. |

`ctrl-c` exits cleanly and flushes the log files.

## How model detection works

`llmtop` collects model entries from four sources and deduplicates by `model_id`:

1. **Local APIs** — when running, the following endpoints are queried each tick and report the canonical loaded model id and (where available) its size on disk and resident bytes:
   - **ollama** `GET /api/ps` on `127.0.0.1:11434`,
   - **omlx** `GET /v1/models/status` on `127.0.0.1:8000` (or `5741`),
   - **LM Studio** `GET /api/v0/models` (`state == "loaded"`), with fallback to `/v1/models` if v0 isn't available.

2. **Per-process FFI scan.** For each matched process, `proc_pidinfo` is called twice — once to list open file descriptors (`PROC_PIDLISTFDS` → `PROC_PIDFDVNODEPATHINFO`) and once to walk the VM region map (`PROC_PIDREGIONPATHINFO`) — so we see both currently-open weight files and mmap'd-but-closed weights. Files ≥ 50 MB matching `.gguf` / `.ggml` / `.safetensors` / `.bin` are reported with their on-disk size and resident bytes (`pri_pages_resident * page_size`).

3. **Cmdline file paths.** Args that look like a weight-file path (contains `/`, extension in `[gguf, ggml, safetensors, bin]`, stem not in `{readme, license, notice, changelog, tokenizer, config}`) are reported as `parent/stem`.

4. **Cmdline HF flags.** The cmdline is scanned for known model flags whose value is a HuggingFace-style repo id (`org/repo`). Recognized flags: `--model`, `--model-name`, `--model-id`, `--model-path`, `--repo-id`, `--hf-model`, `--hf-repo`, and the short form `-m`. Both `--flag value` and `--flag=value` are accepted. The value must contain exactly one `/`, with both halves non-empty and using only `[A-Za-z0-9._-]`; anything that looks like a flag, a local path, or a URL is rejected. This is the layer that catches MLX/`mlx-vlm`/`transformers` workloads which read safetensors into the Metal heap and then close the file descriptor — i.e. the cases where layer 2 has nothing to attribute.

Residency percent close to 100% means the model is fully paged in — anything lower (especially with active swap-outs) means parts are getting evicted.

## Output schemas

### `--once` / `--jsonl`

```json
{
  "wall_ts": "...",
  "memory": { ... },
  "soc": { ... },
  "models": [
    { "source": "cmdline",
      "model_id": "mlx-community/Qwen2.5-VL-7B-Instruct-4bit",
      "size_bytes": null,
      "resident_bytes": null,
      "process_name": "Python",
      "pid": 95462 }
  ]
}
```

`source` is one of `ollama`, `omlx`, `lmstudio`, `files`, `cmdline`.

## Why not `top` / `htop` / `asitop`?

- `top` and `htop` show RSS, but conflate the model with everything else the process is doing.
- `asitop` is great for power and frequency, but doesn't track which model weights are resident.
- `llmtop` is specifically about: *is my model in unified memory, and is it staying there?*

## Security

See [SECURITY.md](SECURITY.md) for how to report vulnerabilities.

## License

Apache License 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
