# EAGLE — Master Plan

> **EAGLE** is the **local, privacy-first** version of MERCY.
> It runs on one computer, keeps its database on that computer, works without internet, and
> **no data ever leaves the device — not even when the computer is online.** The internet is used
> only to download EAGLE itself and the model files, and a download carries no personal data.
>
> The repository is public. Anyone can clone it and run `./eagle.sh` to get their own EAGLE.

The AI model plan is in its own file: **[AI_MODELS.md](AI_MODELS.md)**.

---

## 0. At a glance

| Topic | MERCY (today) | EAGLE (plan) |
|---|---|---|
| Where it runs | Vercel (cloud) | Your own computer (`127.0.0.1`) |
| Database | Neon Postgres (cloud) | **PGlite** (embedded Postgres + pgvector, in `data/`) |
| AI | Gemini → Groq → OpenRouter (all cloud) | **Local models only, several at once** (AI_MODELS.md): Ollama, Hugging Face in Jupyter, your own model, other local servers |
| Embeddings | Gemini API | **Local** (Ollama `bge-m3` / `nomic-embed-text`) |
| Modes | Public MERCY + Owner mode + ObS (`/kneel`) | **One mode**: always owner powers, no "Owner mode" label anywhere |
| Sign-in | Google OAuth + owner password + `/to-me` | **Local password (Argon2id) + optional TOTP**, no Google |
| Admin | Visitor-facing admin + owner admin | **Only you** (single admin = single user) |
| Voice | Browser Web Speech API (audio goes to Google) + Edge TTS (text goes to Microsoft) | **whisper.cpp / faster-whisper** (STT) + **Piper / MMS-TTS** (TTS), all local |
| Python tools | Vercel Python function | Local Python subprocess (no network) |
| Timers / automations | External cron (`/api/tick`) | EAGLE's own in-process scheduler |
| Run | `npm run dev` / Vercel deploy | **`./eagle.sh`** |
| Internet | Needed for everything | **Not used**, except for downloads you start (install, update, models), which carry no personal data |
| Platform | Any browser | **Linux** (Ubuntu first), macOS; Windows later through WSL2 (§4.3) |
| Phone | Any browser | Optional, **directly from the laptop** over its own Wi-Fi hotspot or USB — never through the internet (§5.4) |

---

## 1. Principles (they override every other decision)

1. **Local only.** EAGLE never sends personal data off the computer — no prompts, chats, memory,
   files, voice or logs — whether the internet is connected or not. There is no "Online mode".
   This is enforced in code (§5), not left to the model or to a setting.
2. **Data stays on the device.** Chats, memory, notes, knowledge, embeddings and voice all live in
   `data/`. Personal data never goes into git (`.gitignore` + a pre-commit check).
3. **Downloads are the only internet use, and they are visible.** Installing, updating and
   downloading a model are started by you, say what they download, carry no personal data, and are
   listed in Admin → Privacy → **Network log**. After setup, EAGLE works with the internet unplugged.
4. **One user, one mode.** No visitors, no public persona, no Inbox. You = admin = owner.
5. **Reproducible and public.** No keys, names or photos are hard-coded in the repo. All
   personalization happens through the first-run wizard and the admin panel and is saved in `data/`.
6. **The MERCY repository is never touched.** Code is **copied** from MERCY at a fixed commit,
   chosen when Phase 0 starts: the latest MERCY `main` (at the time of writing `c47e2b4`, which
   adds background jobs, chat summaries, backup, hands-free voice and MCP over `d745b16`, plus the
   agent skills / lessons / evals work that follows it). The commit used is written in
   `docs/UPSTREAM.md`; no change, PR or commit is made to MERCY.

---

## 2. What is kept, removed and added

### 2.1 Kept (ported, running locally)

