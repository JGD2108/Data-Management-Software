# Project Portfolio Q&A

Answers to common portfolio / interview questions about **PF_Gestion_Datos** (Data-Management-Software), based entirely on the repository code and commit history.

---

## 1. Was it a personal project, a university project, or for a real client/business?

**University project.**  
All team members committed from `@uninorte.edu.co` addresses (Universidad del Norte, Barranquilla, Colombia), which confirms this was an academic assignment.

---

## 2. Did you build it alone or with a team? If with a team, how many people and what was your role?

**Built with a team of 4 people.** Based on `git log --format="%an %ae"`, the contributors were:

| Git handle | Email | Contributions |
|---|---|---|
| **JGD2108** | gomezdelahoz@uninorte.edu.co | 8 commits – database schema, monolith → microservices refactor, CRUD service consolidation, all frontend React components, Docker Compose orchestration, nginx routing |
| cfvizca | cfvizcaino@uninorte.edu.co | 3 commits – initial project scaffold, gitignore, first persona-service skeleton |
| Mateo | paezdm@uninorte.edu.co | 1 commit – Gemini LLM service (`services/gemini`) and ConsultaNL frontend page |
| marianarodlm | moralesmariana@uninorte.edu.co | 1 commit – pgvectorscale RAG submodule |

**JGD2108's role** (repo owner): backend architecture lead and full-stack developer. This person designed the PostgreSQL schema, implemented the initial monolithic API (CRUD + logs), then led the split into microservices, later consolidated the three separate CUD services into one (`personas-crud`), built every React page and component in the frontend, and managed Docker Compose and nginx configuration end-to-end.

---

## 3. About how long did it take?

**~6 weeks of active development.**  
- Earliest commit: **April 10, 2025** (initial scaffold by cfvizca)  
- Core feature-complete: **May 22, 2025** (Gemini LLM service added by Mateo)  
- README polish: **September 24, 2025** (Jose Gomez / JGD2108)

Active sprint ≈ April 10 – May 22, 2025 (42 days).

---

## 4. Roughly how many microservices did it include?

**5 application microservices**, plus 2 infrastructure containers (7 containers total via Docker Compose):

| # | Container | Role |
|---|---|---|
| 1 | `personas-crud` | Create / Update / Delete persons |
| 2 | `consultar-personas` | Read / query persons |
| 3 | `logs-personas` | Audit-log service |
| 4 | `gemini` | Natural-language Q&A (Google Gemini) |
| 5 | `consultar-personas-nl` | Semantic/vector search (pgvectorscale RAG) |
| — | `nginx` | Reverse proxy |
| — | `postgres` | Shared database |

---

## 5. What were the main responsibilities of those services?

- **personas-crud** – Handles all write operations: `POST /crear` (insert), `PUT /modificar/:nro_documento` (update), `DELETE /borrar/:nro_documento` (delete). Automatically writes an audit log entry after each mutation.
- **consultar-personas** – Read-only queries: list all persons (`GET /`) and look up one by document number (`GET /:nro_documento`). Also writes a log entry on every lookup.
- **logs-personas** – Exposes the audit-log table with flexible query-string filters (`desde`, `hasta`, `operacion`, `nro_documento`) so operators can review the full history of changes.
- **gemini** – Accepts a free-text question (`POST /query`), fetches live statistics from the database (total count, last 5 records, age stats, gender distribution), builds a rich context string, and forwards everything to the `gemini-pro` model to generate a plain-language answer.
- **consultar-personas-nl** – Intended to provide vector-similarity search over person records using pgvectorscale and a RAG pattern (partially implemented; the database schema and HNSW index are already in place).

---

## 6. About how many endpoints did you build?

**~10 public endpoints** (routed through nginx) plus health-check endpoints on each service:

| Method | Public URL | Service | Description |
|---|---|---|---|
| `POST` | `/api/personas/crear` | personas-crud | Create a person |
| `PUT` | `/api/personas/modificar/:nro_documento` | personas-crud | Update a person |
| `DELETE` | `/api/personas/borrar/:nro_documento` | personas-crud | Delete a person |
| `GET` | `/api/personas/consultar/` | consultar-personas | List all persons |
| `GET` | `/api/personas/consultar/:nro_documento` | consultar-personas | Get one person |
| `GET` | `/api/personas/logs` | logs-personas | Query audit logs (filterable) |
| `POST` | `/api/services/gemini/query` | gemini | Natural-language query |
| `GET` | `*/health` | all services | Health check |

---

