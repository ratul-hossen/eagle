# EAGLE — AI Model Plan

How EAGLE connects to AI models: the six ways a user can bring a model, how each one is set up in
a few clicks, where the data goes with each, how several models work together, and how to deal
with the weaknesses of local models.

Main plan: [PLAN.md](PLAN.md).

---

## 1. The six ways to bring a model

| # | Way | What the user does | Runs on | Tier |
|---|---|---|---|---|
| 1 | **Ollama — local model** | Picks a model in EAGLE; EAGLE pulls it | Your computer | 🟢 Local |
| 2 | **Ollama — cloud model** | Signs in to Ollama, picks a `-cloud` model | Ollama's servers | 🔴 Cloud |
| 3 | **Hugging Face → Jupyter (local)** | Opens the EAGLE notebook, types the model id, runs all cells | Your computer | 🟢 Local |
| 4 | **Hugging Face → Google Drive → Colab** | Opens the EAGLE Colab notebook, types the model id, runs all cells, pastes one code into EAGLE | A Google VM | 🟡 Remote |
| 5 | **Gemini API / Groq API** | Pastes an API key | Google / Groq | 🔴 Cloud |
| 6 | **Your own trained model** | Uses way 1, 3 or 4 with the model's folder or file | Wherever you run it | 🟢 or 🟡 |

**Any number of these can be connected at the same time** (§9): for example a local Ollama model
for everyday chat, a fine-tuned model in Jupyter for one agent, and Gemini kept as an opt-in
option for hard questions.

Anything else that speaks the OpenAI-compatible API (LM Studio, llama.cpp server, vLLM,
OpenRouter, …) can be added as **"Other (OpenAI-compatible)"** with a URL and an optional key.

---

## 2. Easy setup — the design

Setup must take minutes, not an afternoon. Five mechanisms make that happen.

### 2.1 One "Connect a model" screen

Admin → AI → **Connect a model** (also step 3 of the first-run wizard) shows six cards, one per
way above. Each card is a short guided flow — never a blank form:

| Card | Steps in EAGLE |
|---|---|
| Ollama (on this computer) | EAGLE finds Ollama → shows models that fit this machine → **Download** → done |
| Ollama Cloud | **Sign in to Ollama** (EAGLE runs `ollama signin` and shows the link) → pick a cloud model → confirm the cloud warning → done |
| Hugging Face in Jupyter | **Open notebook** (EAGLE starts Jupyter with the notebook) → type the model id → *Run all* → EAGLE connects by itself |
| Hugging Face in Colab | **Open in Colab** (button) → type the model id → *Run all* → paste the **connection code** → done |
| Gemini / Groq | Paste key(s) → EAGLE lists the models the key can use → pick → done |
| Your own model | Choose a folder or file → EAGLE detects the format and picks the right route (§8) |

Every flow ends with an automatic **Test** (a short chat, a tool call, and for embedding models
an embedding) and shows the result in plain words: "Works. Can use tools. About 18 words/second."

### 2.2 Auto-detect

On the Models page and during setup, EAGLE checks the usual local ports and offers what it finds:

| Port | Server |
|---|---|
| 11434 | Ollama |
| 8001 | EAGLE Jupyter notebook |
| 1234 | LM Studio |
| 8080 | llama.cpp server |
| 8000 | vLLM |

"Found Ollama with 3 models — connect?" One click. Only loopback addresses are scanned.

### 2.3 Connection codes (no copying URLs and tokens by hand)

The notebooks never ask the user to copy a URL and a token separately.

- **Local Jupyter (way 3)**: EAGLE shows a short **pairing code** (6 characters, valid 10 minutes).
  The notebook's first cell asks for it, then registers itself with EAGLE at
  `http://127.0.0.1:4810/api/models/pair` — EAGLE appears as "Connected ✅" in both places.
  Loopback only, so nothing leaves the machine.
