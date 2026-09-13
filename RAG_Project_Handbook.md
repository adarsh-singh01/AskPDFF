# RAG PDF Q&A Project — Documentation & Interview Preparation Handbook

> This handbook is built directly from your actual source code (`client/` and `server/`), not generic RAG theory. Every claim is traced to a real file/function. Where the code doesn't answer a question, this document says so explicitly instead of guessing.

---

## Table of Contents

1. Project Overview
2. Complete Technology Stack
3. Folder and File Structure
4. Complete System Architecture
5. PDF Upload & Ingestion Pipeline
6. PDF Processing (Extraction, OCR, Chunking)
7. Embeddings
8. Pinecone Vector Database
9. The Complete RAG Pipeline
10. Question-Answering Workflow (End-to-End Trace)
11. RAG vs. General Gemini Fallback
12. Authentication & Authorization
13. Appwrite Database Architecture
14. Conversation & Message Flow
15. Background Ingestion Worker
16. Document Lifecycle
17. Retention & Limits
18. API Documentation
19. Frontend Architecture
20. Streaming Architecture
21. Environment Variables & Configuration
22. Security Analysis
23. Performance & Scalability
24. Error Handling & Failure Scenarios
25. Important Design Decisions
26. Bugs, Fixes & Debugging Lessons
27. Debugging Methodology
28. Complete End-to-End Examples
29. RAG Concepts You Must Know
30. RAG Evaluation
31. Interview Pitches (30s / 2min / 5min / RAG-in-1min)
32. Interview Questions (Beginner / Intermediate / Advanced, 75 total)
33. Interview Follow-Up Chains
34. "Did You Really Build This?" Questions
35. Improvements You Can Mention
36. Testing Strategy
37. Deployment & Production Readiness
38. "If I Had to Rebuild It"
39. Cheat Sheet
40. Final Interview Cheat Sheet

---

## Chapter 1 — Project Overview

### What is this project, in plain language?

You built a website where a person can upload a PDF and then chat with it. They type a question, and the app tries to answer it **using the actual content of the PDF they uploaded**. If it can't find the answer inside the PDF, it falls back to a general-purpose answer from Google's Gemini AI model instead of just saying "I don't know."

### What problem does it solve?

Reading long PDFs to find one specific fact is slow. This app lets someone skip the reading and just ask.

### Who would use it?

Anyone who has a PDF (a research paper, a manual, notes, a contract) and wants direct answers instead of manually searching — the README frames it as useful for "research, study, or decision-making."

### What happens when a user uploads a PDF? (plain language)

1. The browser sends the PDF file to the backend.
2. The backend reads the text out of the PDF (or runs OCR if the PDF is a scanned image, not real text).
3. The backend chops the text into small overlapping chunks (~800 characters each).
4. The backend saves a record of the document and its chunks in a database (Appwrite), and creates a background "job" saying "please process this."
5. A background worker (running inside the same Flask process, on its own thread) picks up that job, converts each chunk into a vector (a list of numbers representing its meaning) using Google's embedding model, and stores those vectors in Pinecone (a vector database).
6. Once every chunk is stored, the document is marked `completed`.

### What happens when the user asks a question? (plain language)

1. The browser sends the question (plus the conversation ID) to the backend.
2. The backend checks: does this conversation have at least one `completed` document?
3. If yes: it turns the question into a vector too, and asks Pinecone "give me the 5 most similar chunks, but only chunks that belong to this user and this conversation."
4. Those 5 chunks get pasted into a prompt template along with the question and recent chat history, and that whole prompt is sent to Gemini.
5. Gemini's answer streams back to the browser piece by piece.

### What happens when the answer exists in the PDF?

Gemini is instructed (via the prompt in `ai_service.py::generate_rag_response`) to answer using only the given context. If it can, the browser shows that answer directly — this is the "RAG answer."

### What happens when the answer does NOT exist in the PDF?

The RAG prompt explicitly tells Gemini: *"If the answer is not in the provided context, just say, 'Answer is not available in the context'."* The backend (`routes.py::process_documents_without_voice`) checks the RAG output for that exact phrase. If found, it throws away that attempt and calls `generate_fallback_response()` instead — a second Gemini call with **no PDF context at all**, just the question and history. The user sees a prefix: *"I couldn't find relevant information in your uploaded documents. General Gemini response:"* followed by the general answer.

### Main features (from the actual code, not the README's aspirational list)

- Email/password authentication via Appwrite (`appwrite(service)/auth.js`)
- Google OAuth login via Appwrite's hosted OAuth flow (`account.createOAuth2Session`) — **you've said you are not using this right now**
- PDF upload (max 2 files per your frontend validation in `ChatInterface.jsx`, 5MB each — enforced again server-side)
- Automatic OCR fallback for scanned PDFs (`documents.py::_run_ocrmypdf`)
- Background, asynchronous ingestion (Appwrite `ingestion_jobs` + a worker thread in `ingestion_worker.py`)
- Semantic retrieval scoped to the current user AND the current conversation (Pinecone metadata filter)
- Streamed, incremental chat responses (NDJSON over a single HTTP response)
- Gemini fallback when nothing relevant is found in the PDF
- Per-user daily prompt limit (`user_service.py`, 100/day, resets on the calendar day)
- Conversation history with delete (which also purges chunks, jobs, documents, messages, and Pinecone vectors for that conversation)
- Voice input via the browser's native Web Speech API (`hooks/useSpeech.js`)
- Automatic retention/cleanup of old documents per user (`retention.py`)

### Major components

| Component | Technology | Role |
|---|---|---|
| Frontend | React + Vite | UI, auth state, chat, file upload, streaming render |
| Backend API | Flask | REST endpoints, auth verification, orchestration |
| Auth/DB | Appwrite | User accounts, JWTs, all relational data (conversations, messages, documents, chunks, jobs, limits) |
| Vector DB | Pinecone | Stores chunk embeddings; semantic similarity search |
| Embeddings | Gemini (`gemini-embedding-001`) | Turns text into vectors |
| LLM | Gemini (`gemini-3.5-flash-lite` in code — see Chapter 26 for why this is likely a bug) | Generates the final answer |
| Background worker | Python `threading.Thread` inside Flask | Converts pending chunks into vectors asynchronously |

### High-level architecture (actual, not generic)

```text
                         ┌──────────────────┐
                         │      User        │
                         └────────┬──────────┘
                                  │
                                  ▼
                     ┌────────────────────────┐
                     │  React + Vite Frontend  │
                     │ (client/src)            │
                     └────────────┬────────────┘
                                  │ fetch() + Bearer JWT
                                  ▼
                     ┌────────────────────────┐
                     │     Flask Backend       │
                     │   (server/api/routes.py)│
                     └───────┬───────┬─────────┘
                              │       │
                 ┌────────────┘       └─────────────┐
                 ▼                                   ▼
        ┌──────────────────┐                ┌──────────────────────┐
        │ Appwrite          │                │ Background Worker    │
        │ (Auth + Database) │                │ (ingestion_worker.py) │
        └──────────────────┘                └───────────┬───────────┘
                                                          │
                                    ┌─────────────────────┼─────────────────────┐
                                    ▼                                           ▼
                          ┌──────────────────┐                     ┌──────────────────┐
                          │ Gemini Embeddings │                     │ Pinecone Vector DB│
                          │ (gemini-embedding-001)                   │ (per-chunk vectors)│
                          └──────────────────┘                     └──────────────────┘

  Chat-time retrieval also goes:  Flask → Pinecone (similarity search) → Flask → Gemini LLM → streamed back to Frontend
```

---

## Chapter 2 — Complete Technology Stack

### Frontend

| Tech | Where it's used | Why | What breaks if removed |
|---|---|---|---|
| **React 18** | Whole `client/src` tree | Component-based UI | No UI framework |
| **Vite** | Build tool, `vite.config.js`, `npm run dev` | Fast dev server/build | No dev server |
| **React Router DOM v6** | `App.jsx` (`/`, `/login`, `/signup`, `/chat`, `/privacy-policy`) | Client-side routing | No page navigation |
| **Redux Toolkit** | `redux/store.js`, `authSlice.js`, `convoSlice.js` | Global state for auth (`userData`, `jwt`) and conversations/messages | Components would need prop-drilling or context |
| **shadcn/ui + Radix primitives** | `components/ui/*` (Button, Alert, Sheet, AlertDialog, Card, Input, Label) | Accessible, unstyled component primitives styled with Tailwind | Would need to hand-roll accessible dialogs/sheets |
| **Tailwind CSS** | `tailwind.config.js`, className usage everywhere | Utility-first styling | No styling system |
| **react-markdown + remark-gfm** | `ChatInterface.jsx` | Renders Gemini's Markdown-formatted answers (tables, lists, etc.) as real HTML | Bot answers would show raw Markdown syntax |
| **react-syntax-highlighter (dracula theme)** | `ChatInterface.jsx`'s custom `code` renderer for `ReactMarkdown` | Syntax-highlights fenced code blocks in answers | Code blocks would be plain text |
| **appwrite (JS SDK)** | `appwrite(service)/auth.js` | Talks to Appwrite Auth (`Account`) directly from the browser | No login |
| **Native Web Speech API** (`SpeechRecognition`/`webkitSpeechRecognition`) | `hooks/useSpeech.js` | Voice-to-text mic input | No voice input. Note: the package `react-speech-recognition` is installed in `package.json` but **not actually imported anywhere** — the hook talks to the browser API directly. |
| **@react-oauth/google** | Installed, imported nowhere | Dead dependency — Google login is done through Appwrite's own OAuth flow instead | No effect if removed |

### Backend

| Tech | Where | Why | What breaks if removed |
|---|---|---|---|
| **Flask 3.1** | `app.py`, `api/routes.py` | HTTP server, routing | No API |
| **Flask-Cors** | `app.py`: `CORS(app, supports_credentials=True, resources={r"/api/*": {"origins": os.getenv("FRONTEND_URL")}})` | Lets the Vite dev server (a different origin) call the API from the browser | Browser blocks cross-origin requests |
| **Flask-Limiter** | `app.py`: `Limiter(get_remote_address, app=app, default_limits=["60 per minute"])` | Basic global rate limiting by IP | No protection against request flooding |
| **gunicorn + gevent** | `requirements.txt`, `Dockerfile` CMD | Production WSGI server or the local `python app.py` dev server — one or the other actually runs the app | Not used in your local `python app.py` runs; relevant for the Docker/production path only |
| **python-dotenv** | `app.py`: `load_dotenv()` | Loads `server/.env` into `os.environ` | Env vars wouldn't load locally |
| **appwrite (Python SDK)** | `app.py`, `api/*.py` | Server-side Appwrite client (Databases, Account) | No database access |
| **LangChain (core, community, text-splitters, google-genai, pinecone)** | `documents.py`, `vector_store.py`, `ai_service.py` | `PyPDFLoader`, `RecursiveCharacterTextSplitter`, `GoogleGenerativeAIEmbeddings`, `ChatGoogleGenerativeAI`, `PineconeVectorStore` — a common interface over PDF loading, chunking, embeddings, chat models, and vector stores | Each of those would need to be hand-integrated against the raw Google/Pinecone SDKs |
| **google-generativeai** | Imported in `ai_service.py` (`import google.generativeai as genai`) but **`genai` is never actually called** in that file — all generation goes through `ChatGoogleGenerativeAI` from `langchain_google_genai` instead | Dead import in the current file | No effect if removed |
| **pypdf (via `PyPDFLoader`)** | `documents.py` | Extracts text from native (non-scanned) PDFs | No PDF text extraction |
| **unstructured[pdf]** | `requirements.txt` | Installed as a dependency, but `documents.py` does not import or call anything from `unstructured` — extraction is done entirely via `PyPDFLoader` | Currently unused in the actual pipeline |
| **tenacity** | `vector_store.py`: `@retry(...)` on `add_documents_with_retry` | Retries failed Pinecone writes with exponential backoff | Pinecone insert failures would not be retried |
| **threading** (`threading.Thread`, `threading.Lock`) | `vector_store.py` (singleton lock), `ingestion_worker.py` (worker thread + lock) | In-process background processing, thread-safety for lazy singletons | No async ingestion; embedding/vector-store objects could be re-initialized unsafely |

### AI / RAG

| Tech | Where | Purpose |
|---|---|---|
| **Google Gemini Embeddings** (`models/gemini-embedding-001`) | `vector_store.py::get_embedding_function` | Converts text (chunks and questions) into 3072-dimension vectors |
| **Google Gemini Chat model** (`gemini-3.5-flash-lite` as written in code) | `ai_service.py` | Generates both the RAG answer and the fallback answer |
| **LangChain `PyPDFLoader`** | `documents.py` | PDF → LangChain `Document` objects, one per page |
| **LangChain `RecursiveCharacterTextSplitter`** | `documents.py::split_documents` | Splits page text into 800-character chunks with 80-character overlap |
| **LangChain `PineconeVectorStore`** | `vector_store.py` | Wraps Pinecone as a LangChain vector store; exposes `.add_documents()` and `.as_retriever()` |

### Vector Database

| Concept | Actual value/behavior in this project |
|---|---|
| **Index** | Single index, name from `PINECONE_INDEX_NAME` env var |
| **Namespace** | Not used — this cannot be determined to be set anywhere in `vector_store.py`; the code never passes a `namespace` argument, so Pinecone's default namespace is used for everyone |
| **Vector ID** | The chunk's SHA-256 content hash (`chunkHash` / `ids_to_add` in `ingestion_worker.py`) — **not** a random UUID |
| **Metadata per vector** | `{chunk_hash, file_hash, document_id, conversation_id, user_id}` (set in `ingestion_worker.py::process_job`) |
| **Similarity search** | `store.as_retriever(search_kwargs={"k": 5, "filter": search_filter})` in `routes.py` |
| **Metadata filter used at query time** | `{"user_id": user_id, "conversation_id": conversation_id}` |

### Database / Backend Services

| Concept | Actual implementation |
|---|---|
| **Appwrite Auth** | Email/password (`account.createEmailPasswordSession`) and Google OAuth (`account.createOAuth2Session`) |
| **Appwrite Database** | 7 collections: `conversations`, `messages`, `user_limits`, `users`, `documents`, `chunks`, `ingestion_jobs` |
| **Queries used** | `Query.equal`, `Query.order_asc`, `Query.order_desc`, `Query.limit` — no `Query.orderBy` on multiple fields, no cursor pagination anywhere in the code |
| **Relationships** | Appwrite native "Relationship" attributes are **not confirmed** to be configured — `appwrite_utils.py::rel_id()` is written defensively to handle the value being a plain string OR a relationship-style dict, which suggests the author was unsure/flexible about this rather than committing to one schema design. This cannot be fully determined without inspecting the live Appwrite console schema. |

---

## Chapter 3 — Folder and File Structure

```text
RAG-main/
│
├── Dockerfile                     # Production container build (gunicorn + gevent + OCR system deps)
├── README.md                      # High-level (and partly outdated) description
│
├── client/                        # React + Vite frontend
│   ├── index.html
│   ├── vite.config.js
│   ├── tailwind.config.js
│   ├── package.json
│   └── src/
│       ├── main.jsx                       # React root, wraps <App> in Redux <Provider>
│       ├── App.jsx                        # <BrowserRouter> + 5 <Route>s
│       ├── config/conf.js                 # Reads every VITE_* env var into one object
│       ├── appwrite(service)/auth.js       # AuthService class: login/signup/logout/JWT/Google OAuth
│       ├── utils/apiClient.js             # apiFetch(): attaches a fresh JWT to every backend call
│       ├── redux/
│       │   ├── store.js
│       │   ├── authSlice.js               # { status, userData, jwt }
│       │   └── convoSlice.js              # conversations, activeConversationId, messages + 3 thunks
│       ├── hooks/
│       │   ├── useChat.js                 # Upload → poll status → stream chat — the core client logic
│       │   └── useSpeech.js               # Browser SpeechRecognition wrapper
│       ├── pages/LandingPg.jsx            # Marketing page; pings backend on mount
│       └── components/
│           ├── ChatInterface.jsx          # Main chat screen
│           ├── ConversationHistory.jsx    # Sidebar: list/select/delete conversations
│           ├── Signup.jsx                 # Shared login/signup form (email+password, Google button)
│           ├── ProcessingAnimation.jsx    # "Thinking..." bouncing dots
│           ├── PrivacyPolicy.jsx
│           ├── Home.jsx / HowItWorks.jsx / Navbar.jsx / Footer.jsx / Logo.jsx / Card.jsx  # Marketing/landing UI
│           └── ui/*.jsx                   # shadcn/ui primitives (button, input, alert, sheet, alert-dialog, card, label)
│
└── server/                        # Flask backend
    ├── app.py                     # App factory: CORS, rate limiter, Appwrite client, worker startup
    ├── requirements.txt
    └── api/
        ├── auth.py                # @auth_required decorator — verifies the user's JWT per request
        ├── routes.py              # Every HTTP route lives here
        ├── documents.py           # PDF loading, OCR fallback, chunking, hashing
        ├── vector_store.py        # Pinecone + Gemini embeddings singletons, retryable insert
        ├── ai_service.py          # Prompt construction + Gemini calls (RAG and fallback) + save message
        ├── user_service.py        # Prompt-limit accounting + conversation creation/update
        ├── appwrite_utils.py      # Status constants, rel_id() helper, Users-collection resolver
        ├── ingestion_worker.py    # Background thread: picks up ingestion_jobs, embeds, writes to Pinecone
        └── retention.py           # Per-user cleanup: keeps only the newest N completed docs
```

### File-by-file responsibility (non-trivial files only)

