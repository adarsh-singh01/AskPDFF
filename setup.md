# RAG Project — Full Local Setup Guide (Windows)

I inspected every file in `server/` and `client/src/`, not just the README. Two important **code-vs-README discrepancies** found first:

> ⚠️ **Discrepancy 1 — Vector DB**: README says ChromaDB. The actual code (`server/api/vector_store.py`) uses **Pinecone** (`langchain_pinecone`, `pinecone`). ChromaDB is not used anywhere in the code. The `Dockerfile`'s `CHROMA_PATH` env var is leftover/unused.
>
> ⚠️ **Discrepancy 2 — Google OAuth Client ID**: `VITE_GOOGLE_OAUTH_CLIENT_ID` is defined in `client/.env.sample` and read in `conf.js`, but **never actually used anywhere in the code**. Google login is done entirely through Appwrite's `account.createOAuth2Session(OAuthProvider.Google, ...)`, which means the real Google Client ID/Secret must be configured **inside the Appwrite Console**, not in `.env`. The `.env` variable appears to be dead/unused.
>
> ⚠️ **Discrepancy 3**: `server/requirements.txt` does **not** list `ocrmypdf` or `tenacity`, even though `documents.py` calls `ocrmypdf` as a subprocess and `vector_store.py` imports `tenacity`. `tenacity` installs transitively (as a dependency of `langchain-core`), but `ocrmypdf` does not — it must be installed separately.

---

## 1. Software to install first

