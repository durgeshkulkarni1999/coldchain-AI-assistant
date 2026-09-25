# Cold-Chain AI Assistant

An AI agent that lets logistics dispatchers ask plain-English questions about refrigerated fleet incidents, such as *"Which trucks are breaching temperature limits and what should we do?"*. It answers by combining three sources:

- **Fleet telemetry** from a legacy SQL Server database: GPS, IoT temperature, cargo condition, delay and congestion risk.
- **Live corridor conditions** (weather and congestion) from the [Open-Meteo](https://open-meteo.com/) API.
- **Company SOPs** (standard operating procedures), retrieved from a Pinecone vector index.

It replies with an executive summary, a telemetry table and an action plan that cites the relevant SOP rule.

## How it works

```
                 ┌───────────────────────── Streamlit UI (src/ui.py) ─────────────────────────┐
                 │   🧊 Dispatch Console (chat)            🛡️ Security & Audit Logs (admin)   │
                 └───────────────┬───────────────────────────────────────▲─────────────────────┘
                                 │                                       │ writes every step
                                 ▼                                       │
                 LangGraph agent (src/orchestrator.py) ──────► FDE_VIEWS.AgentAuditLog
                  reasoner LLM ⇄ ToolNode
                                 │
        ┌────────────────────────┼─────────────────────────────┐
        ▼                        ▼                             ▼
 query_telemetry_db     fetch_corridor_conditions      search_compliance_sop
 SQL Server view        Open-Meteo REST API            Pinecone (SOP embeddings)
 FDE_VIEWS.VW_ACTIVE_FLEET                             data/policy/*.md|pdf|txt|csv|xlsx
```

The agent's tools live in `src/agent_tools.py`, and its response format is set by `src/prompts/system_prompt.txt`.

### Security model

The database imitates a messy legacy system. Its raw table is `dbo.TBL_SC_FLEET_HIST_RAW`, with cryptic column names like `IOT_TEMP_VAL_C`. The agent never touches it:

- **Readable names:** a view, `FDE_VIEWS.VW_ACTIVE_FLEET`, renames the columns to plain English.
- **Least-privilege account:** the agent connects as `USR_FDE_RO`, which can only `SELECT` from that view. It is denied direct reads of the raw table and all writes to `dbo`.
- **Tamper-resistant audit trail:** the agent can `INSERT` into `FDE_VIEWS.AgentAuditLog`, but can't read, update or delete it. Only an admin can view the log, from the UI.
- **Query guard:** `query_telemetry_db` rejects any SQL that doesn't start with `SELECT`.

## Project layout

```
├── data/
│   ├── raw/dynamic_supply_chain_logistics_dataset.csv   # source fleet telemetry
│   └── policy/Cold_Chain_Incident_SOP_v2.md             # SOPs to embed into Pinecone
├── scripts/
│   ├── ingest_legacy_data.py          # CSV → SQL Server (legacy schema)
│   ├── setup_security_and_views.sql   # view, read-only agent login, audit table
│   └── ingest_sop_pinecone.py         # data/policy → Pinecone (incremental)
├── src/
│   ├── agent_tools.py                 # the 3 LangChain tools
│   ├── orchestrator.py                # LangGraph agent (also a terminal chat)
│   ├── prompts/system_prompt.txt
│   └── ui.py                          # Streamlit app
└── requirements.txt
```

## Prerequisites

- **Python 3.12**
- **Docker**, to run SQL Server 2022 locally
- **A [Pinecone](https://www.pinecone.io/) API key** (the free tier is enough)
- **An LLM**, either:
  - an OpenAI API key, or
  - [Ollama](https://ollama.com/) with `qwen2.5:7b` pulled, for a fully local model

SQL Server is reached through `pymssql`, which installs with `pip`. You don't need a separate ODBC driver.

## Setup

### 1. Install dependencies

```bash
git clone https://github.com/durgeshkulkarni1999/coldchain-AI-assistant.git
cd coldchain-AI-assistant
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

The requirements include `torch` and `sentence-transformers`, so the first install takes a few minutes.

### 2. Create `.env`

Create a `.env` file in the project root:

```ini
PINECONE_API_KEY=your-pinecone-key
OPENAI_API_KEY=your-openai-key          # needed when Agent_llm or Embeddings_model is OPENAI

Embeddings_model=LOCAL                  # LOCAL (BAAI/bge-m3, 1024-dim) or OPENAI (1536-dim)
Local_Embedding_Model=BAAI/bge-m3
Agent_llm=OPENAI                        # OPENAI (gpt-4o-mini) or OLLAMA (qwen2.5:7b)

SQL_SERVER_HOST=localhost
SQL_SERVER_PORT=1433
SQL_ADMIN_USER=sa
SQL_ADMIN_PASSWORD=YourStrong!Passw0rd  # must match MSSQL_SA_PASSWORD in step 3
SQL_AGENT_USER=USR_FDE_RO
SQL_AGENT_PASSWORD=AgentPassword2026!   # must match the password in setup_security_and_views.sql

# HF_TOKEN=your-huggingface-token       # optional, speeds up the first model download
```

Use the same `Embeddings_model` value for ingestion (step 6) and for running the app. Each mode writes to its own Pinecone index: `fde-sop-index-local` or `fde-sop-index-openai`.

### 3. Start SQL Server

```bash
docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=YourStrong!Passw0rd" \
  -p 1433:1433 --name legacy-mssql -d mcr.microsoft.com/mssql/server:2022-latest
```

Wait about 20 seconds, until `docker logs legacy-mssql` shows *"SQL Server is now ready for client connections"*. SQL Server needs at least 2 GB of RAM.

### 4. Load the fleet telemetry

```bash
python scripts/ingest_legacy_data.py
```

This loads the CSV into `dbo.TBL_SC_FLEET_HIST_RAW` (about 32k rows). It replaces the table on every run.

### 5. Create the view, the agent account and the audit table

```bash
docker exec -i legacy-mssql /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P "YourStrong!Passw0rd" -C < scripts/setup_security_and_views.sql
```

Run this once. It isn't idempotent, so a second run fails with "already exists" errors.

### 6. Embed the SOPs into Pinecone

```bash
python scripts/ingest_sop_pinecone.py
```

This creates the Pinecone index if needed and only re-embeds files in `data/policy/` that have changed. It tracks those changes with a hash cache in `data/cache/`. To force a full re-ingest, for example into a new Pinecone project, delete `data/cache/ingestion_hash_cache.json`.

## Running

**Web UI:**

```bash
streamlit run src/ui.py
```

Open http://localhost:8501.

- **🧊 Dispatch Console:** chat with the agent. For example: *"Show me shipments with temperature above 4°C near Long Beach and tell me what to do."*
- **🛡️ Security & Audit Logs:** sign in with `SQL_ADMIN_USER` / `SQL_ADMIN_PASSWORD` to see every step the agent took.

**Terminal chat, without the UI:**

```bash
python src/orchestrator.py        # type 'exit' to quit
```

**Smoke-test the three tools:**

```bash
python src/agent_tools.py
```

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `ModuleNotFoundError` | The venv isn't active. Run `source .venv/bin/activate`. |
| `Invalid object name 'FDE_VIEWS.AgentAuditLog'` | Step 5 wasn't run, or was run from an older script version without the audit table. |
| `Login failed for user 'USR_FDE_RO'` | `SQL_AGENT_PASSWORD` in `.env` doesn't match the password in `setup_security_and_views.sql`. |
| SOP search returns nothing | `Embeddings_model` differs between ingestion and runtime, or a stale `data/cache/` skipped ingestion. Delete the cache and rerun step 6. |
| `HF_TOKEN` warning | Harmless. Set `HF_TOKEN` to raise Hugging Face download rate limits. |

> **Note:** the passwords above are for local development only. Before deploying anywhere reachable from outside, change the `sa` and agent passwords and keep them out of git.
