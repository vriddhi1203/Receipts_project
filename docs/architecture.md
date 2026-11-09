# Architecture
 
## The rule
 
The backend is laid out in four layers. **Every import points inward.**
 
A module may import from its own layer or any layer inside it. Never
outward. `domain` does not know `application` exists. `application` does not
know Postgres exists.
 
```
interfaces  →  composition  →  infrastructure  →  application  →  domain
```
 
The `frontend/` service is outside all of this. It talks to the backend over
HTTP and never imports it.
 
## The layers
 
| Layer | Contains | May import | Must never import |
|---|---|---|---|
| `domain` | Models, value objects, metrics, errors. Pure Python. | stdlib, `dataclasses` | anything else in `receipts`; any third-party library |
| `application` | `ports/` (abstract base classes) and `use_cases/` (one per user action) | `domain` | `infrastructure`, `interfaces`, `psycopg`, `httpx`, `fastapi` |
| `infrastructure` | Implementations of the ports — Postgres repositories, loaders, embedders, LLM clients | `application.ports`, `domain`, any library | `interfaces`, `composition`, `application.use_cases` |
| `composition.py` | The only file that names concrete classes. Builds and wires objects. | everything inward | `interfaces` |
| `interfaces` | CLI and HTTP API. Thin: parse input, call a use case, format output. | `composition`, `application` types, `domain` | `infrastructure` |
 
The important consequence: a use case receives its dependencies as *ports*,
so a test can hand it fakes. The `application` test suite needs no database,
no network and no API key.
 
## Layout
 
```
backend/receipts/
├── domain/              models.py · values.py · metrics.py · errors.py
├── application/
│   ├── ports/           the abstract base classes
│   └── use_cases/       IngestDocument · AnswerQuestion · RunEvaluation · …
├── infrastructure/
│   ├── loaders/         text · pdf · web
│   ├── chunking/        fixed_token · sentence
│   ├── embedding/       sentence_transformer · api
│   ├── llm/             provider clients, retry
│   ├── prompts/         reads the templates in backend/prompts/
│   └── persistence/     connection · migrations · mappers · repositories
├── interfaces/
│   ├── cli/
│   └── api/
├── composition.py
└── config.py
```
 
## Where do I add…
 
| What you're adding | Where it goes | What else changes |
|---|---|---|
| A new document format | `infrastructure/loaders/` | Register it in `composition.py`. Nothing else. |
| A new chunking strategy | `infrastructure/chunking/` | `composition.py` |
| A different embedding model | `infrastructure/embedding/` | `composition.py`, and **re-embed everything** — stored vectors from another model are not comparable |
| A different LLM provider | `infrastructure/llm/` | `composition.py` |
| A new metric | `domain/metrics.py` | The evaluation use case |
| A new field on a stored object | `domain/models.py`, a migration, `persistence/mappers.py` | The repository that reads it |
| A new thing a user can do | `application/use_cases/` + a port if it needs new I/O | `composition.py`, then a CLI or API entry point |
| A new API endpoint | `interfaces/api/` | Nothing inward — it calls an existing use case |
 
If adding something means editing a file in a layer *inside* where it
belongs, the abstraction is probably wrong. That's worth raising rather than
working around.
 
## Checks
 
There is no automated enforcement yet. Two questions catch most violations
in review:
 
- Does anything in `domain/` import a library?
- Does any use case name a concrete class?
A ~20 line `pytest` using the `ast` module can check both. Planned for
week 8, once there's enough code to be worth protecting.
 
## Schema
 
The schema is defined by the migration files in
`backend/receipts/infrastructure/persistence/migrations/`, applied in order.
Read them — numbered `CREATE TABLE` statements describe the database more
accurately than prose ever will, and they cannot drift out of date.
 
Migrations are append-only. Once a file has been applied, it is never
edited; write a new one.
 
## See also
 
- [`INSTRUCTIONS.md`](../INSTRUCTIONS.md) — branching, commits, PRs, the board