- **Colab (way 4)**: the notebook's last cell prints one **connection code**, e.g.
  `eagle1:Q2xhdWRl…` — a base64 bundle of the tunnel URL, the bearer token and the end-to-end
  encryption key. Pasting it into EAGLE fills in everything. The code is shown once and expires
  when the Colab session ends.

### 2.4 Ready-made notebooks with one setting

`notebooks/eagle_local_hf.ipynb` and `notebooks/eagle_colab.ipynb` have **one cell to edit**:

```python
MODEL = "Qwen/Qwen3-8B"        # a Hugging Face model id, or a folder path for your own model
QUANTIZE = "auto"              # auto | 4bit | 8bit | none
```

Everything else (installing packages, checking the GPU, choosing a server, downloading, loading,
serving, connecting) is automatic. Before downloading, the notebook checks that the model fits the
GPU/RAM and suggests a 4-bit load or a smaller model if it doesn't.

### 2.5 The same thing from the terminal

```bash
./eagle.sh model add ollama qwen3:8b
./eagle.sh model add ollama-cloud gpt-oss:120b-cloud
./eagle.sh model add jupyter                 # starts Jupyter with the notebook and a pairing code
./eagle.sh model add colab "eagle1:Q2xh…"
./eagle.sh model add gemini                  # asks for the key without echoing it
./eagle.sh model add own ./my-finetune/      # detects the format, picks the route
./eagle.sh model list | test <name> | remove <name>
```

---

## 3. Privacy tiers — the rule behind every way

| Tier | Badge | Where the data goes | When allowed |
|---|---|---|---|
| **local** | 🟢 Local | Nowhere (stays on this computer) | Always |
| **remote** | 🟡 Remote | A machine you control but that isn't this one (a Colab VM) | Online mode only, after confirming once |
| **cloud** | 🔴 Cloud | A company's servers (Ollama Cloud, Google, Groq) | Online mode only, after confirming once |

### 3.1 Enforcement in code

- A model is `local` only if **the model itself runs on this computer**, not merely because its
  URL is loopback. This matters for Ollama: **cloud models are called through the local Ollama
  server** at `127.0.0.1:11434`, yet the prompt goes to Ollama's servers. EAGLE therefore marks an
  Ollama model as cloud when its name has the `-cloud` / `:cloud` tag or Ollama reports a remote
  host for it (`/api/show`, `/api/tags`). If EAGLE can't tell, it treats the model as cloud.
- Remote/cloud requests are blocked in Offline mode. For Ollama cloud models this check happens in
  EAGLE before the request reaches the Ollama server, since the request itself goes to loopback.
- Every other remote/cloud request goes through `egressFetch()` (PLAN.md §5.2): allowlist + Network log.

### 3.2 Fallback rule

- **Never fall back silently to a less private tier.** If a local model fails, EAGLE does not
  quietly use a cloud model (unlike MERCY, which falls back from Gemini to Groq on its own).
- Fallback goes only to the **same or a more private** tier.
- When every local model fails, the chat shows a card: "The local model isn't working (reason…).
  Send this message to Gemini?" [Send once] [No], with exactly what would be sent.

### 3.3 Data scope for remote and cloud models

By default a remote/cloud model receives only the current message and EAGLE's generic system
prompt. Memory, notes, knowledge chunks, chat history and tool use are separate toggles per model,
all off by default (PLAN.md §5.4). Knowledge-base **embeddings are always made locally**.

---

## 4. Way 1 — Ollama, local model 🟢

### 4.1 Why it is the default

- One-command install on Linux/macOS/Windows; CPU, NVIDIA/AMD GPUs and Apple Silicon.
- Manages downloads, versions and deletion (`ollama pull`, `ollama rm`).
- Streaming and **tool calling**, plus an embedding API.
- Listens only on `127.0.0.1:11434` by default.

EAGLE talks to Ollama through its **native API** (`/api/chat`, `/api/embed`, `/api/tags`,
`/api/pull`, `/api/show`), because it gives download progress, model details and per-request
options such as `num_ctx`.

