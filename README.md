# LoveLace

**Generate 3Blue1Brown-style math animations and explore interactive graphs, all in plain English.**

LoveLace turns natural language into math visuals. Describe a concept and it writes and renders an animated [Manim](https://www.manim.community/) video, in the style of 3Blue1Brown, that walks through the idea step by step. Describe a function and it plots it as a live 2D or 3D graph you can keep changing by talking to it. Around those two features is a multimodal AI teaching assistant: you can speak, sketch on a whiteboard, or type, and it answers using your own course materials.

> Built at **TreeHacks 2026** · [Devpost](https://devpost.com/software/lovelace-310caj) · [Live demo](https://tree-hacks-project.vercel.app)

---

## Why

Faced with an expression like

$$\int_0^\infty \left(\sum_{k=1}^\infty \frac{(-1)^k x^{2k}}{(2k)!}\right) e^{-\alpha x}\,dx$$

most students go straight to a chatbot for the answer. LoveLace is meant to build intuition instead. It works the way good office hours do: you think out loud, draw, make mistakes, and see the math move.

## Features

### 🎬 Natural language → Manim videos
- Describe a concept ("show why the derivative of sin is cos", "visualize a linear transformation") and Claude writes Python Manim code for it.
- The code renders asynchronously on a dedicated worker. The finished MP4 is stored in Vercel Blob and plays in the app.
- The prompts make each animation follow step-by-step tutorial reasoning, so the video teaches the idea rather than just looking good.

### 📈 Natural language → interactive graphs
- Plot and edit 2D and 3D graphs by describing them, from a simple line to a 3D sinusoidal surface.
- A custom agent layer on top of the **Desmos API** turns loose or imprecise requests into reliable changes to the graph.
- Change parameters by voice or text and watch how each one changes the curve.

### 🧑‍🏫 Multimodal AI teaching assistant
- **Voice**: real-time speech input with GPT-4o transcription, plus text-to-speech replies.
- **Whiteboard**: sketch or write out steps. A vision model reads your work and catches mistakes in partially correct reasoning.
- **Text**: a normal chat when you just want to type.

### 📚 Course-aware answers (RAG)
- Upload PDFs, lecture recordings, or notes. They are chunked, embedded, and indexed in Supabase.
- For each question, the most relevant chunks are retrieved and passed to the model, so answers use your class's notation, cite your materials, and stick to the syllabus.

---

## Architecture

```
┌──────────────┐   prompt    ┌─────────────────┐  Manim code  ┌──────────────────┐
│  Next.js UI  │ ──────────▶ │  /api/chat      │ ───────────▶ │  Render worker   │
│  (workspace) │             │  (Claude + RAG) │   job enq.   │  (Flask + Manim) │
└──────┬───────┘             └────────┬────────┘              └────────┬─────────┘
       │ poll /api/animation/jobs/[id]│ job JSON                       │ MP4 + callback
       ▼                              ▼                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    Vercel Blob  (manim-jobs/*.json, manim-renders/*.mp4)     │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Animation pipeline**

1. The user's prompt goes to `POST /api/chat` with `mode: "animation"`.
2. Claude generates Python Manim code (`lib/session/animation-prompt.ts`).
3. The app writes a render job to Blob (`manim-jobs/<jobId>.json`) and enqueues the worker.
4. The worker (`worker/render_worker.py`) renders the scene and calls `POST /api/animation/callback`.
5. The callback stores the video at `manim-renders/<jobId>.mp4` and marks the job completed.
6. The frontend polls `GET /api/animation/jobs/[id]` and plays the video.

### Tech stack

| Layer | Tools |
| --- | --- |
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS 4 |
| LLMs | Anthropic Claude (reasoning, Manim code generation), OpenAI GPT-4o / GPT-4o-mini (vision, prompt shaping) |
| Voice | OpenAI Realtime (WebRTC), `gpt-4o-transcribe`, Whisper, OpenAI TTS |
| Graphing | Desmos API with a custom NL → graph agent layer |
| Animation | Manim Community 0.19, Python, Flask render worker (Docker / Render) |
| Retrieval | OpenAI `text-embedding-3-small`, Supabase |
| Storage & hosting | Vercel, Vercel Blob |

### Project layout

```
app/
  workspace/          main workspace UI (chat, whiteboard, graphs, animations)
  session/            tutoring session view
  api/
    chat/             LLM orchestration (chat + animation modes)
    animation/        job status + worker callback
    realtime/         OpenAI Realtime session tokens
    tts/              text-to-speech
    upload/           course material ingestion
components/           Whiteboard, SpirographCanvas
lib/
  rag/                chunking, embeddings, hashing
  lecture/            transcription + ingestion
  whiteboard/         vision analysis of sketches
  session/            retrieval + animation prompt building
  supabase/  upload/  security/
worker/               Python Manim render worker
```

---

## Getting started

### 1. Install and run the web app

```bash
npm install
npm run dev
```

### 2. Environment variables

Create `.env.local`:

```bash
ANTHROPIC_API_KEY=...
OPENAI_API_KEY=...
BLOB_READ_WRITE_TOKEN=...

SUPABASE_URL=...
SUPABASE_SERVICE_ROLE_KEY=...

RENDER_WORKER_URL=https://your-render-worker.onrender.com
RENDER_WORKER_SECRET=your-shared-secret
RENDER_CALLBACK_SECRET=your-callback-secret
# optional, if the callback origin differs from the request origin
# RENDER_CALLBACK_URL=https://your-app-domain.com/api/animation/callback

# optional overrides
# OPENAI_REALTIME_MODEL=...
# OPENAI_TTS_MODEL=...
# OPENAI_TTS_VOICE=...
```

### 3. Deploy the Manim render worker

The worker needs this environment variable:

```bash
RENDER_WORKER_SECRET=your-shared-secret
```

On Render (or any Python host):

- **Runtime:** Python
- **Build command:** `pip install -r worker/requirements.txt`
- **Start command:** `python worker/render_worker.py`

---

## Roadmap

- **Faster renders**: GPU-accelerated Manim rendering, so videos come back close to real time.
- **Spanish language support**, to reach more learners.
- More graph types and longer, multi-scene animations.

## Team

Built at TreeHacks 2026 by **Aanika Tangirala**, **Om Shah**, **Hannah Cui**, and **Kailana Baker-Matsuoka**.

Originally developed at [aanikat07/TreeHacks-Project](https://github.com/aanikat07/TreeHacks-Project).
