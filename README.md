# llmtop

A realtime memory, swap, and model-residency monitor for **MLX** and other local LLM workloads on Apple Silicon.

`top` doesn't tell you whether your 70B model is actually pinned in unified memory or quietly being paged to swap. `llmtop` does.

## What it shows

- **System memory** — wired / active / inactive / compressed / free, plus total RAM.
- **Swap activity** — used vs. total, instantaneous in/out rate, and cumulative counters. Swap-out rate is highlighted red because for an LLM workload it usually means you're about to evict weights.
- **Memory pressure** — kernel pressure level (`Normal` / `Warning` / `Critical` from `kern.memorystatus_vm_pressure_level`), compressor and decompressor throughput in MB/s, file-backed pageout rate (red when non-zero — catches mmap'd weight eviction *before* swap moves), and purgeable headroom.
- **`iogpu.wired_limit_mb`** — the GPU wired-memory cap (`auto` or whatever you've sysctl'd).
- **Active models** — unified view of resident models, with size on disk vs. resident memory and a residency `%`. Pulled from:
  - the **ollama** local API (`/api/ps` on `127.0.0.1:11434`),
  - the **omlx** local API (`/v1/models/status`, using the API key in `~/.omlx/settings.json`),
  - **per-process open file descriptors** for `.safetensors` / `.gguf` / `.mlx` weights (authoritative — works for `mlx_lm`, `vllm`, `llama.cpp`, LM Studio, etc.),
  - process command-line as a fallback.
- **Matching processes** — top-N by RSS, filtered by name/cmdline patterns.

## Install

`llmtop` is a single-file [PEP 723](https://peps.python.org/pep-0723/) script. With [`uv`](https://github.com/astral-sh/uv) installed, it bootstraps its own dependencies on first run — no venv to manage.

```sh
chmod +x llmtop
./llmtop
```

Optional — drop it on your `PATH`:

```sh
install -m 755 llmtop /usr/local/bin/llmtop
llmtop
```

Requires Python ≥ 3.11 and macOS (relies on `vm_stat` and `sysctl`). Built and tested on Apple Silicon.

## Usage

```sh
./llmtop                            # live TUI, default filters, 1s refresh
./llmtop -i 0.5                     # 0.5s refresh
./llmtop -m ollama -m llama         # custom process filters (repeatable)
./llmtop --log run.csv              # also append CSV alongside the TUI
./llmtop --jsonl run.jsonl          # also append JSONL
./llmtop --no-tui --log run.csv     # headless logger (great for benchmarks)
./llmtop --pane procs               # show only the matching-processes pane
./llmtop --pane system --pane models  # show only those two panes
```

### Flags

| Flag | Description |
| --- | --- |
| `-i`, `--interval` | Sample interval in seconds (default `1.0`). |
| `-m`, `--match` | Substring to match against process name + cmdline. Repeatable. Defaults: `python, mlx, omlx, ollama, llama, lm-studio, lmstudio, vllm`. |
| `--log PATH` | Append a CSV row per sample (one column per metric). |
| `--jsonl PATH` | Append a JSON object per sample (full process + model breakdown). |
| `--no-tui` | Skip the TUI; print a one-line summary per tick. Use with `--log`/`--jsonl` for unattended runs. |
| `--pane NAME` | Show only this pane. Repeatable. Choices: `system`, `pressure`, `models`, `procs`. Defaults to all four. |

`ctrl-c` exits cleanly and flushes the log files.

## Output schemas

### CSV (`--log`)

`ts, wired_mb, active_mb, compressed_mb, free_mb, swap_used_mb, swap_total_mb, swapins_total, swapouts_total, swapin_rate, swapout_rate, top_pid, top_name, top_rss_mb`

### JSONL (`--jsonl`)

One object per sample, including the full top-8 process list and every detected model with size + resident bytes. Suitable for piping into `jq`, DuckDB, or a notebook.

## How model detection works

For each matched process, `llmtop` inspects `/proc`-style open file descriptors via `psutil` and groups any open `.safetensors` / `.gguf` / `.mlx` files (≥ 50 MB) by a derived **model id**:

- HuggingFace cache layout (`models--org--repo/snapshots/<hash>/<file>`) → `org/repo`.
- Single-file ggufs → filename.
- Otherwise → containing directory name.

Resident bytes are the process RSS; size on disk is the sum of weight files. Residency percent close to 100% means the model is fully paged in — anything lower (especially with active swap-outs) means parts are getting evicted.

When ollama or omlx are running, their local APIs supply authoritative numbers and override the heuristic for those processes.

## Why not `top`/`htop`/`asitop`?

- `top` and `htop` show RSS, but conflate the model with everything else the process is doing.
- `asitop` is great for power and frequency, but doesn't track which model weights are resident.
- `llmtop` is specifically about: *is my model in unified memory, and is it staying there?*

## Security

See [SECURITY.md](SECURITY.md) for how to report vulnerabilities.

## License

Apache License 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