- **Python 3.10 or 3.11** (Dockerfile uses 3.11; README says 3.9+ — either works)
- **Node.js 18+** and npm
- **Git** (optional, you already have the code)
- **Ghostscript**, **Tesseract OCR**, **Poppler** — required by `ocrmypdf` (confirmed by the project's own `Dockerfile`, which installs `tesseract-ocr ghostscript poppler-utils` alongside `ocrmypdf`)

---

## 2. Backend setup (Flask)

```bash
cd server
python -m venv venv
.\venv\Scripts\activate
pip install -r requirements.txt
pip install ocrmypdf
```

**OCR system tools (Windows) — simplest path:**
- Ghostscript: install from ghostscript.com (Windows installer), keep default install (adds itself to PATH)
- Tesseract OCR: install the "UB-Mannheim" Windows build, and make sure its install folder is added to PATH
- Poppler: download "Poppler for Windows" release, unzip, add its `bin` folder to PATH

(Don't troubleshoot further now — only revisit this if you actually get an OCR error.)

---

## 3. Frontend setup (React + Vite)

```bash
cd client
npm install
```

---

## 4 & 5. `.env` files and where each value comes from

### `server/.env`
```env
GOOGLE_API_KEY=YOUR_GEMINI_API_KEY
VITE_APPWRITE_ENDPOINT=https://cloud.appwrite.io/v1
VITE_APPWRITE_PROJECT_ID=YOUR_APPWRITE_PROJECT_ID
VITE_APPWRITE_API_KEY=YOUR_APPWRITE_API_KEY
VITE_APPWRITE_DATABASE_ID=YOUR_APPWRITE_DATABASE_ID
VITE_APPWRITE_CONVERSATIONS_COLL_ID=YOUR_CONVERSATIONS_COLLECTION_ID
VITE_APPWRITE_MESSAGES_COLL_ID=YOUR_MESSAGES_COLLECTION_ID
VITE_APPWRITE_USER_LIMITS_COLL_ID=YOUR_USER_LIMITS_COLLECTION_ID
VITE_APPWRITE_USERS_COLL_ID=YOUR_USERS_COLLECTION_ID
VITE_APPWRITE_DOCUMENTS_COLL_ID=YOUR_DOCUMENTS_COLLECTION_ID
VITE_APPWRITE_CHUNKS_COLL_ID=YOUR_CHUNKS_COLLECTION_ID
VITE_APPWRITE_INGESTION_JOBS_COLL_ID=YOUR_INGESTION_JOBS_COLLECTION_ID
PINECONE_API_KEY=YOUR_PINECONE_API_KEY
PINECONE_INDEX_NAME=YOUR_PINECONE_INDEX_NAME
FRONTEND_URL=http://localhost:5173
MAX_FILE_BYTES=5242880
MAX_CHUNKS_PER_DOC=70
DOCS_KEEP_PER_USER=3
DOCS_MAX_AGE_DAYS=10
CHUNK_CAP_PER_USER=800
ADMIN_TOKEN=any_random_string_you_choose
```
Where values come from:
- `GOOGLE_API_KEY`: Google AI Studio → Get API Key
- `VITE_APPWRITE_*` (endpoint/project/database/collections): Appwrite Console (see section 6)
- `VITE_APPWRITE_API_KEY`: Appwrite Console → your project → **Overview → Integrations → API Keys** → create a key with `databases.read`/`databases.write` scopes (server key, keep secret)
- `PINECONE_API_KEY` / `PINECONE_INDEX_NAME`: Pinecone Console (section 8)
- `FRONTEND_URL`: must exactly match where Vite runs — `http://localhost:5173`
- `MAX_FILE_BYTES`, `MAX_CHUNKS_PER_DOC`, `DOCS_KEEP_PER_USER`, `DOCS_MAX_AGE_DAYS`, `CHUNK_CAP_PER_USER`: optional, code has defaults (5MB, 70, 3, 10, 800) if left blank — values above are just the defaults spelled out
- `ADMIN_TOKEN`: you invent this yourself; only used for the `/api/admin/*` routes (optional for normal use)

### `client/.env`
```env
VITE_GOOGLE_OAUTH_CLIENT_ID=NEEDS_MANUAL_CONFIGURATION
VITE_APPWRITE_PROJECT_ID=YOUR_APPWRITE_PROJECT_ID
VITE_APPWRITE_DATABASE_ID=YOUR_APPWRITE_DATABASE_ID
VITE_APPWRITE_COLLECTION_ID=NOT_ACTUALLY_USED
VITE_APPWRITE_USERS_COLL_ID=YOUR_USERS_COLLECTION_ID
VITE_APPWRITE_USER_LIMITS_COLL_ID=YOUR_USER_LIMITS_COLLECTION_ID
VITE_EMAIL_ADDRESS=your-contact-email@example.com
VITE_APPWRITE_ENDPOINT=https://cloud.appwrite.io/v1
VITE_BACKEND_URL=http://127.0.0.1:5000
VITE_BASE_URL=http://localhost:5173
```
- `VITE_GOOGLE_OAUTH_CLIENT_ID`: **NEEDS MANUAL CONFIGURATION** — as noted, this variable is unused by the code. Google login actually needs its Client ID/Secret set inside Appwrite Console (section 6), not here. You can leave this blank/placeholder — it won't break anything.
- `VITE_APPWRITE_COLLECTION_ID`: read into `conf.js` but never referenced anywhere else in the code — you can put any placeholder, it has no effect.
- `VITE_EMAIL_ADDRESS`: just a support email shown on the Privacy Policy page — any address you want.
- `VITE_BACKEND_URL` / `VITE_BASE_URL`: must match the actual ports the two servers run on (5000 and 5173).

---

## 6. Appwrite setup

1. Go to Appwrite Cloud (cloud.appwrite.io) → create account → **Create Project**. Copy the **Project ID** and the endpoint (`https://cloud.appwrite.io/v1`).
2. **Authentication**: Console → Auth → Settings → enable **Email/Password**.
3. **Google OAuth**: Console → Auth → Settings → OAuth2 Providers → enable **Google** → paste a Google Cloud OAuth Client ID + Secret (create these in Google Cloud Console → Credentials → OAuth Client ID → Web application → Authorized redirect URI = the callback URL Appwrite shows you on that same screen).
4. **API Key**: Console → Overview → Integrations → API Keys → create key → scopes: `databases.read`, `databases.write` (or "Databases" full access) → copy the key.
5. **Database**: Console → Databases → Create Database → copy the **Database ID**.
6. **Create these 7 collections**, exactly as the code expects (attribute names are case-sensitive):

| Collection | Attributes (type, notes) |
|---|---|
| **conversations** | `title` (string, 50), `userId` (string, 50 — Appwrite **Auth** user id), `lastMessageAt` (string/datetime, 50) |
| **messages** | `conversationId` (string, 50), `senderType` (string, 20 — values `"user"`/`"bot"`), `content` (string, large e.g. 10000), `timestamp` (string/datetime, 50) |
| **user_limits** | `userId` (string, 50), `promptCount` (integer), `lastResetDate` (string, 20) |
| **users** | `userId` (string, 50 — Auth user id), `email` (string, 255), `status` (boolean), `joined` (string, 50) |
| **documents** | `userId` (string, 50 — **Users collection** document id, not Auth id), `conversationId` (string, 50), `fileHash` (string, 100), `fileName` (string, 255), `lastUsedAt` (string, 50), `status` (string, 20 — `pending`/`processing`/`completed`/`failed`), `chunkCount` (integer) |
| **chunks** | `chunkId` (string, 200), `documentId` (string, 50), `conversationId` (string, 50), `userId` (string, 50), `fileHash` (string, 100), `chunkHash` (string, 100), `text` (string, large e.g. 3000) |
| **ingestion_jobs** | `userId` (string, 50), `documentId` (string, 50), `conversationId` (string, 50), `fileHash` (string, 100), `status` (string, 20), `attempts` (integer), `errorMessage` (string, 1000) |

   Set all attributes as **required except**: `chunkCount` (default 0), `attempts` (default 0), `errorMessage` (default empty, not required).

   Copy each collection's ID into the matching `.env` variable.

7. **Permissions**: For all 7 collections, go to **Settings → Permissions** and enable **Document Security**. You don't need to manually add roles — the backend uses an API key (full admin access) for every database call, and for `conversations`/`messages`/`user_limits` the code itself sets per-document permissions at creation time (requires Document Security to be ON to accept that).

   NEEDS MANUAL CONFIGURATION: whether `userId` on `documents`/`chunks`/`ingestion_jobs` should be a native Appwrite "Relationship" attribute — a code comment in `appwrite_utils.py` calls them relationships, but the rest of the code treats them as plain strings. Use **String** type (simplest, works with the code as written); I can't determine from the code alone that a true Relationship attribute is required.

---

## 7. Gemini API setup

- Go to https://aistudio.google.com/apikey → Create API Key → copy into `GOOGLE_API_KEY` in `server/.env`.
- Code uses models `gemini-embedding-001` (embeddings) and `gemini-2.5-flash-lite` (chat) — no extra setup needed beyond the key.

---

## 8. Pinecone setup

- Go to https://www.pinecone.io → sign up → **Create Index**.
- Since Gemini's `gemini-embedding-001` outputs 3072-dimension vectors (noted in the code's own comment), create the index with **dimension = 3072** and any metric (code doesn't specify one, `cosine` is the safe default).
- Copy the **API key** → `PINECONE_API_KEY`.
- Copy the **Index name** you chose → `PINECONE_INDEX_NAME`.

---

## 9. OCRmyPDF setup

Already covered in section 2 — `pip install ocrmypdf` inside the venv, plus Ghostscript + Tesseract + Poppler on your Windows PATH. Nothing further needed for initial setup.

---

## RUNNING THE PROJECT

**Terminal 1 — backend:**
```bash
cd server
.\venv\Scripts\activate
python app.py
```

**Terminal 2 — frontend:**
```bash
cd client
npm run dev
```

Open: **http://localhost:5173**
Backend runs at: **http://127.0.0.1:5000** (you don't open this in a browser, the frontend calls it)

---

## VERIFICATION CHECKLIST

- [ ] Backend starts with no errors, `http://127.0.0.1:5000/api/health` returns `{"status":"ok"}`
- [ ] Frontend loads at `http://localhost:5173`
- [ ] Email/password signup + login works
- [ ] Google login works (redirects through Appwrite)
- [ ] Upload a normal (text) PDF → status moves to "completed"
- [ ] Upload a scanned/image-only PDF → triggers OCR path, still completes
- [ ] Ask a question about the uploaded PDF → get a relevant answer (RAG path)
- [ ] Ask something unrelated to the PDF → falls back to plain Gemini answer
- [ ] Conversation history/messages persist and reload correctly on revisit
- [ ] Deleting a conversation removes it and its messages/documents/chunks

Send me the exact error text when you hit one and I'll diagnose it.