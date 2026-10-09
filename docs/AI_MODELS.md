# EAGLE — AI Model Plan

The ways EAGLE runs AI models, where the data goes with each one, how each is connected, which
model does which job, and how to deal with the weaknesses of local models.

Main plan: [PLAN.md](PLAN.md).

---

## 1. Core idea: one API, many backends

In MERCY, Gemini, Groq and OpenRouter all expose an **OpenAI-compatible** `/chat/completions`
endpoint, so MERCY has a single streaming client (`src/lib/llm/chat.ts`). EAGLE extends that
client, because the local runtimes expose the same API:

| Backend | Endpoint | Runs on |
|---|---|---|
| **Ollama** | `http://127.0.0.1:11434/v1` | Your computer |
| **Jupyter + Hugging Face** (your notebook) | `http://127.0.0.1:8001/v1` | Your computer |
| LM Studio / llama.cpp server / vLLM | `http://127.0.0.1:<port>/v1` | Your computer |
| **Colab** (model stored in Drive) | `https://<tunnel>/v1` | A Google VM (your account) |
| **Gemini** | `https://generativelanguage.googleapis.com/v1beta/openai` | Google cloud |
| **Groq** (as in MERCY) | `https://api.groq.com/openai/v1` | Groq cloud |
| **Grok** (xAI) | `https://api.x.ai/v1` | xAI cloud |

Adding a backend means adding a **provider row** (in the database), not new code.

### 1.1 Provider row

```jsonc
{
  "id": "ollama-main",
  "kind": "ollama",              // ollama | openai-compatible | colab | gemini | groq | grok
  "baseUrl": "http://127.0.0.1:11434/v1",
  "tier": "local",               // local | remote | cloud  (see §2)
  "apiKeys": [],                 // for cloud; encrypted at rest; several keys = MERCY's rotation
  "models": ["qwen3:8b"],
  "capabilities": { "tools": true, "vision": false, "embeddings": false },
  "contextTokens": 8192,
  "enabled": true,
  "dataScope": { "memory": false, "notes": false, "knowledge": false, "history": 6, "tools": false } // remote/cloud only
}
```

### 1.2 Role → model

Each job can use a different model (Admin → AI → Models):

| Role | Job | Default (local) |
|---|---|---|
| `chat` | Conversation, RAG answers | Mid-size model (7–9B) |
| `orchestrator` | Team lead (decides who does what) | A model good at tool calling |
| `agent` | NEELA/ALEX… using tools | Same or a smaller model |
| `embedding` | Knowledge search | `bge-m3` |
| `vision` (optional) | Understanding images | A vision model (Gemma 3 / Qwen-VL family) |
| `utility` | Chat titles, memory extraction, summaries | A small fast model (1–4B) |
| `stt` / `tts` | Voice | whisper / Piper / MMS (PLAN.md §8) |

