![Python](https://img.shields.io/badge/python-3.10%2B-3776AB?logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/flask-2.3-000000?logo=flask&logoColor=white)

# AI Research Assistant

A personal document vault with a hand-built multi-strategy search engine and encrypted, password-protected sharing links.

> **Note on the repository name:** despite being named `Distributed-File-System`, this is a single-process Flask application backed by SQLite or PostgreSQL — not a distributed system. The README below describes what the code actually does.

Upload research files, and the app extracts and indexes their text (including OCR for images) so you can find them again later — by filename, tag, summary, or full content — using one of three search algorithms implemented from scratch. Documents can also be shared externally through time-limited, download-capped links protected by a password, without ever exposing the underlying file path.

## Features

- **Authentication** — signup/login with bcrypt-hashed passwords and server-side sessions; every route below the login wall re-checks `session['user_id']`.
- **File upload with validation** — accepts TXT, PDF, DOC, DOCX, PNG, JPG, JPEG, and GIF, enforcing a configurable max file size (default 10 MB) and a max page/word-derived page count (default 50 pages) before anything touches disk.
- **Automatic text extraction & indexing** — TXT read directly, PDF via PyPDF2, DOCX via `python-docx`, and images via Tesseract OCR (`pytesseract`); extracted text is stored per document so it's searchable.
- **Tagging** — add/remove free-form tags per document, scoped to the owning user.
- **Three hand-implemented search algorithms** — BFS, DFS, and A*, all searching the same field hierarchy (filename → tags → summary → content) but with different traversal and ranking strategies; every query is logged with algorithm, result count, and execution time for later analysis.
- **Secure sharing** — generate a share link with a required password, optional expiration date, and optional max-download cap. Links carry a Fernet-encrypted token (not a raw document ID), and view/download counts are tracked server-side so a link can be revoked or expire on its own.
- **Dual database backend** — runs against a bundled SQLite file with zero configuration, or against PostgreSQL via `DATABASE_URL`, through a thin compatibility layer so the same `%s`-style queries work against either engine.

## Tech Stack

- **Backend:** Python, Flask, Jinja2
- **Database:** SQLite (default, zero-config) or PostgreSQL via `psycopg2`
- **Auth & crypto:** `bcrypt` for password hashing, `cryptography` (Fernet) for encrypted share tokens
- **Document processing:** `PyPDF2`, `python-docx`, `Pillow` + `pytesseract` (OCR)
- **Serving:** `gunicorn`, deployable to Render (`render.yaml` included)

## Getting Started

### Prerequisites

- Python 3.10+ (the codebase uses modern type-hint syntax)
- Optionally, a PostgreSQL database — SQLite is used automatically if none is configured
- [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) installed locally if you want text extraction from images to work

### Installation

```bash
git clone https://github.com/DharambirAgrawal/Distributed-File-System.git
cd Distributed-File-System/ai_research_assistant
pip install -r requirements.txt
```

### Configuration

All configuration is optional — the app runs out of the box against a bundled SQLite database. To customize it, create a `.env` file in `ai_research_assistant/`:

```env
# Optional: use PostgreSQL instead of the bundled SQLite database
DATABASE_URL=postgresql://username:password@host:5432/database_name

# Optional: override where the SQLite database file lives
SQLITE_DB_PATH=/absolute/path/to/ai_research.db

# Flask session secret — set this to a real secret in any non-local deployment
SECRET_KEY=change-me-in-production

# Optional: key used to encrypt share-link tokens (falls back to a key derived from SECRET_KEY)
SHARE_ENCRYPTION_KEY=

# Optional: upload limits
MAX_UPLOAD_SIZE_MB=10
MAX_UPLOAD_PAGE_LIMIT=50
```

### Run it

```bash
python main.py
```

Then open `http://localhost:5000`, sign up, and log in.

## Usage

1. **Sign up / log in** — accounts are isolated; every query and file operation is scoped to the logged-in user.
2. **Upload a document** — files are stored under `uploads/user_<id>/`, and their text is extracted and indexed immediately.
3. **Tag it** — add tags from the dashboard to make it easier to find later.
4. **Search** — go to `/search`, pick BFS, DFS, or A*, and enter a query. Results show which field matched (filename, tag, summary, or content), a context snippet, and a relevance score.
5. **Share it** — from the dashboard, generate a share link with a password and (optionally) an expiration date or a download limit. Send the resulting `/share/<token>` URL to anyone; they'll need the password to view or download the file.

## How It Works

### Request flow

`main.py` is a single Flask app with route handlers delegating to a small set of service modules: `user/auth.py` (authentication), `storage/uploader.py` (upload + file management), `search/algorithms.py` (search), and `share/share_link.py` + `share/decrypt.py` (secure sharing). There's no background worker or message queue — every request is handled synchronously in the same process.

### Storage layer

`db/database.py` picks an engine at import time based on `DATABASE_URL`: if it's unset or unparseable, the app falls back to a local SQLite file (`db/ai_research.db` by default). Application code is written once against `psycopg2`-style queries (`%s` placeholders, `RETURNING id`), and a `SQLiteCursorWrapper` translates those into SQLite-compatible calls — including simulating `RETURNING id` via `cursor.lastrowid` — so the rest of the codebase doesn't need to know which backend is active.

### Upload & text-extraction pipeline

On upload, `FileUploader` validates the extension, enforces size and page-count limits (reading PDFs/DOCX in-memory to count pages before ever saving them), then saves the file to a per-user directory with automatic de-duplication of filenames. `utils/file_utils.py` then extracts text based on file type — direct read for TXT, `PyPDF2` for PDF, `python-docx` for DOCX, and Tesseract OCR for images — and the result is stored in the `documents.full_text_content` column, which is what the search engine actually queries against.

### Search engine

`search/algorithms.py` implements three traversal strategies over the same four-field hierarchy (filename → tags → summary → full text):

- **BFS** searches every document's filenames first, then every document's tags, then summaries, then content — a breadth-first pass across fields.
- **DFS** searches each document top-to-bottom through the same fields before moving to the next document — a depth-first pass across documents.
- **A\*** scores every document with a heuristic (`f = g + h`) that weights filename word overlap highest, then tag overlap, then summary overlap, then raw term frequency in the content — with a length penalty so shorter, denser matches aren't buried by longer documents — then returns the best-matching field per document in score order.

Match scoring combines `difflib.SequenceMatcher` similarity with exact-substring and word-overlap boosts, and every search is logged to a `search_logs` table with the algorithm used, result count, and execution time in milliseconds — enough to compare the three strategies against each other over real usage.

### Secure sharing

`ShareService` never puts a raw document ID in a link. Instead, `create_share` builds a JSON payload (`{doc, nonce, generated_at}`) and encrypts it with Fernet symmetric encryption, using either a configured `SHARE_ENCRYPTION_KEY` or a key derived from `SECRET_KEY` via SHA-256. The resulting token *is* the URL path segment (`/share/<token>`), so a link can't be tampered with or guessed. The share password is stored as a bcrypt hash, separate from the encryption key, and `ShareAccessManager` gates access on three independent conditions checked on every request — not revoked, not expired, and downloads remaining under `max_downloads` — before the password is even checked. A successful password check unlocks viewing for that browser session and increments a view counter; the actual file download is a second gated step that increments a separate download counter and fails closed once the cap is hit.

## Project Structure

```
ai_research_assistant/
├── main.py                # Flask app & route handlers
├── requirements.txt
├── user/
│   └── auth.py             # Signup/login, bcrypt hashing
├── storage/
│   └── uploader.py          # Upload validation, per-user storage, tagging
├── search/
│   └── algorithms.py        # BFS / DFS / A* search engine
├── share/
│   ├── share_link.py        # Fernet-encrypted share tokens, share CRUD
│   └── decrypt.py           # Share access/authorization gating
├── db/
│   ├── database.py          # SQLite/PostgreSQL compatibility layer
│   ├── models.sql           # PostgreSQL schema
│   └── models_sqlite.sql    # SQLite schema
├── utils/
│   └── file_utils.py        # Text extraction (PDF/DOCX/TXT/OCR)
├── templates/                # Jinja2 templates
└── uploads/                  # Per-user file storage (created at runtime)
```

## API Endpoints

| Method | Route | Description |
|---|---|---|
| GET | `/` | Redirects to dashboard or login |
| GET/POST | `/login` | User login |
| GET/POST | `/signup` | User registration |
| GET | `/logout` | Clear session |
| GET | `/dashboard` | File list + active shares |
| GET/POST | `/upload` | Upload a file |
| GET | `/delete_file/<id>` | Delete a file |
| POST | `/create_share/<document_id>` | Create a password-protected share link |
| POST | `/revoke/<share_id>` | Revoke a share link |
| POST | `/add_tag/<id>` | Add a tag |
| GET | `/remove_tag/<id>/<tag>` | Remove a tag |
| GET/POST | `/search` | Search with BFS/DFS/A* |
| GET | `/view_document/<id>` | View extracted content + tags |
| GET/POST | `/share/<token>` | Password gate for a shared file |
| GET | `/share/<token>/download` | Download a shared file (after auth) |

## Testing

`test_app.py` is a lightweight smoke test that exercises signup, login, and dashboard access against a locally running instance (`python test_app.py` while the app is up on `localhost:5000`) — it's a manual integration check rather than a `pytest` suite.

## Deployment

`render.yaml` at the repo root configures a Render web service (Python 3.12, `gunicorn`-ready). Set `DATABASE_URL` and `SECRET_KEY` as environment variables in production; without `DATABASE_URL`, the deployed instance will fall back to the bundled SQLite file, which is fine for a demo but not for multi-instance or persistent production use.