| MERCY feature | In EAGLE |
|---|---|
| Chat UI (`ChatApp`, `Message`, `Markdown`, sidebar, history) | Same design. The "Owner mode" pill, unlock/exit buttons and visitor starters are removed |
| 3D eagle avatar (`Eagle3D`, `LivingEagle`, orb, image) | Same — already local (three.js, no model download) |
| Team / agents (NEELA, ALEX, Tony, Friday + custom) | Same runner; default names are generic and renameable (no one's personal team in a public repo) |
| Tool tiers `read / safe / act` + Approve card | Kept exactly — central to security |
| Long-term memory (`[[remember|…]]`) | Same, encrypted (§6.3) |
| Private notes | Same, encrypted |
| Knowledge base + structure-aware chunker + hybrid RAG (vector + full-text + RRF) | Same algorithm on PGlite; local embeddings |
| Timers, alarms, automations (`schedule.ts`) | Same logic, in-process scheduler (no external cron) + desktop notifications |
| Document tools (PDF/Word/PPT/Excel → text, Markdown → PDF/Word, Bangla shaping) | Same Python code (`python/mercy_tools` → `python/eagle_tools`) as a local subprocess |
| Admin panel layout, save bar, sections | Same look; new section list (§7) |
| Flow view (agents at work) | Same |
| Key pool / slot rotation (`keypool.ts`) | The fallback idea only, across local models (no API keys) |
| Router (knowledge vs web) | Knowledge only (no web search); optional offline Wikipedia (Kiwix) takes the "web" role |
| Chat summaries ("what did we talk about last week?") | Same, made on the scheduler by the `utility` model; encrypted with memory |
| Background jobs (big work done a step at a time, report at the end) | Same, on the in-process scheduler — no 60-second limit locally, so steps can be longer |
| File attachments in the chat (PDF, Word, Excel, photos) | Same composer; read by the local Python tools; photos by a local vision model |
| Hands-free voice loop (silence detection, barge-in, sentence-by-sentence speech) | Same hook; STT/TTS go to the local voice server (§8) |
| Automation templates | Same (morning brief without email/calendar until Phase 7: uses the local calendar and notes) |
| MCP client | Same, and better suited locally: **stdio** MCP servers on this computer too (filesystem, git, SQLite…), never remote HTTP ones (they would send data out) |
| Usage page | Same, plus per-model speed (tokens/s) for local models |
| **Agent skills, lessons from feedback, evals** | Same — and more important locally (§9.1) |

### 2.2 Removed

| MERCY feature | Why |
|---|---|
| Public MERCY / visitor identity / visitor scope policy | EAGLE has no visitors |
| Owner mode toggle, `/to-me`, `/exit`, "Owner mode until…" | One mode — always the owner |
| `/kneel`, `/getup`, ObS identity | One mode (requirement) |
| Inbox (visitor messages) | No visitors |
| Portfolio sync, `/api/sync`, cron | Needs internet and is personal (Ratul's portfolio) |
| `/embed` widget, portfolio framing headers | Part of a public website |
| Google OAuth sign-in, `ADMIN_EMAILS` | Online dependency; replaced by a local password |
| Gemini, Groq, OpenRouter, key pool, Tavily web search, maps | Send data to outside services |
| Google Calendar / Drive / Gmail, Telegram | Send data to outside services; replaced by the local calendar, local files and desktop notifications |
| Visitor chat rate limits | Single user (sign-in brute-force limits stay) |
| Neon HTTP driver, Vercel config, `vercel.json` | Local database |
| `msedge-tts`, Web Speech API as defaults | Send data to outside servers → local voice (§8) |
| `next/font/google` | Downloads from Google Fonts at build time → fonts bundled with `next/font/local` |
| `public/owner.png`, `seed/knowledge/ratul-profile.md`, Ratul-specific text | Public repo; no personal data |
| Web push (VAPID) | Push goes through Google/Apple/Mozilla servers; EAGLE uses the browser's local Notification API instead |
| Inbox alerts on Telegram | No inbox (no visitors) |
| GitHub connector | Sends data to GitHub; for local repositories a local `git` MCP server does the job |

### 2.3 New (EAGLE only)

- **`./eagle.sh`** — install, setup wizard, run, update, backup (§4).
- **Local-only network layer**: loopback-only model calls, a download-only gateway and the **Network log** (§5).
- **Connect a model**: one screen with guided cards (Ollama, Hugging Face in Jupyter, your own model, other local servers), auto-detect, pairing code, automatic tests (AI_MODELS.md §2).
- **Phone access without the internet** (optional, off by default): the laptop's own hotspot or USB (§5.4).
- **Local files tool**: a sandbox folder (`data/files/`) agents can read and write — nothing outside it.
- **Local calendar** (ICS file) instead of Google Calendar; you can import an `.ics` file you exported yourself.
- **Lock screen**: auto-locks when idle; unlock with password/TOTP.
- **Encrypted backup / restore** (one password-encrypted `.eagle-backup` file).
- **Offline knowledge (optional)**: Kiwix/ZIM (offline Wikipedia) — "web-like" search without internet.

---

## 3. Architecture

```
┌──────────────────────────── your computer ─────────────────────────────┐
│                                                                        │
│  Browser (localhost:4810)                                              │
│    │  same-origin only (CSP connect-src 'self')                        │
│    ▼                                                                   │
│  EAGLE server (Next.js, bound to 127.0.0.1)                            │
│    ├─ auth: password + TOTP, session cookie                            │
│    ├─ chat / agent runner / tools (read · safe · act)                  │
│    ├─ RAG (chunker + hybrid search)                                    │
│    ├─ scheduler (timers, automations)                                  │
│    ├─ llm/  → provider registry ──┬─► Ollama        127.0.0.1:11434    │
│    │                              ├─► Jupyter/HF    127.0.0.1:8001     │
│    │                              └─► other local server  127.0.0.1    │
│    ├─ voice/ → whisper (STT), piper/mms (TTS) [local python]           │
│    ├─ pytools/ → python subprocess (no network)                        │
│    ├─ net/download → only: EAGLE updates, packages, model files        │
│    └─ db/ → PGlite  ─────────────► data/db/ (pgvector + FTS)           │
│                                                                        │
│  data/  (gitignored)  db/ files/ models/ voices/ backups/ logs/        │
└────────────────────────────────────────────────────────────────────────┘
        Nothing personal ever crosses this line, online or not.
```

### 3.1 Stack

| Part | Choice | Why |
|---|---|---|
| App | **Next.js (same version as MERCY) + React + Tailwind** | Keeps the frontend identical to MERCY; code can be copied directly |
| Run mode | `next build` once, then `next start -H 127.0.0.1` | Fast production build, no dev server |
| Database | **PGlite** (`@electric-sql/pglite` + `vector` extension), data dir `data/db` | Embedded Postgres (WASM), so no database server to install. MERCY's schema, pgvector, `tsvector` full-text and RRF SQL run almost unchanged. MERCY's build log already tested its SQL on PGlite |
| Database alternative | System Postgres + pgvector (`DATABASE_URL`) | For more data / performance; one adapter, two backends |
| AI runtime | **Ollama** (default) | One-command install, OpenAI-compatible API, CPU and GPU, easy model management |
| Python | `python/.venv` (created by eagle.sh) | Document tools, voice, Jupyter bridge |
| Auth | Argon2id password hash + TOTP (RFC 6238) | No online service needed |

**Database adapter**: MERCY uses the Neon `sql\`...\`` tagged template. EAGLE's `src/lib/db.ts`
provides a small adapter with the same tagged-template API on top of PGlite's `query()`, so the
rest of the code (rag, memory, schedule, agents) runs with few changes.

**Embedding dimensions**: MERCY uses a fixed `vector(768)`. Local models differ (bge-m3 = 1024,
nomic = 768). Plan: `settings.embedding = {model, dims}`; schema setup creates `vector(dims)`;
changing the model offers "Re-index all" in admin (recreate the chunks table and re-embed). The
`embed_key` logic (re-embed only changed chunks) stays.

### 3.2 Repository layout

```
eagle/
├── eagle.sh                 ← the only entry point
├── README.md                ← five-line quick start
├── LICENSE
├── docs/
│   ├── PLAN.md              ← this file
│   ├── AI_MODELS.md         ← AI model details
│   ├── SECURITY.md          ← threat model: what is protected and what isn't
│   ├── UPSTREAM.md          ← which MERCY commit the code was copied from
│   └── ADMIN.md             ← admin panel guide
├── config/
│   └── eagle.example.toml   ← port, model and voice defaults (no secrets)
├── notebooks/
│   └── eagle_local_hf.ipynb ← local Jupyter: HF model → OpenAI-compatible server
├── python/
│   ├── eagle_tools/         ← MERCY's document tools (copied)
│   ├── eagle_voice/         ← whisper STT + piper/mms TTS server
│   └── requirements*.txt    ← pinned
├── scripts/                 ← migrate, seed, doctor, backup (TS)
├── seed/knowledge/          ← generic templates ("about-me.example.md"), nothing personal
├── src/                     ← Next.js app (ported from MERCY)
└── data/                    ← .gitignore — ALL personal data
    ├── db/  files/  models/  voices/  backups/  logs/
    └── secrets.json         ← chmod 600 (session key, wrapped data key)
```

---

## 4. `./eagle.sh` — how it runs

```bash
git clone https://github.com/<you>/eagle && cd eagle
./eagle.sh            # first time: setup wizard; afterwards: start
```

### 4.1 Commands

| Command | What it does |
|---|---|
| `./eagle.sh` | Installs if needed, starts, and opens the browser (the first-run setup page if EAGLE isn't set up yet) |
| `./eagle.sh setup --terminal` | The same first-run setup in the terminal, for people who prefer it |
| `./eagle.sh start` / `stop` / `status` | Server on/off, PID file `data/eagle.pid` |
| `./eagle.sh doctor` | Checks everything: Node, Python, Ollama, models, disk, port, file permissions, database integrity |
| `./eagle.sh models` | List / pull / recommend Ollama models for this hardware |
| `./eagle.sh update` | `git pull` (verified tag) → `npm ci` → build → migrate. **The only command that needs internet on its own** (says so first) |
| `./eagle.sh backup` / `restore <file>` | Encrypted backup |
| `./eagle.sh reset-password` | Forgotten password, from the terminal (§6.2.3) |
| `./eagle.sh phone on` / `off` | Phone access over the laptop's hotspot or USB (§5.4) |
| `./eagle.sh uninstall` | Removes the app; asks separately whether to delete `data/` |

### 4.2 First run

The terminal only installs and starts EAGLE. Everything personal — including the password — is
set in the browser.

**In the terminal (`./eagle.sh`):**

1. **Checks**: OS (**Linux** — Ubuntu/Debian first, then Fedora/Arch; macOS; on Windows it explains WSL2, §4.3), Node ≥ 20, Python ≥ 3.10, `git`, free disk, RAM, GPU (`nvidia-smi` / Apple Silicon).
2. **Install (needs internet — this is the install step, and it says so)**: `npm ci` (exact versions from the lockfile),
   `python -m venv python/.venv && pip install -r requirements.txt --require-hashes`.
   If Ollama is missing it shows the install command (it never runs `curl | sh` without your confirmation).
3. **Database**: PGlite init + migrate + generic seed. `data/` created with `chmod 700`, `umask 077`.
4. **Build**: `next build` (telemetry off: `NEXT_TELEMETRY_DISABLED=1`).
5. **Start** on `127.0.0.1:4810` (configurable port).
6. **Setup link**: because no password exists yet, it creates a one-time **setup token**, prints
   `http://127.0.0.1:4810/setup#<token>` and opens it in the default browser. Only that link can
   open the setup page, so no other program or website on the computer can claim EAGLE first.
   The token is stored only as a SHA-256 hash and expires in 30 minutes (run `./eagle.sh` again
   for a new one).

**In the browser (`/setup`), a short step-by-step page in MERCY's design:**

1. **Language** (Bangla / English) — the setup page itself switches language.
2. **About you**: your name, what to call the assistant, timezone (auto-detected; MERCY
   hard-codes `Asia/Dhaka`, EAGLE makes it configurable).
3. **Password**: typed twice, with a strength meter (at least 10 characters; a long passphrase is
   suggested). Stored only as an Argon2id hash (§6.2).
4. **Recovery key**: shown once (§6.2.3). *Download* or *Print*, then type its last 4 characters
   to confirm it was saved. Setup can't finish without this step.
5. **Two-step sign-in (optional)**: TOTP QR code for an authenticator app (works offline).
6. **Connect a model**: the "Connect a model" cards (AI_MODELS.md §2), with the models that fit
   this machine pre-selected and a download progress bar. Can be skipped and done later in admin.
7. **Voice (optional)**: download the local voice models.
8. **Done** → the chat opens, signed in.

After setup EAGLE runs fully **with the internet switched off**. A self-test
(`doctor --offline`) proves it: it runs chat + RAG + voice with networking disabled
(Linux network namespace / `unshare`, or with the proxy off).

### 4.3 Linux first, Windows through WSL2

EAGLE is built and tested on **Linux** first (the owner's laptop runs Ubuntu), then macOS.
Windows users run it inside **WSL2** — Windows' built-in Linux ("Windows Subsystem for Linux"):
one command in PowerShell (`wsl --install`) installs Ubuntu inside Windows, and `./eagle.sh` runs
there unchanged; the browser on Windows opens it at `127.0.0.1`. A native Windows installer is not
planned for v1. **No Docker** (decided): it complicates GPU access and Ollama, and `./eagle.sh`
already sets everything up.

### 4.4 Running in the background (optional)

`./eagle.sh service install` → a `systemd --user` unit on Linux, a `launchd` plist on macOS, so
EAGLE starts at login.

---

## 5. Network and privacy layer (the most important part)

### 5.1 The rule

EAGLE sends **no personal data** off the computer, whether the internet is on or off. There is no
online mode, no cloud model, no web search and no outside connector. The only network traffic
EAGLE ever starts is a **download you asked for**:

| Download | When | What it sends |
|---|---|---|
| EAGLE's code and packages | `./eagle.sh` install, `./eagle.sh update` | The package names (from the lockfiles) |
| A model | Admin → Connect a model → Download, `./eagle.sh models pull` | The model's name |
| Voice models | Setup step 7 or Admin → Voice | The voice's name |

Each download says what it will fetch and from where before it starts. A computer that is never
connected again after setup loses nothing but updates.

### 5.2 Enforced in code

- **Loopback only for everything that carries data.** Model servers, the voice server and Python
  tools are reached through `localFetch()`, which accepts only `127.0.0.1` / `::1` / `localhost`
  and a port EAGLE knows. EAGLE warns if Ollama listens on `0.0.0.0` (anyone on the network could use it).
- **Downloads go through one module**, `src/lib/net/download.ts` → `download(kind, url)`. It
  accepts only a fixed allowlist of hosts (the npm and PyPI registries, `ollama.com`'s model
  registry through the Ollama app, `huggingface.co`, GitHub for EAGLE's own releases), only GET,
  never a request body, and never from a code path that has chat, memory or file data.
- A lint rule forbids raw `fetch` / `http.request` / `net` in server code; only `localFetch` and
  `download.ts` may use them. CI checks it, and a test runs the whole app with the network
  namespace closed (`unshare -n`) to prove chat, RAG, agents, voice and documents work offline.
- Ollama **cloud** models are detected and refused (AI_MODELS.md §3), because they are called
  through the local Ollama app but run on Ollama's servers.
- **Network log** (Admin → Privacy): every download with time, host, size and purpose. Bodies are
  not logged. An empty log after setup is the expected state.

### 5.3 Browser side

- **CSP**: `default-src 'self'; connect-src 'self'; img-src 'self' data: blob:; font-src 'self'; frame-ancestors 'none'`.
  The browser cannot make third-party requests either (no CDN, analytics or fonts).
- Fonts (Inter, JetBrains Mono, Noto Sans Bengali) ship in the repo.
- The browser's Web Speech API (in Chrome, audio goes to Google's servers) is **never used**;
  speech recognition is local whisper (§8).
- MERCY's web push (it goes through Google/Apple/Mozilla) is not copied; reminders use the
  browser's local Notification API.

### 5.4 Using EAGLE from your phone — without the internet (optional, off by default)

The phone talks **directly to the laptop**, never through the internet or any outside server:

| Way | How |
|---|---|
| **Laptop hotspot** (recommended) | Admin → Phone → **Turn on**. EAGLE starts the laptop's own Wi-Fi hotspot (NetworkManager on Linux), with a strong password. The phone joins it; no router or internet is involved |
| **USB** | Phone connected by cable with USB tethering; EAGLE listens on that USB network only |
| **Home Wi-Fi** | Possible, but only if you choose it: the router sees encrypted traffic between the two devices, and nothing goes to the internet |

Rules when phone access is on:

- EAGLE listens on that one network interface only (never `0.0.0.0`), and only while it is on;
  it switches itself off after a set time (default 2 hours).
- **Pairing by QR code**: the laptop shows a QR code with a one-time code; the phone scans it,
  signs in with the password (+ TOTP if set) and is added to Admin → Security → Devices, where it
  can be removed.
- **HTTPS with EAGLE's own local certificate** (made on this laptop; the phone trusts it once
  during pairing), so the hotspot can't read the traffic.
- The phone gets the same chat and admin, including voice: the phone records, the laptop transcribes.
- The phone needs no app (the PWA from MERCY: "Add to Home Screen").
## 6. Security

### 6.1 Threat model

| Threat | Protection |
|---|---|
| Someone else on the network reaches EAGLE | Bound to `127.0.0.1` only. Phone access is optional and off by default; when on: one interface only, QR pairing, HTTPS (local certificate), password + TOTP, auto-off (§5.4) |
| Another website in the browser calls EAGLE's API (CSRF / DNS rebinding) | `Host` header check (only `localhost:<port>` / `127.0.0.1:<port>`), `Origin` check on every POST, `SameSite=Strict` cookie |
| Another user on the same machine | `data/` is `chmod 700`, secrets `600`; sensitive columns encrypted (§6.3) |
| Stolen laptop | Full-disk encryption recommended (LUKS / FileVault / BitLocker) + app-level encryption + encrypted backups |
| Prompt injection (a file, page or email saying "ignore previous…") | MERCY's tool tiers: an `act` tool always needs your **Approve**; tool output is data and can never approve. Offline, there is also far less outside content |
| An agent wandering the file system | File tools work only inside the `data/files/` sandbox (path traversal check, symlinks resolved) |
| Network or exec through Python tools | Subprocess without network (`unshare -n` on Linux when available), timeout, memory limit, no shell |
| Supply chain (npm/pip) | Lockfile + `npm ci`, pip `--require-hashes`, Dependabot alerts, `npm audit` in CI; few dependencies |
| Sign-in brute force | MERCY's rate-limit logic (5 failures / 15 min) in the local database |
| Malware in a model file | Only `.gguf` / `safetensors` (no pickle `.bin`); Ollama's official library / verified HF repos; sha256 shown |
| Secrets in logs | API keys masked (last 4), bodies not logged |
| Personal data pushed to the public repo | `.gitignore` + pre-commit hook (blocks `data/`, `.env*`, `*.gguf`, `secrets.json`) |

### 6.2 Passwords, sign-in and recovery

#### 6.2.1 Sign-in

- **One password** for everything: the chat and the admin panel. Set on the first-run page (§4.2).
- Optional **TOTP** second step.
- Session: a random 256-bit token in an httpOnly, `SameSite=Strict` cookie. The database stores
  only its **SHA-256 hash**, so a copied database can't be used to sign in. Sessions are revocable
  in Admin → Security.
- 30 minutes idle → **lock screen** (work continues in the background, the UI is locked);
  12 hours maximum (MERCY's owner-session logic reused).
- **Re-confirm for sensitive actions**: turning on phone access, pairing a phone, changing the
  password, a new recovery key, backup/restore and erase ask for the password
  again if it wasn't entered in the last 10 minutes.
- Failed attempts: 5 within 15 minutes lock sign-in for the rest of those 15 minutes (MERCY's rate
  limit, local). Recovery attempts have the same limit.
- Admin → Security: active sessions, recent sign-ins, failed attempts (MERCY's `security_log` reused).

#### 6.2.2 Changing the password

Admin → Security → **Change password**: current password + new password twice. Only the data
key's wrapping changes (§6.3), so nothing is re-encrypted and nothing is lost. All other
sessions are signed out.

#### 6.2.3 Forgot password

EAGLE has no email and no server, so recovery works with things only you have:

| Way | Where | Result |
|---|---|---|
| **Recovery key** | Sign-in page → **Forgot password?** → enter the recovery key → set a new password | Everything kept. A new recovery key is shown (the old one stops working) |
| **Recovery key from the terminal** | `./eagle.sh reset-password` → enter the recovery key | Same as above, for when the browser page isn't reachable |
| **No recovery key** | `./eagle.sh reset-password --erase-encrypted` (type `ERASE` to confirm) | A new password is set, but the encrypted data — memory, private notes, chats, API keys, connector tokens, TOTP — **cannot be recovered and is deleted**. Knowledge files, settings, agents, timers and models are kept |

The recovery key is a 24-word phrase (or the same as a 32-character code), generated at setup.
EAGLE stores it only as an Argon2id hash plus a second wrapping of the data key (§6.3). The
honest part: if both the password and the recovery key are lost, encrypted data is gone — that is
exactly what keeps it safe from anyone else. The setup page and the Security page say this in plain words.

The terminal reset needs shell access to this computer's user account, which counts as being the owner.

#### 6.2.4 Hashing — used wherever possible

A secret is **hashed** when EAGLE only needs to check it, and **encrypted** when EAGLE must use
the original value (for example to send an API key to a provider).

| Secret | Stored as | Why |
|---|---|---|
| Password | **Argon2id** hash (64 MB memory, t=3, unique salt) | Only checked; slow by design against guessing |
| Recovery key | **Argon2id** hash (+ wraps the data key) | Only checked |
| TOTP backup codes | **Argon2id** hash each, one-time use | Only checked |
| Session tokens | **SHA-256** hash | Only checked; a copied database gives no sessions |
| Setup token, pairing codes (Jupyter) | **SHA-256** hash + expiry, one-time use | Only checked |
| Phone pairing keys, the local HTTPS certificate's private key | AES-256-GCM **encrypted** / file `600` | Needed to talk to a paired phone |
| TOTP secret | AES-256-GCM **encrypted** | Needed to compute codes |
| Model files, backups | **SHA-256** checksum (backups: HMAC) | Detects tampering or corruption |

All comparisons use constant-time equality (`timingSafeEqual`), as in MERCY. Hashing uses
well-known libraries (`argon2`/`@node-rs/argon2`, Node's `crypto`) — no home-made cryptography.

### 6.3 Encryption at rest

- A random 256-bit **data key**, wrapped twice and stored in `secrets.json`: once with a key
  derived from the password (Argon2id), once with a key derived from the recovery key. Changing
  the password or the recovery key only re-wraps it; data is not re-encrypted.
- AES-256-GCM for: memory (facts, chat summaries, lessons), private notes, chat messages, skills.
- **Honest trade-off**: knowledge chunk text and embeddings are not encrypted, because search
  (full-text + vector) needs plaintext. That is why **full-disk encryption** is strongly
  recommended; `doctor` checks for it and warns.
- Optional "vault mode" (later): lock `data/db` with gocryptfs/age while EAGLE is stopped.

### 6.4 Backup

`./eagle.sh backup` → `data/` (database dump + files) → `tar` → password-encrypted with
**age** / AES-GCM → `data/backups/eagle-YYYYMMDD.eagle-backup`. EAGLE never uploads backups
anywhere; copy the file yourself if you want it elsewhere. Optional automatic daily local backup
(keeps the last 7).

---

## 7. Admin panel (you only)

MERCY's admin shell (sidebar, save bar, ⌘S, mobile pills) stays the same. Sign-in is EAGLE's
own password (§6.2); there is no Google sign-in.

**How to open it** — both ways, like MERCY plus a shortcut:

- An **admin icon at the bottom left** of the chat sidebar (next to the theme toggle) opens the
  admin panel. On phones it is in the sidebar drawer's footer. Keyboard: `Ctrl/⌘ + ,`.
- The admin panel is its own page at **`/admin`** (as in MERCY), with a **Back to chat** link.
  It can be bookmarked or opened directly.

**Everything is set from here.** After installation nothing needs a terminal or a config file:
models, phone access, voice, identity, branding, agents, skills, backups — all in the admin panel. The
terminal is only for installing, updating and the password reset without a browser.

Sections:

| Group | Section | Contents |
|---|---|---|
| — | **Dashboard** | Model status (🟢 Ollama ready), network mode, disk use, knowledge count, scheduled tasks, last backup |
| You | **Profile** | Your name, language, timezone, how you want it to work (MERCY's owner Profile) |
| | **Memory** | Remembered facts (edit/delete) |
| | **Private notes** | Notes |
| EAGLE | **Identity** | Name, personality, style, rules, greeting, voice — **one identity** (no public/owner/ObS split) |
| | **Branding** | Name, tagline, accent colour, avatar (eagle / orb / image) |
| | **Knowledge** | Upload (json/md/txt + **pdf/docx/pptx/xlsx** via local Python), edit, preview chunks, re-index |
| Team | **Agents** | Team, tools per agent, voices |
| | **Timers & automations** | Same as MERCY |
| | **Files** | Browse the `data/files/` sandbox |
| AI | **Connect a model** | The six guided cards, auto-detect, pairing/connection codes (AI_MODELS.md §2) |
| | **Models** | Every connected model with tier, status, abilities and eval score; roles, per-agent models, presets, compare; context length, temperature, data scope, keys; hardware info |
| Privacy | **Network** | Proof of local-only: the **Network log** (downloads only), the offline self-test result, which local servers EAGLE talks to |
| | **Phone** | Turn phone access on/off (hotspot / USB / home Wi-Fi), QR pairing, paired devices, auto-off timer |
| | **Security** | Change password, new recovery key, TOTP on/off, sessions, security log, lock timeout |
| | **Backup** | Back up now, restore, schedule |
| System | **System** | Database health, re-index all, logs, version, update instructions |

---

## 8. Voice — fully local

| | MERCY | EAGLE |
|---|---|---|
| Speech → text | Browser Web Speech API (Chrome sends audio to Google) | **faster-whisper** or **whisper.cpp** (`small`/`medium` multilingual — Bangla + English), served by `python/eagle_voice` on `127.0.0.1:8002` |
| Text → speech | Edge TTS (Microsoft, online) | English: **Piper** (fast, CPU, many female voices). Bangla: **`facebook/mms-tts-ben`** (Hugging Face, local), or a Piper Bangla voice if one is available |
| Fallback | Browser voice | The browser's **local** voices (`speechSynthesis`, OS voices, no network) |

MERCY's `SentenceBuffer` (speak after the first sentence, prefetch the next), the voice-mode loop
and the avatar states stay; only the `/api/tts` and STT backends change. Recording uses
`MediaRecorder` in the browser → `/api/stt` (localhost) → whisper. Voice models live in
`data/voices/` and are an optional download during setup.

---

## 9. Owner powers (all of them, in the one mode)

Status of every MERCY owner tool in EAGLE:

| Tool | Tier | Offline | Note |
|---|---|---|---|
| `get_time`, `calculate` | read | ✅ | Timezone from config |
| `set_timer`, `list_timers`, `cancel_timer`, `create_automation` | safe | ✅ | In-process scheduler + desktop notifications (browser Notification API) |
| `search_knowledge` | read | ✅ | Local RAG |
| `save_note` | safe | ✅ | Encrypted |
| `make_document`, `document_to_pdf`, `count_tokens`, `text_stats` | safe/read | ✅ | Local Python |
| **new** `file_list`, `file_read`, `file_write` | read/safe/act | ✅ | `data/files/` only; overwrite/delete = act |
| **new** `calendar_local_*` | read/safe | ✅ | ICS file |
| **new** `offline_wiki_search` | read | ✅ | If a Kiwix ZIM is installed |
| `web_search`, `maps_*`, `calendar_*` (Google), `drive_*`, `gmail_*`, `message_me` (Telegram), `github_*` | — | ❌ | **Not in EAGLE**: they send data out (§5.1) |

Automations (e.g. "a brief every morning at 8") also run offline, with a local model. When the
computer is off, automations don't run (it isn't a 24/7 cloud). On the next start EAGLE shows
what was missed, reusing MERCY's catch-up logic (`nextDue`).

### 9.1 Making the agents good at their jobs (more important with local models)

Local 3–9B models are weaker than Gemini at following long instructions, at tool calling and at
Bangla. EAGLE closes most of that gap by giving the agents better instructions and letting them
learn, instead of needing a bigger model. All of this comes from MERCY:

| Mechanism | What it is | Why it helps a small model |
|---|---|---|
| **Skills (playbooks)** | Named instruction documents for a kind of task ("Formal email to a professor", "Weekly report", "Study plan"), each with a one-line "when to use". An agent's prompt lists only the names; `use_skill` loads the full text when needed (or it is included directly when the task clearly matches) | Short prompts with exact steps work far better on small models than one long system prompt. Skills are Markdown in `data/` — the user can write their own, and EAGLE can propose one after a task ("Save this as a skill?", needs Approve) |
| **Lessons from feedback** | 👍/👎 on every answer; a 👎 with a note, or a correction in the chat ("no, more formal"), becomes a short lesson for the agent that did the work, added to its prompt (≤ 15 per agent) | The model doesn't need to be retrained: the same mistake is not repeated |
| **Evals** | Test cases per agent (prompt + rubric + expected tools), run on demand in a dry-run sandbox (no real side effects) and scored by a judge model plus fixed checks; history per model | Choose models and prompts by measurement (merged with AI_MODELS.md §11.2). Locally, a run costs only time, so it can run after every model or prompt change, or nightly |
| **Worked examples** | A 👍 answer can be saved as an example for its skill | Small models copy a format from one good example better than from a description |
| **Self-check** | For bigger tasks the agent checks its draft against the task (answered everything? facts from the tool output?) before reporting | Costs one extra local call — affordable locally (no API bill), off for quick chats |
| **Per-agent model** | AI_MODELS.md §10.2 | A strong tool model where it matters, a small fast one elsewhere |

Lessons, skills and examples are part of the encrypted backup. A future option (not planned for
v1): export good examples + lessons as a small fine-tuning dataset for "your own model"
(AI_MODELS.md, way 6).

---

## 10. Frontend — like MERCY, one mode

- `ChatApp`, `Message`, `Markdown`, `VoiceOverlay`, `team.tsx`, `FlowCanvas` and `Eagle3D` are ported.
- Removed: the "Owner mode" pill/timer, the unlock form, `/exit`, `/kneel`, visitor suggestions, the "Hiring?" link.
- Starter suggestions are tasks (like MERCY's owner starters), personalised.
- **Admin icon at the bottom left** of the sidebar → `/admin` (§7).
- New pages in MERCY's design: `/setup` (first run), `/login` (with **Forgot password?**), `/recover`, and the lock screen.
- A small status in the header: model name and a 🔒 "Local" mark (with the Network log one click away).
- A **model picker** in the composer (local models only).
- Light/dark, `prefers-reduced-motion` and the Bangla font stay as they are.
- The assistant's default name is **EAGLE** and can be changed in admin (whoever clones it can name it).

---

## 11. Personalization and the public repo

- The repo has only **generic** defaults: an identity template, an empty knowledge base, a generic agent team.
- Everything personal goes through the wizard + admin into `data/`. Ratul's profile, photo, name
  and voice lines are not copied.
- `config/eagle.example.toml` (non-secret defaults) → `data/config.toml`.
- **Import from MERCY** (optional script): a MERCY export (knowledge files, identity, memory) as
  JSON → imported into EAGLE. It only reads the export file; MERCY is not changed.
- **License**: **AGPL-3.0-only** (decided; see `LICENSE`). Anyone may use, study, modify and share
  EAGLE. Whoever distributes a modified version, or lets other people use one over a network, must
  publish its full source under the same license — so every version of EAGLE stays auditable,
  which fits a privacy tool. Running your own unmodified or modified copy for yourself requires
  nothing. Every source file carries an `SPDX-License-Identifier: AGPL-3.0-only` header.
- `CONTRIBUTING.md`, issue templates, `SECURITY.md` (how to report a vulnerability).

---

## 12. Phases (build order)

At the end of every phase: `typecheck`, `lint`, `test`, `next build` clean + **offline self-test**.

### Phase 0 — Repository bootstrap
- [ ] Copy `src/`, `python/`, `scripts/` and config from the latest MERCY `main` (§1.6; record the commit in `docs/UPSTREAM.md`; MERCY stays read-only).
- [ ] Remove personal data: `owner.png`, `ratul-profile.md`, voice lines; Ratul/Dhaka hard-coding → config.
- [ ] Remove Vercel, Neon, `api/py`, embed, sync, inbox, kneel, Google auth, web push, Telegram, GitHub, Google connectors, maps, web search, Gemini/Groq and the key pool (all send data out, §5.1).
- [ ] `.gitignore`, LICENSE, README, pre-commit hook, CI (lint + typecheck + test + raw-fetch check).

### Phase 1 — Local foundation
- [ ] PGlite adapter (`db.ts`) + schema (MERCY's public and owner databases merged into one).
- [ ] Local auth: `/setup` with setup token, Argon2id password, recovery key, TOTP, hashed sessions, lock screen, re-confirm, rate limit.
- [ ] Forgot password: `/recover` + `./eagle.sh reset-password` (with and without the recovery key).
- [ ] Admin icon (bottom left) + `/admin` route.
- [ ] One mode: `isOwner()` → always true when signed in; owner-only branches simplified.
- [ ] `eagle.sh` (checks, install, setup link, start/stop/status/doctor, `setup --terminal`).
- [ ] Local fonts, CSP, Host/Origin checks, telemetry off.

### Phase 2 — Local AI
- [ ] Provider registry (database) + generic OpenAI-compatible client (extending MERCY's `chat.ts`).
- [ ] Ollama (native API): health, model list, pull with progress, chat (streaming + tools), embeddings; cloud-model detection.
- [ ] Hugging Face in Jupyter: `eagle_local_hf.ipynb` (one settings cell), pairing endpoint.
- [ ] Your own model: format detection, GGUF import, HF folder / LoRA routes, convert to GGUF.
- [ ] Other OpenAI-compatible endpoints (LM Studio / llama.cpp / vLLM) + auto-detect of local ports.
- [ ] Roles, per-agent models, chat model picker, `@model`, compare, presets.
- [ ] Configurable embedding dimensions + re-index all.
- [ ] Admin → Models, Providers. Hardware-based recommendations.

### Phase 3 — Privacy layer
- [ ] `localFetch` (loopback only) + `download.ts` (allowlisted downloads, GET only) + lint rule + CI check.
- [ ] Ollama cloud-model detection and refusal.
- [ ] Network log page; encryption at rest (data key, wrapping, column encryption).
- [ ] Offline self-test (`doctor --offline`, network namespace closed) in CI.

### Phase 4 — Owner powers
- [ ] Agent runner + team + flow view (tool calling tested with local models, AI_MODELS.md §7).
- [ ] Memory, notes, timers, automations (in-process scheduler), notifications.
- [ ] Python document tools as a local subprocess; PDF/Office upload in Knowledge.
- [ ] File sandbox tools, local calendar.
- [ ] Background jobs, chat summaries, automation templates on the in-process scheduler.
- [ ] Skills, lessons from feedback, worked examples, self-check, evals (§9.1) — run the eval set against each recommended local model and publish the scores in `docs/AI_MODELS.md`.
- [ ] MCP: local stdio servers (sandboxed to `data/files/` where they touch files) + remote ones through the gateway.

### Phase 5 — Local voice
- [ ] `python/eagle_voice`: whisper STT + Piper/MMS TTS server.
- [ ] Local `/api/stt`, `/api/tts`; a MediaRecorder path in `useVoice`.
- [ ] Voice mode, team voices (a Piper voice per agent).

### Phase 6 — Phone access without the internet (optional)
- [ ] Laptop hotspot (NetworkManager) and USB tethering; bind to that interface only; auto-off.
- [ ] Local certificate authority, QR pairing, device list and removal; PWA on the phone.

### Phase 7 — Local extras
- [ ] Local MCP servers over stdio (filesystem in `data/files/`, git, SQLite), sandboxed.
- [ ] Offline Wikipedia (Kiwix/ZIM) as the "web" for the router.
- [ ] ICS import for the local calendar.

### Phase 8 — Hardening and release
- [ ] Backup/restore, service install, update with signed-tag verification.
- [ ] `SECURITY.md`, `ADMIN.md`, screenshots.
- [ ] Tests on Ubuntu (the owner's laptop) and one other Linux, then macOS; a WSL2 guide for Windows.
- [ ] Tag `v1.0.0`.

---

## 13. Risks and honest limits

| Risk | Response |
|---|---|
| Local models are weaker than MERCY's Gemini, especially in **Bangla** and **tool calling** | Model choices in AI_MODELS.md; skills, lessons and evals (§9.1); a separate (bigger) model for agents; fewer tools |
| Low-RAM computers (8 GB) | Small models (3–4B), smaller context, voice optional |
| Slow on CPU only | Streaming, small models, lower `num_ctx`; GPU used automatically when present |
| Not 24/7 (computer off = automations off) | Show missed tasks + catch up |
| No cloud model to fall back on when a local model is weak | Skills, lessons and evals (§9.1); recommend the biggest model the laptop can run; be honest in the answer when unsure |
| No web search, so no fresh news or prices | Said plainly in the answer; offline Wikipedia for general knowledge |
| A phone on home Wi-Fi shares the network with other devices | Hotspot or USB recommended; HTTPS + pairing + auto-off on any network |
| PGlite is single-process and slower with large data | System Postgres as an alternative backend |
| Encryption vs search trade-off | Recommend full-disk encryption + `doctor` check |

---

## 14. Decisions

All decided (2026-10-10, by the owner):

1. **Models**: local only. Gemini, Groq, Ollama Cloud and Colab are not supported (they send data out).
2. **License**: AGPL-3.0-only (§11).
3. **Platform**: runs on the owner's own laptop — Linux (Ubuntu) first, then macOS; Windows through WSL2 (§4.3).
4. **Docker**: no.
5. **Phone**: optional, off by default, only directly from the laptop (hotspot / USB, or home Wi-Fi if chosen) — never through the internet (§5.4).
6. **Name**: "EAGLE" by default; name, look and avatar are changed in Admin → **Branding** (and asked once in the first-run setup).
