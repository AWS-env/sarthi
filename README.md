# Bhumi Vikas Sahayak (Sarthi)

**Track 1: Bhumi Vikas Sahayak** · Claude Impact Lab × Cerebras Café Compute, Bhopal · 11 October 2026

**Team:** Clover · Members: Medhansh, Prakhar, Shrey, Shreyas

A question-answering tool for the **MP Bhumi Vikas Rules, 2012**. A clerk asks in English, Hindi or Hinglish and gets a short answer in the same language, with the rule number, the exact rule text it came from, and the PDF page. When the rules are silent, the tool says so instead of guessing.

> **The tool points to the rule; the clerk decides.** The rules data is consolidated to **6 March 2025**.

---

## Contents

- [Who it is for](#who-it-is-for)
- [What works today](#what-works-today)
- [How it works](#how-it-works)
- [Measured quality](#measured-quality)
- [Getting started](#getting-started)
- [Configuration](#configuration)
- [Data pipeline](#data-pipeline)
- [Evaluation scripts](#evaluation-scripts)
- [Apple Watch app](#apple-watch-app)
- [API reference](#api-reference)
- [Security model](#security-model)
- [Deployment](#deployment)
- [Project layout](#project-layout)
- [Data sources](#data-sources)

---

## Who it is for

The user is a **building-permission clerk at the Bhopal Municipal Corporation counter**. A citizen asks something like *"My plot is 200 sq m on a 9 m road. How much gap do I leave at the front, and what fee do I pay?"* The clerk needs the right rule number and its exact wording within a minute.

The 108-page rulebook is hard to use at a counter for three reasons:

- Fee and setback numbers sit in tables spread across the book, and tables are easy to misread.
- The rules keep changing, so the clerk has to know which version an answer comes from.
- Questions come in English and Hindi, and text copied from the official Hindi PDF comes out garbled.

The full brief is in [`reference/PROBLEM-STATEMENT.md`](reference/PROBLEM-STATEMENT.md).

## What works today

| Feature | Where | Status |
|---|---|---|
| Landing page ("Sarthi") | `apps/web/app/page.tsx` | Works |
| Email + password sign-in / sign-up (Supabase), English/Hindi toggle | `apps/web/app/login/` | Works |
| Protected chat screen with multi-turn conversation | `apps/web/app/chat/`, `apps/web/components/ChatPanel.tsx` | Works |
| Cited answers: query rewrite → hybrid search → rerank → LLM answer | `apps/api/app/chat.py` | Works |
| "Not found" reply when nothing in the rulebook is relevant | `apps/api/app/chat.py` (`NOT_FOUND_BELOW`) | Works |
| Questions about a named rule ("what does rule 23 say") fetch that rule directly | `apps/api/app/retrieval.py` | Works |
| Voice input via Sarvam AI speech-to-text | `apps/web/components/MicButton.tsx`, `apps/api/app/stt.py` | Works |
| "Listen" playback of an answer via Sarvam AI text-to-speech | `apps/web/components/SpeakButton.tsx`, `apps/api/app/speak.py` | Works |
| Clickable citations open the official English or Hindi PDF at the cited page | `apps/web/components/ChatPanel.tsx`, `apps/web/public/rules/` | Works |
| Apple Watch app: ask by voice, hear the answer, see the rule number and PDF page | `watch/Sarthi/` | Works. See [Apple Watch app](#apple-watch-app) |
| WhatsApp bot (Cloud API webhook): text and voice-note questions, answered with rule citations; voice notes also get a spoken reply | `apps/web/app/webhooks/whatsapp/route.ts`, `apps/api/app/tts.py` | Works |
| ElevenLabs phone agent that answers caller questions from the same rulebook | `apps/web/app/webhooks/elevenlabs/route.ts` | Works |
| Neo4j graph of rules and cross-references (optional) | `apps/api/scripts/index_graph.py` | Built |

## How it works

```
Browser ──► Next.js 16 (apps/web)  ──── x-bff-secret ────►  FastAPI (apps/api)
            · proxy.ts checks the                            · /chat  /stt  /speak  /tts  /health
              Supabase session                                     │
            · /api/chat, /api/stt,                                 ├─► Cerebras  (query rewrite + answer)
              /api/tts, /api/py/* forward to                       ├─► Qdrant Cloud (dense + BM25 hybrid)
              FastAPI server-side                                  ├─► fastembed, local (embeddings + reranker)
                                                                   └─► Sarvam AI (speech-to-text + text-to-speech, optional)
WhatsApp (Meta) ─► /webhooks/whatsapp (HMAC) ─ x-bff-secret ──►  FastAPI
ElevenLabs agent ─► /webhooks/elevenlabs (x-tool-secret) ─ x-bff-secret ──►  FastAPI
Apple Watch ─────────── x-bff-secret (direct, no Supabase) ──►  FastAPI
```

### Answering a question (`POST /chat`)

1. **Rewrite.** Cerebras turns the latest message (English, Hindi or Hinglish), plus up to 6 earlier turns, into one standalone English search query. Earlier turns are used only to resolve words like "it" or "that". The query keeps all numbers, units and rule numbers.
2. **Hybrid search.** The query is embedded locally with `BAAI/bge-small-en-v1.5` (dense) and `Qdrant/bm25` (sparse). Qdrant fetches the top 20 from each and merges them with Reciprocal Rank Fusion. Only chunks marked `answerable` are searched, so case-law "COMMENTS" blocks are never used as a source.
3. **Named rules.** If the query mentions a rule ("rule 42-A"), that rule's chunks are fetched directly and added to the candidates.
4. **Rerank.** A local cross-encoder (`Xenova/ms-marco-MiniLM-L-6-v2`) rescores the candidates and keeps the top 5.
5. **Not-found gate.** If the best rerank score is below `NOT_FOUND_BELOW` (default `-4`), the API returns a fixed "the rules appear to be silent on this" reply and does not call the LLM.
6. **Answer.** Cerebras writes a short answer (temperature 0) using **only** the 5 excerpts. It replies in the user's language, cites rules as **Rule N**, quotes the key wording exactly, and copies table numbers without estimating. The response includes `sources` (rule id, title, PDF page).

If `CEREBRAS_MODEL` fails, the API retries once with `CEREBRAS_FALLBACK_MODEL`. Reasoning models' `<think>` blocks are stripped from the output.

### WhatsApp bot (`/webhooks/whatsapp`)

1. **Verify.** Meta's `POST` is rejected unless its `x-hub-signature-256` matches `WHATSAPP_APP_SECRET`. The route returns `200` straight away and does the rest in the background (`after()`), so Meta does not retry.
2. **Text message.** The question goes to `/chat` as a single message. The reply's `**bold**` becomes WhatsApp `*bold*`, and a `Sources:` line lists the cited rules with their PDF pages. Replies are capped at 4,096 characters.
3. **Voice note.** The audio (up to ~30 s) is downloaded from Meta and sent to `/stt`. The transcript is answered like a text message and sent back in italics above the answer. The answer is then spoken by `/tts` in the detected language and sent as a voice note.
4. **Anything else** (images, stickers, etc.) gets a "please send text or a voice message" reply.

### Tech stack

| Layer | Choice |
|---|---|
| Frontend | Next.js 16.4 (App Router, Cache Components), React 19.3, Tailwind CSS 4, GSAP |
| Auth | Supabase (`@supabase/ssr`, httpOnly cookies) |
| Backend | FastAPI 0.115 on Python 3.12, managed with [uv](https://docs.astral.sh/uv/) |
| Vector search | Qdrant Cloud, with named vectors `dense` (384-dim) and `bm25` (sparse) in one collection |
| Embeddings / rerank | [fastembed](https://github.com/qdrant/fastembed), running locally on CPU (about 70 MB of models) |
| LLM | Cerebras Inference (OpenAI-compatible chat completions) |
| Speech | Sarvam AI `saarika:v2.5` for speech-to-text (automatic language detection) and `bulbul:v3` for text-to-speech |
| Monorepo | npm workspaces + Turborepo |

## Measured quality

These figures measure **search**: is the right rule among the 5 excerpts given to the LLM?

### Sample questions (`eval_retrieval.py`)

The 15 in-scope questions from [`reference/resources/sample_questions.md`](reference/resources/sample_questions.md):

| Method | Right rule in top 5 | Verified table in top 5 |
|---|---|---|
| Hybrid search only | 13 / 15 | 4 / 6 |
| Hybrid + MiniLM reranker (**used**) | **15 / 15** | **6 / 6** |

### Stress set (`eval_stress.py`, saved in `apps/api/data/stress_results.json`)

| Set | Right rule in top 5 | Ranked first |
|---|---|---|
| 28 harder English questions | 28 / 28 | 26 / 28 |
| 13 Hindi questions, searched as-is | 3 / 13 | 2 / 13 |
| Same 13 Hindi questions, translated to English by an LLM first | 11 / 13 | 10 / 13 |

**Out-of-scope questions** (6, e.g. "Who is the Chief Minister?", "How do I pay property tax online?") had a top rerank score of -5.3 or lower. The lowest in-scope question scored -3.65. The `-4` threshold falls between the two, which is the basis for the not-found gate.

Translating Hindi questions into English lifts search from 3 / 13 to 11 / 13, which is why `/chat` always rewrites the question into English first.

### Method

- The "right rule" for each question was set by the team from the rulebook. See `apps/api/data/golden_retrieval.json` and `golden_stress.json`.
- `golden_multilingual.json` holds 146 items covering the same intents in English, Hindi and Hinglish, plus edge cases.
- The 14 extracted tables were compared by hand against the rendered PDF pages.

## Getting started

### Prerequisites

- Node.js 20+ and npm 11
- Python 3.12 and [uv](https://docs.astral.sh/uv/)
- A [Supabase](https://supabase.com) project, with **Authentication → Email → "Confirm email" turned off** (otherwise sign-up shows a "confirmation is on" error)
- A [Qdrant Cloud](https://cloud.qdrant.io) cluster (the free tier is enough)
- A [Cerebras](https://cloud.cerebras.ai) API key
- *Optional:* a [Sarvam AI](https://www.sarvam.ai) key for voice, and a Meta WhatsApp Cloud API app

### Setup

```bash
# 1. Install dependencies
npm install
cd apps/api && uv sync && cd ../..

# 2. Create env files (git-ignored) and fill them in. See "Configuration" below.
cp apps/web/.env.example apps/web/.env.local
cp apps/api/.env.example apps/api/.env

# 3. Check that Qdrant is reachable (the Neo4j check is optional)
cd apps/api && uv run python -X utf8 scripts/check_connections.py

# 4. Index the rulebook into Qdrant once (downloads ~70 MB of models on first run; safe to re-run)
uv run python -X utf8 scripts/index_qdrant.py
cd ../..
```

### Run

```bash
npm run dev:api   # terminal 1 → FastAPI on http://127.0.0.1:8000
npm run dev:web   # terminal 2 → Next.js on http://localhost:3000
```

Open http://localhost:3000, sign up, and go to `/chat`.

To test on a phone on the same Wi-Fi, run `npm run dev:lan -w api` instead, which binds FastAPI to `0.0.0.0`.

> `-X utf8` matters on Windows, where the scripts print Hindi text and read UTF-8 files.

## Configuration

### `apps/web/.env.local`

| Variable | Required | Purpose |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | yes | Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | yes | Supabase public (anon) key |
| `BFF_SHARED_SECRET` | yes | Random string; must match the API's value |
| `BACKEND_URL` | yes | FastAPI base URL, e.g. `http://127.0.0.1:8000` |
| `DEMO_EMAIL`, `DEMO_PASSWORD` | no | One-click demo login on `/login`; the account is created on first use |
| `WHATSAPP_VERIFY_TOKEN` | WhatsApp only | Token Meta sends when you save the callback URL |
| `WHATSAPP_APP_SECRET` | WhatsApp only | Used to verify the `x-hub-signature-256` header |
| `WHATSAPP_ACCESS_TOKEN` | WhatsApp only | Graph API token for sending replies |
| `WHATSAPP_GRAPH_VERSION` | no | Defaults to `v23.0` |
| `ELEVENLABS_TOOL_SECRET` | voice agent only | Shared secret the ElevenLabs agent sends as `x-tool-secret` |

### `apps/api/.env`

| Variable | Required | Purpose |
|---|---|---|
| `BFF_SHARED_SECRET` | yes | Requests without it get `401` |
| `QDRANT_URL`, `QDRANT_API_KEY` | yes | Qdrant Cloud cluster |
| `QDRANT_COLLECTION` | yes | Collection name; defaults to `bhumi_rules` in the example |
| `CEREBRAS_API_KEY` | yes | Cerebras Inference key |
| `CEREBRAS_MODEL` | yes | Primary chat model |
| `CEREBRAS_FALLBACK_MODEL` | no | Used if the primary model fails |
| `NOT_FOUND_BELOW` | no | Rerank score threshold for "not found"; default `-4` |
| `SARVAM_API_KEY` | voice only | Sarvam speech-to-text, and text-to-speech for WhatsApp voice replies |
| `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE` | no | Only for `index_graph.py` / `check_connections.py` |

Generate a shared secret with `openssl rand -hex 32`. No API keys or secrets are committed to this repository.

## Data pipeline

All derived data lives in `apps/api/data/` and is built from `reference/` by the scripts in `apps/api/scripts/`. The scripts never modify `reference/`. The derived files are already committed, so you only need to re-run the pipeline if you change it.

```
reference/data/bhumi_vikas_rules.jsonl ──► clean_rules.py ──► rules_clean.jsonl ─┐
reference/resources/…_EN.pdf ──► extract_tables.py ──► tables_raw.json           │
                                   build_tables.py ──► tables.json ──────────────┤
                                                                                 ▼
                                                 chunk_rules.py ──► chunks.jsonl
                                                                                 │
                                                 index_qdrant.py ──► Qdrant  ◄───┘
                                                 index_graph.py  ──► Neo4j (optional)
```

| Step | Script | Output |
|---|---|---|
| Clean | `clean_rules.py` | `rules_clean.jsonl`: **106 rules → 111 records**. Removes running page headers, moves amendment footnotes into dated `amendments` metadata, and splits nested rules (12-A, 17-a, 25(A), 42-A, 93(a)) into their own records. Hidden page/footnote markers let later steps track pages. |
| Extract tables | `extract_tables.py` | `tables_raw.json`: ruled tables pulled from the priority PDF pages with pdfplumber |
| Curate tables | `build_tables.py` | `tables.json`: **14 priority tables** (rules 56, 21, 52, 42 Tables 4/5/5-A, 53), printed as in the PDF, with numeric `bands` for lookup tables |
| Chunk | `chunk_rules.py` | `chunks.jsonl`: **431 chunks** split along the rule structure (sub-rules, clauses, definitions, tables). Each chunk keeps its rule number, title, part, PDF page range and amendment dates, and is capped at ~1800 characters so bge-small does not truncate it. Case-law chunks are marked `answerable: false`. |
| Index | `index_qdrant.py` | Upserts dense + BM25 vectors. Point ids are derived from `chunk_id`, so re-running updates points in place. |

```bash
cd apps/api
uv run python -X utf8 scripts/clean_rules.py
uv run python -X utf8 scripts/extract_tables.py
uv run python -X utf8 scripts/build_tables.py
uv run python -X utf8 scripts/chunk_rules.py
uv run python -X utf8 scripts/index_qdrant.py
```

## Evaluation scripts

Run all of these from `apps/api` with `uv run python -X utf8 scripts/<name>`.

| Script | What it checks | LLM calls |
|---|---|---|
| `eval_retrieval.py [--rerankers]` | Sample questions: right rule / verified table in the top 5. `--rerankers` also compares MiniLM with `jina-reranker-v1-tiny-en`. | none |
| `eval_stress.py` | Harder English and Hindi questions, Hindi both as-is and translated; out-of-scope scores; whether a threshold separates them. Writes `data/stress_results.json`. | ~15 small |
| `eval_determinism.py` | Same input gives the same retrieval, the same model output at temperature 0, and the same normalised cache key. Takes about 2 minutes. | ~120 small |
| `build_multilingual_set.py` | Regenerates `data/golden_multilingual.json` | none |
| `check_connections.py` | Qdrant / Neo4j connectivity and counts (never prints secrets) | none |

## Apple Watch app

`watch/Sarthi/` is a SwiftUI watchOS app called "Sarthi" that lets the user ask by voice without using a keyboard.

<p align="center">
  <img src="apps/web/public/Watch_face.png" alt="Sarthi on Apple Watch: navy screen with a domed-pavilion mark, a cream mic button labelled &quot;Ask a rule&quot;, the transcribed question below it, and a Bhopal lakefront skyline along the bottom" width="260">
</p>

**Flow**

1. Tap the mic. The built-in watchOS recorder records up to 30 s (`.m4a`, wide-band speech).
2. The audio goes to `POST /stt`, which returns a transcript from Sarvam AI.
3. The transcript goes to `POST /chat` as a single-turn question.
4. The watch shows the rule number and title, the answer, and the source line *"MP Bhumi Vikas Rules 2012, PDF page N"*. It also reads the answer aloud with `AVSpeechSynthesizer`. The rule shown is the first one cited in the reply, and falls back to the first source if no rule is cited. If there are no sources, it shows "Not found".

**Requirements:** Xcode with the watchOS SDK. The deployment target is watchOS 27.0.

**Setup**

1. Create `watch/Sarthi/Sarthi Watch App/Secrets.swift`. It is git-ignored, so never commit it:

   ```swift
   import Foundation

   enum Secrets {
       /// Must equal BFF_SHARED_SECRET in apps/api/.env
       static let bffSharedSecret = "<same value as apps/api/.env>"
       /// Your Mac's LAN IP, used only on a real watch
       static let macLANAddress = "192.168.x.x"
   }
   ```

2. Start the backend:
   - **Simulator:** `npm run dev:api` is enough, because the simulator reaches `http://127.0.0.1:8000`.
   - **Real watch:** run `npm run dev:lan -w api` so FastAPI listens on `0.0.0.0:8000`. Keep the watch (or its paired iPhone) on the same Wi-Fi as the Mac.
3. Open `watch/Sarthi/Sarthi.xcodeproj` and run the **Sarthi Watch App** scheme.

## API reference

The browser only calls the Next.js routes. Every FastAPI route requires the `x-bff-secret` header, and the interactive docs (`/docs`, `/redoc`, `/openapi.json`) are turned off.

| Next.js route | Auth | Forwards to | Notes |
|---|---|---|---|
| `POST /api/chat` | Supabase session | `POST /chat` | JSON chat request |
| `POST /api/stt` | Supabase session | `POST /stt` | Raw audio bytes (has its own route because the generic proxy would corrupt binary data) |
| `POST /api/tts` | Supabase session | `POST /speak` | Reads an answer aloud; returns base64 WAV clips |
| `* /api/py/<path>` | Supabase session | `* /<path>` | Generic text/JSON passthrough, e.g. `/api/py/health` |
| `GET/POST /webhooks/whatsapp` | Meta verify token / HMAC signature | `POST /chat`, `POST /stt`, `POST /tts` | Sits outside `/api` so Meta can reach it without a session. Returns `200` immediately and replies in the background |
| `POST /webhooks/elevenlabs` | `x-tool-secret` header | `POST /chat` | Server tool for the ElevenLabs phone agent; returns plain-text answer + cited rule numbers |

### `POST /chat`

```jsonc
// request
{ "messages": [
    { "role": "user",  "text": "Mera ghar 10 meter chaudi road pe hai, aage kitni jagah chhodni padegi?" }
] }
// role is "user" or "model"; 1–40 messages, each ≤ 4000 chars; the last one must be from the user

// response
{ "reply": "…**Rule 56**…",
  "sources": [ { "rule_id": "56", "title": "…", "page": 77 } ] }
```

Errors: `400` if the last message is not from the user, `401` without the secret, `502` if the LLM is unavailable, `503` if the LLM is not configured.

### `POST /stt`

Send raw audio in the body (`content-type` such as `audio/webm`, up to 5 MB / ~30 s). The response is `{ "transcript": "…", "language_code": "hi-IN" }`.

### `POST /speak`

Send `{ "text": "…" }`. Markdown and tables are stripped, long answers are split on sentence boundaries, and the language is picked from the script (Devanagari → `hi-IN`, otherwise `en-IN`). The response is `{ "audios": ["<base64 WAV>", …] }`, played back to back by the Listen button. Errors: `503` if `SARVAM_API_KEY` is not set or Sarvam is busy, `502` if Sarvam fails.

### `POST /tts`

Send `{ "text": "…", "language_code": "hi-IN" }` (`language_code` is optional; unsupported or missing codes fall back to `hi-IN`). Markdown/WhatsApp formatting is stripped and long text is cut to 2,500 characters on a sentence boundary. The response is Ogg/Opus audio (`audio/ogg`), the format WhatsApp needs for voice notes. Errors: `503` if `SARVAM_API_KEY` is not set, `502` if Sarvam fails.

### `GET /health`

Returns `{ "status": "ok" }`.

## Security model

- **BFF gateway.** The browser never reaches FastAPI. (The Apple Watch app calls FastAPI directly with the shared secret.) Next.js checks the Supabase session with `getUser()`, which revalidates with Supabase on every call (`getSession()` is not trusted). It then calls FastAPI server-side with `BFF_SHARED_SECRET`, which FastAPI compares in constant time.
- **Route protection.** `apps/web/proxy.ts` (the Next.js 16 replacement for middleware) redirects signed-out users from `/chat` to `/login` and returns `401` for `/api/*`.
- **Cookies.** Session cookies are `httpOnly`, and `secure` in production.
- **WhatsApp.** Webhook requests are rejected unless their HMAC-SHA256 signature matches `WHATSAPP_APP_SECRET`.
- **ElevenLabs tool.** `/webhooks/elevenlabs` requires the `x-tool-secret` header, compared in constant time with `ELEVENLABS_TOOL_SECRET`.
- **No personal data** is stored by this app. Chat history is not saved on the server. Audio and questions are sent to Sarvam AI and Cerebras respectively.

## Deployment

`vercel.json` deploys `apps/web` as a Vercel service. The FastAPI backend is hosted separately (any container or VM host that can run `uvicorn app.main:app`), then set `BACKEND_URL` and `BFF_SHARED_SECRET` in the Vercel project. Turborepo passes `BACKEND_URL`, `BFF_SHARED_SECRET` and `WHATSAPP_*` through to the build.

The backend downloads about 70 MB of fastembed models on first request and loads them into memory, so give it at least ~1 GB of RAM.

```bash
npm run build   # turbo run build
npm run lint    # turbo run lint
```

## Project layout

```
.
├── apps/
│   ├── web/                      Next.js 16 frontend + BFF gateway
│   │   ├── app/
│   │   │   ├── page.tsx          landing page ("Sarthi")
│   │   │   ├── login/            sign-in / sign-up (server actions)
│   │   │   ├── chat/             protected chat screen
│   │   │   ├── api/chat/         → FastAPI /chat
│   │   │   ├── api/stt/          → FastAPI /stt (raw audio)
│   │   │   ├── api/tts/          → FastAPI /speak (Listen button)
│   │   │   ├── api/py/[...path]/ generic passthrough
│   │   │   ├── webhooks/whatsapp/ WhatsApp bot (text + voice notes)
│   │   │   └── webhooks/elevenlabs/ ElevenLabs phone agent server tool
│   │   ├── components/           ChatPanel, MicButton, SpeakButton
│   │   ├── public/rules/         official EN/HI rulebook PDFs (citation links)
│   │   ├── lib/supabase/         server-side Supabase client
│   │   └── proxy.ts              session check + route protection
│   └── api/                      FastAPI backend (Python 3.12, uv)
│       ├── app/
│       │   ├── main.py           app + shared-secret dependency
│       │   ├── chat.py           rewrite → retrieve → answer
│       │   ├── retrieval.py      hybrid search + rerank
│       │   ├── stt.py            Sarvam speech-to-text
│       │   ├── speak.py          Sarvam text-to-speech (browser Listen)
│       │   └── tts.py            Sarvam text-to-speech (WhatsApp voice replies)
│       ├── data/                 derived data + golden eval sets
│       └── scripts/              data pipeline, indexing, evaluation
├── reference/                    official Track 1 data pack (read-only)
├── watch/Sarthi/                 Apple Watch app (SwiftUI, watchOS)
├── turbo.json · vercel.json · package.json
└── LICENSE
```

## Data sources

- **`reference/`** is the official Track 1 data pack: rules as JSONL, full text by page, the English and Hindi PDFs, and sample questions, consolidated to 6 March 2025. Source: https://mptownplan.gov.in/NewsPDF/bvn06032025E.pdf. The English PDF is the source of truth. See [`reference/data/DATA_DICTIONARY.md`](reference/data/DATA_DICTIONARY.md).
- Everything in `apps/api/data/` is derived from that pack by the scripts above.
- No other data was added, no personal data is included, and no live government system was used.

### whatsapp qr

<img width="656" height="656" alt="whatsapp_hi_qr" src="https://github.com/user-attachments/assets/90be2a8c-9bca-4636-a0e7-ae7bc9614337" />

### Demo video of calling agent 

https://youtube.com/shorts/aXNzBXW7eJY?feature=share


### Demo video of the Watch  
https://youtu.be/zQ51H6ht0Bo?si=HhqQssPlog7aTL-F


*README last reviewed: 18 October 2026.*