### 4.2 Setup

1. EAGLE checks for Ollama. If it is missing, it shows the official install command and runs it only after you confirm.
2. It checks that Ollama listens on loopback (warns about `OLLAMA_HOST=0.0.0.0`: anyone on the LAN could use it).
3. It recommends a chat model and an embedding model for this machine (§4.3) and downloads them with a progress bar.
4. After the download **no internet is needed**.

### 4.3 Which model fits which machine

Rule of thumb (4-bit Q4_K_M): **~0.6 GB of RAM/VRAM per 1B parameters**, plus 0.5–2 GB for context.

| Machine | Chat model size | Examples (family) |
|---|---|---|
| 8 GB RAM, no GPU | 3–4B | Qwen3 4B, Gemma 3 4B, Llama 3.2 3B, Phi-4-mini |
| 16 GB RAM / 8 GB VRAM | 7–9B | **Qwen3 8B** (default), Llama 3.1 8B |
| 32 GB RAM / 12–16 GB VRAM / Apple 32 GB | 12–14B | Gemma 3 12B, Qwen3 14B |
| 24 GB+ VRAM / Apple 64 GB | 27–32B | Gemma 3 27B, Qwen3 32B |

> Model names change quickly. The recommendation list lives in `config/models.json` and is updated
> with each EAGLE release; `./eagle.sh models` shows what fits the current machine.

| Job | Recommendation | Why |
|---|---|---|
| Chat (Bangla + English) | **Qwen3** or **Gemma 3** family | Multilingual; the default is chosen by the Bangla eval (§10.2) |
| Agents / tool calling | **Qwen3** (8B+) or Llama 3.1 8B | Stable tool-call format |
| Utility (titles, summaries, memory) | Qwen3 1.7B/4B, Gemma 3 1B–4B | Fast |
| Embedding | **`bge-m3`** (1024 dims, multilingual, good Bangla) | A Bangla question finds an English chunk, as with MERCY's Gemini embeddings |
| Embedding (low RAM, mostly English) | `nomic-embed-text` (768 dims) | Small and fast |
| Vision | Gemma 3 (4B+) or the Qwen-VL family | Images and screenshots |

### 4.4 Settings EAGLE sets for you

- **`num_ctx`**: Ollama's default context is small and it silently cuts longer prompts. EAGLE's
  prompt (system + memory + 6 knowledge chunks + history) needs about 8k tokens, so EAGLE sends
  `num_ctx` with every request (default 8192, up to the model's limit).
- `keep_alive` "30m": the model stays loaded, so the next answer starts fast.
- `temperature`: chat 0.7, agents 0.2, utility 0.
- `think` on/off for reasoning models; the thinking is never shown, only the answer (as in MERCY).

---

## 5. Way 2 — Ollama, cloud model 🔴

Ollama can also run large models on its own servers. They are used through the same local Ollama
app, so for the user it feels like way 1 — but **the prompt leaves the computer**.

### 5.1 Setup

1. Card "Ollama Cloud" → **Sign in to Ollama** → EAGLE runs `ollama signin` and shows the sign-in link.
2. EAGLE lists the available cloud models (the ones tagged `-cloud`).
3. Pick one → EAGLE shows the cloud warning once → done.

Alternative without the local app: an Ollama API key with the host `https://ollama.com`
(stored encrypted, like the Gemini/Groq keys).

### 5.2 Rules

- Tier **cloud**: Online mode only, red badge on every answer, data scope off by default (§3.3).
- Read Ollama's own policy on how cloud prompts are handled before sending anything private.
- Useful for: very large models that no home computer can run, used for non-private questions.

---

## 6. Way 3 — Hugging Face model in Jupyter (local) 🟢

For any Hugging Face model, including ones that have no Ollama/GGUF version, experiments, and
fine-tuned models.

### 6.1 What the user does

1. EAGLE → Connect a model → **Hugging Face in Jupyter** (or `./eagle.sh model add jupyter`).
   EAGLE starts Jupyter from its own Python environment (`python/.venv`, with everything
   installed) and opens `notebooks/eagle_local_hf.ipynb`.
2. Types the model id in the one settings cell: `MODEL = "Qwen/Qwen3-8B"`.
3. **Run all**. The notebook asks for the pairing code that EAGLE shows, and connects.

### 6.2 What the notebook does

1. **Checks** the GPU (CUDA / Apple MPS / CPU) and free memory; estimates whether the model fits
   and picks 4-bit / 8-bit / full precision (`QUANTIZE = "auto"`).
2. **Downloads once** with `huggingface_hub.snapshot_download` into `data/models/hf/<name>/`
   — `safetensors` only, never pickle `.bin`. Gated models (Llama, Gemma) ask for a Hugging Face
   token, used only for the download.
3. **Locks offline**: `HF_HUB_OFFLINE=1`, `TRANSFORMERS_OFFLINE=1` — no more internet.
4. **Loads** with `transformers` (`device_map="auto"`), or with `llama-cpp-python` for a GGUF file.
5. **Serves** an OpenAI-compatible API on **`127.0.0.1:8001`** (never `0.0.0.0`): `/v1/models` and
   `/v1/chat/completions` with streaming, using the model's own chat template (with tools when the
   template supports them). Every request must carry EAGLE's bearer token.
