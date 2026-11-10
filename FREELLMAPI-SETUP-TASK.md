# FreeLLMAPI — Local Setup Task

**Read this whole file before running anything.** This is a task brief for you (Claude Code) to set up an already-cloned FreeLLMAPI repo on my machine.

---

## Context you need

**What FreeLLMAPI is:** a self-hosted, OpenAI-compatible proxy that aggregates the *free tiers* of ~29 LLM providers behind one `/v1` endpoint. It has no models of its own — it is purely a router. It stores my provider API keys encrypted in a local SQLite DB, picks the best available model per request, and fails over when one provider is rate-limited.

**Repo:** `tashfeenahmed/freellmapi` (MIT licensed)

**My machine:**
- macOS, Apple Silicon (M5 Pro)
- Homebrew and GitHub CLI (`gh`) installed
- Git identity is handled by conditional `includeIf` config — personal email inside `~/personal`, office email everywhere else. **This is already correct. Do not "fix" my git config.**

**Where the repo is:** I cloned it already. Find it first — do not re-clone. Try in this order:
1. `ls ~/personal/freellmapi` — most likely location
2. `find ~ -maxdepth 4 -type d -name freellmapi -not -path "*/node_modules/*" 2>/dev/null`

If you find more than one, ask me which. If you find none, ask me for the path. **Do not clone a fresh copy.**

**Goal state:** router running locally on `:3001`, dashboard on `:5173`, at least one provider key working, and a successful test request that shows which upstream model served it.

---

## Hard rules — do not violate these

1. **Never commit, print, or echo the contents of `.env`.** It holds the master encryption key for my provider credentials.
2. **Never commit, print, or echo any provider API key or the unified `freellmapi-...` key.** If you need to reference a key in a command, use a shell variable or tell me to paste it myself.
3. **Do not set `DEV_MODE=true`.** The server will fall back to a database-stored development key when `DEV_MODE=true` and `NODE_ENV` is not production. That fallback must never be used with real provider keys. Generate a proper `ENCRYPTION_KEY` instead.
4. **Do not push anything to any remote.** No commits to this repo at all unless I explicitly ask.
5. **Do not touch anything under `~/docprime`.** That is office work (Bitbucket, Policybazaar group). This tool is for personal use only and must not be wired into office projects.
6. **Do not modify my global Claude Code / Codex / Cursor config** in this task. Wiring coding agents to this router is explicitly out of scope — see "Out of scope" at the bottom.
7. **Stop and ask me** at every point marked 🛑 **HUMAN GATE**. Those steps need a browser and my accounts; you cannot and should not attempt them.

---

## Step 1 — Preflight

Verify the environment before touching anything.

```bash
node -v          # need v20 or higher
npm -v
sw_vers -productVersion
```

- If Node is missing or below 20: run `brew install node`, then re-check.
- Confirm you are in the right directory and it is the real repo: `package.json` should exist and `git remote -v` should point at `tashfeenahmed/freellmapi`.

Report the versions back to me before continuing.

---

## Step 2 — Read before you trust

Before installing, skim these and flag anything that looks off:

- `package.json` — check the `scripts` block (`dev`, `build`, `test`) and scan `dependencies` for anything that looks unrelated to an Express + React + SQLite app.
- `.env.example` — list every variable it expects, so we know what is configurable.
- Check for install-time scripts: `grep -n "postinstall\|preinstall\|prepare" package.json`

If a postinstall script exists, show it to me and wait for my OK before running `npm install`.

Tell me in one short paragraph what the scripts and dependencies look like. If anything seems suspicious, say so plainly — I would rather bail now than after keys are in it.

---

## Step 3 — Install dependencies

```bash
npm install
```

Native SQLite modules will compile for arm64 — this takes a few minutes and is normal. If it fails, capture the full error and check for these known causes before improvising:
- missing Xcode command line tools → `xcode-select --install`
- Node version mismatch with prebuilt binaries → try `npm rebuild`
- Python missing for node-gyp → `brew install python`

Do not silently switch to `--force` or `--legacy-peer-deps`. If peer deps conflict, show me the conflict first.

---

## Step 4 — Generate the encryption key

This is mandatory for startup. It is the AES-256-GCM master key that encrypts my provider credentials at rest.

```bash
cp .env.example .env
ENCRYPTION_KEY="$(node -e 'console.log(require("crypto").randomBytes(32).toString("hex"))')"
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$ENCRYPTION_KEY" > .env
```

Then verify without leaking the value:

```bash
# should print the key NAMES only, never the values
cut -d= -f1 .env
# should print 64 (32 bytes as hex)
awk -F= '/^ENCRYPTION_KEY=/{print length($2)}' .env
```

**Confirm `.env` is gitignored:**

```bash
git check-ignore -v .env
```

If that returns nothing, `.env` is **not** ignored. Add `.env` to `.gitignore`, then re-run the check. Also confirm nothing sensitive is already staged: `git status --short`.

**Tell me to back up this key somewhere safe (password manager).** If I lose it, every stored provider key becomes unreadable and I have to re-add them all. Do not put it in a file in the repo, in Notes, or anywhere in the project directory.

---

## Step 5 — Start the server

`npm run dev` is a **long-running foreground process**. Do not block your session on it.

```bash
npm run dev > /tmp/freellmapi-dev.log 2>&1 &
echo $! > /tmp/freellmapi.pid
sleep 15
tail -40 /tmp/freellmapi-dev.log
```

Expected: server listening on `:3001`, Vite dev server on `:5173`.