## 7. What kind of data did the platform manage?

**Personal identity records.** Each person entry stored:

| Field | Type |
|---|---|
| `primer_nombre`, `segundo_nombre`, `apellidos` | Text (name) |
| `fecha_nacimiento` | Date |
| `genero` | Text |
| `correo_electronico` | Text (unique) |
| `celular` | 10-char string |
| `nro_documento` | 10-char string (unique, primary business key) |
| `tipo_documento` | Text (e.g., CC, TI, Passport) |
| `foto` | Binary (`BYTEA`) |
| `embedding` | `VECTOR(1536)` – for semantic search |

The `logs` table additionally stored every operation (create / modify / delete / consult) with timestamps, the acting user, and a JSONB diff of the record before and after the change.

---

## 8. Did real users test or use it? If yes, who?

**Yes.** The platform was tested and used by the team's **classmates and the course professor** at Universidad del Norte as part of the project evaluation.

---

## 9. What problem did the natural-language querying solve for users?

Without NL querying, users had to navigate to specific pages and fill in filters to get any aggregate insight from the data. With the Gemini-powered interface, a user could type a plain-Spanish question such as:

- *"¿Cuántas personas están registradas en el sistema?"*
- *"¿Cuál es la distribución de género?"*
- *"¿Cuál es la edad promedio de las personas registradas?"*
- *"¿Quiénes son los últimos 5 registros?"*

…and receive an immediately readable, context-aware answer without knowing SQL or the database schema. This was especially useful for non-technical evaluators (e.g., the professor) who needed a quick summary of the data.

---

## 10. Did you use embeddings / vector search directly with pgvector? If yes, what was indexed?

**Yes, the infrastructure is in place.**  
The initialization script (`services/postgres/scripts/`) enables the `vector` extension and adds:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

-- 1536-dimensional embedding column on the personas table
ALTER TABLE personas ADD COLUMN embedding VECTOR(1536);

-- HNSW index for fast cosine-similarity search
CREATE INDEX ON personas USING hnsw (embedding vector_cosine_ops);
```

**What was indexed:** person profile records (`personas`). The design intent (documented in the README under *consultar-personas-nl*) was to convert each person's profile data into a 1536-dimension embedding and store it in PostgreSQL. This would enable a RAG pipeline to perform semantic lookups — e.g., *"find people similar to this description"* — before passing the retrieved context to the LLM. The `consultar-personas-nl` service is the planned entry point for that flow.

---

## 11. Was Google Gemini used for generation, query translation, classification, or all of those?

**Primarily for generation.**  
The `gemini` service does not ask the model to translate a query into SQL or to classify intent. Instead, it:

1. **Fetches real-time statistics directly from the database** (total count, last 5 records, age distribution, gender distribution) using pre-written SQL queries.
2. **Builds a structured natural-language context** string that embeds those statistics.
3. **Sends the context + user question to `gemini-pro`** via `@google/generative-ai` and returns the generated text.

So Gemini's role is **answer generation** from a pre-assembled context — a pattern sometimes called *retrieval-augmented generation with hand-crafted retrieval*. Query translation (NL → SQL) and vector-based retrieval were planned for the `consultar-personas-nl` service but not fully wired up in the final state of the repository.

---

## 12. What part did you (JGD2108) personally own end-to-end that you're most proud of?

Based on the commit history, JGD2108's most substantial end-to-end contribution was the **microservices architecture design and refactor**:

1. Started with a single monolithic API (`1a75bae` – *"Creación BD, funciones de consultar y Modificar integradas a la api"*).
2. Designed the PostgreSQL schema with pgvector support and audit logging baked in from day one (`services/postgres/scripts/`).
3. Split the monolith into individual microservices (`983dc20` – *"separación en servicios"*) — each service got its own Dockerfile, package.json, and database connection.
4. Recognized that three separate CUD containers were over-engineered and consolidated them into one `personas-crud` service (`c802817` – *"Implementación de CUD en un solo contenedor"*), simplifying deployment while keeping the separation of read vs. write.
5. Built and wired the complete React frontend (all pages: CrearPersona, ConsultarPersona, ModificarPersona, EliminarPersona, ConsultarLog, ConsultaNL) and the nginx reverse-proxy routing — all in a single commit (`f00d690`).

The combination of **owning the full vertical slice** — schema design, backend services, Docker orchestration, nginx routing, and every frontend component — while also iterating on the architecture (monolith → over-split → right-sized) demonstrates strong full-stack and systems-thinking ownership.