6. **Pairs** with EAGLE (§2.3) and prints "Connected to EAGLE ✅".

Next time: open the notebook and **Run all** — the model is already downloaded and the pairing is
remembered. When the notebook stops, EAGLE shows the model as 🔴 offline and uses the next local model.

### 6.3 Shortcut for GGUF models

If a Hugging Face model has a **GGUF** version, Jupyter isn't needed: EAGLE pulls it straight into
Ollama (`ollama pull hf.co/<user>/<repo>:<quant>`). The Hugging Face card offers this
automatically when it finds a GGUF version.

---

## 7. Way 4 — Hugging Face → Google Drive → Colab 🟡

For users without a GPU who want a bigger model. The model is kept in Google Drive so it is
downloaded only once, and each Colab session loads it from Drive.

### 7.1 The honest part first

On this path the **prompt and its context are processed on a Google VM**, the model is stored in
Drive, and the tunnel provider sits between EAGLE and Colab. It breaks EAGLE's "no data leaves the
device" principle, so it is tier **remote** 🟡: off by default, Online mode only, badge on every answer.

### 7.2 What the user does

1. EAGLE → Connect a model → **Hugging Face in Colab** → **Open in Colab**
   (opens `notebooks/eagle_colab.ipynb` from the EAGLE repo on GitHub).
2. Chooses *Runtime → Change runtime type → GPU*.
3. Types the model id in the settings cell → **Run all** → allows Drive access when asked.
4. Copies the **connection code** from the last cell → pastes it into EAGLE → done.

### 7.3 What the notebook does

```
1. Mount Drive → MyDrive/EAGLE/models/
2. Model already in Drive?  yes → load it from Drive
                            no  → download from Hugging Face into Drive (only the first time)
3. Load on the GPU (vLLM if supported, otherwise transformers / llama.cpp), 4-bit if needed to fit the T4
4. Serve on 127.0.0.1:8000 inside the VM, with a random bearer token
5. Start a cloudflared quick tunnel → https://<random>.trycloudflare.com
6. Print the connection code: tunnel URL + token + end-to-end key
```

### 7.4 Security

- **Bearer token** (32 random bytes): the URL alone is useless.
- **End-to-end encryption** (on by default): request and response bodies are encrypted with
  AES-GCM using the key in the connection code, so the tunnel only sees ciphertext. The Colab VM
  still sees plaintext, because the model runs there.