Verify both are actually up:

```bash
curl -s -o /dev/null -w "server:%{http_code}\n" http://localhost:3001/v1/models
curl -s -o /dev/null -w "dashboard:%{http_code}\n" http://localhost:5173
```

If port 3001 or 5173 is already occupied, tell me what is holding it (`lsof -i :3001`) rather than reassigning ports on your own.

Give me the log tail and both status codes. Then tell me the dashboard URL to open.

---

## Step 6 — 🛑 HUMAN GATE: I get the provider keys

**Stop here. This is my job, not yours.** Do not try to browse to these sites, do not try to automate signup, do not ask me to paste keys into your terminal.

Print me a clean checklist of where to sign up, and why each one matters. Start with these three — all card-free, and they cover fast/smart/coding:

| Provider | Console | What it gets me |
|---|---|---|
| Groq | console.groq.com | fastest inference, Llama 3.3 70B |
| Google AI Studio | aistudio.google.com | Gemini 2.5 Pro — smartest model in the free tier |
| Cerebras | cloud.cerebras.ai | very fast, Qwen 3 Coder |

Optional second wave once the first three work: OpenRouter, Mistral, Cohere, Cloudflare Workers AI, NVIDIA NIM, HuggingFace.

Then tell me to do this **in the dashboard UI at http://localhost:5173**, and wait for me to say I'm done:

1. **Keys** page → paste each provider key → save
2. Confirm each key's status dot goes green (it runs a health check)
3. **Models** page → drag the Fallback Chain into priority order — top entry is tried first
4. **Keys** page header → copy the unified `freellmapi-...` key

I will paste keys into the dashboard myself, in the browser. You never see them.

---

## Step 7 — Verify routing end to end

Once I confirm keys are in, test it. **I will export the unified key into your shell myself** — ask me to run `export FLA_KEY=...` rather than asking me to paste it into a message.

```bash
curl -sS http://localhost:3001/v1/chat/completions \
  -H "Authorization: Bearer $FLA_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"auto","messages":[{"role":"user","content":"Reply with exactly: OK"}]}' \
  -D /tmp/fla-headers.txt | head -c 2000

grep -i "x-routed-via" /tmp/fla-headers.txt
```

Every response carries an `X-Routed-Via: <platform>/<model>` header — that tells us which upstream actually served it. **The test only passes if that header is present.** A 200 with no routing header means something is wrong.

Also sanity-check the catalog:

```bash
curl -sS http://localhost:3001/v1/models -H "Authorization: Bearer $FLA_KEY" | head -c 1000
```

Then confirm failover works: on the Models page I'll disable my top-priority model, you re-run the chat request, and we check that `X-Routed-Via` names a *different* model. That proves the fallback chain is live and not just decorative.

---

## Step 8 — Write me a local usage note

Create `NOTES-adityaraj.md` **in the repo root** (and add it to `.gitignore` — this is my scratch file, not a contribution). Include:

- Exact command to start and stop the server, including the background-process invocation and `kill $(cat /tmp/freellmapi.pid)`
- Where the SQLite DB lives on disk, and which files must be backed up together (DB + the `ENCRYPTION_KEY`) — they are useless apart
- Which providers I added and the date
- A ready-to-paste Node and Python snippet pointing an OpenAI client at `http://localhost:3001/v1` with `model: "auto"`
- The `auto:*` routing strategies the dashboard exposes, so I know my options beyond plain `auto`
- Any error I hit during setup and what fixed it

---

## Out of scope — do not do these

- **Do not** run `npx freellmapi setup-claude`, `setup-codex`, `setup-cursor`, or any other generator. Do not touch `~/.claude`, `~/.codex`, or Cursor settings. I already run Claude Code on real Opus/Sonnet and I am not downgrading my coding agent to a free-tier pool.
- **Do not** wire this into anything under `~/docprime`.
- **Do not** expose the server beyond localhost. No `HOST_BIND=0.0.0.0`, no tunnels, no ngrok. It is single-user by design and has no multi-tenant auth model.
- **Do not** enable the Premium / signed-catalog paid feed. Not buying it right now.
- **Do not** deploy this anywhere — no Docker push, no Fly, no Railway, no Netlify.

---

## Things I already know, so don't re-explain them

I've read the project's own disclaimer and I'm going in with eyes open:

- This is for personal experimentation and prototyping, **not production**. If I build something real on it, I swap in a paid API before shipping.
- No frontier models. The free catalog tops out around Llama 3.3 70B, GLM-4.5, Qwen 3 Coder, and Gemini 2.5 Pro. No Opus- or GPT-5-class reasoning here.
- Effective quality degrades through the day as top models hit daily caps and the router falls down the chain. Resets at UTC midnight.
- No SLA, variable latency.
- My relationship with each upstream provider is still governed by the ToS I accepted on signup — proxying the traffic through this router doesn't change that, and staying compliant is on me. This is exactly why it stays personal-only and off office projects.

What I *do* want it for: backend LLM calls in my personal projects (`dualmind-chatbot`, `studynotion-v2`) during development, so I'm not burning paid credits on prototyping — with an OpenAI-compatible seam that makes switching to a paid provider a one-line change later.

---

## Reporting format

Work through the steps in order. After each step, give me:

- ✅ / ⚠️ / ❌ and one line on what happened
- the exact commands you ran
- any error output verbatim, not paraphrased

**Do not skip ahead past a failure.** If a step fails, stop, show me the error, and propose a fix — don't work around it silently. And don't mark a step ✅ unless its verification command actually passed.
