# LLM Setup on OpenClaw: Nvidia Free API & Local Ollama

> This document details how we configured two distinct LLM backends in OpenClaw: Nvidia's free cloud API (via Nvidia NIM) and a fully local Ollama stack. Both are viable depending on task type, available hardware, and whether you need cloud-level reasoning or pure local privacy.

---

## Overview

OpenClaw supports pluggable LLM providers through its auth-profile system. By default it ships pointing at Anthropic's API, but you can redirect it to any OpenAI-compatible endpoint. This opens the door to two setups we actively use:

| Backend | Provider Key | Endpoint | Cost |
|---|---|---|---|
| Nvidia NIM (cloud) | `nvidia` (OpenAI-compatible) | `https://integrate.api.nvidia.com/v1` | Free tier available |
| Local Ollama | `ollama` | `http://127.0.0.1:11434` | Free, hardware only |

Both connect to OpenClaw the same way — through `auth-profiles.json` — but they serve different roles in the workflow.

---

## Part 1: Nvidia Free LLM (Nvidia NIM)

### What Is Nvidia NIM?

Nvidia NIM (Nvidia Inference Microservices) is Nvidia's hosted inference platform available at [build.nvidia.com](https://build.nvidia.com). It provides free API access to a wide catalog of open-weight models — including Llama, Mistral, Qwen, DeepSeek, and others — through an OpenAI-compatible REST API. The free tier includes a generous number of credits to get started without a payment method.

### Why Use It?

- **No local hardware requirement** — runs in Nvidia's data center on high-end GPUs
- **Free tier** — enough credits to run real workloads before needing to think about cost
- **OpenAI-compatible** — the API format matches the OpenAI spec, so OpenClaw connects to it with minimal configuration
- **Access to large models** — you can run 70B+ parameter models that would be impossible to run locally on consumer hardware
- **Good for reasoning-heavy tasks** — when the local model gives a weak or uncertain answer, routing to a hosted large model is the fallback

### Getting an Nvidia NIM API Key