- The notebook never logs prompts. Ending the session (`runtime.unassign()`) deletes the VM;
  EAGLE removes the tunnel host from its allowlist.

### 7.5 Limits

- Free Colab: the GPU (T4, ~15 GB VRAM) isn't always available, sessions last a few hours and
  disconnect when idle. EAGLE shows "Colab disconnected" and uses the next model.
- Colab's terms may restrict tunnels and some uses; check them before relying on this.
- The same notebook works on Kaggle with small changes.

---

## 8. Way 5 — Gemini API and Groq API 🔴

The same providers MERCY uses, with the same OpenAI-compatible endpoints.

| Provider | Endpoint | Setup |
|---|---|---|
| Gemini | `https://generativelanguage.googleapis.com/v1beta/openai` | Paste one or more keys from Google AI Studio |
| Groq | `https://api.groq.com/openai/v1` | Paste one or more keys from the Groq console |

- After the key is pasted, EAGLE calls the provider's `/models` endpoint and lists the models that
  key can use, so the user picks from a list instead of typing names.
- Several keys per provider rotate exactly like MERCY's `keypool.ts` (least recently used first,
  cooldowns on 429/401/5xx, a health table in admin).
- Keys are stored **encrypted** in `data/`; no `.env` editing.
- **On the Gemini free tier, Google may use prompts to improve its products** — read the terms,
  and keep private work on local models.
- Tier cloud: Online mode, red badge, data scope off by default.

---

## 9. Way 6 — Your own trained model

EAGLE accepts the formats that training normally produces and picks the route automatically
(card "Your own model", or `./eagle.sh model add own <path>`):

