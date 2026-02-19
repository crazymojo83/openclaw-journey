# 🦞 Gateway Crashes, Model Woes, and Going Full Local (Feb 19, 2026)

> **TL;DR:** Three separate issues hit on the same day — a rogue cron job crashing the gateway via an EBADF bug, NVIDIA's Llama 3.3 70B rejecting parallel tool calls, and the gateway not recovering after crashes. All three fixed. Setup is now 100% local on Ollama with a self-healing gateway.

---

## Issue 1: Gateway Keeps Shutting Down

### Symptom
The OpenClaw gateway was repeatedly crashing and not coming back. Running `openclaw gateway status` showed:

```
Runtime: stopped (state Ready, last run time 2/19/2026 6:45:53 AM)
Service is loaded but not running (likely exited immediately).
```

### Root Cause (Two-Part)

**Part A — Rogue cron job causing timeouts:**

A cron job named `voicebox-setup` (id: `6410e30e...`) was running every 6 hours. Its payload tried to run:

```
curl -L https://voicebox.sh/download.sh | bash
voicebox config --tts-engine qwen3 --output-format wav
voicebox modulate --pitch 400 --speed 1.2
```

But the `exec` tool was blocked by OpenClaw security (`exec host not allowed`), so the embedded agent just hung. After 60 seconds the cron tool timed out with:

```
[tools] cron failed: gateway timeout after 60000ms
```

This timeout triggered a gateway self-restart via `SIGUSR1`.

**Part B — EBADF crash during Windows restart:**

When the gateway received `SIGUSR1`, it attempted a "full process restart" by spawning a new child `node.exe` and exiting the parent. On Windows, the child inherited the parent's `stdout`/`stderr` file descriptors. When the parent exited (and the scheduled task's cmd.exe shell closed), those handles became invalid. The child then immediately crashed:

```
[openclaw] Uncaught exception: Error: EBADF: bad file descriptor, write
    at writeSync (node:fs:915:3)
    at SyncWriteStream._write (node:internal/fs/sync_write_stream:27:5)
    at Writable.write (node:internal/streams/writable:508:10)
    at console.log ...
```

The cycle: **cron timeout → restart signal → EBADF crash → gateway dead → cron timeout again.**

### Fix

**1. Disable the problematic cron job directly in `~/.openclaw/cron/jobs.json`** (gateway was too dead to use the CLI):

```json
{
  "id": "6410e30e-826d-42bc-a741-d0c0e45b2992",
  "name": "voicebox-setup",
  "enabled": false,
  ...
}
```

**2. Add a restart loop to `~/.openclaw/gateway.cmd`** so the gateway auto-recovers from crashes instead of staying dead:

```batch
:loop
"C:\Program Files\nodejs\node.exe" C:\Users\steve\AppData\Roaming\npm\node_modules\openclaw\dist\index.js gateway --port 18789
set EXIT_CODE=%ERRORLEVEL%
if %EXIT_CODE% == 0 goto :eof
echo [%date% %time%] Gateway exited (code %EXIT_CODE%). Restarting in 5 seconds...
timeout /t 5 /nobreak >nul
goto loop
```

- If node exits cleanly (code 0), the loop stops.
- If node crashes for any reason, it waits 5 seconds and restarts.

> **Warning:** OpenClaw updates may regenerate `gateway.cmd` and overwrite this loop. Check it after updates.

---

## Issue 2: `400 This model only supports single tool-calls at once!`

### Symptom
After restoring the gateway, all agent interactions failed with:

```
400 This model only supports single tool-calls at once! This model only supports single tool-calls at once!
```

### Root Cause

The primary model was `nvidia/meta/llama-3.3-70b-instruct` via NVIDIA's API. Llama 3.3 70B on NVIDIA doesn't support parallel tool calls — it hard-rejects the request with a 400 when OpenClaw sends multiple tools in a single turn.

Attempted fix: adding `"params": { "parallel_tool_calls": false }` to the model config entry. This didn't work — the NVIDIA API for this model rejects the `parallel_tool_calls` parameter entirely.

### Finding: Live Model Testing

Tested all promising NVIDIA models via the API. Key results:

