# AI PDF Chatbot

A full-stack application for uploading PDF documents and chatting with their contents using a **Retrieval-Augmented Generation (RAG)** pipeline — powered entirely by local AI models via Ollama.

---

## Architecture Overview

```
┌─────────────────────────┐         1. Upload PDFs          ┌─────────────────────────────┐
│  Frontend (Next.js)     │ ──────────────────────────────▶  │  Backend (LangGraph)        │
│  - React UI w/ chat     │                                  │  - Ingestion Graph           │
│  - Upload .pdf files    │  ◀──────────────────────────────  │  + Vector embedding via      │
│                         │         2. Confirmation           │    SupabaseVectorStore       │
└─────────────────────────┘                                  └─────────────────────────────┘
        (storing embeddings in DB)

┌─────────────────────────┐         3. Ask questions         ┌─────────────────────────────┐
│  Frontend (Next.js)     │ ──────────────────────────────▶  │  Backend (LangGraph)        │
│  - Chat + SSE stream    │                                  │  - Retrieval Graph           │
│  - Display sources      │  ◀──────────────────────────────  │  + Chat model (Ollama)       │
│                         │         4. Streamed answers       │                              │
└─────────────────────────┘                                  └─────────────────────────────┘
```

- **Supabase** is used as the vector store to store and retrieve relevant documents at query time.
- **Ollama** runs `llama3:8b` for chat and `nomic-embed-text` for embeddings — fully local, no external API keys needed.
- **LangGraph** orchestrates the "graph" steps for ingestion, retrieval, and generating responses.
- **Next.js** (React) powers the user interface for uploading PDFs and real-time chat.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | Next.js 14, TypeScript, Tailwind CSS, shadcn/ui, Vercel AI SDK |
| **Backend** | TypeScript, LangGraph, LangChain |
| **LLM** | Ollama — `llama3:8b` |
| **Embeddings** | Ollama — `nomic-embed-text` (768 dimensions) |
| **Vector Store** | Supabase — PostgreSQL + pgvector |

---

## Prerequisites

- [Node.js](https://nodejs.org/) v18+
- [Yarn](https://yarnpkg.com/) — `npm install -g yarn`
- [Ollama](https://ollama.com/)
- [Python 3.11+](https://python.org/)
- [LangGraph CLI](https://pypi.org/project/langgraph-cli/) — `pip install langgraph-cli`

---

## Getting Started

### 1. Clone and install

```bash
git clone <your-repo-url>
cd ai-pdf-chatbot

yarn install
cd backend && yarn install && cd ..
cd frontend && yarn install && cd ..
```

### 2. Pull Ollama models

```bash
ollama pull llama3:8b
ollama pull nomic-embed-text
```

Verify with `ollama list`.

### 3. Set up Supabase

1. Create a project at [supabase.com](https://supabase.com/)
2. Copy your **Project URL** and **Service Role Key** from **Settings → API**
3. Open the **SQL Editor** and create the following:

   - Enable the `vector` extension
   - A `documents` table with columns: `id` (bigserial), `content` (text), `metadata` (jsonb), and `embedding` (vector of **768** dimensions)
   - An **HNSW index** on the `embedding` column using `vector_cosine_ops`
   - A `match_documents` function that accepts `query_embedding` (vector 768), `match_count` (int, default 5), and `filter` (jsonb, default '{}') — returns matching rows ordered by cosine similarity
   - **Row Level Security** policies to allow read and insert access

   > Refer to `GUIDELINES.md` for the complete SQL script.

### 4. Configure environment variables

**`backend/.env`**

```env
SUPABASE_URL=https://your-project-id.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key-here
```

**`frontend/.env`**

```env
NEXT_PUBLIC_LANGGRAPH_API_URL=http://localhost:2024
# LANGCHAIN_API_KEY=
LANGGRAPH_INGESTION_ASSISTANT_ID=ingestion_graph
LANGGRAPH_RETRIEVAL_ASSISTANT_ID=retrieval_graph
LANGCHAIN_TRACING_V2=false
LANGCHAIN_PROJECT="pdf-chatbot"
```

> **Note:** `LANGCHAIN_API_KEY` should remain commented out for local development.

---

## Run

Open two terminals:

**Terminal 1 — Backend**

```bash
cd backend
yarn langgraph:dev
```

Wait for `Server running at ::1:2024`

**Terminal 2 — Frontend**

```bash
cd frontend
yarn dev
```

Wait for `Ready on http://localhost:3000`

Open [http://localhost:3000](http://localhost:3000), upload a PDF, and start chatting.

---

## Project Structure

```
ai-pdf-chatbot/
├── backend/                         # TypeScript
│   ├── src/
│   │   ├── ingestion_graph/         # PDF chunking + embedding pipeline
│   │   │   └── graph.ts
│   │   ├── retrieval_graph/         # Query → retrieve → respond pipeline
│   │   │   ├── graph.ts
│   │   │   ├── configuration.ts
│   │   │   ├── prompts.ts
│   │   │   └── state.ts
│   │   └── shared/                  # Supabase retriever, embedding config
│   │       └── retrieval.ts
│   ├── langgraph.json               # LangGraph server config
│   └── package.json
├── frontend/                        # TypeScript
│   ├── app/
│   │   ├── api/chat/route.ts        # Chat API endpoint
│   │   └── page.tsx                 # Main chat page
│   ├── components/                  # Chat UI, PDF viewer
│   ├── constants/
│   │   └── graphConfigs.ts          # Model selection
│   ├── lib/
│   │   └── langgraph-server.ts      # LangGraph client setup
│   └── package.json
└── README.md
```

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `LANGCHAIN_API_KEY is not set` | Keep it commented out in `frontend/.env` |
| `expected 1536 dimensions, not 768` | Re-run the Supabase SQL setup — ensure 768 dimensions |
| `Could not find function match_documents` | Re-run the Supabase SQL setup — ensure the `filter` parameter exists |
| `Connection refused :2024` | Run `yarn langgraph:dev` in `backend/` |
| `Connection refused :11434` | Run `ollama serve` |
| Chat ignores PDF content | Verify rows exist in Supabase → `documents` table |