Each role has an **ordered list** (like MERCY's slot list). If the first is busy or fails, the next
is tried — following the rule in §2.2.

---

## 2. Privacy tiers — the most important rule

| Tier | Badge | Where the data goes | When allowed |
|---|---|---|---|
| **local** | 🟢 Local | Nowhere (loopback) | Always |
| **remote** | 🟡 Remote | Another machine that is yours (a Colab VM) | Online mode only, provider on, tunnel host allowlisted |
| **cloud** | 🔴 Cloud | A company's servers (Google, Groq, xAI) | Online mode only, provider on |

### 2.1 Tier enforcement (in code)

- A `tier: local` provider whose `baseUrl` host is **not loopback cannot be saved**
  (`127.0.0.1`, `::1`, `localhost`). A cloud URL can't be marked "local" by mistake.
- Remote/cloud requests only go through `egressFetch()` (PLAN.md §5.2) and are rejected in Offline mode.

### 2.2 Fallback rule (different from MERCY!)

In MERCY, when Gemini fails it silently moves on to Groq. In EAGLE:

- **Never fall back silently to a less private tier.** A failing local model never moves to the cloud.
- Fallback only to the **same or a more private** tier (cloud → local OK; local → cloud NO).
- When every local model fails, the chat shows a card: "The local model isn't working (reason…).
  Send this message to Gemini?" [Send once] [No] — and the card lists exactly what would be sent
  (the message, no context).

---

## 3. Ollama (default, fully offline)

### 3.1 Why Ollama

- One-command install (Linux/macOS/Windows); CPU + NVIDIA/AMD GPU + Apple Silicon (Metal).
- Manages downloads, versions and deletion itself (`ollama pull`, `ollama rm`).
- OpenAI-compatible API (streaming + **tool calling**) and a native embedding API.
- Listens only on `127.0.0.1:11434` by default.

### 3.2 Install and setup (what eagle.sh does)

1. Checks `ollama --version`. If missing, it **shows** the official install command and runs it only after you confirm.
2. `curl 127.0.0.1:11434/api/version` — is the server running?
3. Checks that `OLLAMA_HOST` is loopback (`0.0.0.0` → warning: anyone on the LAN could use your model).
4. Recommends models for the hardware (§3.3) → `ollama pull <chat>` + `ollama pull <embedding>`.
5. After the download, **no more internet is needed**.

### 3.3 Which model for which hardware

Rule of thumb (Q4_K_M quantization): **~0.6 GB of RAM/VRAM per 1B parameters** + 0.5–2 GB for context.

| Your machine | Chat model size | Examples (family) |
|---|---|---|
| 8 GB RAM, no GPU | 3–4B | Qwen3 4B, Gemma 3 4B, Llama 3.2 3B, Phi-4-mini |
| 16 GB RAM / 8 GB VRAM | 7–9B | **Qwen3 8B** (default), Llama 3.1 8B, Gemma 3 (12B is tight) |
| 32 GB RAM / 12–16 GB VRAM / Apple M-series 32 GB | 12–14B | Gemma 3 12B, Qwen3 14B |
| 24 GB+ VRAM / Apple 64 GB | 27–32B | Gemma 3 27B, Qwen3 32B |

> Model names and versions change quickly. The repo keeps a recommendation list in
> `config/models.json` (updated with each EAGLE release), and `./eagle.sh models` shows what fits
> your machine. Check [ollama.com/library](https://ollama.com/library) for the latest at setup time.

### 3.4 Which model for which job

| Job | Recommendation | Why |
|---|---|---|
| Chat (Bangla + English) | **Qwen3** or **Gemma 3** family | Both are multilingual; Gemma 3 was trained on many languages, Qwen3 is strong at tool calling. The default is chosen by testing both in Bangla (§7) |
| Agents / tool calling | **Qwen3** (8B+) or Llama 3.1 8B | Stable tool-call format |
| Utility | Qwen3 1.7B/4B, Gemma 3 1B–4B | Fast; enough for titles and summaries |
| Embedding | **`bge-m3`** (1024 dims, multilingual, good Bangla, 8k context) | Cross-lingual like MERCY's Gemini embeddings: a Bangla question matches an English chunk |
| Embedding (low RAM, mostly English) | `nomic-embed-text` (768 dims) | Small, fast |
| Vision | Gemma 3 (4B+) or the Qwen-VL family | Images and screenshots |

### 3.5 Ollama settings EAGLE sets for you

- **`num_ctx`**: Ollama's default context is small. EAGLE's prompt = system + memory (~2–3k) +
  6 RAG chunks (~3k) + history, so every request sends `options.num_ctx` (default 8192, up to the
  model's limit). Without it, Ollama silently cuts the prompt and answers get worse.
- `keep_alive`: "30m" (the model stays in RAM, so the next answer is fast). Configurable in admin.
- `temperature`: chat 0.7, agents/tools 0.2, utility 0.
- `think`: thinking on/off for reasoning models (Qwen3), like MERCY's `REASONING_EFFORT`; the
  thinking is never shown (as in MERCY), only the final answer is streamed.

### 3.6 Bringing your own / Hugging Face models into Ollama

A **GGUF** model on Hugging Face can be pulled directly: `ollama pull hf.co/<user>/<repo>:<quant>`.
Or a local GGUF file:

```text
# data/models/Modelfile
FROM ./my-model.Q4_K_M.gguf
PARAMETER num_ctx 8192
```
`ollama create my-model -f data/models/Modelfile` → select `my-model` in EAGLE.
No Jupyter needed — this is the **easiest way** to run a Hugging Face model.

---

## 4. Jupyter Notebook + Hugging Face (local)

When: the model has no GGUF, or you want to experiment (a fine-tuned model, a LoRA adapter, a
custom pipeline), or you want to run something through `transformers`.

### 4.1 How it works

```
EAGLE ──HTTP (loopback, bearer token)──► small server inside the Jupyter kernel :8001
                                           └─ transformers model (GPU/CPU)
```

Cells in `notebooks/eagle_local_hf.ipynb` (in the repo):

1. **Config**: `MODEL_DIR = "../data/models/hf/<name>"`, port `8001`, `EAGLE_TOKEN` (copied from EAGLE's admin).
2. **Download once** (internet): `huggingface_hub.snapshot_download(repo_id, local_dir=MODEL_DIR,
   allow_patterns=["*.safetensors","*.json","tokenizer*"])` — safetensors only, no pickle `.bin`.
3. **Offline lock**: `os.environ["HF_HUB_OFFLINE"]="1"; os.environ["TRANSFORMERS_OFFLINE"]="1"`
   — from here on the libraries don't touch the internet.
4. **Load**: `AutoModelForCausalLM.from_pretrained(MODEL_DIR, device_map="auto", torch_dtype="auto")`
   (+ optional `bitsandbytes` 4-bit, + optional LoRA via `PeftModel`).
5. **Serve**: a small FastAPI app — `GET /v1/models`, `POST /v1/chat/completions` (SSE streaming
   via `TextIteratorStreamer`), using the tokenizer's `apply_chat_template` (with tools when the
   template supports them). `uvicorn` in a background thread on **`host="127.0.0.1"`** — never
   `0.0.0.0`. Every request must carry `Authorization: Bearer <EAGLE_TOKEN>`.
6. **Status cell**: "Connected to EAGLE ✅" — calls `/v1/models` to confirm.

Alternative servers (instead of, or next to, the notebook):
- `llama-cpp-python[server]` — GGUF, CPU friendly.
- `vllm serve <dir> --host 127.0.0.1` — Linux + NVIDIA, fast, with a tool-calling parser.

### 4.2 EAGLE side

- Admin → AI → Providers → **Add → Jupyter / custom (OpenAI-compatible)** → URL `http://127.0.0.1:8001/v1`, token.
- **Test** button: `/v1/models` + a short chat + a tool-call test → capabilities detected
  automatically (a model that can't call tools is marked "chat only" and not used for agents).
- The tier is automatically `local` (loopback). When the notebook stops, health turns 🔴 and the
  next model is used (local only).

---

## 5. Colab + Google Drive (opt-in, **data leaves the device**)

When: your computer has no GPU but you want to run a bigger model.

### 5.1 The honest part first

On this path your **prompt and context are processed on a Google VM**, the model lives in Drive,
and the tunnel provider (e.g. Cloudflare) sits in the middle of the traffic. It **breaks** EAGLE's
"no data leaves the device" principle — so it is tier **remote** 🟡, off by default, only
available in Online mode, and the chat shows the badge every time.

### 5.2 Flow

```
Drive: MyDrive/eagle/models/<model>/ (uploaded / downloaded once)
   │
Colab (eagle_colab.ipynb):
   1. drive.mount()  → load the model (vLLM or llama.cpp, T4 GPU)
   2. random EAGLE_TOKEN (or pasted from EAGLE) + optional E2E key
   3. server on 127.0.0.1:8000 (inside the Colab VM)
   4. cloudflared quick tunnel → https://<random>.trycloudflare.com
   5. prints URL + token (copy them into EAGLE)
   │
EAGLE (Online mode) ── egressFetch, only that exact host allowlisted ──► tunnel ──► Colab server
```

### 5.3 Security measures

- **Bearer token** (32 random bytes) — someone who finds the URL still can't use it without the token.
- **End-to-end payload encryption (optional, recommended)**: EAGLE and the notebook encrypt
  request/response bodies with AES-GCM using a shared key (HKDF from the token). The tunnel
  (Cloudflare) only sees ciphertext. (The Colab VM still sees plaintext — the compute happens there.)
- The notebook never logs or prints prompts; `runtime.unassign()` at the end of a session.
- Only that session's exact tunnel host is allowlisted; it is removed from EAGLE when the session ends.
- Default data scope: the message only (memory/notes/knowledge off).

### 5.4 Limits

- Free Colab: a GPU (T4, ~15 GB VRAM) is not always available, sessions are limited (a few hours), and idle sessions disconnect.
- **Colab's terms** may restrict some uses (remote proxies / tunnels) — check before using; Colab Pro is more relaxed.
- Alternatives: a Kaggle notebook (same notebook, minor changes), or your own GPU server (tier `remote`, same flow).

---

## 6. Cloud: Gemini, Groq, Grok (opt-in, like MERCY)

- MERCY's `keypool.ts` as is: several keys, slot = (provider, key, model), least recently used
  first, cooldowns for 429/401/5xx, a health table in admin. Keys are stored **encrypted** in
  `data/` (no env file needed).
- Gemini: MERCY's same OpenAI-compatible endpoint. **On the free tier Google may use prompts and
  responses to improve its products** — read their terms; use a paid tier or a local model for private work.
- Groq: a fast fallback, as in MERCY.
- Grok (xAI): `https://api.x.ai/v1`, OpenAI-compatible — a new provider kind, same client.
- Every request: Online mode + provider on + data scope (PLAN.md §5.4) + redaction + Network log.
- **Cloud embeddings are never used** — knowledge-base embeddings are always local (otherwise the
  entire knowledge base would be uploaded to the cloud while indexing).

---

## 7. Dealing with the weaknesses of local models

### 7.1 Tool calling (agents)

Small local models make tool-call mistakes (bad JSON, wrong tool, loops). Plan:

1. **Fewer tools**: MERCY already has per-agent tool lists — locally, at most 6 tools per agent.
2. **Strict schema**: validate tool arguments against the JSON schema; on invalid JSON, one "fix your JSON" retry, then fail.
3. **Step limit**: `MAX_AGENT_STEPS` 3–4 locally (5 in MERCY).
4. **Temperature 0.2** for the tool roles.
5. **A big orchestrator, small workers**: role → model mapping (§1.2).
6. **Simple mode**: with a weak model, drop the team and give one agent all the tools (configurable).
7. Because `act` tools need Approve, even a wrong tool call can't do harm.

### 7.2 Evals — measure which model is good

`eval/` in the repo:
- `eval/tools.jsonl` — 40–60 prompts with the expected tool and arguments ("set an alarm for 7 tomorrow morning" → `set_timer`).
- `eval/bangla.jsonl` — Bangla questions + a knowledge chunk → does the answer contain the key fact?
- `eval/rag.jsonl` — retrieval: is the expected chunk in the top 6? (compares embedding models).

`./eagle.sh eval <model>` → a score; Admin → Models shows each model's score. Default models are
chosen by these scores, not by guesswork.

### 7.3 Bangla

- Embeddings: `bge-m3` is cross-lingual; Postgres's `simple` full-text config handles Bangla words (tested in MERCY).
- Chat: Bangla is acceptable on 7B+ models and weak on 3–4B. If the wizard's language is Bangla, a slightly bigger model is recommended.
- The system prompt says "answer in the language the user writes in" (as in MERCY).

### 7.4 Context budget

The prompt builder trims to the model's `contextTokens`, by priority: system/policy > current
message > RAG chunks (6 → 4 → 3) > memory (relevant facts first) > history (oldest dropped first).

### 7.5 Speed

- Always stream; `keep_alive` keeps the model in RAM for a fast first token.
- 3–4B models on CPU-only machines; Ollama uses a GPU automatically when there is one.
- Utility jobs (titles, memory) run in the background on a small model and never hold up the main answer.

---

## 8. Model file security and licenses

- Only `safetensors` / `GGUF` (pickle `.bin`/`.pt` files are never loaded — code execution risk).
- Sources: Ollama's official library or well-known Hugging Face organisations; admin shows the source and sha256.
- Models are **never committed** (`data/models/` is gitignored, `*.gguf` blocked by the pre-commit hook).
- Models have their own licenses (Qwen: mostly Apache-2.0; Gemma: Gemma Terms of Use; Llama:
  Llama Community License). EAGLE ships no models; users download them. The README says so.

---

## 9. Summary — which path when

| You want | Path | Tier |
|---|---|---|
| The most private and the simplest | **Ollama** + `qwen3`/`gemma3` + `bge-m3` | 🟢 |
| A GGUF model from Hugging Face | `ollama pull hf.co/...` or a Modelfile | 🟢 |
| A Hugging Face model via transformers / fine-tuned / experiments | **Jupyter notebook** server on :8001 | 🟢 |
| No GPU, a big model, some risk is acceptable | **Colab + Drive** + tunnel + token + E2E | 🟡 |
| The smartest answers, for work where privacy matters less | **Gemini / Groq / Grok** | 🔴 |
