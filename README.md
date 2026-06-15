# ReAct Agent

A full-stack **ReAct (Reason + Act)** AI agent that streams its internal reasoning loop — Thought, Action, Observation, and Final Answer — to a live web UI in real time.

---

## What It Does

The agent receives a natural-language question, then iteratively:
1. **Thinks** — reasons about what it knows and what it needs to find out
2. **Acts** — runs a web search if more information is needed
3. **Observes** — reads the search result and updates its reasoning
4. **Answers** — produces a final answer once it has enough information

Every step is streamed to the browser as it happens via Server-Sent Events (SSE).

---

## Architecture

```
Browser (React)
    │  POST /api/agent/run  (SSE stream)
    ▼
Express Server (Node.js / TypeScript)
    │  stdin JSON  ──►  stdout JSON-lines
    ▼
Python ReAct Agent
    ├── LLM: OpenAI gpt-4o-mini
    └── Tool: DuckDuckGo web search
```

### Request Flow

1. User enters an OpenAI API key and a question in the React UI.
2. The React frontend opens an SSE connection to `POST /api/agent/run`.
3. The Express server spawns `python3 run_agent.py`, passes the payload via stdin, and pipes stdout back as SSE `data:` events.
4. The Python agent runs the ReAct loop — calling OpenAI and (optionally) searching the web — printing one JSON event per step.
5. The frontend parses each event and renders it in the Execution Trace panel in real time.

---

## Project Structure

```
├── artifacts/
│   ├── api-server/               # Express 5 API (TypeScript)
│   │   └── src/
│   │       ├── index.ts          # Server entry point
│   │       ├── app.ts            # Express app setup
│   │       └── routes/agent.ts   # POST /api/agent/run — SSE bridge
│   └── react-agent-ui/           # React + Vite frontend
│       └── src/
│           ├── App.tsx           # App shell & theme provider
│           └── pages/home.tsx    # Main UI — SSE consumer & trace renderer
│
├── react_agent/                  # Python ReAct agent (do not modify)
│   ├── agent.py                  # ReActAgent class — core reasoning loop
│   ├── llm.py                    # OpenAI wrapper (gpt-4o-mini)
│   ├── search.py                 # DuckDuckGo web search tool
│   ├── run_agent.py              # stdin → stdout bridge called by Node
│   └── requirements.txt          # Python dependencies
│
├── lib/
│   ├── api-spec/                 # OpenAPI contract (source of truth)
│   ├── api-zod/                  # Zod schemas generated from OpenAPI
│   └── db/                       # Drizzle ORM schema + migrations
│
└── scripts/                      # Shared utility scripts
```

---

## Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, Tailwind CSS, Radix UI, Lucide Icons |
| Backend | Node.js 24, Express 5, TypeScript 5.9 |
| AI Agent | Python 3.11, OpenAI SDK (`gpt-4o-mini`) |
| Search | DuckDuckGo (via `requests`) |
| Streaming | Server-Sent Events (SSE) |
| DB / ORM | PostgreSQL, Drizzle ORM |
| API contract | OpenAPI 3.1, Orval codegen, Zod validation |
| Package mgmt | pnpm workspaces |

---

## Getting Started

### Prerequisites

- **Node.js 24+** and **pnpm**
- **Python 3.11+**
- An **OpenAI API key** (`sk-...`)
- A **PostgreSQL** database (connection string in `DATABASE_URL`)

### Install dependencies

```bash
# Node packages
pnpm install

# Python packages
pip install -r react_agent/requirements.txt
```

### Environment variables

Create a `.env` file (or set these in your environment):

```env
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
SESSION_SECRET=your-session-secret
```

### Run in development

```bash
# Start the API server (port 5000)
pnpm --filter @workspace/api-server run dev

# Start the React frontend
pnpm --filter @workspace/react-agent-ui run dev
```

Both services are reverse-proxied through a single shared proxy — the frontend is at `/` and the API is at `/api`.

### Other useful commands

```bash
# Full typecheck across all packages
pnpm run typecheck

# Regenerate API hooks and Zod schemas from the OpenAPI spec
pnpm --filter @workspace/api-spec run codegen

# Push DB schema changes (dev only)
pnpm --filter @workspace/db run push
```

---

## How the ReAct Loop Works

The agent prompt enforces a strict format:

```
Thought: <internal reasoning>
Action: Search[<query>]
```

or, when ready:

```
Thought: <final reasoning>
Final Answer: <answer>
```

Each iteration:
1. The LLM receives the full conversation history (system prompt + all prior thought/action/observation turns).
2. If it emits `Action: Search[...]`, the agent executes the search and appends the result as an `Observation:`.
3. The loop continues (up to 6 iterations by default) until `Final Answer:` is produced.

### SSE event types

| `type` | Description |
|---|---|
| `thought` | The model's internal reasoning step |
| `action` | A search query the agent is executing |
| `observation` | The search result returned |
| `final_answer` | The agent's concluded answer |
| `error` | Any error from the Python process |
| `done` | Stream is complete |

---

## Frontend Features

- **Dark terminal-style UI** with colour-coded step types
- **API key stored in `localStorage`** — no server-side key storage
- **Real-time streaming** — each step appears as Python emits it
- **Light / dark mode** toggle

---

## Python Agent Files (do not modify)

The core agent logic is intentionally kept separate and should not be modified:

- `react_agent/agent.py` — ReAct reasoning loop
- `react_agent/llm.py` — OpenAI chat completions wrapper
- `react_agent/search.py` — DuckDuckGo search tool
- `react_agent/main.py` — standalone CLI entry point

The `run_agent.py` bridge (reads stdin, writes stdout) is the only integration point with Node.js.

---

## License

MIT