| Model | Tool Support | Notes |
|---|---|---|
| `meta/llama-3.3-70b-instruct` | Single only | Hard 400 error |
| `meta/llama-4-scout-17b-16e-instruct` | **Parallel ✓** | 3 simultaneous tool calls confirmed |
| `meta/llama-4-maverick-17b-128e-instruct` | Unclear | Hit token limit in test |
| `mistralai/mistral-large-2-instruct` | Unclear | Didn't call tools in test |
| `nvidia/llama-3.1-nemotron-70b-instruct` | Unclear | Didn't call tools in test |

`meta/llama-4-scout-17b-16e-instruct` was the only model that demonstrated genuine parallel tool call support — it called `get_weather` (twice, for different cities) and `web_search` in a single response.

### Fix
Switched primary model to `nvidia/meta/llama-4-scout-17b-16e-instruct` in `openclaw.json`.

---

## Issue 3 / Final Setup — Going Fully Local

### Motivation
NVIDIA's API behavior was unpredictable (policy changes, model limitations). With an RTX 5060 Ti on-hand, there was no reason to depend on cloud APIs.

### Hardware
- **GPU:** NVIDIA GeForce RTX 5060 Ti — **16 GB VRAM**
- **RAM:** 64 GB
- **CPU:** Intel Core Ultra 7 265F (20 cores)

### Model Audit

Checked all installed Ollama models for tool-calling capability:

| Model | Size | Tools | Thinking | Context | Verdict |
|---|---|---|---|---|---|
| `gpt-oss:20b` | 13 GB | ✓ | ✓ | 131K | **Best — use as primary** |
| `qwen3:8b` | 5.2 GB | ✓ | ✓ | 41K | Good fallback |
| `qwen2.5-coder:14b` | 9.0 GB | ✓ | ✗ | 32K | Coding tasks |
| `deepseek-r1:14b` | 9.0 GB | **✗** | ✓ | 131K | **Removed — no tools** |
| `llama3.2:latest` | 2.0 GB | ? | ✗ | — | Too small |

`deepseek-r1:14b` was pulled for reasoning but has **no tool support** — useless for OpenClaw. Removed.

### New Model Pulled

**`qwen3:14b`** (9.3 GB, Q4_K_M) — significant upgrade over `qwen3:8b`, tools + thinking, fits comfortably in 16 GB VRAM.

```
ollama pull qwen3:14b
ollama rm deepseek-r1:14b
```

### Final `openclaw.json` Model Config

```json
"agents": {
  "defaults": {
    "model": {
      "primary": "ollama/gpt-oss:20b",
      "fallbacks": ["ollama/qwen3:14b", "ollama/qwen3:8b"]
    },
    "models": {
      "ollama/gpt-oss:20b": {},
      "ollama/qwen3:14b": {},
      "ollama/qwen3:8b": {},
      "ollama/qwen2.5-coder:14b": {}
    }
  }
}
```

**Primary:** `gpt-oss:20b` — 20.9B parameters, MXFP4 quantization, tools + thinking, 131K context. Fits in 16 GB VRAM at 13 GB.
**Fallback chain:** `qwen3:14b` → `qwen3:8b`
**Available for routing:** `qwen2.5-coder:14b` for code-heavy tasks.

---

## Summary of All Changes Made

| File | Change |
|---|---|
| `~/.openclaw/cron/jobs.json` | Disabled `voicebox-setup` cron job (`enabled: false`) |
| `~/.openclaw/gateway.cmd` | Added restart loop — auto-recovers from crashes |
| `~/.openclaw/openclaw.json` | Switched primary model to `ollama/gpt-oss:20b`, added fallbacks, removed NVIDIA as primary |

---

## Lessons Learned

- **Check cron jobs if the gateway keeps crashing.** An agent-created cron job with a hanging task is enough to trigger the restart loop and expose the Windows EBADF bug.
- **`deepseek-r1` has no tool support.** It's a reasoning/thinking model only. Don't use it as a primary model in OpenClaw.
- **NVIDIA's hosted models have inconsistent parallel tool call support.** If you must use NVIDIA, test models explicitly before committing.
- **The Windows EBADF crash is a known OpenClaw bug** on Windows with scheduled-task managed gateways. The restart loop in `gateway.cmd` is a practical workaround until it's fixed upstream.
- **16 GB VRAM is plenty for great local agentic AI.** `gpt-oss:20b` at 13 GB fits with headroom, gives tools + thinking + 131K context.