| What you have | EAGLE's route | Tier |
|---|---|---|
| A **GGUF** file | Imported into Ollama (`ollama create` with a generated Modelfile) | 🟢 |
| A full **Hugging Face model folder** (`config.json` + `*.safetensors`) | Jupyter notebook (way 3) with `MODEL = "<folder>"`; optional one-click **convert to GGUF** (llama.cpp's `convert_hf_to_gguf.py`, then quantize) so it runs in Ollama | 🟢 |
| A **LoRA adapter** (`adapter_config.json` + `adapter_model.safetensors`) | Jupyter: base model + adapter (`peft`); or Ollama: Modelfile with `FROM <base>` + `ADAPTER <path>` when the base model is in Ollama | 🟢 |
| A model you trained in Colab and saved to Drive | Colab notebook (way 4) with `MODEL = "/content/drive/MyDrive/…"` | 🟡 |
| A model on your own Hugging Face account (private repo) | Way 3 or 4 with the repo id + your HF token | 🟢 / 🟡 |

Before connecting, EAGLE checks:

- **Format**: only `safetensors` and `GGUF` (pickle `.bin` / `.pt` can execute code when loaded, so they are refused).
- **Chat template**: if the tokenizer has none, EAGLE asks which family it was trained from
  (Llama / Qwen / Gemma / Mistral / ChatML) and uses that template.
- **Tool calling**: the automatic test decides whether the model can be used by agents or only for chat.
- **Fit**: whether it fits this machine's memory, and at which quantization.

Optional: `./eagle.sh eval <model>` runs EAGLE's eval set (§10.2) so you can compare your model
with the stock models on Bangla answers, tool calls and knowledge answers.

---

## 10. Using many models at once

Every connected model shows on the Models page with its tier, status and abilities. They can work together in five ways.

### 10.1 Roles

Each job has an ordered list of models (Admin → AI → Models):

| Role | Job | Typical choice |
|---|---|---|
| `chat` | Everyday conversation, knowledge answers | Local 7–9B |
| `orchestrator` | Leads the team (who does what) | The best local tool-calling model |
| `agent` | NEELA/ALEX… using tools | Same, or smaller |
| `utility` | Chat titles, memory, summaries | Local 1–4B |
| `embedding` | Knowledge search | `bge-m3` (always local) |
| `vision` | Images | A local vision model |

If the first model in a list is busy or offline, the next one is used, following the fallback
rule (§3.2).

### 10.2 A model per agent

Each agent in Admin → Agents can have its own model: e.g. a fine-tuned writing model for the agent
that drafts documents, a strong tool model for the planner.

### 10.3 Model picker in the chat

The composer has a model picker (default: the `chat` role). Picking a model applies to that chat
only. A remote or cloud model turns the composer border red and says where the message will go.
Typing `@gemini` or `@my-finetune` at the start of a message sends just that message to that model.

### 10.4 Compare

**Compare** sends one question to two or three models side by side (cloud ones only in Online
mode). Useful for choosing defaults or checking your own model against a stock one.

### 10.5 Presets

A preset saves a whole set of choices in one name, switchable from the header:

- **Private** (default): only local models.
- **Power**: local for everything, plus one cloud model allowed for `chat` with consent.
- **Experiment**: your Jupyter model for `chat`, Ollama for the rest.

---

## 11. Dealing with the weaknesses of local models

### 11.1 Tool calling

Small local models make tool-call mistakes (bad JSON, wrong tool, loops):

1. At most 6 tools per agent locally (MERCY already has per-agent tool lists).
2. Tool arguments validated against the JSON schema; one "fix your JSON" retry, then a clean failure.
3. `MAX_AGENT_STEPS` 3–4 locally (5 in MERCY); temperature 0.2 for tool roles.
4. A stronger orchestrator with smaller workers (§10.1).
5. **Simple mode**: with a weak model, one agent with all tools instead of a team.
6. `act` tools always need your Approve, so a wrong tool call can't do harm.

### 11.2 Evals — choose models by measurement

`eval/` in the repo:

- `eval/tools.jsonl` — 40–60 requests with the expected tool and arguments ("set an alarm for 7 tomorrow morning" → `set_timer`).
- `eval/bangla.jsonl` — Bangla questions + a knowledge chunk → does the answer contain the key fact?
- `eval/rag.jsonl` — is the expected chunk in the top 6? (compares embedding models)

`./eagle.sh eval <model>` gives a score, shown next to each model on the Models page. Defaults
are chosen by these scores.

### 11.3 Bangla

- `bge-m3` embeddings are cross-lingual; Postgres's `simple` full-text config handles Bangla words.
- Bangla is acceptable on 7B+ models and weak on 3–4B; choosing Bangla in the wizard recommends a bigger model.

### 11.4 Context and speed

- The prompt is trimmed to each model's context: system > current message > knowledge chunks
  (6 → 4 → 3) > memory > history (oldest first).
- Always streamed; `keep_alive` keeps models loaded; utility jobs run in the background on a small model.

---

## 12. Model file security and licenses

- Only `safetensors` and `GGUF` are loaded. Pickle files are refused.
- Sources and sha256 are shown on the Models page.
- Models are **never committed** (`data/models/` is gitignored; `*.gguf` and `*.safetensors` are
  blocked by the pre-commit hook).
- Models have their own licenses (Qwen: mostly Apache-2.0; Gemma: Gemma Terms of Use; Llama:
  Llama Community License). EAGLE ships no models; users download them under those licenses.

---

## 13. Summary

| You want | Way | Tier |
|---|---|---|
| The simplest and most private | **1 — Ollama local** | 🟢 |
| A very large model, privacy not critical | **2 — Ollama cloud** | 🔴 |
| Any Hugging Face model on your own machine | **3 — Jupyter** (or Ollama if it has a GGUF) | 🟢 |
| A big model without your own GPU | **4 — Drive + Colab** | 🟡 |
| The strongest hosted models | **5 — Gemini / Groq** | 🔴 |
| Your own fine-tuned or trained model | **6 — own model**, through way 1, 3 or 4 | 🟢 / 🟡 |
| Several at once | Roles, per-agent models, picker, compare, presets (§10) | per model |
