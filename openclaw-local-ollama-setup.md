# 🦞 Running OpenClaw with Local Ollama LLMs (No API Key Required)

> **TL;DR:** OpenClaw defaults to Anthropic's API. You can ditch the cloud dependency and run fully local models using Ollama — but there's a non-obvious config step that will bite you if you skip it. This post documents exactly what we had to do to get it working.

---

## Why Go Local?

If you're already running an Ollama stack on a machine with a decent GPU, paying for a cloud LLM API just to power your agentic tooling feels a little backwards. This setup lets OpenClaw talk directly to your local model instance — zero cloud calls, zero API keys, full control.

We got this working on Windows with OpenClaw `2026.2.15` and Ollama running on `127.0.0.1:11434`.

---

## Prerequisites

### What You'll Need
- **OpenClaw** installed (version `2026.2.15` tested)
- **Ollama** running with at least one model pulled
- Basic familiarity with editing JSON files and running PowerShell

### Verify Ollama Is Up and Running

```powershell
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

Also confirm the endpoint is accessible:
```powershell
curl http://localhost:11434/api/tags
```

---

## The Problem OpenClaw Ships With

Out of the box, OpenClaw will try to route everything through Anthropic's API. The moment you try to run it without a valid API key, you'll see:

```
No API key found for provider "anthropic"
```

Even after setting the provider to `ollama` in `openclaw.json`, OpenClaw may silently prefix your model with `anthropic/` — so `qwen3:8b` becomes `anthropic/qwen3:8b` and fails. Not ideal. Here's the actual fix.

---

## Step-by-Step Configuration

### Step 1: Run the Ollama Auth Setup

OpenClaw has its own auth profile system. Start by adding a profile for Ollama:

```powershell
openclaw models auth add
```

When prompted:

| Prompt | What to Enter |
|---|---|
| Provider id | `ollama` |
| Token method | Press **Enter** (accept default: `paste-token`) |
| Paste token | `x` *(Ollama doesn't use real tokens — just put anything)* |
| Does this token expire? | `No` |

This creates a profile called `ollama:manual` in OpenClaw's config.

---

### Step 2: The Critical Fix — Add `baseURL` to the Auth Profile

Here's the part that isn't obvious: the auth profile created above is **missing the endpoint URL**. Without it, OpenClaw has no idea where to send requests.

Open the file:
```powershell
notepad C:\Users\<yourusername>\.openclaw\agents\main\agent\auth-profiles.json
```

Replace the contents with this:

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

The magic line is `"baseURL": "http://127.0.0.1:11434"`. Without it, nothing works.

> ⚠️ **Note:** If `auth-profiles.json` doesn't exist yet, you'll need to create it. The directory path is:
> `C:\Users\<yourusername>\.openclaw\agents\main\agent\`

---

### Step 3: Set Your Default Model

Now point OpenClaw at the right model with the correct provider prefix:

```powershell
openclaw models set ollama/qwen3:8b
```

> **Watch the prefix!** If you run `openclaw models set qwen3:8b` without `ollama/`, it may default to `anthropic/qwen3:8b` — which defeats the whole purpose.

Verify:
```powershell
openclaw models status
```

Expected output:
```
Model                 Input  Ctx   Local  Auth           Tags
ollama/qwen3:8b       -      -     -      ollama:manual  default
```

If the Auth column shows `ollama:manual` and there's no `missing` tag — you're done. 🎉

---

## Model Recommendations

Here's how the models in our setup stack up for different use cases:

| Model | Size | Best For |
|---|---|---|
| `llama3.2:latest` | 2.0 GB | Fast prototyping, quick tasks |
| `qwen3:8b` | 5.2 GB | Balanced general purpose (our default) |
| `qwen2.5-coder:14b` | 9.0 GB | Code-heavy tasks — highly recommended |
| `deepseek-r1:14b` | 9.0 GB | Reasoning-heavy tasks |
| `gpt-oss:20b` | 13 GB | When you want the big brain |

For most agentic workflows, `qwen3:8b` hits the sweet spot. If you're using OpenClaw for code generation and review, swap in `qwen2.5-coder:14b` — it's noticeably better on structured coding tasks.

---

## Troubleshooting

**"Config gets overwritten after running `openclaw doctor`"**
This is a known quirk. The `doctor` command and some other internal operations may clear the `models` section in `openclaw.json`. Rely on the CLI (`openclaw models set`) and the auth profile file — those are more stable.

**"Still seeing `anthropic/` prefix on the model"**
Always use the full `ollama/modelname` syntax with the set command. OpenClaw defaults to Anthropic if no provider is specified.

**"Model appears set but still errors at runtime"**
Check three things: Ollama is running (`ollama ps`), the model is loaded (`ollama list`), and `baseURL` is in your `auth-profiles.json`. If all three are good, restart OpenClaw.

**"Error: unknown option '--endpoint'"**
The `--endpoint` flag doesn't exist in the current CLI. The endpoint has to be set directly in `auth-profiles.json` as `baseURL`. This is what tripped us up early on.

---

## Full Working `auth-profiles.json`

For reference, here's our complete working auth profile:

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

---

## Final Thoughts

Once it's working, it's solid. The combination of OpenClaw's agentic tooling with a locally-hosted model is genuinely powerful — and the privacy story is hard to beat. No data leaving your network, no API costs, and you can hot-swap models as needed.

If you're running VRAM-heavy models (14B+), make sure your GPU is handling inference — you can check with `ollama ps` to see what's loaded and whether it's on GPU or CPU. The performance difference is significant.

Happy to answer questions in the comments — this repo is tracking our full OpenClaw journey as we push it further.

---

*Tested on: OpenClaw 2026.2.15 | Ollama | Windows 11 | GPU-accelerated inference*