**`server/app.py`**
- Purpose: application factory + startup wiring.
- Calls: `CORS()`, `Limiter()`, builds a **privileged** Appwrite `Client` using `VITE_APPWRITE_API_KEY` (an API key, not a user session — this client has full admin rights over the database, bypassing per-document permissions), stores all collection IDs as both `app.*` attributes and module-level globals, registers `api` blueprint at `/api`, eagerly calls `get_vector_store()` once at boot (so the first real user request doesn't pay Pinecone/embedding client init latency), starts the ingestion worker thread.
- Called by: nothing — it's the entry point (`python app.py`).

**`server/api/auth.py`**
- `get_user_client(jwt_token)`: builds a **separate**, non-privileged Appwrite `Client` scoped to one user's JWT via `client.set_jwt(...)`.
- `auth_required(f)`: a decorator; reads `Authorization: Bearer <jwt>`, builds that user-scoped client, calls `Account(user_client).get()` (this is literally "ask Appwrite: is this JWT valid, and who is it?"), and injects the resulting Appwrite user dict as `kwargs['user']` into the wrapped route function. Returns HTTP 401 on any failure.
- Called by: every protected route in `routes.py` via the `@auth_required` decorator.

**`server/api/documents.py`**
- `_run_ocrmypdf`: shells out to the `ocrmypdf` CLI.
- `load_documents(file_array)`: orchestrates PyPDFLoader → OCR-fallback-if-needed → returns LangChain `Document`s.
- `split_documents`: `RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=80)`.
- `calculate_chunk_ids`: builds `"{source}:{page}:{index}"` IDs (used as `chunkId`, not the Pinecone vector ID — see Ch. 8).
- `compute_sha256_from_stream` / `compute_chunk_hash`: SHA-256 hashing for whole-file dedup and per-chunk content identity (the latter becomes the Pinecone vector ID).
- `build_chunk_records`: maps chunk objects → the exact dict shape the `chunks` Appwrite collection expects.
- `prioritize_chunks`: naive keyword-overlap sort of chunks vs. the user's prompt (see Ch. 6 for why this currently has limited effect).
- Called by: `routes.py::upload_documents`.

**`server/api/vector_store.py`**
- `get_embedding_function()` / `get_vector_store()`: thread-safe lazy singletons for the Gemini embedding client and the Pinecone-backed LangChain vector store.
- `add_documents_with_retry`: the only place vectors are actually written, wrapped in `tenacity.retry`.
- `reset_vector_store()`: nukes the **entire** Pinecone index (all users, all data) — admin-only.
- Called by: `routes.py` (retrieval, deletion), `ingestion_worker.py` (writes), `retention.py` (deletion).

**`server/api/ai_service.py`**
- `generate_rag_response` / `generate_fallback_response`: build the two prompt variants and call `ChatGoogleGenerativeAI(...).stream(prompt)`.
- `save_message_to_db`: persists one chat message (user or bot) to the `messages` collection.
- Called by: `routes.py::process_documents_without_voice`.

**`server/api/user_service.py`**
- `get_user_prompt_limit`: the daily-quota logic (100/day, resets by calendar date string comparison).
- `create_conversation` / `update_conversation_timestamp`.
- Called by: `routes.py`.

**`server/api/appwrite_utils.py`**
- Status string constants shared by `routes.py`, `ingestion_worker.py`, `retention.py`.
- `rel_id`: normalizes a value that might be a plain ID string, a relationship dict (`{"$id": ...}`), or a list of either.
- `get_or_create_user_document_id`: finds-or-creates a row in the `users` collection keyed by the Appwrite Auth user ID. **Its return value is currently unused for identity purposes** in the live `routes.py` (see Ch. 26) — it still runs (and still creates rows) but the ID it hands back isn't what gets stored as `userId` on documents/conversations anymore.

**`server/api/ingestion_worker.py`**
- The only background execution path in the project. See Ch. 15 in depth.

**`server/api/retention.py`**
- Per-user cleanup, triggered only as a side effect of a successful ingestion (`ingestion_worker.py::process_job` calls it at the end). See Ch. 17.

---

## Chapter 4 — Complete System Architecture

### Logical layers (as actually implemented)

```text
UI  (React components, hooks)
 ↓
API client  (apiClient.js::apiFetch, or raw fetch in convoSlice.js — see Ch. 19 for the inconsistency)
 ↓
Flask routes  (routes.py) — auth check, validation, orchestration
 ↓
Services  (documents.py, ai_service.py, user_service.py, vector_store.py, retention.py, ingestion_worker.py)
 ↓
Data / AI backends  (Appwrite Databases, Pinecone, Gemini)
```

### RAG-specific ingestion architecture

```text
PDF (werkzeug FileStorage)
 ↓
documents.py::load_documents        → PyPDFLoader, OCR fallback if scanned
 ↓
documents.py::split_documents       → RecursiveCharacterTextSplitter(800, 80)
 ↓
documents.py::calculate_chunk_ids   → deterministic "source:page:index" chunk IDs
 ↓
documents.py::build_chunk_records   → dict per chunk (chunkId, documentId, conversationId, userId, fileHash, chunkHash, text)
 ↓
Appwrite `chunks` collection         (text stored here, NOT in Pinecone)
 ↓
Appwrite `ingestion_jobs` collection (status=pending)   ← this is where the HTTP request ENDS (202 Accepted)
 ↓  (asynchronously, on the worker thread)
ingestion_worker.py::process_job
 ↓
vector_store.py::get_embedding_function  → gemini-embedding-001
 ↓
vector_store.py::add_documents_with_retry → PineconeVectorStore.add_documents(batch, ids=chunk_hashes)
 ↓
Pinecone index (vector + metadata only — NOT the raw text)
```

### Query-time (retrieval + generation) architecture

```text
User question
 ↓
routes.py::process_documents_without_voice
 ↓
Check: any COMPLETED document for (userId, conversationId)?
 ↓ yes
vector_store.py::get_vector_store().as_retriever(k=5, filter={user_id, conversation_id})
 ↓
Pinecone similarity search (question is embedded on the fly by the same LangChain retriever)
 ↓
Up to 5 chunk texts returned as LangChain Document objects
 ↓
ai_service.py::generate_rag_response(question, chunks, history)  → prompt string → Gemini.stream()
 ↓
If response contains "Answer is not available in the context":
    ai_service.py::generate_fallback_response(question, history) → Gemini.stream() (no context)
 ↓
NDJSON stream → HTTP response → frontend (hooks/useChat.js) → incremental UI render
```

A key point for interviews: **the raw chunk text lives in Appwrite's `chunks` collection, not in Pinecone.** Pinecone only ever receives the vector + a small metadata dict. But at *query* time, LangChain's retriever returns the matched vectors *with their original page_content* — this works because when the vectors were written (`ingestion_worker.py`), the LangChain `Document` objects passed to `add_documents` had `page_content=ch.get('text', ...)` set from the Appwrite chunk row, and Pinecone (via the LangChain integration) stores that text too, typically inside the vector's metadata, so it can be returned at query time without a second Appwrite lookup. **The exact internal storage format Pinecone uses for that text field is inside the LangChain/Pinecone SDK and is not fully visible in the project's own code — this cannot be determined beyond what's described here.**

---

## Chapter 5 — PDF Upload & Ingestion Pipeline (Deep Trace)

```text
User selects PDF in <ChatInterface.jsx>
        ↓
handleFileChange() — client-side check: is .pdf? ≤5MB? ≤2 files?
        ↓
useChat.js::submitChat() builds FormData(file, prompt, conversationId)
        ↓
apiFetch('/api/documents/upload', POST)         [utils/apiClient.js]
        ↓
Flask route: routes.py::upload_documents        [@auth_required]
        ↓
auth.py::auth_required — validates JWT, injects `user`
        ↓
appwrite_utils.py::get_or_create_user_document_id() — ensures a `users` row exists (side effect only — see Ch. 26)
        ↓
Per file: extension check (.pdf), size check (MAX_FILE_BYTES), documents.py::compute_sha256_from_stream()
        ↓
Conversation resolved: user_service.py::create_conversation() (new) or update_conversation_timestamp() (existing)
        ↓
Duplicate check: Query.equal(conversationId) + Query.equal(fileHash) on `documents` collection
        ↓  (not a duplicate)
documents.py::load_documents([file])            — PyPDFLoader (+ OCR if scanned)
        ↓
documents.py::split_documents()                 — 800 chars / 80 overlap
        ↓
Chunk-count guard: > MAX_CHUNKS_PER_DOC (70) → HTTP 400, upload rejected entirely
        ↓
documents.py::calculate_chunk_ids() + prioritize_chunks()
        ↓
Appwrite: create one `documents` row              {userId, conversationId, fileHash, fileName, status: "pending", chunkCount}
        ↓
Appwrite: create one `chunks` row PER CHUNK        {chunkId, documentId, conversationId, userId, fileHash, chunkHash, text}
        ↓
routes.py::create_ingestion_job()                  → Appwrite `ingestion_jobs` row {status: "pending", attempts: 1}
        ↓
HTTP 202 response: {message, conversationId, files: [{filename, status: "queued"|"skipped_duplicate"}]}
        ↓  ═══════════ HTTP REQUEST ENDS HERE ═══════════
        ↓  (background thread, unrelated to this HTTP request)
ingestion_worker.py — picks the job up on its next 5-second poll
        ↓
Marks job PROCESSING, marks document PROCESSING
        ↓
Reads all `chunks` rows for that document from Appwrite (NOT from the original upload request — a second read)
        ↓
vector_store.py::get_embedding_function() + get_vector_store()
        ↓
Batches of 16 chunks → add_documents_with_retry() → Pinecone
        ↓
Document marked COMPLETED {chunkCount, lastUsedAt}; job marked COMPLETED
        ↓
retention.py::enforce_retention_for_user() runs
```

### Why is ingestion separated from the HTTP request?

Embedding every chunk of a PDF (potentially dozens of Gemini embedding calls) and writing them to Pinecone can take several seconds to tens of seconds for a large file. If this ran synchronously inside `upload_documents`, the HTTP request would hang open that whole time, risking client/proxy timeouts and blocking that Flask worker thread from serving anyone else. By returning `202 Accepted` immediately after **persisting the raw chunks and a job record**, the frontend is freed to poll a cheap status endpoint (`/api/documents/status`) instead of holding one long connection open. This is a classic "accept now, process later" pattern — even though here it's implemented with a simple in-process polling thread rather than a real message queue (see Ch. 25 for the trade-off).

### Why chunk PDFs at all?

- Gemini's embedding model and LLM both have finite context. You can't usefully embed or retrieve "the whole PDF" as one unit — a whole-document vector would blur every topic in the file into one average, destroying the ability to find *specific* answers.
- Chunking to ~800 characters keeps each vector's meaning focused on a small, coherent slice of the document.

### Why overlap (80 chars)?

Without overlap, a sentence that happens to span a chunk boundary can be split so that neither half contains the complete idea, and semantic search on either half individually may miss it. An 80-character overlap (~10%) means boundary content usually appears whole in at least one of the two neighboring chunks.

### What happens if chunks are too small / too large? (reasoned from this implementation, not measured)

- Too small: more precise localization of an answer, but each chunk carries less context, so Gemini may receive a sentence fragment without enough surrounding meaning; also more chunks = more embedding calls, and the project's `MAX_CHUNKS_PER_DOC = 70` limit gets hit faster, causing more uploads to be rejected outright.
- Too large: retrieval becomes less precise (a big chunk may contain the answer buried among unrelated text, and the vector represents an "average" of everything in it), but fewer total chunks are needed.
- This project fixed both values (800 / 80) — there's no adaptive/dynamic chunking logic in the code.

### Duplicate detection

Two independent hash layers:
- **File-level**: `fileHash` (SHA-256 of the whole uploaded file) — used in `upload_documents` to skip re-processing an identical file already uploaded to the same conversation.
- **Chunk-level**: `chunkHash` (SHA-256 of just that chunk's text) — used as the **Pinecone vector ID itself**. This means if the exact same chunk text is ever embedded twice (e.g., the same paragraph appearing in two different documents), Pinecone's `add_documents(..., ids=[...])` will simply **overwrite** the existing vector at that ID rather than creating a duplicate — a natural, incidental form of content-level dedup, though it wasn't necessarily the primary design goal.

### Ingestion job retry behavior

- **Within one job's processing**: `add_documents_with_retry` retries the Pinecone write itself up to 5 times with exponential backoff (`tenacity`), for transient Pinecone/network failures during a single job's run.
- **Across job attempts**: `ingestion_worker.py::process_job` increments `attempts` every time it starts processing a job, but **nothing in the code re-queues a job that ends in `JOB_FAILED`** — once failed, it stays failed until an admin manually intervenes via `/api/admin/rebuild-index` (and even that route only re-queues currently-`COMPLETED` documents, not failed ones — see Ch. 26 for why this is a real gap).
- **Crash recovery**: `recover_stuck_jobs()` runs once at worker startup and resets any job stuck in `PROCESSING` (or the legacy string `"running"`) back to `PENDING`, so a Flask restart mid-ingestion doesn't leave jobs permanently stuck.

---

## Chapter 6 — PDF Processing

### Extraction strategy

`documents.py::load_documents` tries `PyPDFLoader` first (native text extraction — fast, no OCR needed). It decides a PDF is "scanned" (i.e., needs OCR) using **two heuristics**, either of which triggers OCR:

```python
if not pypdf_docs:
    is_scanned = True   # no pages/text at all
elif sum(len(doc.page_content) for doc in pypdf_docs) / len(pypdf_docs) < 500:
    is_scanned = True   # average <500 characters/page — likely an image-only PDF
```

If OCR runs (`_run_ocrmypdf`, using `ocrmypdf -l eng --output-type pdf --force-ocr`), the code then **re-extracts with `PyPDFLoader` on the OCR'd output** and compares total character counts: OCR's output is only used if it produced meaningfully more text (`total_chars_ocr > total_chars_pypdf * 1.1`, i.e., at least 10% more) than the original pass, or if the original pass produced nothing at all. Otherwise it silently keeps the original `pypdf` extraction. This protects against OCR occasionally producing garbage/noise on a PDF that actually had fine native text.

### What happens when extraction fails entirely?

If neither `pypdf` nor OCR produces any `Document` objects, `load_documents` returns an empty list for that file. Back in `routes.py::upload_documents`, `split_documents([])` yields no chunks, and the code returns:
```python
if not chunks:
    return jsonify({"error": f"{file.filename} contained no extractable text"}), 400
```
No `documents` row, no `chunks` rows, and no ingestion job are ever created for that file — it fails cleanly before touching the database.

### Chunking implementation

```python
RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=80,
    length_function=len,
    is_separator_regex=False,
)
```

- **What it does**: tries to split on paragraph breaks first, then sentences, then words, then raw characters — only falling back to a harder cut when a "nicer" boundary can't keep chunks under 800 characters. This is why it's "recursive" — it recurses through a priority list of separators.
- **Why used**: naive fixed-length slicing (`text[0:800]`, `text[800:1600]`, ...) frequently cuts mid-word or mid-sentence; `RecursiveCharacterTextSplitter` biases toward semantically cleaner breaks while still guaranteeing a hard size ceiling.
- **Concrete effect in this project**: for a typical text-heavy PDF page (roughly 2,000–3,000 characters), you'll get roughly 3–4 chunks per page, each overlapping its neighbor by 80 characters.

---

## Chapter 7 — Embeddings

### What is an embedding? (plain language)

A piece of text gets converted into a long list of numbers (a vector) such that texts with similar *meaning* end up as nearby points in that number-space — even if they don't share any of the same words. "The cat sat on the mat" and "A feline rested on a rug" would land close together; "The stock market crashed" would land far away.

### Why does RAG need embeddings?

Because the retrieval step needs to find chunks that are *relevant to the meaning* of the user's question, not just chunks that literally contain the same words. Embeddings are what make that possible.

### How THIS project generates embeddings

- **Model**: `models/gemini-embedding-001`, via `langchain_google_genai.GoogleGenerativeAIEmbeddings` (`vector_store.py::get_embedding_function`).
- **Dimension**: the code's own comment states this model *"Sends in 3072 dimension output no matter what you give in for output_dimensionality attribute"* — i.e., 3072-dimensional vectors regardless of any dimension parameter you might try to pass.
- **Where generated**:
  - **At ingestion time**: inside `PineconeVectorStore.add_documents(...)` — called from `ingestion_worker.py`. The LangChain `PineconeVectorStore` wrapper internally calls the embedding function on each chunk's text before writing to Pinecone. Your code never calls the embedding model directly by name at this point; it's hidden inside the vector store abstraction.
  - **At query time**: inside `store.as_retriever(...).invoke(user_prompt)` in `routes.py` — again, the retriever embeds the raw question text internally before doing the similarity search.
- **When**: ingestion-time embedding happens once per chunk, asynchronously, in the background worker. Query-time embedding happens synchronously, once per user question, inside the `/api/prompt/text-file` request.
- **How stored**: as vectors + metadata in the Pinecone index (see Ch. 8) — never persisted anywhere in Appwrite.
- **How later used**: Pinecone's nearest-neighbor search compares the question's fresh embedding against every stored chunk vector (scoped by the metadata filter) and returns the closest ones.

```text
Text Chunk ("The invoice due date is March 15, 2024...")
    ↓  GoogleGenerativeAIEmbeddings (gemini-embedding-001)
[0.0134, -0.221, 0.087, ... ]   ← 3072 numbers
    ↓
Pinecone  (stored with metadata: user_id, conversation_id, document_id, file_hash, chunk_hash)
```

### Why not keyword search?

Keyword search would only find "due date" if the user typed those exact words. If a user asks "when does the invoice need to be paid?", none of those words might literally appear in the PDF chunk, but the *meaning* overlaps heavily — which is exactly what embeddings capture and keyword search cannot.

---

## Chapter 8 — Pinecone Vector Database

### What Pinecone is / why used here

Pinecone is a managed vector database — it stores vectors and answers "find me the K vectors closest to this query vector" extremely fast, at scale, with optional metadata filtering. This project uses it purely as that similarity-search engine; it does **not** use Pinecone for anything relational (that's Appwrite's job).

### Index / namespace

- Index name comes from `PINECONE_INDEX_NAME`.
- **Namespace**: not used anywhere in `vector_store.py` — the code never sets one, so every vector for every user lives in the same default namespace of the same single index.

### Vector IDs

The Pinecone vector ID is the **chunk's content SHA-256 hash** (`chunkHash`), passed as `ids=ids_to_add` in `ingestion_worker.py`. This is deliberate and different from using a random UUID: it means re-embedding identical text is idempotent (same ID → overwrite, not duplicate).

### Metadata stored per vector

Set in `ingestion_worker.py::process_job`:

```python
meta = {
    'chunk_hash': chunk_hash,
    'file_hash': file_hash,
    'document_id': document_id,
    'conversation_id': conversation_id,
    'user_id': auth_user_id or user_id,
}
```

| Field | Why it exists |
|---|---|
| `user_id` | **Security-critical.** Used at query time to guarantee one user's question can never retrieve another user's chunks. |
| `conversation_id` | **Security/UX-critical.** Scopes retrieval to the current conversation only, so a chunk uploaded in Conversation A never leaks into Conversation B's answers — and lets `DELETE /api/conversations/<id>` cleanly wipe just that conversation's vectors (`store.delete(filter={'conversation_id': conversation_id})`). |
| `document_id` | Lets `retention.py::prune_document` delete exactly one document's vectors (`store.delete(filter={'document_id': doc_id})`) without touching siblings. |
| `file_hash` | Traceability back to the source file; not used in any retrieval filter in the current code, but present for debugging/auditing which physical file a vector came from. |
| `chunk_hash` | Same value as the vector's own ID, stored redundantly inside the metadata too — useful for debugging/logging without needing to separately track the ID. |

### Why filtering by `user_id` + `conversation_id` matters (security)

At query time (`routes.py`):
```python
search_filter = {"user_id": user_id, "conversation_id": conversation_id}
retriever = store.as_retriever(search_kwargs={"k": 5, "filter": search_filter})
```

If this filter were removed (or if `user_id`/`conversation_id` were ever written incorrectly at ingestion time), the top-5 similarity search would run across **every vector belonging to every user in the entire Pinecone index**. A malicious or simply curious user could ask a broad question and receive chunks of *someone else's uploaded PDF* as "context," which the LLM might then quote back to them — a direct, severe cross-user data leak. This filter is the *entire* access-control mechanism for retrieval; Pinecone itself has no concept of "this vector belongs to user X" beyond what's in that metadata dict and whether the query happens to filter on it.

```text
User Question
      ↓
Retriever (embeds question via gemini-embedding-001)
      ↓
Pinecone similarity search over ALL vectors in the index
      ↓
Metadata Filter: user_id == <this user>  AND  conversation_id == <this conversation>
      ↓
Top-5 most similar chunks (from that user+conversation's own vectors only)
```

### Similarity search, conceptually

Pinecone computes a distance/similarity score (commonly cosine similarity, though **the exact metric configured for this index cannot be determined from the code** — it's an index-level setting made when the index was created, not something set in `vector_store.py`) between the question's embedding and candidate vectors passing the metadata filter, and returns the `k=5` with the best scores.

---

## Chapter 9 — The Complete RAG Pipeline

**R**etrieval — finding the 5 most relevant stored chunks for a given question, scoped to the right user/conversation (Ch. 8).
**A**ugmentation — inserting those 5 chunks' raw text into the prompt sent to the LLM, so the LLM has that specific information available even though it wasn't in its training data.
**G**eneration — Gemini reads the augmented prompt and writes an answer.

```text
                 OFFLINE / INGESTION
                         │
                         ▼
                       PDF                     (documents.py::load_documents)
                         │
                         ▼
                    Text Chunks                (documents.py::split_documents, 800/80)
                         │
                         ▼
                    Embeddings                 (vector_store.py, gemini-embedding-001, inside add_documents)
                         │
                         ▼
                     Pinecone                  (vector + {user_id, conversation_id, document_id, file_hash, chunk_hash})
                         │
              ───────────┼───────────
                         │
                         ▼
                 ONLINE / QUERY
                         │
                   User Question               (routes.py::process_documents_without_voice)
                         │
                         ▼
                   Vector Search                (store.as_retriever(k=5, filter=...))
                         │
                         ▼
                 Relevant Chunks                (up to 5 LangChain Document objects)
                         │
                         ▼
              Prompt + Retrieved Context         (ai_service.py::generate_rag_response)
                         │
                         ▼
                      Gemini                     (ChatGoogleGenerativeAI(...).stream(prompt))
                         │
                         ▼
                     Final Answer                (streamed as NDJSON `rag_chunk` events)
```

### The actual RAG prompt template (`ai_service.py::generate_rag_response`)

```text
You are an expert RAG assistant that answers questions based on the provided documents and previous conversation.
Provide a detailed answer to the question based on the following context.
If the answer is not in the provided context, just say, "Answer is not available in the context".
Don't provide the wrong answer.

Previous conversation:
{history_str_prompt}

Context from documents:
{context}

Question:
{user_question}

Answer:
```

`context` is just `"\n\n".join(doc.page_content for doc in context_documents)` — the raw text of up to 5 chunks, concatenated with blank lines between them, no further ranking, deduplication, or citation markers.

---

## Chapter 10 — Question-Answering Workflow (End-to-End Trace)

```text
User types a question, clicks Send
 ↓
ChatInterface.jsx::onSubmit
 ↓
useChat.js::submitChat(input, files, callbacks)
 ↓
apiClient.js::apiFetch('/api/prompt/text-file', POST FormData{prompt, conversationId, history})
 ↓
routes.py::process_documents_without_voice   [@auth_required]
 ↓
auth.py::auth_required → user["$id"]
 ↓
appwrite_utils.py::get_or_create_user_document_id (side effect only, see Ch. 26)
 ↓
user_service.py::get_user_prompt_limit → 429 if exhausted
 ↓
user_service.py::create_conversation OR update_conversation_timestamp
 ↓
ai_service.py::save_message_to_db(sender_type="user")
 ↓
databases.list_documents(status=COMPLETED, userId, conversationId) — "is there anything to retrieve from?"
 ↓
  if yes → vector_store.py::get_vector_store().as_retriever(k=5, filter={user_id, conversation_id}).invoke(question)
 ↓
context_documents = [] or up to 5 chunks
 ↓
generate_stream() generator function begins:
    if context_documents: ai_service.py::generate_rag_response(...) → stream "rag_chunk" events
    if buffer contains "Answer is not available in the context":
        ai_service.py::generate_fallback_response(...) → stream "fallback_start" then "fallback_chunk" events
    ai_service.py::save_message_to_db(sender_type="bot", content=final_answer_for_db)
    yield final "metadata" event: {conversationId, promptsRemaining}
 ↓
Flask Response(generate_stream(), mimetype="application/x-ndjson")
 ↓
useChat.js reads response.body via ReadableStream reader, splits on "\n", JSON.parses each line
 ↓
Redux: addMessage({type:"bot", content: finalBotText}) once the stream ends
 ↓
ChatInterface.jsx re-renders with the new message (via useSelector on conversation.messages)
```

### What happens if retrieval returns zero documents?

Two distinct "zero" cases, both ending at the same fallback path but for different reasons:
1. **No completed documents at all** for this (userId, conversationId) — the `docs_ready.get("total", 0) > 0` check is false, so `context_documents` stays `[]` and retrieval (the Pinecone call) never even runs.
2. **Documents exist and are completed, but the retriever's top-5 search returns nothing relevant** — `docs = retriever.invoke(user_prompt)`; if `docs` is empty/falsy, `context_documents = []` too.

In **either** case, `if context_documents:` is false, so `rag_response_buffer = "Answer is not available in the context"` is set directly (Gemini is never even called for the RAG attempt), and the code immediately proceeds to the fallback branch.

---

## Chapter 11 — RAG vs. General Gemini Fallback

### How the switch is actually decided (inspected, not guessed)

```python
if context_documents:
    rag_stream = generate_rag_response(user_prompt, context_documents, history)
    for chunk in rag_stream:
        ...
        rag_response_buffer += chunk_content
else:
    rag_response_buffer = "Answer is not available in the context"

if "Answer is not available in the context" in rag_response_buffer:
    # switch to fallback
```

So the decision to fall back is a **combination**:
- If there were no context documents at all → the sentinel string is set directly (no Gemini call happens for the RAG path).
- If there *were* context documents but Gemini itself decided (per the prompt's own instruction) that the answer isn't in them → Gemini's own output contains that sentinel string, and the backend detects it via a plain substring check (`in`).

There is **no separate confidence score, retrieval-score threshold, or classifier** — it is purely: "did the RAG attempt's text happen to contain this exact phrase?" This means if a PDF's actual content ever legitimately contained the literal phrase "Answer is not available in the context" (extremely unlikely, but illustrative), the backend would incorrectly trigger a fallback. **This is a real, if minor, fragility of a substring-match approach.**

### RAG answer vs. general Gemini answer — why the distinction matters

- A RAG answer is grounded in the user's own uploaded content — it can reference specifics that only exist in that PDF.
- A general Gemini answer is Gemini's own knowledge, with **no guarantee of relation to the uploaded PDF at all**.
- The frontend visually marks this by prefixing the fallback text: `"I couldn't find relevant information in your uploaded documents. General Gemini response:\n"` — this is the *only* signal the user gets that they're looking at an ungrounded answer; there's no separate UI badge/color, it's baked into the streamed text itself.

---

## Chapter 12 — Authentication & Authorization

### Registration / Login / Logout (frontend: `appwrite(service)/auth.js`)

- `createAccount({email, password})` → `account.create(ID.unique(), email, password)`, then immediately calls `login()`.
- `login({email, password})` → first defensively tries `account.get()` + `deleteSession('current')` to kill any stale session, then `account.createEmailPasswordSession(email, password)`.
- `googleLogin()` → `account.createOAuth2Session(OAuthProvider.Google, `${BaseUrl}/chat`, `${BaseUrl}/login`)` — Appwrite handles the entire OAuth handshake; the app just gets redirected to `/chat` on success or `/login` on failure. **(You are not using this path right now.)**
- `logout()` → `account.deleteSession('current')`.
- `getCurrentUser()` → `account.get()` — returns `null` on any failure (used to check "am I logged in?").
- `getJWT()` → `account.createJWT()` — this is the short-lived token sent to the Flask backend on every API call.

### Backend verification (`server/api/auth.py`)

There is no backend-side "login" endpoint at all — the backend **never issues its own session or token**. It only ever *verifies* a JWT that Appwrite itself issued to the frontend:

```python
def get_user_client(jwt_token):
    user_client = Client()
    user_client.set_endpoint(...); user_client.set_project(...)
    user_client.set_jwt(jwt_token)
    return user_client

@auth_required
def decorated_function(*args, **kwargs):
    jwt_token = request.headers['Authorization'].split(' ')[1]
    user = Account(get_user_client(jwt_token)).get()
    kwargs['user'] = user
```

This means Appwrite is the actual source of truth for "is this token valid, and who does it belong to" — Flask just asks Appwrite that question on every protected request. There is no local JWT-signature verification, no session store, and no backend-side token blacklist.

### How the authenticated user's ID flows through the RAG pipeline

```text
Appwrite Auth session (browser)
       ↓  account.createJWT()
JWT string
       ↓  Authorization: Bearer <jwt>  (apiClient.js)
Flask: auth.py::auth_required → Account(user_client).get() → user["$id"]
       ↓  (this exact string, unchanged, in the current code)
routes.py: passed as `userId` when creating/querying `documents`, `conversations` rows
       ↓
documents.py::build_chunk_records(..., user_id=user["$id"], ...) → `chunks.userId`
       ↓
ingestion_worker.py::process_job → meta['user_id'] = auth_user_id or rel_id(job/doc userId)
       ↓
Pinecone vector metadata: user_id
       ↓
routes.py retrieval filter: {"user_id": user_id}  where user_id = user["$id"] from THIS request's auth
```

**Why consistent identity is critical**: the retrieval filter and the ingestion metadata must use the *exact same string* for the same person, every time, or a document ingested under one identity value will never be found by a query filtered on a different (even if logically "the same person") identity value. See Chapter 26 for a real historical instance of this exact class of bug in an earlier version of this codebase (the `users_doc_id` vs. `user["$id"]` inconsistency).

---

## Chapter 13 — Appwrite Database Architecture

| Collection | Purpose | Key fields (from actual code) | Created by | Read by | Updated by | Deleted by |
|---|---|---|---|---|---|---|
| `users` | Mirrors each Appwrite Auth user as an app-level row | `userId`, `email` (truncated to 30 chars!), `status`, `joined` | `appwrite_utils.py::get_or_create_user_document_id` | Same function (lookup) | Never | Never |
| `conversations` | One chat thread | `title` (truncated to 50 chars), `userId`, `lastMessageAt` | `user_service.py::create_conversation` | `routes.py` (list/get/ownership check) | `user_service.py::update_conversation_timestamp` | `routes.py::delete_conversation` |
| `messages` | One chat turn (user or bot) | `conversationId`, `senderType` ("user"/"bot"), `content`, `timestamp` | `ai_service.py::save_message_to_db` | `routes.py::get_messages` | Never | `routes.py::delete_conversation` (bulk wipe) |
| `documents` | One uploaded PDF | `userId`, `conversationId`, `fileHash`, `fileName`, `lastUsedAt`, `status`, `chunkCount` | `routes.py::upload_documents` | `routes.py` (status polling, retention) | `ingestion_worker.py` (status transitions), `retention.py` (status→FAILED on prune) | `routes.py::delete_conversation` (bulk), never individually except via prune's status flip |
| `chunks` | One text chunk of one document | `chunkId`, `documentId`, `conversationId`, `userId`, `fileHash`, `chunkHash`, `text` | `routes.py::upload_documents` | `ingestion_worker.py` (reads to embed) | Never | `retention.py::prune_document`, `routes.py::delete_conversation` |
| `ingestion_jobs` | One background-processing task | `userId`, `documentId`, `conversationId`, `fileHash`, `status`, `attempts`, `errorMessage` | `routes.py::create_ingestion_job` | `ingestion_worker.py` | `ingestion_worker.py` (status transitions) | `routes.py::delete_conversation` |
| `user_limits` | Daily prompt quota per user | `userId`, `promptCount`, `lastResetDate` | `user_service.py::get_user_prompt_limit` | Same function | Same function | Never |

### Relationship diagram (actual)

```text
Appwrite Auth User (identified by user["$id"])
 │
 ├── users            (1 mirror row, keyed by userId == Auth id)
 │
 ├── conversations    (many, keyed by userId)
 │       │
 │       └── messages         (many, keyed by conversationId)
 │
 └── documents        (many, keyed by userId + conversationId)
         │
         ├── chunks           (many, keyed by documentId + conversationId + userId)
         └── ingestion_jobs   (one per document, keyed by documentId + conversationId + userId)
```

**Note on the `users` collection**: it is populated but, in the currently pasted `routes.py`, its returned document ID (`users_doc_id`) is fetched and then simply **not used** anywhere as the actual `userId` value stored on `documents`/`conversations`/etc. — those all now consistently use the raw Appwrite Auth `user["$id"]` instead. The `users` collection is therefore currently more of a side-effect audit log than a foreign key target. This cannot be confirmed as intentional vs. leftover from a refactor — see Chapter 26.

---

## Chapter 14 — Conversation & Message Flow

1. **Conversation creation**: happens lazily — the very first upload or chat message with `conversationId == "null"` (a literal string, sent by the frontend when there's no active conversation yet) triggers `user_service.py::create_conversation`, which generates a fresh Appwrite `ID.unique()` document ID. That document's `$id` *is* the conversation ID from then on — there's no separate "conversation ID generator"; it's just Appwrite's own document ID.
2. **Document association**: every `documents`/`chunks`/`ingestion_jobs` row created during an upload carries that same `conversationId` string.
3. **User question**: sent with the current `conversationId` (or `"null"` for a brand-new chat).
4. **Message storage**: both the user's question and the bot's final answer are saved via `ai_service.py::save_message_to_db`, each as its own row in `messages`.
5. **Assistant response**: saved only once, at the *end* of the stream (`final_answer_for_db`), not chunk-by-chunk — so a message row always contains the complete final answer, never a partial one, even though the user sees it arrive incrementally.
6. **Conversation retrieval**: `GET /api/conversations` lists all conversations for `user["$id"]`; `GET /api/conversations/<id>` first re-verifies `convo["userId"] == user["$id"]` (ownership check) before returning that conversation's messages.

### Why `conversation_id` is used in the RAG metadata/filter

Without it, uploading a second, unrelated PDF in a brand-new chat would let that new chat's questions retrieve chunks from *every previous conversation that user ever had* — including PDFs that have nothing to do with the current topic, polluting answers with irrelevant context. Scoping by `conversation_id` keeps each chat's knowledge base isolated to what was actually uploaded *in that chat*.

---

## Chapter 15 — Background Ingestion Worker

### Why it exists

See Chapter 5 — to keep the upload HTTP request fast and to isolate potentially slow/retriable embedding work from the request/response cycle.

### Lifecycle

```text
app.py startup
    ↓
ingestion_worker.py::start_worker_once(app)
    ↓  (guarded by _worker_started + _worker_lock so it only ever starts once)
daemon Thread("ingestion-worker") begins _run()
    ↓
recover_stuck_jobs()            ← runs exactly once, at startup
    ↓
LOOP forever:
    load_pending_jobs(limit=3)  ← Query.equal(status=PENDING), order_asc($createdAt), limit 3
    if none: sleep(5)
    else: process_job(job) for each of up to 3, sequentially, on the SAME thread
```

There is exactly **one** worker thread, processing **up to 3 jobs per loop iteration, one at a time** (not in parallel) — this is a simple, single-consumer polling design, not a real task queue (no Celery/RQ/SQS).

### Job states

```text
PENDING  ──(worker picks it up)──▶  PROCESSING  ──(success)──▶  COMPLETED
                                        │
                                        └──(any exception)──▶  FAILED   (stays FAILED — no automatic retry of the whole job)

Also:  PROCESSING  ──(app crashes/restarts)──▶  recover_stuck_jobs() on next boot ──▶  PENDING (with errorMessage: "recovered after restart")

Legacy compatibility: a job found with the old status string "running" is also treated as stuck and reset to PENDING.
```

### Batch processing

Chunks for one document are embedded/upserted to Pinecone in batches of 16 (`BATCH_SIZE`), each batch retried up to 5 times independently (`add_documents_with_retry`). This bounds how much a single Pinecone API call has to handle and limits how much work is lost/retried if one batch fails partway through a large document.

### Failure handling

On any exception inside `process_job`'s `try` block, the job is marked `FAILED` with `errorMessage=str(exc)`, and a *second*, nested `try` attempts to also mark the parent `document` as `FAILED` (in case the first failure happened before the document row existed or was reachable — defensive double-try).

---

## Chapter 16 — Document Lifecycle

```text
(HTTP request) Uploaded, hashed, chunked, row created
   ↓
status = "pending"          (documents row created in routes.py::upload_documents)
   ↓  (worker picks up the matching ingestion_jobs row)
status = "processing"        (ingestion_worker.py::process_job)
   ↓  (all chunk batches embedded + written to Pinecone successfully)
status = "completed"         (chunkCount + lastUsedAt set)
   ↓
Used for Retrieval            (routes.py checks status == COMPLETED before allowing retrieval for this doc's conversation)
   ↓  (triggered automatically after every successful ingestion, for that user)
Retention / Cleanup           (retention.py::enforce_retention_for_user)
   ↓  (if selected for pruning)
status = "failed"             (retention.py::prune_document reuses the FAILED status to mean "removed")
   ↓
Chunks deleted from Appwrite + vectors deleted from Pinecone (document row itself is NOT deleted, just marked failed)
```

Alternative failure branch: `pending → processing → failed` (genuine ingestion error, `errorMessage` populated) — this looks *identical* in the `status` field to a retention-pruned document, which only has an empty `errorMessage` and no distinguishing flag. **This is a real ambiguity in the schema, not a guess** — both paths write the same string `"failed"`.

### Duplicate uploads

Re-uploading the exact same file (same `fileHash`) to the **same conversation** is always skipped as `"skipped_duplicate"` — regardless of whether the original document ultimately succeeded or failed. This means **a document that failed ingestion can never be retried by simply re-uploading the same file to the same conversation** — the duplicate check runs before any status check. The only ways to actually retry are: upload to a *different* conversation, or have an admin use `/api/admin/rebuild-index` (which, again, only targets already-`COMPLETED` docs, not failed ones — so today there is genuinely no built-in path to retry a failed ingestion for the same file+conversation combination).

---

## Chapter 17 — Retention & Limits

| Limit | Env var | Code fallback default | Your current `server/.env` value | Enforced where |
|---|---|---|---|---|
| Max file size | `MAX_FILE_BYTES` | 5,242,880 (5MB) | 5242880 | `routes.py::upload_documents` |
| Max chunks per document | `MAX_CHUNKS_PER_DOC` | 70 | 70 | `routes.py::upload_documents` |
| Docs kept per user (most recent) | `DOCS_KEEP_PER_USER` | 100 (code default in `retention.py`) | 3 | `retention.py::enforce_retention_for_user` |
| Max document age (days) | `DOCS_MAX_AGE_DAYS` | 100 (code default) | 10 | same |
| Chunk cap per user (across kept docs) | `CHUNK_CAP_PER_USER` | 8000 (code default) | 800 | same |
| Daily prompt quota | (hardcoded, no env var) | `MAX_PROMPTS_PER_DAY = 100` in `user_service.py` | n/a — not configurable via env | `user_service.py::get_user_prompt_limit` |
| Files per upload (client-side only) | n/a | 2, hardcoded in `ChatInterface.jsx::handleFileChange` | n/a | Frontend only — **not re-validated on the backend**, see Ch. 22 |

### Why these limits exist (reasoned from where they're enforced)

- `MAX_FILE_BYTES` / `MAX_CHUNKS_PER_DOC`: **cost + performance controls** — cap how much text one upload can force through the embedding pipeline and how many Gemini embedding calls one document can trigger.
- `DOCS_KEEP_PER_USER` / `DOCS_MAX_AGE_DAYS` / `CHUNK_CAP_PER_USER`: **storage + cost controls** — Pinecone and Appwrite storage both cost money and/or have plan limits; this keeps a single user's footprint bounded over time by automatically pruning old data.
- `MAX_PROMPTS_PER_DAY`: **cost + abuse control** — caps how many Gemini LLM calls (the most expensive part per-request) one user can trigger per day.

None of these are framed in the code as *security* controls in the sense of access control — they're resource/cost governance. The one genuine security-relevant limit missing entirely from the backend is a server-side cap on the **number of files per upload request** (see Ch. 22).

---

## Chapter 18 — API Documentation

### `GET /api/health`
- Auth: none. Request: none. Response: `{"status": "ok"}`. Purpose: liveness check.

### `GET /api/ping`
- Auth: none. Response: `{"status": "awake"}`. Called by `LandingPg.jsx` on mount to "wake up" a possibly cold/sleeping backend host.

### `GET /api/user/prompt-limit`
- Auth: required. DB: `user_limits`. Response: `{promptsRemaining, maxPrompts}`. Frontend caller: `useChat.js` (on mount / jwt change).

### `GET /api/conversations`
- Auth: required. DB: `conversations` (`Query.equal(userId)`, `order_desc($createdAt)`). Response: raw array of conversation documents. Frontend: `convoSlice.js::fetchConversations`.

### `GET /api/conversations/<conversation_id>`
- Auth: required. Validation: ownership check (`convo["userId"] != user["$id"]` → 403). DB: `conversations` (get one), `messages` (list, `order_asc($createdAt)`). Errors: 403 unauthorized, 500 on any other exception. Frontend: `convoSlice.js::fetchMessages`.

### `DELETE /api/conversations/<conversation_id>`
- Auth: required. Validation: ownership via `rel_id(conv.get('userId')) != user['$id']` → 403; 404 if the conversation doesn't exist at all. Logic: delete Pinecone vectors filtered by `conversation_id`, then wipe `chunks`, `ingestion_jobs`, `documents`, `messages` rows for that conversation, then delete the conversation row itself. Response: `{"success": true, "message": "Conversation and all orphans wiped completely"}`. Frontend: `convoSlice.js::deleteConversation`.

### `POST /api/documents/upload`
- Auth: required. Request: multipart `file` (one or more), `conversationId`, `prompt`. Validation: `.pdf` extension per file, `MAX_FILE_BYTES` per file; chunk count vs. `MAX_CHUNKS_PER_DOC`. DB: `documents`, `chunks`, `ingestion_jobs`, plus `conversations` (create/update). AI/Vector: none synchronously — only queues work. Response: `202 {message, conversationId, files: [{filename, status}]}`. Errors: `400` (no files / non-PDF / too large / no extractable text / too many chunks). Frontend: `useChat.js::submitChat`.

### `GET /api/documents/status?conversationId=<id>`
- Auth: required. DB: `documents` (`Query.equal(conversationId)`). Response: `{ready, pending, failed, total}`. Frontend: `useChat.js::submitChat` (polling loop, 2s interval).

### `POST /api/prompt/text-file`
- Auth: required. Request: form fields `prompt`, `conversationId`, `history` (JSON string). Validation: rejects if `files` are attached (must use the upload route instead); requires non-empty `prompt`; checks daily prompt quota (429 if exhausted). DB: `messages` (create user + bot), `conversations` (create/update), `documents` (readiness check). AI/Vector: Pinecone retrieval (conditional), Gemini generation (RAG and/or fallback). Response: streamed NDJSON (`Content-Type: application/x-ndjson`), event types `rag_chunk`, `fallback_start`, `fallback_chunk`, `error`, `metadata`. Frontend: `useChat.js::submitChat` (stream reader).

### `POST /api/admin/prune`
- Auth: **not** `@auth_required` — instead gated by `require_admin()` checking header `X-Admin-Token` against env `ADMIN_TOKEN`. Request: JSON `{userId?, conversationId?, fileHash?}` (at least one required). Logic: finds matching `documents` rows and calls `prune_document` on each. Response: `{"pruned": <count>}`.

### `POST /api/admin/rebuild-index`
- Auth: same `require_admin()` pattern. Request: JSON `{dropVectors?: bool}`. Logic: optionally wipes the **entire** Pinecone index, then re-creates an `ingestion_jobs` row (and resets status to `pending`) for every currently-`COMPLETED` document. Response: `{jobsCreated, droppedVectors}`.

---

## Chapter 19 — Frontend Architecture

### Entry point & routing

`main.jsx` mounts `<App/>` wrapped in a Redux `<Provider>`. `App.jsx` defines 5 routes: `/` (`LandingPg`), `/login` and `/signup` (both render the same `Signup` component with a different `variable` prop controlling copy/behavior), `/chat` (`ChatInterface`), `/privacy-policy`.

### State

Two Redux slices:
- `authSlice`: `{status, userData, jwt}` — set by `reduxLogin`/`reduxLogout` actions, dispatched from `Signup.jsx` (after email/password or Google auth) and `ChatInterface.jsx` (on mount, re-checking the session).
- `convoSlice`: `{status, conversations, activeConversationId, messages, error}` — driven by 3 async thunks (`fetchConversations`, `fetchMessages`, `deleteConversation`) plus synchronous reducers (`setActiveConversation`, `addMessage`, `setConversationId`, `startNewChat`).

### API communication — an actual inconsistency worth knowing for an interview

There are **two different patterns** for calling the backend in this codebase:
- `utils/apiClient.js::apiFetch` — calls `authService.getJWT()` fresh, **every single call**, to get a brand-new short-lived JWT, then attaches it as the bearer token. Used by `useChat.js` for prompt-limit, upload, status, and chat requests.
- `redux/convoSlice.js`'s three thunks — use raw `fetch()` directly and attach the JWT **already sitting in Redux state** (passed in as an argument), not a freshly-minted one.

This means if the Redux-stored JWT has expired (Appwrite JWTs are short-lived) but the user hasn't sent a chat message recently (which would refresh it via `apiFetch`), a stale JWT could be used for conversation list/fetch/delete calls, while chat itself would keep working because it always mints a fresh token. **This is a real architectural inconsistency in the code, not a guess about intent** — the two code paths are visibly different.

### File upload UI

`ChatInterface.jsx::handleFileChange` — client-side gate: rejects non-PDF or >5MB files (`alert(...)`), caps at 2 files. This mirrors (but is separate from and less strict/completely-required than) the server-side checks — a user could bypass the client-side check entirely (e.g., via `curl`) and rely solely on the backend's own validation, which is exactly why the backend re-validates independently.

### Streaming render

`useChat.js` reads the fetch response body as a stream (`response.body.getReader()`), decodes chunks, splits on newlines, and parses each complete line as one JSON event; `rag_chunk`/`fallback_chunk` content is appended to a local `streamingResponse` state that `ChatInterface.jsx` renders live via `<ReactMarkdown>` before the final message is committed to Redux.

### Document-status polling

After a successful upload, `useChat.js` polls `/api/documents/status` every 2 seconds in a `while (!isReady)` loop until `ready: true` or `failed > 0` (which throws and aborts the chat send).

---

## Chapter 20 — Streaming Architecture

### Why streaming

Waiting for Gemini to finish an entire answer before showing anything would feel slow, especially for longer answers. Streaming lets the UI show text as Gemini generates it — the perceived latency is much lower even though total generation time is the same.

### Backend side

`routes.py::process_documents_without_voice` defines an inner generator function `generate_stream()` and returns:
```python
Response(generate_stream(), mimetype="application/x-ndjson")
response.headers["Cache-Control"] = "no-cache"
response.headers["X-Accel-Buffering"] = "no"
```
Each `yield` inside the generator writes one line of JSON (`json.dumps(...) + "\n"`) to the HTTP response as soon as it's produced — this is **NDJSON** (newline-delimited JSON), not Server-Sent Events (`text/event-stream`) and not raw token streaming from Gemini directly to the socket; Flask streams the generator's yields as the HTTP response body incrementally. `X-Accel-Buffering: no` is a hint to disable any reverse-proxy response buffering (e.g., nginx) that would otherwise collect the whole response before forwarding it, defeating the purpose of streaming.

### Where the tokens come from

`ChatGoogleGenerativeAI(...).stream(prompt)` returns an iterator of chunk objects; `chunk.content` is the incremental text piece from Gemini for that step. The backend re-wraps each into its own NDJSON line.

```text
Gemini.stream(prompt)
  ↓ (LangChain chunk objects, one per generation step)
chunk.content
  ↓
json.dumps({"type": "rag_chunk", "content": chunk_content}) + "\n"
  ↓  (Flask yields this line into the open HTTP response immediately)
HTTP chunked-transfer response body
  ↓
useChat.js: reader.read() → decoder.decode() → buffer.split("\n") → JSON.parse each complete line
  ↓
setStreamingResponse(prev => prev + data.content)
  ↓
ChatInterface.jsx re-renders <ReactMarkdown>{streamingResponse}</ReactMarkdown> on every state update
```

### What happens if streaming fails partway through

- If Gemini itself throws mid-stream, the `try/except` inside `generate_stream()` catches it and yields one final `{"type": "error", "content": "An error occurred: ..."}` line — the stream still ends normally from an HTTP perspective (no broken connection), and the frontend's `switch(data.type)` appends `"\n\nError: ..."` to the message text.
- If the network connection itself drops, `useChat.js`'s `reader.read()` would throw, caught by the outer `catch` in `submitChat`, which restores the user's original input (`callbacks.onRestoreInput`) and shows an error bubble — unless it's an `AbortError` (the user navigated away or sent a new message, aborting the previous `AbortController`), which is silently ignored.

---

## Chapter 21 — Environment Variables & Configuration

**All values below are variable names and purposes only — no real secrets are reproduced.**

### `server/.env`

| Variable | Controls | Read in |
|---|---|---|
| `GOOGLE_API_KEY` | Auth for Gemini embeddings + chat model | `vector_store.py`, implicitly by `langchain_google_genai` |
| `VITE_APPWRITE_ENDPOINT` | Appwrite API base URL | `app.py`, `auth.py` |
| `VITE_APPWRITE_PROJECT_ID` | Which Appwrite project | `app.py`, `auth.py` |
| `VITE_APPWRITE_API_KEY` | **Privileged** server-side Appwrite credential — full database access, bypasses per-document permissions | `app.py` |
| `VITE_APPWRITE_DATABASE_ID` | Which Appwrite database | `app.py` |
| `VITE_APPWRITE_CONVERSATIONS_COLL_ID` / `_MESSAGES_COLL_ID` / `_USER_LIMITS_COLL_ID` / `_USERS_COLL_ID` / `_DOCUMENTS_COLL_ID` / `_CHUNKS_COLL_ID` / `_INGESTION_JOBS_COLL_ID` | Which collection is which | `app.py` |
| `PINECONE_API_KEY` | Auth for Pinecone | `vector_store.py` |
| `PINECONE_INDEX_NAME` | Which Pinecone index | `vector_store.py` |
| `FRONTEND_URL` | Allowed CORS origin for `/api/*` | `app.py` |
| `MAX_FILE_BYTES`, `MAX_CHUNKS_PER_DOC` | Upload limits | `routes.py` |
| `DOCS_KEEP_PER_USER`, `DOCS_MAX_AGE_DAYS`, `CHUNK_CAP_PER_USER` | Retention limits | `retention.py` |
| `ADMIN_TOKEN` | Shared secret for `/api/admin/*` routes | `routes.py::require_admin` |

### `client/.env`

| Variable | Controls | Frontend-safe? |
|---|---|---|
| `VITE_APPWRITE_PROJECT_ID`, `VITE_APPWRITE_ENDPOINT`, `VITE_APPWRITE_DATABASE_ID` | Which Appwrite project/DB the browser SDK talks to | Yes — these are meant to be public; Appwrite's browser SDK is designed to run with only these (no API key) |
| `VITE_APPWRITE_USERS_COLL_ID`, `VITE_APPWRITE_USER_LIMITS_COLL_ID` | Read in `conf.js`, but **never actually used** in any client component to call `databases.*` directly (confirmed: no `databases.` call exists anywhere in `client/src`) | n/a — effectively dead config on the client |
| `VITE_APPWRITE_COLLECTION_ID` | Read in `conf.js`, referenced nowhere else | Dead |
| `VITE_GOOGLE_OAUTH_CLIENT_ID` | Read in `conf.js`, referenced nowhere else — real Google OAuth config lives in the Appwrite Console, not here | Dead in current code |
| `VITE_EMAIL_ADDRESS` | Support email shown on `PrivacyPolicy.jsx` | Yes |
| `VITE_BACKEND_URL` | Base URL the frontend calls for all `/api/*` requests | Yes |
| `VITE_BASE_URL` | Used to build the Google OAuth redirect URLs (`${BaseUrl}/chat`, `${BaseUrl}/login`) | Yes |

### Dangerous configuration mistakes to watch for

- `VITE_APPWRITE_API_KEY` must **never** end up in `client/.env` or any `VITE_*`-prefixed variable that Vite would bundle into the browser — Vite exposes every `VITE_*` variable in the built JS bundle. It's correctly only in `server/.env` in this project, but the naming convention (`VITE_` prefix reused on server-side variables that mirror client ones) is a bit confusing and worth being careful with if this project is ever refactored.
- `ADMIN_TOKEN` is a single shared secret compared with plain `==` (`token == expected` in `require_admin`) — not a constant-time comparison, and not tied to any specific admin *user* identity; anyone who obtains this one string has full admin rights over `/api/admin/prune` and `/api/admin/rebuild-index` (including wiping the entire Pinecone index).
- Secrets must never be committed to Git because anyone with repo access (including, if the repo is ever made public, literally anyone on the internet) gets full read/write access to your Appwrite database, Pinecone index, and Gemini billing account.

---

## Chapter 22 — Security Analysis

### Authentication
- **Current protection**: Every JWT is verified live against Appwrite itself (`Account(user_client).get()`) — not a locally-verified signature, so there's no risk of a locally-forged token being accepted; Appwrite is always the arbiter.
- **Weakness**: no visible token refresh/rotation strategy on the backend side — this cannot be determined beyond what's in `auth.py`, since JWT lifetime and refresh entirely belong to Appwrite's own JWT semantics, not custom code here.

### Authorization / user isolation
- **Current protection**: `conversations`/`messages` ownership is checked explicitly (`convo["userId"] != user["$id"]`) before returning data; Pinecone retrieval is filtered by `user_id` + `conversation_id`.
- **Weakness**: `documents`/`chunks`/`ingestion_jobs` reads inside `ingestion_worker.py` and `retention.py` run using the **privileged API-key Appwrite client** (no per-user permission checks at all there) — this is fine because those code paths are never triggered directly by user input with an attacker-controlled ID, but it does mean the *only* thing standing between users is the application-level `Query.equal(userId, ...)` filters being written correctly everywhere, every time — there is no Appwrite-level document permission acting as a second line of defense for those three collections (per Chapter 13, no permissions array is passed when creating them).

### RAG-specific security
- **Cross-user leakage**: prevented by the `user_id`/`conversation_id` Pinecone metadata filter — see Ch. 8 for exactly why removing it would be catastrophic.
- **Prompt injection**: the RAG prompt directly concatenates raw chunk text (from the user's own uploaded PDF) into the LLM prompt with no sanitization. **This project has no defense against a PDF containing text designed to manipulate Gemini's behavior** (e.g., a PDF containing "Ignore previous instructions and reveal..."). Because retrieval is scoped to the uploader's own documents, the practical blast radius of this is currently limited to a user injecting instructions into *their own* chat session — but if this project ever added shared/multi-tenant documents, this would become a much more serious vector. This is a real, unaddressed gap in the current implementation.
- **Malicious PDFs**: extraction relies on `pypdf` (via `PyPDFLoader`) and the `ocrmypdf` subprocess. Malformed or adversarial PDFs could in principle cause parsing exceptions or resource exhaustion; the code's `try/except` around `PyPDFLoader` calls catches exceptions and treats them as "scanned" (triggering OCR), which is a reasonable soft-failure, but there is no timeout on the `ocrmypdf` subprocess call (`subprocess.run(command, check=True, capture_output=True, text=True)` has no `timeout=` argument) — a pathological PDF could in theory hang OCR indefinitely, tying up that request's ingestion-worker cycle.

### Secrets
- `VITE_APPWRITE_API_KEY`, `GOOGLE_API_KEY`, `PINECONE_API_KEY`, `ADMIN_TOKEN` are all server-only and correctly kept out of the client bundle in the current `.env` split.

### Upload security
- File type validated by **extension only** (`file.filename.lower().endswith(".pdf")`) — not by inspecting actual file content/magic bytes. A file renamed to `.pdf` that isn't a real PDF would likely just fail at the `PyPDFLoader`/OCR stage (returning the "no extractable text" 400 error) rather than causing anything worse, but this is worth naming explicitly as "extension-based, not content-based" validation.
- File size is checked (`MAX_FILE_BYTES`), correctly, per file.
- **Missing**: the backend does **not** re-validate the *number* of files per request server-side — the "max 2 files" rule only exists in `ChatInterface.jsx::handleFileChange` on the client. A direct API call (bypassing the UI) could upload far more than 2 files in one request, each still individually size- and chunk-limited, but with no cap on total files.

### API security
- **CORS**: locked to a single configured origin (`FRONTEND_URL`) for `/api/*` — correctly restrictive, not a wildcard.
- **Rate limiting**: global, IP-based, 60 requests/minute (`Flask-Limiter` default) — not per-user and not tuned per-endpoint (e.g., the expensive `/api/prompt/text-file` and the cheap `/api/health` share the same limit bucket by IP).
- **Input validation**: present for file type/size/chunk-count and for required `prompt` text, but there is no explicit length cap on the `prompt` string itself before it's embedded into a Gemini prompt, nor on the client-supplied `history` JSON array size (only the frontend voluntarily slices to the last 10 messages — `messages.slice(-10)` — nothing stops a direct API caller from sending an enormous `history` payload).
- **Error messages**: several routes return `str(e)` directly in the JSON error response (e.g., `documents_ingestion_status`, `get_messages`, `get_conversations`) — this can leak internal exception details (stack-trace-adjacent strings) to the client, which is a minor information-disclosure concern in a production setting.

### Database security
- Access control for the 4 collections that matter most for isolation (`documents`, `chunks`, `ingestion_jobs`, `users`) rests entirely on the backend's own `Query.equal(userId, ...)` filters, since (per Ch. 13) no Appwrite-level permissions array is set on those. `conversations`/`messages`/`user_limits` **do** get explicit Appwrite permissions (`Permission.read/update/delete(Role.user(user_id))`) at creation time, adding a genuine second layer of protection for those three specifically — **except** `save_message_to_db`'s permission logic has a bug (see Ch. 26) that means bot messages get no permissions at all, and user messages get permissions scoped to the literal string `"user"`, not the real user ID.

---

## Chapter 23 — Performance & Scalability

### Current characteristics (from the code, not measured benchmarks)

- **PDF extraction/chunking**: synchronous, inside the upload HTTP request — for a large or OCR-requiring PDF, this can make the upload request itself slow (OCR in particular, via a blocking `subprocess.run` call, is the most expensive synchronous step).
- **Embedding + Pinecone insertion**: asynchronous (background worker), batched at 16 chunks per Pinecone call, with retry.
- **Retrieval Top-K**: fixed at 5 (`k=5`) — no dynamic adjustment based on question type or result count.
- **Gemini latency**: not measurable from code; mitigated for the user's *perceived* latency by streaming, but total wall-clock generation time is unaffected by streaming.
- **Ingestion worker**: single thread, sequential processing of up to 3 jobs per 5-second poll cycle — this is the most obvious scalability bottleneck: **if 10 users each upload a document at once, they are all served by a single sequential worker loop**, not parallel workers.
- **Database queries**: no visible custom Appwrite indexes beyond whatever Appwrite creates automatically for equality queries — this cannot be confirmed either way from the code (index configuration is an Appwrite Console setting, not code).
- **Retention cleanup**: runs synchronously inside `process_job`, immediately after every successful ingestion, for that one user, adding to that worker cycle's total time.

### Bottlenecks (reasoned)

1. **Single ingestion worker thread** — a hard ceiling on ingestion throughput regardless of how many users upload concurrently.
2. **Synchronous OCR inside the upload request** — a large scanned PDF can make the client wait a long time just for the `202 Accepted` response.
3. **No caching layer** anywhere (no cache on embeddings, no cache on repeated identical questions, no cache on Pinecone results).
4. **IP-based global rate limiting** doesn't protect per-user fairness — one user hammering the API from many IPs, or many users behind one shared IP (e.g., a corporate NAT), interact with the limiter in ways that don't map cleanly to "per person" fairness.

### Scaling from 10 → 1,000 → 100,000 users (what would need to change — clearly separated from what exists today)

| Users | What exists today would... | What you'd need to change |
|---|---|---|
| 10 | Work fine — a single worker easily keeps up. | Nothing urgent. |
| 1,000 | The single worker thread becomes a queueing bottleneck during upload spikes; the single Flask process (even with gunicorn/gevent workers per the Dockerfile) shares that one in-process worker thread per process, so scaling Flask horizontally does **not** scale ingestion unless each process's worker also runs (leading to *multiple* competing workers polling the same `ingestion_jobs` table — which the current code has no locking/claiming mechanism to prevent double-processing of the same job across processes). | Move ingestion to a real task queue (Celery/RQ + Redis, or a cloud queue) with multiple independent workers; add row-level job claiming (e.g., an atomic "claim" update) to prevent two workers grabbing the same job. |
| 100,000 | Appwrite's per-collection query patterns (`Query.equal` scans) and Pinecone's single-index, single-namespace design would need real capacity planning; a single global 60/min rate limit bucket becomes meaningless at this scale. | Per-user rate limiting, Pinecone namespace-per-tenant or sharding strategy, Appwrite read replicas/caching, CDN for the frontend, horizontal Flask scaling behind a load balancer, dedicated worker fleet, observability/alerting. |

**What the current project does** vs. **what you would improve for production** is intentionally kept separate throughout this chapter — the current project is a correctly-functioning single-tenant-friendly prototype, not a horizontally-scaled production system, and that's a completely reasonable and expected state for a project at this stage.

---

## Chapter 24 — Error Handling & Failure Scenarios

| Failure | Where detected | Handling | DB state | User-visible behavior |
|---|---|---|---|---|
| PDF upload fails validation (wrong type/too big) | `routes.py::upload_documents` | Early `return jsonify(error), 400` | No rows created | Frontend `throw new Error("File upload failed")` → error message bubble |
| PDF parsing fails entirely | `documents.py::load_documents` (per-file try/except) | Returns empty `Document` list for that file | No rows created for that file (checked via `if not chunks`) | `400 {"error": "<file> contained no extractable text"}` |
| Empty PDF uploaded | Same as above (0 pages → `is_scanned=True` → OCR attempted → still empty) | Same | Same | Same |
| Embedding API (Gemini) fails during ingestion | `vector_store.py::add_documents_with_retry` | `tenacity` retries 5x with backoff; if still failing, exception propagates up | `ingestion_worker.py` catches it → job `FAILED`, document `FAILED` | User sees the document stuck as not-`ready` in the status-polling loop indefinitely until... the poll loop in `useChat.js` actually only breaks on `ready` or `failed > 0` — since `failed` would become `>0`, the frontend does surface `"Backend failed to process document."` |
| Pinecone fails (write) | Same path as above | Same | Same | Same |
| Pinecone fails (query/retrieval) | `routes.py`: `try/except` around the retriever call | `context_documents = []`, logs error, **silently falls through to the Gemini fallback** rather than surfacing an error to the user | No DB state change | User just gets a general Gemini answer with the "couldn't find relevant information" prefix — indistinguishable from a legitimate "no match found" case |
| Appwrite fails | Varies by route — most routes wrap DB calls in `try/except Exception as e: return jsonify({"error": str(e)}), 500` | Generic 500 | Depends on which call failed | Generic error surfaced to frontend, shown as an error bubble or thrown error |
| Gemini fails during chat generation | `routes.py::generate_stream`'s outer `try/except` | Yields `{"type": "error", "content": "An error occurred: ..."}` | The user's message was already saved before generation started; no bot message gets saved for this failed attempt | Error text appended inline to the (empty) bot response in the chat UI |
| Retrieval returns zero results | `routes.py` (`if docs: ... else: context_documents = []`) | Falls straight to fallback path (see Ch. 11) | No special DB state | User gets the fallback-prefixed general answer |
| Authentication fails / invalid token | `auth.py::auth_required` | `return jsonify({'error': 'Invalid or expired token', ...}), 401` | n/a | `apiClient.js` detects `401` → throws `"AUTH_FAILED"` → `useChat.js` catch calls `callbacks.onAuthFailure` → `ChatInterface.jsx::handleLogout` → redirected to `/login` |
| Duplicate PDF upload | `routes.py::upload_documents` (`Query.equal(conversationId, fileHash)`) | Skips that file, marks `"skipped_duplicate"` in the response, no new rows | No new `documents`/`chunks`/`ingestion_jobs` rows | Frontend still receives `202`, proceeds to poll status (which will reflect the *existing* document's status — completed or, per Ch. 16, permanently stuck failed) |
| Ingestion worker thread itself crashes | `ingestion_worker.py::_run`'s outer `try/except` around the loop body | Logs the error, sleeps 2s, loop continues — the *thread* itself doesn't die, only that iteration's unhandled exception is absorbed | Whatever job was mid-processing may be left in `PROCESSING` until the next app restart triggers `recover_stuck_jobs` | Silent from the user's perspective until/unless they're polling that specific document's status |
| Network request fails (frontend) | `apiClient.js` / `useChat.js` `catch` blocks | Distinguishes `AbortError` (silently ignored — intentional cancellation) from real errors (restores input, shows error message) | n/a | Error bubble with `error.message`, or nothing if it was an intentional abort |

---

## Chapter 25 — Important Design Decisions

| Decision | What was chosen | Reason (inferred from code structure) | Alternative | Trade-off |
|---|---|---|---|---|
| Vector DB | Pinecone (not the README's stated ChromaDB) | Managed service — no self-hosted vector index to operate; the `Dockerfile`'s leftover `CHROMA_PATH` suggests Chroma was the original plan before switching | ChromaDB (self-hosted, free, but you manage persistence/scaling yourself) | Pinecone costs money and adds an external dependency, but removes operational burden |
| Embeddings + LLM provider | Google Gemini for both | One vendor for both embeddings and generation — simpler credential/billing surface (`GOOGLE_API_KEY` covers both) | Could mix providers (e.g., OpenAI embeddings + Gemini LLM) | Locked into Gemini's specific embedding dimensionality (3072, fixed) and rate limits/pricing |
| Backend-as-a-service | Appwrite (Auth + DB) instead of a custom Postgres/Auth stack | Avoids building auth, JWT issuance, and a full CRUD API from scratch | Firebase, Supabase, or a hand-rolled Postgres + custom auth | Less control over query capabilities (no complex joins/relations exercised in the code) and you're tied to Appwrite's permission model |
| Background ingestion | An in-process Python `threading.Thread`, polling Appwrite | Simplest possible implementation — no extra infrastructure (no Redis/RabbitMQ/Celery) needed | Celery/RQ + Redis, or a cloud queue (SQS) | Doesn't scale across multiple server processes/instances without risking duplicate job processing (see Ch. 23); no dead-letter queue, no automatic job retry |
| Chunk-level content hashing as vector ID | SHA-256 of chunk text, not a random UUID | Enables idempotent re-embedding (same content → same ID → overwrite not duplicate) | Random UUIDs per chunk | Two *different* chunks that happen to have identical text (e.g., a repeated boilerplate paragraph across documents) would collapse to the same vector — acceptable here since metadata (`document_id`) still lets you scope/delete correctly per-document even if the underlying vector object is shared |
| Retrieval filtering by `user_id` + `conversation_id` | Both, together | Isolates data per-user (security) AND per-conversation (relevance/UX) | Could filter by `user_id` only (documents could be shared across a user's own conversations) | Current design means uploading the same PDF in two different conversations re-processes and re-stores it twice (only deduplicated at the file-hash level *within* a single conversation, not across conversations for the same user) |
| Streaming responses | NDJSON over a plain Flask `Response` generator | Simple to implement in Flask without extra libraries; works with a normal `fetch()` + `ReadableStream` reader on the frontend | Server-Sent Events (`text/event-stream`), WebSockets | NDJSON-over-HTTP is simpler than SSE/WebSockets but has no built-in reconnection/event-id semantics — a dropped connection loses the whole in-flight response with no resume capability |
| Two-tier fallback (RAG then general Gemini) via substring match | Detect `"Answer is not available in the context"` in the RAG output | Simple, no extra classifier/model call needed | A separate "did retrieval actually help?" scoring step, or a structured JSON response from the LLM instead of a magic string | Fragile — depends on the LLM reliably producing that *exact* phrase, and there's no safeguard against that phrase legitimately appearing in real PDF-derived content |

---

## Chapter 26 — Bugs, Fixes & Debugging Lessons

This chapter is deliberately honest — these are real inconsistencies visible in the code you provided, not invented flaws.

### Bug 1 — Invalid Gemini model name

**Where**: `server/api/ai_service.py`, both `generate_rag_response` and `generate_fallback_response`:
```python
model = ChatGoogleGenerativeAI(model="gemini-3.5-flash-lite", temperature=0.3)
```
**Why it's wrong**: as of this project's context, there is no publicly known Gemini model literally named `gemini-3.5-flash-lite` — this reads like a naming mix-up (likely intended to be a real `gemini-2.x-flash-lite`-style identifier).
**Symptom you would see**: an API error from Google/LangChain when this model string doesn't resolve to a real model, surfaced through the `try/except` in `generate_stream()` as a `{"type": "error", ...}` NDJSON event, or a synchronous exception if it fails before streaming starts.
**How to fix**: replace the string with a real, currently-available Gemini model ID from the Google AI Studio / Gemini API model list.
**Why the fix works**: the LangChain wrapper just passes this string through to Google's API — the moment it names a real model, the call succeeds.

### Bug 2 — Incorrect Appwrite permission role in `save_message_to_db`

**Where**: `server/api/ai_service.py::save_message_to_db`:
```python
permissions=[
    Permission.read(Role.user(sender_type)),
    Permission.update(Role.user(sender_type)),
    Permission.delete(Role.user(sender_type)),
] if sender_type != 'bot' else None
```
**Root cause**: `Role.user(...)` expects an **Appwrite Auth user ID**. `sender_type` is literally the string `"user"` or `"bot"` — never an actual user ID. So for a user message, this grants permissions to a role scoped to the literal string `"user"`, not to the real person who sent it; for a bot message, no permissions are set at all.
**Symptom**: none currently visible, **because** every read of the `messages` collection in `routes.py::get_messages` happens through the **privileged API-key Appwrite client**, which bypasses per-document permissions entirely — so this bug has zero functional impact today. It would only matter if some future code path tried to read `messages` using a user-scoped (JWT) Appwrite client instead of the server's API-key client.
**Fix**: `Role.user(user_id)` — the actual Appwrite Auth user ID would need to be passed into `save_message_to_db` (it currently isn't, so this would require adding a parameter).
**Debugging lesson**: this is exactly the kind of bug that "hides" in a codebase because the immediate code path never exercises the broken part — always ask *"who actually reads this data, and through which client?"* before trusting that written permissions are meaningful.

### Bug 3 — `users` collection lookup is now dead code for identity purposes

**Where**: `routes.py::upload_documents` and `routes.py::process_documents_without_voice` both call:
```python
users_doc_id = get_or_create_user_document_id(databases, db_id, users_collection_id, user["$id"], user.get("email"))
```
...but then every subsequent `userId` field written (`documents`, `chunks`, `ingestion_jobs`) uses `user["$id"]` directly (the raw Appwrite Auth ID), not `users_doc_id`. The `users_doc_id` variable is computed and then unused.
**Symptom this *would* cause if it were still relied upon and inconsistently mixed**: exactly the classic RAG identity bug —
```text
Appwrite Auth User ID  ──╮
                         ├─▶ If ingestion writes one and retrieval filters on the other:
Users-collection Doc ID ──╯     PDF ingestion succeeds, but retrieval returns ZERO results
                                 (Pinecone's user_id filter simply never matches anything)
```
This class of bug — **two different-but-plausible "identity" values used inconsistently between write-time and read-time** — is one of the most common real-world RAG debugging traps, and this project's history (an unused `get_or_create_user_document_id` call sitting right next to the code that no longer uses its result) is a living example of exactly how that class of bug gets introduced and then papered over: someone standardized on `user["$id"]` everywhere but didn't clean up the now-redundant lookup call.
**Current state**: not actively broken, because everything downstream consistently uses `user["$id"]`. But it's worth being able to explain in an interview *why* that lookup call still runs (it still creates/maintains `users` rows as a side effect, even though its return value goes unused) and that this is a sign of an incomplete refactor, not intentional design.

### Bug 4 — Permanently-failed document can never be retried via re-upload

Covered in depth in Chapter 16: the duplicate check (`fileHash` + `conversationId`) runs **before** any status check, so re-uploading a file that previously failed ingestion to the *same* conversation is always treated as `"skipped_duplicate"` — silently leaving the user stuck with a document that will never reach `completed`, no matter how many times they retry the same upload in that chat.
**Fix direction**: the duplicate check should also verify `status == DOC_COMPLETED` (or explicitly exclude `DOC_FAILED`) before deciding to skip.

---

## Chapter 27 — Debugging Methodology

### "The answer is in the PDF but RAG says it can't find it" — checklist

```text
1. Was the PDF actually uploaded successfully?
        → check the HTTP response from /api/documents/upload: did it say "queued" or "skipped_duplicate"?
        ↓
2. Was text actually extracted from the PDF?
        → check logs from documents.py::load_documents: "PyPDFLoader found no text..." / "assuming scanned PDF" / "Using native PDF content"
        ↓
3. Were chunks created?
        → check the `chunks` Appwrite collection for rows with the matching `documentId`
        ↓
4. Did ingestion complete?
        → check the `documents` row's `status` field: pending / processing / completed / failed
        → if `failed`, check the corresponding `ingestion_jobs` row's `errorMessage`
        ↓
5. Were embeddings actually generated and written to Pinecone?
        → check ingestion_worker.py logs: "[job:...] batch N complete" and "Ingestion complete, added N new chunks"
        ↓
6. Is the Pinecone metadata correct for those vectors?
        → (requires inspecting Pinecone directly, e.g. via its console/API) confirm `user_id` and `conversation_id`
          on the stored vectors match EXACTLY the values used in the retrieval filter
        ↓
7. Is `user_id` at query time the same string as `user_id` at ingestion time?
        → both should be `user["$id"]` from Appwrite Auth — verify neither side was accidentally
          swapped for a Users-collection document ID (see Bug 3, Chapter 26)
        ↓
8. Is `conversation_id` at query time the same as at ingestion time?
        → especially check the frontend didn't send "null" as a literal string when it should have sent
          the real conversation ID once one existed
        ↓
9. Does the readiness check even let retrieval run?
        → routes.py checks: is there at least one COMPLETED document for (userId, conversationId)?
          if that document is still `pending`/`processing`, retrieval is skipped entirely and you go
          straight to the fallback — this can LOOK like "RAG can't find it" when it's really
          "ingestion isn't done yet"
        ↓
10. Does the retriever actually return anything?
        → add a temporary log of `len(docs)` right after `retriever.invoke(user_prompt)` in routes.py
        ↓
11. Is the retrieved context actually reaching the Gemini prompt?
        → log `context` right before it's inserted into the RAG prompt in ai_service.py::generate_rag_response
        ↓
12. Is the fallback being triggered incorrectly?
        → check whether Gemini's own RAG-path answer happened to contain the literal phrase
          "Answer is not available in the context" even though it clearly had relevant context —
          this is possible if the model is uncertain or the context was borderline relevant
```

### Concrete logging examples already present in this project

```text
"PyPDFLoader found no text for {file_path}, assuming scanned PDF"      → extraction-stage evidence
"OCR successful for '{filename}'"                                       → OCR-stage evidence
"[job:{job_id}] processing batch 1/3 (16 chunks)..."                    → embedding-stage evidence
"[job:{job_id}] Ingestion complete, added {N} new chunks"                → ingestion-completion evidence
"Vector retrieval failed: {e}"                                          → retrieval-stage failure evidence (routes.py)
```

---

## Chapter 28 — Complete End-to-End Examples

### Example 1: Question answered from the PDF

1. User uploads `invoice.pdf`. `documents.py::load_documents` → native text extracted (not scanned) → `split_documents` yields, say, 12 chunks.
2. `documents` row created: `{status: "pending", chunkCount: 12}`. 12 `chunks` rows created. 1 `ingestion_jobs` row created (`status: "pending"`). HTTP `202` returned.
3. Frontend polls `/api/documents/status`; within a few seconds the worker has embedded all 12 chunks in one batch (`BATCH_SIZE=16` covers all 12 in a single Pinecone call) and marked the document `completed`. Status endpoint now returns `{ready: true}`.
4. User asks: *"What is the total amount due?"*
5. `routes.py` confirms a `completed` document exists for this (userId, conversationId) → runs `store.as_retriever(k=5, filter={user_id, conversation_id}).invoke(question)` → Pinecone returns the 5 chunks whose embeddings are closest to the question's embedding — likely including the chunk containing the invoice total.
6. `ai_service.py::generate_rag_response` builds the prompt with those 5 chunks' text as `context` and streams Gemini's answer, e.g., *"The total amount due is $482.10."*
7. That answer never contains the sentinel phrase, so no fallback happens. `save_message_to_db` persists the bot message. Frontend renders it.

### Example 2: Question unrelated to the PDF

1. Same uploaded `invoice.pdf`, already `completed`.
2. User asks: *"What's the capital of France?"*
3. Retrieval still runs (a completed document exists), and Pinecone will still return its top-5 "closest" chunks — vector search always returns *something* if there are any vectors at all (it doesn't return "nothing" just because the match is poor; there's no similarity-score threshold in this code, only a fixed `k=5`).
4. Those irrelevant invoice chunks get pasted into the RAG prompt regardless. Gemini reads the prompt, sees no genuine connection between "capital of France" and invoice content, and — because the prompt explicitly instructs it to say so — replies with (or a response containing) `"Answer is not available in the context"`.
5. `routes.py` detects that phrase in `rag_response_buffer` → discards it → calls `generate_fallback_response("What's the capital of France?", history)` with no document context at all.
6. Frontend shows: *"I couldn't find relevant information in your uploaded documents. General Gemini response:\nParis is the capital of France."*

**Interview-relevant subtlety**: retrieval in this project has **no relevance-score cutoff** — it always returns its top-K nearest vectors even if none of them are meaningfully related to the question. The actual "is this relevant?" judgment is fully delegated to Gemini's own reading of the prompt, not to any numeric similarity threshold in the code.

---

## Chapter 29 — RAG Concepts You Must Know for Interviews

| Concept | Simple explanation | Technical explanation | In THIS project | Interview question |
|---|---|---|---|---|
| RAG | Look things up before answering | Retrieval step supplies external context to an LLM's generation step at inference time | Fully implemented — Pinecone retrieval + Gemini generation | *"Walk me through your RAG pipeline end to end."* |
| Embedding | Turning text into a "meaning fingerprint" of numbers | A dense vector produced by an embedding model such that semantic similarity ≈ vector proximity | `gemini-embedding-001`, 3072 dims | *"Why can't you just store raw text and search it directly?"* |
| Vector database | A search engine specialized for "find similar vectors" | Approximate/exact nearest-neighbor search over high-dimensional vectors, often with metadata filtering | Pinecone | *"Why Pinecone instead of a SQL `LIKE` search?"* |
| Semantic search | Search by meaning, not exact words | Nearest-neighbor search in embedding space | The retrieval step in `routes.py` | *"How does semantic search differ from keyword search here?"* |
| Chunking | Cutting a document into bite-sized pieces before storing | Splitting text so each unit is small enough to embed meaningfully and retrieve precisely | `RecursiveCharacterTextSplitter(800, 80)` | *"Why 800 characters specifically? What would happen with 8,000?"* |
| Chunk overlap | Slightly repeating text between neighboring chunks so nothing gets cut off badly | Overlap preserves boundary-spanning context across adjacent chunks | 80 characters (~10%) | *"What breaks if overlap is 0?"* |
| Top-K | "Give me the best 5 matches" | The retriever returns the `k` nearest vectors by similarity | `k=5`, fixed, no threshold | *"What if K were 1? What if K were 50?"* |
| Metadata filtering | Only search within a specific slice of the database | Pinecone query-time filter restricting candidates before/while ranking by similarity | `{user_id, conversation_id}` | *"How do you prevent User A from retrieving User B's data?"* |
| Context window | How much text an LLM can "see" at once | A hard limit on prompt length | Not explicitly managed in code — `context = "\n\n".join(...)` of up to 5 chunks is assumed to fit | *"What would you do if your context ever risked exceeding the model's context window?"* — **Potential future improvement**, not implemented |
| Grounding | Making sure the answer is backed by real retrieved evidence, not the model's own guesses | The RAG prompt explicitly instructs the model to only use provided context | Implemented via prompt instruction only — no automated fact-checking of the final answer against the context | *"How do you know the answer is actually grounded and not hallucinated?"* — honest answer: *"Right now, I trust the prompt instruction; I don't have an automated grounding check."* |
| Hallucination | The model confidently making things up | Generating plausible but unsupported/incorrect content | Mitigated only by the prompt instruction + fallback labeling; **not measured or prevented at the system level** | *"How would you detect hallucination in your fallback answers?"* |
| Prompt injection | Malicious instructions hidden inside content the model reads | Text embedded in retrieved context that tries to override the system prompt's instructions | **Not implemented/defended against** — see Ch. 22 | *"What if a PDF contained 'ignore all previous instructions'?"* |
| Hybrid search | Combining keyword search + vector search | Not present here | **Not implemented** — retrieval is pure vector similarity | *"Have you considered hybrid search? Why/why not?"* |
| Reranking | A second pass that re-scores the initially retrieved results more carefully | Not present here | **Not implemented** — Pinecone's raw top-5 order is used as-is | *"How would reranking improve this project?"* |
| Query rewriting | Reformulating the user's raw question into a better search query before embedding it | Not present here — the raw `user_prompt` string is embedded directly | **Not implemented** | *"What if the user's question is vague — how would you improve retrieval for that?"* |
| Multi-query retrieval | Generating several variations of the question and retrieving for each | Not present here | **Not implemented** | *"How would you handle a question with multiple sub-parts?"* |
| Context compression | Summarizing/trimming retrieved chunks before sending to the LLM | Not present — raw chunk text is concatenated as-is | **Not implemented** | *"What if your 5 chunks together were too long for the model's context window?"* |

---

## Chapter 30 — RAG Evaluation

**This project currently has no automated evaluation harness of any kind** — this cannot be determined to exist anywhere in the provided code, and this handbook will not pretend otherwise.

### How you *could* evaluate it (proposed, not implemented)

- **Retrieval metrics**: Recall@5 (of the 5 retrieved chunks, did the one containing the true answer appear at all?), Precision@5 (of the 5, how many were actually relevant?), MRR (how high did the correct chunk rank, on average, across many test questions?).
- **Answer quality metrics**: faithfulness (does the generated answer only state things supported by the retrieved chunks?), correctness (does it match a known-correct answer?).
- **Operational metrics**: end-to-end latency (upload → ready, and question → first streamed token), ingestion failure rate, fallback-trigger rate (how often does the system fall back to general Gemini vs. answer from the PDF — a high fallback rate on questions that *should* be answerable from the PDF would indicate a retrieval-quality problem).

### A minimal evaluation dataset you could build for this specific project

| Question | Expected Source (PDF + page) | Expected Answer | Retrieved Chunks (log this at request time) | Correct? |
|---|---|---|---|---|
| "What is the invoice total?" | `invoice.pdf`, page 1 | "$482.10" | (fill in from `context_documents`) | Y/N |
| "What's the capital of France?" | none — should trigger fallback | (general knowledge answer) | (should be irrelevant chunks, or none) | Y/N — checking that fallback correctly triggers, not that RAG "got it right" |

You'd populate this by hand-writing ~20–30 realistic questions against a few real PDFs you control, running them through the actual `/api/prompt/text-file` endpoint, and manually grading whether the retrieved chunks and final answer were correct — there is no existing tooling in the repo to automate this today.

---

## Chapter 31 — Interview Pitches

### 31.1 — 30-Second Pitch

*"It's a RAG-based PDF Q&A web app — React frontend, Flask backend. Users upload PDFs, which get chunked and embedded with Gemini's embedding model into Pinecone. When they ask a question, I do a similarity search in Pinecone scoped to that user and conversation, feed the top matches into a Gemini prompt, and stream the answer back. If nothing relevant is found in their documents, it automatically falls back to a general Gemini answer instead of failing."*

### 31.2 — 2-Minute Pitch

*"The core problem is: reading a long PDF to find one fact is slow, so I built a chat interface on top of your own PDFs. On the frontend, it's React with Redux for auth/conversation state, and it streams the answer token-by-token as it's generated rather than waiting for the whole response.*

*On the backend, Flask handles auth by verifying JWTs against Appwrite — Appwrite's also where I store all the relational data: conversations, messages, documents, chunks, and background job records. When someone uploads a PDF, I extract text with pypdf, falling back to OCR via ocrmypdf if the PDF turns out to be scanned images rather than real text — I detect that by checking if pypdf's average characters-per-page is suspiciously low. I split the text into 800-character chunks with 80-character overlap, and instead of embedding synchronously inside the upload request — which would make people wait — I persist the chunks and a 'job' row, return immediately, and a background thread picks the job up, embeds each chunk with Gemini's embedding model, and writes the vectors into Pinecone along with metadata like user ID, conversation ID, and document ID.*

*At query time, I embed the question and search Pinecone for the 5 closest chunks, but critically filtered by that same user ID and conversation ID metadata — that's the whole security boundary preventing one user's question from ever surfacing another user's document content. Those 5 chunks get pasted into a prompt along with the question and recent history, sent to Gemini, and streamed back as newline-delimited JSON. If Gemini says the answer isn't in the context — which I detect with a simple string match — I make a second Gemini call with no document context at all, so the user still gets a useful general answer instead of a dead end."*

### 31.3 — 5-Minute Pitch (deeper technical version)

Extend the 2-minute pitch with:
- The idempotency detail: Pinecone vector IDs are SHA-256 hashes of chunk content, not random UUIDs, so re-embedding identical text overwrites rather than duplicates.
- The resilience details: exponential-backoff retry on Pinecone writes via `tenacity`, and a startup recovery routine that resets any job stuck `PROCESSING` back to `PENDING` if the app restarted mid-job.
- The retention system: after every successful ingestion, I run a per-user cleanup that keeps only the newest N completed documents, drops anything older than a max age, and caps total chunks per user — both an Appwrite and Pinecone cost control.
- Be candid about the honest limitations you'd improve (see Chapter 35) — this signals maturity, not weakness.

### 31.4 — RAG Pipeline in 1 Minute

*"Offline: PDF → text extraction (with OCR fallback for scans) → 800-character overlapping chunks → each chunk embedded with Gemini's embedding model → stored in Pinecone with metadata tying it to a specific user, conversation, and document. Online: user's question gets embedded the same way, Pinecone returns the 5 closest chunks filtered to that same user+conversation, those chunks get inserted into a prompt telling Gemini to answer only from that context, and if Gemini can't find the answer there, I fall back to a second, contextless Gemini call so the user isn't stuck with 'I don't know.'"*

---

## Chapter 32 — Interview Questions

### Beginner (25)

1. **What is RAG?** Retrieval-Augmented Generation — retrieving relevant external content and inserting it into an LLM's prompt before it generates an answer, rather than relying only on what the model memorized during training.
2. **What is an embedding?** A vector of numbers representing a piece of text's meaning, produced by a model like `gemini-embedding-001` in this project.
3. **Why use Pinecone?** It's a managed vector database purpose-built for fast nearest-neighbor similarity search with metadata filtering — this project uses it to find the chunks most relevant to a question.
4. **What is Gemini used for here?** Two things: generating embeddings (`gemini-embedding-001`) and generating the actual chat answer (the chat model configured in `ai_service.py`).
5. **Why chunk PDFs instead of embedding the whole document?** A whole document's meaning is too broad/blended for one vector to represent usefully, and it wouldn't let you pinpoint *which part* answers a specific question.
6. **What is Appwrite?** A backend-as-a-service providing authentication and a document database, used here for user accounts and every relational table (conversations, messages, documents, chunks, jobs, limits).
7. **What is a vector database?** A database optimized to store vectors and answer "which stored vectors are most similar to this query vector?"
8. **What does "chunk overlap" mean?** Repeating a small slice of text (80 characters here) between consecutive chunks so information near a chunk boundary isn't split awkwardly.
9. **What's the chunk size in this project?** 800 characters, with 80 characters of overlap.
10. **How does the app know a PDF is scanned vs. native text?** `documents.py` checks if `PyPDFLoader` extracted zero pages, or if the average characters-per-page is under 500 — either triggers OCR.
11. **What OCR tool is used?** `ocrmypdf`, invoked as a subprocess.
12. **What happens after a PDF is uploaded?** It's validated, hashed, chunked, and persisted to Appwrite along with a background "ingestion job" — then a `202 Accepted` is returned immediately, without waiting for embedding.
13. **Why return 202 instead of waiting for full processing?** Embedding a document can take noticeable time; returning immediately keeps the upload request fast and lets the frontend poll a lightweight status endpoint instead.
14. **How is a user authenticated?** Appwrite issues a JWT after login; the frontend sends it as a Bearer token on every API call; the backend verifies it by asking Appwrite's own Account API who it belongs to.
15. **What is a Pinecone metadata filter, and why does this project use one?** A condition (`user_id`, `conversation_id`) applied during the similarity search so only the current user's own, current-conversation's own vectors can ever be matched.
16. **What happens if the question isn't answerable from the PDF?** The RAG prompt instructs Gemini to say so with a specific phrase; the backend detects that phrase and falls back to a second, general Gemini call with no document context.
17. **What is streaming, in this context?** Sending the answer to the frontend incrementally, as newline-delimited JSON lines, as Gemini generates it, instead of waiting for the whole response.
18. **What is `Top-K`?** The number of nearest-neighbor results the retriever asks for — fixed at 5 in this project.
19. **Where is the raw chunk text stored — Pinecone or Appwrite?** Appwrite's `chunks` collection is the primary place it's persisted by this project's own code; Pinecone additionally carries the text alongside the vector via the LangChain integration so it can be returned at query time.
20. **What's the difference between `documents`, `chunks`, and `ingestion_jobs` collections?** `documents` = one row per uploaded PDF and its status; `chunks` = one row per text chunk of a document; `ingestion_jobs` = one row per background-processing task tied to a document.
21. **What triggers the background worker to start?** `start_worker_once(app)`, called once at Flask app startup in `app.py`.
22. **What does the daily prompt limit do?** Caps each user to 100 Gemini chat calls per day (`user_service.py::MAX_PROMPTS_PER_DAY`), resetting when the stored `lastResetDate` no longer matches today's date.
23. **How does deleting a conversation work?** It deletes the conversation's Pinecone vectors (filtered by `conversation_id`), wipes all related Appwrite rows (chunks, jobs, documents, messages), then deletes the conversation row itself.
24. **What is retention, in this project?** An automatic per-user cleanup (`retention.py`) that keeps only a limited number of the most recent completed documents, drops documents past a max age, and caps total stored chunks — freeing Pinecone/Appwrite storage over time.
25. **Is Google OAuth actually wired up right now?** Appwrite's OAuth flow exists in the code (`account.createOAuth2Session`), but you've said you're not using it currently.

### Intermediate (25)

1. **Why filter Pinecone retrieval by both `user_id` and `conversation_id` instead of just `user_id`?** So a chunk uploaded in one conversation never surfaces as context in an unrelated conversation for the same user — keeping each chat's knowledge base topically isolated, not just per-account isolated.
2. **Why does chunk overlap matter for retrieval quality?** Without it, a sentence spanning a chunk boundary could be split so that neither resulting chunk fully captures its meaning, hurting the chance either one is retrieved for a question about that content.
3. **Why run ingestion in a background worker instead of during the upload request?** To avoid a slow HTTP request/timeout risk and to let the request thread stay free for other users while embedding happens asynchronously.
4. **How does semantic retrieval actually work at the algorithm level here?** The question is embedded with the same model used for chunks; Pinecone compares that vector against stored vectors (subject to the metadata filter) and returns the `k` closest by whatever similarity metric the index is configured with.
5. **How is user data isolated across the whole pipeline?** Consistently propagating `user["$id"]` from JWT verification through every Appwrite row and into Pinecone's `user_id` metadata field, then filtering on it at query time.
6. **What happens when retrieval returns nothing relevant?** Nothing special at the vector-search level — Pinecone always returns its top-K nearest results regardless of how weak the match is; the *judgment* of relevance is left entirely to Gemini reading the prompt and (if instructed) saying the answer isn't there.
7. **Why stream the response instead of waiting for the full answer?** Lower perceived latency — the user sees progress immediately rather than staring at a blank/loading state for the full generation time.
8. **Why hash chunk text for the Pinecone vector ID instead of using a random UUID?** So re-embedding identical content is idempotent — the write overwrites the existing vector at that deterministic ID instead of creating a duplicate.
9. **How does the app decide a PDF needs OCR?** Two heuristics in `documents.py`: zero pages extracted by `PyPDFLoader`, or average extracted characters-per-page under 500.
10. **How does the app decide whether OCR's output is actually better than the original extraction?** It compares total character counts and only prefers OCR's output if it's at least 10% higher (or if the original had none at all).
11. **What's the actual mechanism that decides RAG vs. fallback?** A plain substring check for the literal phrase `"Answer is not available in the context"` in the RAG attempt's output (or, if there were zero context documents to begin with, that phrase is set directly without even calling Gemini for the RAG path).
12. **Why might that substring-check approach be fragile?** It depends on Gemini reliably reproducing that exact phrase, and doesn't distinguish "no context existed" from "Gemini genuinely judged the context insufficient" — both are treated identically.
13. **How are ingestion jobs recovered after a server restart?** `recover_stuck_jobs()` runs once at worker startup and resets any job stuck in `PROCESSING` (or the legacy `"running"` string) back to `PENDING`.
14. **What happens if a job's Pinecone write keeps failing?** `add_documents_with_retry` retries up to 5 times with exponential backoff (`tenacity`); if it still fails, the whole job is marked `FAILED` with the error message stored.
15. **What prevents duplicate re-processing of the same PDF?** A file-hash + conversation-ID check before extraction even runs — if a document with that exact combination already exists, the upload is skipped as `"skipped_duplicate"`.
16. **What's a real gap in that duplicate-detection logic?** It doesn't check the existing document's status — so a previously *failed* document can never be retried by re-uploading the same file to the same conversation.
17. **Why give the backend its own privileged Appwrite API key instead of always using the user's JWT for database calls?** Server-side orchestration (the background worker, retention, admin routes) needs to operate on data across the whole system, not just what one user's session permissions would allow.
18. **What's a security implication of that privileged API key?** It bypasses Appwrite's own document-level permission checks entirely — the application code's own filtering logic (`Query.equal(userId, ...)`) becomes the *only* thing enforcing per-user isolation for collections that don't have explicit permissions set.
19. **Why is `MAX_CHUNKS_PER_DOC` useful?** It caps how many embedding calls and Pinecone writes a single upload can trigger — a cost and abuse control, not a correctness requirement.
20. **What does the retention system's `CHUNK_CAP_PER_USER` protect against?** Unbounded Pinecone/storage growth from one user repeatedly uploading many/large documents over time.
21. **How is a returning user's daily prompt count reset?** By comparing the stored `lastResetDate` string against today's date (`date.today().isoformat()`) — if they differ, the count resets to reflect the new day's first request.
22. **What's the actual behavior difference between `apiClient.js::apiFetch` and the raw `fetch()` calls in `convoSlice.js`?** `apiFetch` fetches a fresh JWT on every call; the `convoSlice.js` thunks reuse whatever JWT is already sitting in Redux state, which could theoretically be stale.
23. **How does the frontend know when background ingestion has finished?** By polling `GET /api/documents/status?conversationId=...` every 2 seconds until `ready: true` or `failed > 0`.
24. **Why is there no Appwrite-level permission set on the `documents`/`chunks`/`ingestion_jobs` collections?** Those are only ever read/written via the backend's privileged API-key client (never directly from user-scoped sessions), so the code never had to pass a `permissions` array for them — unlike `conversations`/`messages`/`user_limits`, which do set one.
25. **What is the practical effect of `prioritize_chunks` given that oversized uploads are rejected outright rather than truncated?** It reorders chunks before persistence, but since no truncation currently happens on the stored side, it currently has no measurable effect on what gets retrieved — its main potential value would be if future code truncated to the top-N prioritized chunks instead of rejecting the whole upload.

### Advanced (25)

1. **How would you prevent cross-user retrieval, beyond what's here today?** Add a second, independent layer — e.g., Pinecone namespaces per user (rather than relying solely on a metadata filter that a code bug could accidentally omit), plus automated tests that assert a query filtered for User A never returns a vector whose metadata says User B.
2. **How would you improve retrieval quality?** Add a similarity-score threshold (don't blindly return top-5 regardless of how weak the match is), consider hybrid search (combine keyword + vector), and consider reranking the top candidates with a more expensive but more accurate model before building the final prompt.
3. **How would you handle prompt injection from malicious PDF content?** Explicitly instruct the model (in the system portion of the prompt) to treat retrieved context strictly as data, never as instructions, and consider a lightweight scan/sanitization pass over extracted text before it's ever embedded or shown to the LLM.
4. **How would you scale ingestion?** Replace the single in-process polling thread with a real task queue (Celery/RQ + Redis, or a managed cloud queue) supporting multiple independent workers and atomic job claiming, so scaling out Flask instances also scales out ingestion throughput.
5. **How would you evaluate retrieval quality systematically?** Build a small labeled question/answer/expected-source dataset (Chapter 30) and compute Recall@K/Precision@K/MRR against it after any pipeline change, rather than eyeballing individual chats.
6. **How would you reduce hallucinations in the fallback path?** The fallback intentionally has no grounding — you'd want to clearly and durably label it as ungrounded in the UI (which the current text prefix already partially does), and potentially add a disclaimer that persists visually, not just as inline text that could scroll out of view.
7. **How would you handle a change of embedding model in production?** Because vector dimensionality and semantic space differ between models, you'd need to fully re-embed and re-index every existing chunk — you can't mix vectors from two different embedding models in one Pinecone index and expect meaningful similarity comparisons; `admin_rebuild()`'s `dropVectors` + re-queue-all-completed-docs pattern is actually a reasonable starting point for exactly this kind of migration.
8. **How would you migrate to a new Pinecone index without downtime?** Write to both the old and new index during a transition window (dual-write), backfill the new index from Appwrite's `chunks` collection (which still has the raw text, unlike Pinecone-only data), then cut over reads once backfill is verified complete.
9. **How would you implement hybrid search here?** Add a traditional keyword/full-text index (Appwrite doesn't offer this natively for search-quality full-text; you'd likely need a separate search engine or Postgres `tsvector`), retrieve candidates from both keyword and vector search, then merge/deduplicate/rerank before building the prompt.
10. **How would you introduce reranking?** After Pinecone's initial top-K (perhaps widened to top-20), run a smaller, cheap cross-encoder or an LLM-based relevance-scoring pass over each candidate against the question, then keep only the best 5 for the final prompt — trading a bit of latency for materially better precision.
11. **What happens today if two ingestion workers (e.g., two Flask processes) both poll the same `ingestion_jobs` table?** They could both `list_pending_jobs`, both see the same `PENDING` job, and both attempt to process it — there's no atomic "claim" step (no `update_document` with an optimistic lock/conditional check) preventing double-processing; this is a real concurrency gap if this app were ever run with more than one worker-enabled process.
12. **How would you fix that concurrency gap?** Add an atomic claim: e.g., a conditional update ("set status=PROCESSING WHERE status=PENDING") that only one process can win, or move to a queue system that inherently provides single-delivery semantics.
13. **How would you defend against a malicious file disguised as a PDF?** Validate actual file content (magic bytes / MIME sniffing), not just the filename extension, before attempting extraction.
14. **How would you cap resource usage from a pathological OCR job?** Add an explicit `timeout=` to the `subprocess.run` call in `_run_ocrmypdf`, and treat a timeout as an OCR failure that falls back to whatever native extraction produced (if any).
15. **How would you make the RAG/fallback decision less fragile than a substring match?** Have the model return a structured response (e.g., JSON with an explicit `"grounded": true/false` field, or a numeric confidence) instead of relying on it reproducing an exact English sentence, and treat retrieval's own similarity scores as an additional independent signal.
16. **How would you rate-limit per user instead of per IP?** Key `Flask-Limiter` (or a replacement) off the authenticated user ID extracted after `auth_required` runs, rather than the remote IP address, and consider different limits for cheap endpoints (`/api/health`) vs. expensive ones (`/api/prompt/text-file`).
17. **How would you distinguish a genuinely-failed ingestion from a retention-pruned document, given both currently write the same `"failed"` status?** Add a distinct status value (e.g., `"pruned"`) instead of overloading `DOC_FAILED` for two semantically different situations.
18. **How would you add automatic retry for permanently-failed ingestion jobs?** Add a scheduled sweep (or extend the worker loop) that finds `FAILED` jobs under some max-attempts threshold and re-queues them as `PENDING`, distinct from today's manual-only `/api/admin/rebuild-index` path (which also only targets `COMPLETED` docs, not `FAILED` ones).
19. **How would you handle the context-window risk if you ever increased `k` significantly?** Track/estimate token count of the assembled context before sending it to Gemini, and either drop the lowest-ranked chunks or summarize/compress them if the total would exceed the model's context limit.
20. **How would you add observability to this pipeline?** Structured logging with correlation IDs (tie a single question's log lines together), metrics on ingestion latency/failure rate and fallback-trigger rate, and alerting on the ingestion worker's error rate.
21. **How would you test cross-user isolation automatically?** An integration test that ingests a document as User A, then asserts that a retrieval attempt authenticated as User B (even with a semantically similar question) never returns any of User A's chunks — asserting on the Pinecone metadata filter's actual effect, not just trusting the code.
22. **What would you change about how `chunk_hash` doubles as the Pinecone vector ID if two genuinely different documents happened to share an identical paragraph?** Today, the second document's identical chunk would simply overwrite the first's vector at that ID — functionally harmless for retrieval (the content is identical, so the vector is correct either way), but you'd want to confirm that Pinecone's stored metadata after such an overwrite reflects whichever document's data was written *last*, which could subtly affect metadata-based deletion (deleting one document could inadvertently remove a shared vector another document still "needs" if you ever changed the ID scheme to be less content-derived).
23. **How would you handle Gemini API outages gracefully at scale?** Add a circuit breaker / fallback provider strategy, and clearly surface a "service temporarily degraded" state to users rather than a generic error string, especially for the embedding step (which blocks ingestion entirely) vs. the chat step (which only blocks that one message).
24. **How would you version your Pinecone metadata schema for future changes?** Add a schema-version field to each vector's metadata so a future migration script can identify and selectively re-process only older-schema vectors.
25. **What's the single most impactful next investment for retrieval quality specifically, given the current implementation?** Almost certainly a similarity-score threshold combined with reranking — today, weak matches are never filtered out, so borderline-irrelevant chunks can and do reach the prompt exactly as confidently as strong matches.

---

## Chapter 33 — Interview Follow-Up Chains

**Interviewer: Why did you use Pinecone?**
→ *Managed vector similarity search with metadata filtering, so I don't have to run and scale my own vector index.*
**Follow-up: Why not just store vectors in Postgres with pgvector?**
→ *That's a completely reasonable alternative, especially since I'm already storing everything else in a document database. I chose Pinecone for a purpose-built, managed retrieval experience and to keep vector search fully decoupled from the relational data — but pgvector would remove one external dependency, at the cost of managing indexing/scaling myself.*
**Follow-up: What happens if Pinecone goes down?**
→ *Looking at the code, retrieval failures are caught (`try/except` around the retriever call in `routes.py`) and the system falls back to a general Gemini answer rather than erroring out — so chat keeps working in a degraded mode. Ingestion, though, would fail outright, since `add_documents_with_retry` would exhaust its retries and mark the job/document `FAILED`.*
**Follow-up: How do you prevent one user from retrieving another user's documents?**
→ *A Pinecone metadata filter on `user_id` and `conversation_id`, applied on every retrieval call — see Chapter 8/22 for exactly why this is the entire security boundary and what would happen if it were ever accidentally omitted.*

**Interviewer: Why did you use Gemini specifically?**
→ *One vendor for both embeddings and chat generation simplifies credentials and billing.*
**Follow-up: Why not OpenAI?**
→ *No specific technical blocker — it would just mean managing two providers' credentials/rate limits instead of one, or swapping providers entirely. The LangChain abstraction (`ChatGoogleGenerativeAI` / `GoogleGenerativeAIEmbeddings`) would make swapping providers a relatively contained change.*
**Follow-up: What would you need to change if you switched embedding providers?**
→ *A full re-embed of every existing chunk — different embedding models produce vectors in different, incompatible semantic spaces, so you cannot mix vectors from two providers in one index meaningfully.*

**Interviewer: Why Appwrite instead of a custom database?**
→ *It bundles authentication and a document database together, so I didn't have to build auth from scratch.*
**Follow-up: What if you needed complex relational queries later?**
→ *Appwrite's query capabilities (as used here — `Query.equal`, `order_asc/desc`, `limit`) are simpler than full SQL joins; a genuinely relationally-complex future feature might push me toward Postgres, potentially alongside Appwrite for auth only.*

**Interviewer: How do you know the answer came from the PDF and not Gemini's general knowledge?**
→ *The RAG prompt only ever includes retrieved chunk text as context and explicitly instructs the model to say so if the answer isn't there; if that phrase appears, the backend discards that attempt and clearly labels the next attempt as a "general Gemini response" in the text the user sees. Beyond that instruction and that label, I don't have an automated way to verify the final answer's factual grounding against the chunks — that's an honest limitation, not something I'd claim is airtight.*

**Interviewer: What was the hardest bug you encountered?**
→ *(Tailored to your own experience — but the codebase itself illustrates a great example: an identity-consistency issue where a Users-collection document ID was being resolved via `get_or_create_user_document_id` but the rest of the pipeline had moved to using the raw Appwrite Auth ID directly. If those two had ever been mixed inconsistently between where a document was written and where retrieval filtered, ingestion would silently succeed while retrieval silently returned nothing — a classic "everything looks fine except the one thing that matters" RAG bug.)*

---

## Chapter 34 — "Did You Really Build This?" Questions

| Question | Answer, from actual code |
|---|---|
| Where exactly are embeddings generated? | Inside `PineconeVectorStore.add_documents(...)` (ingestion) and `.as_retriever(...).invoke(...)` (query) — both hidden inside the LangChain vector-store abstraction in `vector_store.py`, not called out as a separate explicit embedding step in your own code. |
| What embedding model are you using? | `models/gemini-embedding-001`. |
| What is the chunk size? | 800 characters. |
| What is the chunk overlap? | 80 characters. |
| Where are vectors stored? | Pinecone, index name from `PINECONE_INDEX_NAME`, no namespace set. |
| What metadata is stored with each vector? | `chunk_hash`, `file_hash`, `document_id`, `conversation_id`, `user_id`. |
| How do you identify the current user? | `user["$id"]`, obtained from Appwrite's `Account.get()` after verifying the request's JWT in `auth.py::auth_required`. |
| How is the conversation ID generated? | It's just Appwrite's own auto-generated document `$id` for the `conversations` collection row created in `user_service.py::create_conversation` — no separate ID scheme. |
| What happens after upload returns HTTP 202? | The HTTP request is over; a background worker thread later picks up the created `ingestion_jobs` row on its next 5-second poll cycle and does the actual embedding/Pinecone-write work. |
| Why is ingestion asynchronous? | To keep the upload request fast and avoid tying up a Flask worker thread for the duration of embedding a whole document. |
| How does the worker know what to process? | It queries the `ingestion_jobs` collection for rows with `status == "pending"`, oldest first, limited to 3 per loop iteration. |
| How does the application know whether documents are ready? | `GET /api/documents/status` aggregates the `status` field across all `documents` rows for a conversation; `ready` is only true if none are pending/processing/failed. |
| What happens when retrieval returns zero documents? | The RAG attempt is skipped entirely (the sentinel "not available" text is set directly), and the system proceeds straight to the general Gemini fallback. |
| Where is the RAG prompt constructed? | `server/api/ai_service.py::generate_rag_response`, a plain Python f-string template. |
| How does the frontend receive a streamed answer? | Via `response.body.getReader()` on the raw `fetch()` response in `hooks/useChat.js`, decoding and JSON-parsing each newline-delimited chunk as it arrives. |

---

## Chapter 35 — Improvements You Can Mention

### Easy
- Fix the invalid `gemini-3.5-flash-lite` model string (Ch. 26, Bug 1).
- Fix the `Role.user(sender_type)` permission bug in `save_message_to_db` (Ch. 26, Bug 2).
- Skip only-truly-completed documents in the upload duplicate check, so failed ingestion can actually be retried (Ch. 26, Bug 4).
  - *Current limitation*: a permanently-failed upload can never succeed via retry in the same conversation.
  - *Proposed solution*: check `status == DOC_COMPLETED` before treating a matching `fileHash` as a skip-worthy duplicate.
  - *Benefit*: users aren't silently stuck.
  - *Trade-off*: none meaningful — this is close to a pure bug fix.

### Medium
- Add a similarity-score threshold to retrieval instead of unconditional top-5.
  - *Current limitation*: even completely irrelevant chunks get passed into the prompt if they happen to be the "least-bad" of what's available.
  - *Proposed solution*: discard candidates below a tuned similarity cutoff before building the context.
  - *Benefit*: fewer weakly-relevant, confusing context injections.
  - *Trade-off*: risk of over-filtering genuinely-relevant borderline matches; needs tuning per use case.
- Move background ingestion to a real task queue (Celery/RQ + Redis).
  - *Current limitation*: single in-process thread; no safe multi-process scaling; no automatic re-queue of failed jobs.
  - *Proposed solution*: a proper worker pool with atomic job claiming.
  - *Benefit*: real horizontal scalability and safer concurrency.
  - *Trade-off*: added infrastructure (Redis or similar) and operational complexity.
- Content-based file-type validation instead of extension-only.
  - *Benefit*: closes a minor but real gap in upload validation.
  - *Trade-off*: negligible.

### Advanced
- Reranking pass after initial retrieval.
  - *Current limitation*: raw vector-similarity order is used as-is for the final prompt.
  - *Proposed solution*: retrieve a wider candidate set (e.g., top-20), rerank with a cheaper cross-encoder or LLM scoring call, keep the best 5.
  - *Benefit*: materially better precision on ambiguous questions.
  - *Trade-off*: added latency and cost per query.
- Per-user rate limiting instead of per-IP.
  - *Benefit*: fairer resource allocation; harder to starve one legitimate user via another's traffic on a shared IP.
  - *Trade-off*: requires the limiter to run *after* auth resolves the user, slightly restructuring middleware order.
- Prompt-injection defense for retrieved PDF content.
  - *Benefit*: closes a real, currently-unaddressed security gap (Ch. 22).
  - *Trade-off*: added prompt complexity; imperfect — no known defense is airtight.

### Production-Scale
- Namespace-per-tenant (or per-user) Pinecone strategy as a second, independent layer of isolation beyond the metadata filter alone.
- Full observability stack: structured/correlated logging, metrics dashboards, alerting on ingestion failure rate and fallback-trigger rate.
- Automated evaluation harness (Chapter 30) run in CI on every pipeline-affecting change, to catch retrieval-quality regressions before they reach users.
- CI/CD with automated tests (Chapter 36) gating deploys.

---

## Chapter 36 — Testing Strategy

**No automated tests for this behavior were found in the provided project.** There is no test directory, test framework configuration, or test file visible in either `client/` or `server/`.

### Proposed unit tests
- `documents.py::split_documents` — given a known input string, assert chunk count and boundaries match expectations for `chunk_size=800, chunk_overlap=80`.
- `documents.py::build_chunk_records` — assert the output dict shape exactly matches the Appwrite `chunks` schema.
- `auth.py::auth_required` — with a mocked Appwrite `Account.get()`, assert a valid token injects `user` and an invalid one returns 401.
- `user_service.py::get_user_prompt_limit` — assert the reset-by-date logic correctly resets `promptCount` when `lastResetDate` differs from today.

### Proposed integration tests
- Upload a real small PDF against a test Appwrite project/Pinecone index; assert `documents`/`chunks`/`ingestion_jobs` rows are created correctly.
- Let the ingestion worker run against that test setup; assert the document eventually reaches `status: "completed"` and that vectors with the expected metadata exist in Pinecone.
- Call `/api/prompt/text-file` against a completed test document; assert a RAG-path answer is produced (not a fallback) for a question whose answer is genuinely in the test PDF.
- Gemini generation itself: mock `ChatGoogleGenerativeAI.stream` to avoid real API calls/costs in CI.

### Proposed end-to-end tests
```text
Upload PDF
 ↓
Wait for ingestion (poll /api/documents/status until ready)
 ↓
Ask a question with a known answer in the PDF
 ↓
Assert the response does NOT contain the fallback prefix
 ↓
Assert the answer text contains the expected fact
```
```text
Ask a question unrelated to any uploaded PDF
 ↓
Assert the response DOES contain "I couldn't find relevant information in your uploaded documents."
```

---

## Chapter 37 — Deployment & Production Readiness

### Current setup (from what's actually in the repo)
- **Local dev**: `python app.py` (Flask's built-in dev server, `debug=True`) for the backend; `npm run dev` (Vite dev server) for the frontend.
- **Production container**: `Dockerfile` builds a Python 3.11-slim image, installs `tesseract-ocr`, `ocrmypdf`, `ghostscript`, `poppler-utils` at the OS level, installs Python deps via `uv`, and runs `gunicorn --worker-class gthread --workers 1 --threads 4 --bind 0.0.0.0:$PORT --timeout 120 app:app`.
- **CORS**: locked to a single `FRONTEND_URL` origin — this must be updated to the real deployed frontend URL in production, not left as `http://localhost:5173`.
- **Database/Vector DB/LLM**: all three (Appwrite, Pinecone, Gemini) are already external managed services reachable over the internet — no local-only dependency there.
- **Secret management**: currently just `.env` files loaded via `python-dotenv` locally; production would need these injected as real environment variables by whatever hosting platform is used (this cannot be determined from the code — no hosting platform config like a `render.yaml`, `fly.toml`, or CI/CD pipeline file is present in what was provided).

### What must change before real production use
- **Single gunicorn worker, 1 process**: the Dockerfile's `--workers 1` means exactly one process — combined with the single in-process ingestion-worker thread, this is fine for low traffic but is a real ceiling; scaling workers naively (without fixing job-claiming, per Ch. 23) risks duplicate job processing.
- **`debug=True`** in `app.py`'s `if __name__ == '__main__'` block must never run in production (it's not what the Dockerfile's `gunicorn` command uses, so this is already correctly avoided in the container path — but worth being explicit that the `python app.py` local-dev entrypoint and the Docker/gunicorn entrypoint are two genuinely different run modes).
- **CORS origin, all collection IDs, and every secret** must be re-pointed to production values — none of this is automated; it's manual `.env` configuration per environment today.
- **No CI/CD pipeline** is present in the provided project — this cannot be confirmed to exist.
- **No monitoring/alerting** is present — failures are currently only visible via application logs.

---

## Chapter 38 — "If I Had to Rebuild It"

```text
1.  Set up the Flask backend skeleton + Appwrite project (Auth + Database + all 7 collections)
2.  Implement JWT-verifying auth middleware (auth.py) before anything else — every other route depends on it
3.  Implement conversation creation/listing (the simplest "real" data flow, no AI yet)
4.  Implement PDF upload with validation + hashing (no chunking/embedding yet — just persist the raw upload metadata)
5.  Implement text extraction + OCR fallback (documents.py) — verify extraction quality before building anything downstream of it
6.  Implement chunking (RecursiveCharacterTextSplitter) — verify chunk boundaries look sensible on real PDFs
7.  Set up Pinecone (index, dimension matching the chosen embedding model) + the embedding client — verify a single manual embed+query round-trip works before automating it
8.  Implement the ingestion worker (synchronous first, to prove correctness; make it a background thread only after the synchronous version works end-to-end)
9.  Implement retrieval with the user_id/conversation_id metadata filter FIRST — build the security boundary in from day one, not as an afterthought
10. Implement the RAG prompt + Gemini generation call (non-streaming first)
11. Implement the fallback-detection logic and the second Gemini call
12. Add streaming (NDJSON) once the non-streaming version is proven correct
13. Build the frontend chat UI against the now-stable API
14. Add conversation history, delete, and document-status polling to the frontend
15. Add the daily prompt limit and retention/cleanup system
16. Add rate limiting, input validation hardening, and admin routes
17. Add automated tests (unit → integration → end-to-end) — retroactively, ideally continuously from step 3 onward instead
18. Add observability/logging, then deploy
```

**Why this order**: authentication and data isolation (steps 2 and 9) are the two things every other feature depends on and the two things hardest to safely retrofit later — building them first, deliberately, avoids exactly the kind of identity-consistency bug class described in Chapter 26. Extraction/chunking/embedding are verified in isolation (steps 5–7) *before* wiring them into an asynchronous worker, so any correctness problems are caught in the simplest possible setting, not buried inside background-thread debugging.

---

## Chapter 39 — Cheat Sheet

**Architecture**: React/Vite frontend → Flask backend → Appwrite (Auth + DB) + Pinecone (vectors) + Gemini (embeddings + LLM). Background worker thread does async ingestion.

**Important files**: `app.py` (wiring), `routes.py` (every endpoint), `documents.py` (extraction/chunking), `vector_store.py` (Pinecone + embeddings), `ai_service.py` (prompts + generation), `ingestion_worker.py` (background processing), `retention.py` (cleanup), `auth.py` (JWT verification), `appwrite_utils.py` (status constants + `rel_id`).

**Important functions**: `load_documents`, `split_documents`, `calculate_chunk_ids`, `build_chunk_records`, `get_vector_store`, `add_documents_with_retry`, `generate_rag_response`, `generate_fallback_response`, `process_job`, `enforce_retention_for_user`, `auth_required`.

**Important env vars**: `GOOGLE_API_KEY`, `PINECONE_API_KEY`, `PINECONE_INDEX_NAME`, `VITE_APPWRITE_API_KEY`, `VITE_APPWRITE_DATABASE_ID` + 7 collection ID vars, `FRONTEND_URL`, `ADMIN_TOKEN`, `MAX_FILE_BYTES`, `MAX_CHUNKS_PER_DOC`, `DOCS_KEEP_PER_USER`, `DOCS_MAX_AGE_DAYS`, `CHUNK_CAP_PER_USER`.

**API endpoints**: `/api/health`, `/api/ping`, `/api/user/prompt-limit`, `/api/conversations` (GET), `/api/conversations/<id>` (GET/DELETE), `/api/documents/upload` (POST), `/api/documents/status` (GET), `/api/prompt/text-file` (POST, streaming), `/api/admin/prune` (POST), `/api/admin/rebuild-index` (POST).

**Appwrite collections**: `users`, `conversations`, `messages`, `documents`, `chunks`, `ingestion_jobs`, `user_limits`.

**Pinecone metadata fields**: `user_id`, `conversation_id`, `document_id`, `file_hash`, `chunk_hash`.

**RAG pipeline**: PDF → extract (+ OCR if scanned) → chunk (800/80) → embed (`gemini-embedding-001`) → store in Pinecone → [query] → embed question → retrieve top-5 (filtered by user_id + conversation_id) → build prompt → Gemini → stream answer → (fallback to general Gemini if "Answer is not available in the context" detected).

**Authentication flow**: Appwrite session/JWT (frontend) → Bearer token → `auth.py::auth_required` verifies against Appwrite `Account.get()` → `user["$id"]` propagated everywhere downstream.

**Important numbers**:

```text
Chunk size:            800 characters
Chunk overlap:         80 characters
Top-K (retrieval):     5
Max file size:         5 MB (5,242,880 bytes)
Max chunks per doc:    70
Max files per upload:  2 (frontend-only enforcement)
Ingestion batch size:  16 chunks per Pinecone write
Embedding dimension:   3072 (gemini-embedding-001, fixed)
Daily prompt limit:    100 (hardcoded, not env-configurable)
Retention (your .env): 3 docs kept, 10-day max age, 800-chunk cap per user
Retry policy:          5 attempts, exponential backoff (min 5s, max 60s, multiplier 2)
Worker poll interval:  5 seconds when idle, up to 3 jobs per cycle
Global rate limit:     60 requests/minute per IP
```

---

## Chapter 40 — Final Interview Cheat Sheet

### 10 Things You Must Remember
1. Retrieval is filtered by `user_id` AND `conversation_id` in Pinecone metadata — that's the whole isolation boundary.
2. Chunking is `RecursiveCharacterTextSplitter(800, 80)`.
3. Embeddings: `gemini-embedding-001`, 3072 dimensions, fixed regardless of requested dimensionality.
4. Pinecone vector ID = SHA-256 hash of the chunk's own text (not a random UUID) — enables idempotent re-embedding.
5. Ingestion is asynchronous: upload returns `202` immediately; a single background thread does the actual embedding.
6. The RAG-vs-fallback decision is a plain substring check for `"Answer is not available in the context"`.
7. Retrieval always returns its top-5 nearest vectors — there's no similarity-score cutoff; relevance judgment is delegated to the LLM.
8. Backend auth never issues its own tokens — it only verifies Appwrite-issued JWTs live against Appwrite's own Account API.
9. The backend's own Appwrite client uses a privileged API key that bypasses per-document permissions — app-level query filters are the real access control for most collections.
10. Retention (`retention.py`) automatically prunes old/excess documents per user after every successful ingestion.

### 10 Technical Terms You Must Know
RAG, embedding, vector database, semantic/similarity search, chunking, chunk overlap, Top-K, metadata filtering, context window, grounding/hallucination.

### 10 Architecture Points
React/Vite frontend • Flask backend • Appwrite for Auth+DB • Pinecone for vectors • Gemini for embeddings+LLM • single background worker thread • NDJSON streaming • JWT-based auth verified live against Appwrite • per-user retention • daily prompt quota.

### 10 Likely Interview Questions
"Walk me through your RAG pipeline." • "Why Pinecone?" • "How do you isolate user data?" • "What happens if retrieval finds nothing?" • "Why is ingestion asynchronous?" • "How does streaming work end to end?" • "What's your biggest security gap right now?" • "How would you scale ingestion?" • "What's a bug you found and fixed?" • "How would you evaluate answer quality?"

### 10 Strong Improvements You Can Mention
Similarity-score threshold on retrieval • reranking • real task-queue ingestion with atomic job claiming • per-user rate limiting • prompt-injection defenses • content-based (not extension-based) file validation • automated evaluation harness • CI with unit/integration/e2e tests • fixing the retry-a-failed-upload gap • distinguishing "pruned" from "failed" document status.

### 5 Things Interviewers May Challenge You On
1. "Your fallback detection is a string match — isn't that fragile?" → Yes, honestly; explain the alternative (structured model output / confidence score).
2. "What stops two background workers from double-processing the same job if you scaled to multiple processes?" → Nothing today — explain the atomic-claim fix.
3. "How do you know your RAG answers are actually grounded, not hallucinated?" → Only via the prompt instruction; no automated grounding check exists.
4. "Why does `save_message_to_db` grant permissions to `Role.user(sender_type)`?" → It's a bug (Chapter 26); explain why it's currently harmless (privileged API-key client bypasses it) but should be `Role.user(user_id)`.
5. "What happens to a document that fails ingestion — can the user ever recover?" → Not by re-uploading the same file to the same conversation today; explain the fix.

### 30-Second Pitch
See Chapter 31.1.

### 2-Minute Pitch
See Chapter 31.2.

### RAG Explanation (1 minute)
See Chapter 31.4.

### Complete Architecture Diagram
```text
                         ┌──────────────────┐
                         │      User        │
                         └────────┬──────────┘
                                  ▼
                     ┌────────────────────────┐
                     │  React + Vite Frontend  │
                     └────────────┬────────────┘
                                  │ Bearer JWT
                                  ▼
                     ┌────────────────────────┐
                     │     Flask Backend       │
                     └───────┬───────┬─────────┘
                              │       │
                 ┌────────────┘       └─────────────┐
                 ▼                                   ▼
        ┌──────────────────┐                ┌──────────────────────┐
        │    Appwrite       │                │  Background Worker    │
        │ (Auth + Database) │                │ (ingestion_worker.py) │
        └──────────────────┘                └───────────┬───────────┘
                                                          │
                                    ┌─────────────────────┼─────────────────────┐
                                    ▼                                           ▼
                          ┌──────────────────┐                     ┌──────────────────┐
                          │ Gemini Embeddings │                     │     Pinecone       │
                          └──────────────────┘                     └──────────────────┘
                                                          ▲
                  Query time: Flask → Pinecone (filtered search) → Flask → Gemini LLM → stream → Frontend
```