1. Go to [build.nvidia.com](https://build.nvidia.com)
2. Create a free account
3. Navigate to any model page (e.g., `meta/llama-3.1-70b-instruct`)
4. Click **Get API Key** — this generates a key prefixed with `nvapi-`
5. Copy the key immediately; it is only shown once

The free tier credits are tied to the account and apply across all models in the NIM catalog.

### Configuring OpenClaw for Nvidia NIM

#### Step 1: Add an Auth Profile for Nvidia

Run the auth setup command:

```powershell
openclaw models auth add
```

When prompted:

| Prompt | What to Enter |
|---|---|
| Provider id | `openai` *(NIM uses the OpenAI-compatible API format)* |
| Token method | Press **Enter** (accept default: `paste-token`) |
| Paste token | Your `nvapi-...` key |
| Does this token expire? | `No` |

#### Step 2: Manually Set the `baseURL` in the Auth Profile

Just like with Ollama, the CLI does not automatically write the endpoint URL. You must add `baseURL` manually.

Open:
```powershell
notepad C:\Users\<yourusername>\.openclaw\agents\main\agent\auth-profiles.json
```

Add or update the Nvidia profile so it looks like this:

```json
{
  "version": 1,
  "profiles": {
    "nvidia-nim:manual": {
      "type": "token",
      "provider": "openai",
      "token": "<your nvapi- key here>",
      "baseURL": "https://integrate.api.nvidia.com/v1"
    },
    "ollama:manual": {
      "type": "token",
      "provider": "ollama",
      "token": "x",
      "baseURL": "http://127.0.0.1:11434"
    }
  },
  "lastGood": {
    "openai": "nvidia-nim:manual",
    "ollama": "ollama:manual"
  },
  "usageStats": {
    "nvidia-nim:manual": {
      "lastUsed": 0,
      "errorCount": 0
    },
    "ollama:manual": {
      "lastUsed": 0,
      "errorCount": 0
    }
  }
}
```

> The key field `"baseURL": "https://integrate.api.nvidia.com/v1"` is what redirects all OpenAI-protocol calls away from OpenAI's servers and to Nvidia's NIM infrastructure.

#### Step 3: Set the Model

Pick a model from the NIM catalog. Model names must match exactly what Nvidia uses (check the model's page on build.nvidia.com for the exact string).

```powershell
openclaw models set openai/meta/llama-3.1-70b-instruct
```

Verify:
```powershell
openclaw models status
```

Expected output:
```
Model                                  Input  Ctx   Local  Auth                Tags
openai/meta/llama-3.1-70b-instruct     -      -     -      nvidia-nim:manual   default
```

### Models Available on Nvidia NIM (Examples)

| Model | Parameters | Notes |
|---|---|---|
| `meta/llama-3.1-8b-instruct` | 8B | Fast, good for quick tasks |
| `meta/llama-3.1-70b-instruct` | 70B | Strong reasoning, recommended for complex tasks |
| `mistralai/mistral-7b-instruct-v0.3` | 7B | Efficient, multilingual |
| `qwen/qwen2-7b-instruct` | 7B | Good code and instruction following |
| `deepseek-ai/deepseek-coder-6.7b-instruct` | 6.7B | Code-focused |

For the most current list, browse the full catalog at [build.nvidia.com/explore/discover](https://build.nvidia.com/explore/discover).

---

## Part 2: Local LLM via Ollama

### What Is Ollama?

[Ollama](https://ollama.com) is a local model runner that handles downloading, quantizing, and serving open-weight LLMs on your own machine. It exposes an HTTP API that is partially compatible with the OpenAI spec. OpenClaw has a native `ollama` provider that connects to it directly.

Everything runs on-device. No internet connection required after the model is pulled. No data leaves the machine.

### Prerequisites

- **Ollama installed** — download from [ollama.com](https://ollama.com)
- **Nvidia GPU with VRAM** — Ollama will use the GPU automatically for inference if drivers are installed; CPU fallback works but is significantly slower
- **OpenClaw** — version `2026.2.15` or later tested

### Hardware Considerations

Model size determines minimum VRAM. Rough estimates:

| Model Size | Min VRAM | Notes |
|---|---|---|
| 2B–4B | 3–4 GB | Runs on most modern GPUs |
| 7B–8B | 6–8 GB | Comfortable on mid-range GPUs |
| 14B | 10–12 GB | Requires higher-end GPU |
| 32B | 24+ GB | Typically multi-GPU or large VRAM workstation |

To check GPU VRAM usage during inference:
```powershell
ollama ps
```

This shows which models are loaded, which device (GPU or CPU) is handling inference, and current VRAM consumption.

### Pulling Models

Before OpenClaw can use a model, Ollama needs to have it downloaded locally:

```powershell
# Pull a specific model
ollama pull qwen3:8b

# Pull a code-specialized model
ollama pull qwen2.5-coder:14b

# List what's available locally
ollama list
```

Example output from our setup:
```
NAME                 ID              SIZE      MODIFIED
llama3.2:latest      a80c4f17acd5    2.0 GB    ✓
deepseek-r1:14b      c333b7232bdb    9.0 GB    ✓
qwen3:8b             500a1f067a9f    5.2 GB    ✓
qwen2.5-coder:14b    9ec8897f747e    9.0 GB    ✓
gpt-oss:20b          17052f91a42e    13 GB     ✓
```

### Verify the Ollama Endpoint

Before touching OpenClaw config, confirm Ollama is responding:

```powershell
# Check what models are available via API
curl http://localhost:11434/api/tags

# Quick generation test
curl -s http://localhost:11434/api/generate `
  -d '{"model": "qwen3:8b", "prompt": "Hello", "stream": false}' | jq -r '.response'
```

If the first command returns a JSON list of models, Ollama is running and accessible.

### Configuring OpenClaw for Ollama

#### Step 1: Register the Ollama Auth Profile

```powershell
openclaw models auth add
```

| Prompt | What to Enter |
|---|---|
| Provider id | `ollama` |
| Token method | Press **Enter** (accept default: `paste-token`) |
| Paste token | `x` *(Ollama has no token requirement — any value works)* |
| Does this token expire? | `No` |

#### Step 2: Add `baseURL` to the Auth Profile

The CLI does not write the endpoint. You must add it manually:

```powershell
notepad C:\Users\<yourusername>\.openclaw\agents\main\agent\auth-profiles.json
```

The critical addition is the `"baseURL"` field:

```json
{
  "version": 1,
  "profiles": {
    "ollama:manual": {
      "type": "token",
      "provider": "ollama",
      "token": "x",
      "baseURL": "http://127.0.0.1:11434"
    }
  },
  "lastGood": {
    "ollama": "ollama:manual"
  },
  "usageStats": {
    "ollama:manual": {
      "lastUsed": 0,
      "errorCount": 0
    }
  }
}
```

Without `baseURL`, OpenClaw has no idea where to send requests and the setup silently fails.

#### Step 3: Set the Active Model

Always include the `ollama/` provider prefix:

```powershell
openclaw models set ollama/qwen3:8b
```

> **Important:** Running `openclaw models set qwen3:8b` (without the prefix) causes OpenClaw to silently route to `anthropic/qwen3:8b`, which will error. Always use the full `ollama/<modelname>` format.

Confirm it's set correctly:
```powershell
openclaw models status
```

Expected:
```
Model                 Input  Ctx   Local  Auth           Tags
ollama/qwen3:8b       -      -     -      ollama:manual  default
```

If the `Auth` column shows `ollama:manual` and there is no `missing` tag, the setup is complete.

### Local Model Recommendations

Based on testing across different task types:

| Model | Size | Best For |
|---|---|---|
| `llama3.2:latest` | 2.0 GB | Fast prototyping, quick lookups |
| `qwen3:8b` | 5.2 GB | Balanced general-purpose (recommended default) |
| `qwen2.5-coder:14b` | 9.0 GB | Code generation, review, and debugging |
| `deepseek-r1:14b` | 9.0 GB | Reasoning-heavy tasks, multi-step problems |
| `gpt-oss:20b` | 13 GB | Complex tasks requiring larger context |

For most agentic OpenClaw workflows, `qwen3:8b` is the go-to. For code-specific tasks, `qwen2.5-coder:14b` is noticeably better — worth the extra VRAM.

---

## Part 3: Switching Between Backends

Both profiles can coexist in `auth-profiles.json`. Switching the active model at runtime is a single command:

```powershell
# Switch to Nvidia NIM (cloud, large model)
openclaw models set openai/meta/llama-3.1-70b-instruct

# Switch back to local Ollama
openclaw models set ollama/qwen3:8b
```

### When to Use Which

| Situation | Recommended Backend |
|---|---|
| Quick task, drafting, lookups | Local Ollama — faster, no latency |
| Sensitive data — nothing should leave the machine | Local Ollama — zero egress |
| Complex reasoning, large context | Nvidia NIM — access to 70B+ models |
| Local GPU is under load or VRAM is full | Nvidia NIM — offload to cloud |
| No internet available | Local Ollama only |
| Benchmarking or testing model quality | Nvidia NIM — consistent baseline |

---

## Part 4: Common Issues and Fixes

### "No API key found for provider 'anthropic'"
OpenClaw defaults to Anthropic. You either have not set a model yet, or a previous `openclaw doctor` run cleared your model config. Re-run `openclaw models set ollama/<model>` or `openclaw models set openai/<model>`.

### "Still seeing `anthropic/` prefix on the model"
You used `openclaw models set <model>` without a provider prefix. Always specify `ollama/` or `openai/` explicitly.

### "Model set but still errors at runtime" (Ollama)
Check three things in order:
1. Is Ollama running? (`ollama ps`)
2. Is the model pulled? (`ollama list`)
3. Is `baseURL` present in `auth-profiles.json`?

If all three are confirmed, restart OpenClaw.

### "Error: unknown option '--endpoint'"
The `--endpoint` flag does not exist in the OpenClaw CLI. The endpoint is set exclusively through `baseURL` in `auth-profiles.json`.

### "`openclaw doctor` wiped my model config"
The `doctor` command is known to clear the `models` section in `openclaw.json`. This is a quirk of the tool. Do not rely on direct edits to `openclaw.json` for model config — use `openclaw models set` via CLI and keep `auth-profiles.json` as the stable source of truth.

### Nvidia NIM: "401 Unauthorized"
The `nvapi-` key is invalid, expired, or was not set in `auth-profiles.json`. Re-paste the key directly into the `"token"` field in the profile — do not let any whitespace sneak in.

### Nvidia NIM: "429 Too Many Requests"
Free tier credits are exhausted or rate-limited. Either wait for the limit window to reset or switch to the local Ollama backend temporarily.

---

## Part 5: Reference Config

A complete `auth-profiles.json` with both backends configured:

```json
{
  "version": 1,
  "profiles": {
    "nvidia-nim:manual": {
      "type": "token",
      "provider": "openai",
      "token": "<your nvapi- key>",
      "baseURL": "https://integrate.api.nvidia.com/v1"
    },
    "ollama:manual": {
      "type": "token",
      "provider": "ollama",
      "token": "x",
      "baseURL": "http://127.0.0.1:11434"
    }
  },
  "lastGood": {
    "openai": "nvidia-nim:manual",
    "ollama": "ollama:manual"
  },
  "usageStats": {
    "nvidia-nim:manual": {
      "lastUsed": 0,
      "errorCount": 0
    },
    "ollama:manual": {
      "lastUsed": 0,
      "errorCount": 0
    }
  }
}
```

---

## Summary

- **Nvidia NIM** gives access to hosted large models (70B+) for free through a simple API key and an OpenAI-compatible endpoint — useful for reasoning-heavy tasks or when local hardware can't handle the model size.
- **Ollama** runs models entirely on local hardware (GPU-accelerated), with zero data egress and no cost beyond electricity — the default for privacy-sensitive or latency-critical work.
- Both connect to OpenClaw through the same `auth-profiles.json` mechanism, and switching between them is a one-line CLI command.
- The `baseURL` field in the auth profile is non-negotiable for both backends — it is not written by the CLI and must be added manually.

---

*Tested on: OpenClaw 2026.2.15 | Ollama | Nvidia NIM | Windows 11 | GPU-accelerated local inference*
