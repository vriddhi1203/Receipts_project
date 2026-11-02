# Receipts 

A question-answering system that only answers from documents you give it —
with citations — and says "I don't know" when the documents don't cover the
question.
 
Point it at a body of text (lecture notes, documentation, handbooks, papers),
ask a question in plain English, and get back an answer with numbered
references to the exact passages it came from.
 
```
> How many days of attendance do I need to sit the exam?
 
You need 75% attendance in each course to be eligible for the
end-semester exam [1]. Medical exemptions require documentation
submitted within 7 days [3].
 
  [1]  handbook.pdf, p.12 — "a minimum of 75% attendance in each course…"
  [3]  circular-2026-04.pdf — "medical exemption requests must be…"
```
 
Built without a RAG framework. Every stage — loading, token-based chunking,
embedding, vector search, prompt construction, evaluation — is implemented
directly, so the mechanism is visible rather than hidden behind a library.
 
---
 
## Status
 
The only part of this README that changes week to week.
 
| Capability | State |
|---|---|
| Database + project scaffolding | ✅ working |
| Document ingestion (PDF, text, web) | ⬜ not started |
| Token-based chunking | ⬜ not started |
| Embeddings + vector search | ⬜ not started |
| Cited answers from an LLM | ⬜ not started |
| Evaluation harness (Recall@k, MRR) | ⬜ not started |
| HTTP API | ⬜ not started |
| Streamlit UI | ⬜ not started |
 
---
 
## Architecture
 
Two services in one repository:
 
- **`backend/`** — Python. All the logic, laid out in four layers under one
  rule: **every import points inward.** `domain` (pure Python models and
  metrics) ← `application` (use cases and the abstract ports they depend on)
  ← `infrastructure` (Postgres, HTTP clients, embedding models) ←
  `interfaces` (CLI, API).
- **`frontend/`** — a Streamlit app. A client and nothing more: it talks to
  the backend over HTTP and never imports it.
Postgres with the `pgvector` extension does double duty — ordinary
relational storage and vector similarity search — so there's no separate
vector database.
 
Full detail, including what may import what and why: [`docs/architecture.md`](docs/architecture.md).
 
---
 
## Prerequisites
 
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Git
- An API key for a language model provider (not needed until answers are
  implemented — see Status)
Python is not required on your machine. Everything runs in containers.
 
---
 
## Setup
 
```bash
git clone <repo-url>
cd receipts
 
cp .env.example .env
# open .env and fill in the values — see Configuration below
 
docker compose up -d --build
```
 
Check it came up:
 
```bash
docker compose ps
docker compose exec db psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT 1"
```
 
Both services should show as running. If `backend` shows `Exited`, see
Troubleshooting.
 
---
 
## Usage
 
Commands run **inside the backend container**, where the paths you type are
the paths the code sees.
 
In Docker Desktop: open **Containers**, click the `backend` container, open
the **Exec** tab. You'll land in `/app`. Or from a terminal:
 
```bash
docker compose exec backend bash
```
 
Then, depending on what's implemented (see Status):
 
```bash
python -m receipts ingest data/raw/handbook.pdf   # load a document
python -m receipts tokens "organic chemistry"     # inspect tokenization
python -m receipts search "attendance policy"     # retrieval only, no LLM
python -m receipts ask "how many days do I need?" # the full answer
python -m receipts ask "..." --no-rag             # same question, no context
python -m receipts evaluate                       # score against the golden set
```
 
Put the documents you want to ask about in `data/raw/`. That folder is
mounted into the container and is **not** committed to git.
 
Stop everything with `docker compose down`. Your data survives — it lives in
a named volume, not in the containers.
 
---
 
## Project layout
 
```
receipts/
├── docker-compose.yml
├── .env.example               every key, no values
├── docs/                      architecture, corpus notes, results
├── data/raw/                  your documents — gitignored
├── eval/                      the golden question set
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── receipts/
│   │   ├── domain/            models, metrics — pure Python
│   │   ├── application/       ports/ and use_cases/
│   │   ├── infrastructure/    Postgres, embedders, LLM clients
│   │   ├── interfaces/        cli/ and api/
│   │   ├── composition.py     the only file that names concrete classes
│   │   └── config.py
│   └── tests/
└── frontend/                  Streamlit app
```
 
---
 
## Configuration
 
All values live in `.env`, which is never committed. `.env.example` lists
every key with no values.
 
| Variable | Used by | Notes |
|---|---|---|
| `POSTGRES_USER` | db, backend | |
| `POSTGRES_PASSWORD` | db, backend | |
| `POSTGRES_DB` | db, backend | |
| `DATABASE_URL` | backend | Host is `db`, not `localhost` |
| `EMBEDDING_MODEL` | backend | Defaults to a local sentence-transformers model |
| `LLM_PROVIDER` | backend | Which provider to use |
| `LLM_API_KEY` | backend | Keep it out of git — see Troubleshooting |
| `API_URL` | frontend | `http://backend:8000` |
 
---
 
## Troubleshooting
 
**`backend` container shows `Exited (0)`.** A container with no long-running
process exits as soon as it starts, and a stopped container has no shell to
exec into. Until the API server exists, the backend service needs
`command: sleep infinity` in `docker-compose.yml`.
 
**`ModuleNotFoundError: No module named 'receipts'`.** You're in the wrong
directory. Run from `/app` inside the container. There's no packaging config
yet, so the package is only importable from the project root.
 
**`connection refused` talking to the database.** Inside a container,
`localhost` means *that container*. Use the compose service name — `db` —
as the host.
 
**Folders from the layout above are missing after cloning.** Git does not
track empty directories. Any folder with no file in it won't come down with
a clone; an `__init__.py` or `.gitkeep` fixes it.
 
**You committed a secret.** Rotate the key immediately. Removing the commit
is not enough — it's already in the history, and possibly in someone else's
clone.
 
---
 
## Contributing
 
Branching, commit format, pull requests, issues and the board are all
documented in [`INSTRUCTIONS.md`](INSTRUCTIONS.md). Read it before your first
branch.
 
The short version: one issue → one branch off `develop` → one PR →
`Closes #N` in the description → squash merge.
 
Claude finished the response
