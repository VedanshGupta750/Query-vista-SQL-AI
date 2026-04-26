# QueryVista — AI-Powered Database Migration & Dual-DB Intelligence Platform

> Migrate data between SQL and NoSQL databases using AI-generated schema mapping, human-in-the-loop approval, and then query both databases side-by-side with plain English.

---

## Table of Contents

- [What is QueryVista](#what-is-queryvista)
- [The Problem It Solves](#the-problem-it-solves)
- [How It Works — The 4-Phase Architecture](#how-it-works--the-4-phase-architecture)
- [Supported Migration Pipelines (8 Directions)](#supported-migration-pipelines-8-directions)
- [System Architecture](#system-architecture)
- [SQLAI — Post-Migration Dual-DB Intelligence](#sqlai--post-migration-dual-db-intelligence)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Setup & Installation](#setup--installation)
- [API Reference](#api-reference)
- [Frontend — Migration Wizard UI](#frontend--migration-wizard-ui)
- [CLI — Command Line Interface](#cli--command-line-interface)
- [End-to-End User Journey](#end-to-end-user-journey)
- [Security Considerations](#security-considerations)
- [License](#license)

---

## What is QueryVista

QueryVista is a full-stack ETL (Extract, Transform, Load) platform that lets you migrate data between completely different database systems — SQL to NoSQL, NoSQL to SQL, or NoSQL to NoSQL — without writing a single line of migration code.

The platform uses **Azure OpenAI GPT-4o** to analyze your source database schema and generate a complete migration blueprint (field mappings, type conversions, embedding strategies). You review and approve the plan before anything executes. After migration, a built-in **AI Query Analyzer** lets you query both the original and migrated databases simultaneously using plain English, comparing results side-by-side to validate the migration.

**Three interfaces:**
- **Web UI** — 5-step browser-based migration wizard with progress tracking
- **CLI** — Interactive terminal wizard for power users
- **SQLAI Dashboard** — Post-migration dual-database query analyzer

---

## The Problem It Solves

| Problem | How QueryVista Solves It |
|---------|------------------------|
| Engineers spend hundreds of hours writing custom ETL scripts for each migration | AI generates the entire migration plan automatically from schema analysis |
| Migration logic is a black box — you don't know if data mapped correctly until production breaks | Human-in-the-loop review lets you inspect and modify every field mapping before execution |
| QA teams must rewrite the same query in SQL and then again in MongoDB syntax to verify data | Plain English queries execute against both databases simultaneously, results displayed side-by-side |
| Schema translation between SQL and NoSQL is error-prone (types, relationships, nested documents) | GPT-4o understands both paradigms and handles type coercion, flattening, embedding strategies |

---

## How It Works — The 4-Phase Architecture

Every migration follows four phases:

```
Phase 1: DISCOVERY       Extract schema, metadata, relationships from the source database
                         (tables, columns, types, PKs, FKs, indexes, row counts)

Phase 2: AI ARCHITECT    Send schema to Azure GPT-4o, which generates a JSON migration
                         blueprint with field mappings, type conversions, and strategies

Phase 3: HUMAN REVIEW    User reviews the AI plan in a visual editor. Can modify field
                         names, change strategies, or provide natural language feedback
                         ("embed orders inside users") — AI regenerates the plan

Phase 4: EXECUTION       ETL engine reads source data, applies transformations from the
                         approved plan, writes to target database with progress tracking
```

```
 Source DB ──> Schema Extraction ──> Azure GPT-4o ──> Migration Plan (JSON)
                                                           |
                                                    Human Reviews & Edits
                                                           |
                                                    Approved Plan
                                                           |
                                          ETL Engine (read, transform, write)
                                                           |
                                                      Target DB
                                                           |
                                              SQLAI: Query Both DBs with English
```

### Phase 1: Discovery (Schema Extraction)

**For SQL databases (MySQL, PostgreSQL):**
- Uses SQLAlchemy Inspector to extract table names, column names and types, primary keys, foreign keys, indexes, and row counts
- Captures nullable constraints, default values, and relationships

**For MongoDB:**
- Samples 100 documents per collection
- Infers field types from actual values (handles mixed types)
- Captures collection names, field structures, and sample documents

**For CouchDB:**
- Queries the REST API (`_all_dbs`, `_all_docs`)
- Samples documents, infers field types
- Filters out system databases (prefixed with `_`)

### Phase 2: AI Architect (Plan Generation)

The extracted schema is sent to Azure OpenAI GPT-4o with a system prompt:
> "You are a database migration architect"

The AI returns a structured JSON plan:
```json
{
  "collections": [
    {
      "source": "users",
      "target": "users",
      "strategy": "flat",
      "field_mappings": [
        {"source_field": "user_id", "target_field": "_id", "type": "string"},
        {"source_field": "created_at", "target_field": "created_at", "type": "datetime"}
      ],
      "notes": "Direct 1:1 mapping, primary key becomes document _id"
    }
  ]
}
```

Strategies include: `flat` (1:1 mapping), `embed` (nest related data), `reference` (keep as foreign key), `normalize`, `denormalize`.

### Phase 3: Human Review (HITL)

The user can:
- Edit field mappings directly in the JSON editor
- Provide natural language feedback (e.g., "Rename `usr_email` to `email`", "Embed addresses inside users instead of separate collection")
- The AI regenerates the plan incorporating the feedback
- This feedback loop can repeat until the user approves

### Phase 4: Execution (ETL)

The pipeline:
1. Reads data from source database in batches
2. Applies field mappings and type conversions from the approved plan
3. Handles type coercion: `datetime` to ISO string, `Decimal` to float, `bytes` to UTF-8, `ObjectId` to string, nested objects to JSON columns or flattened fields
4. Writes to target database (bulk inserts for performance)
5. Reports progress via callback: `on_progress(current_table, total_tables, table_name)`
6. Returns summary: tables migrated, total rows, errors

---

## Supported Migration Pipelines (8 Directions)

### SQL to NoSQL

| # | Source | Target | Key Features |
|---|--------|--------|-------------|
| 1 | MySQL | MongoDB | Field mapping, type coercion, `datetime` to ISO, `Decimal` to float |
| 2 | MySQL | CouchDB | REST API bulk inserts, `doc_type` tagging per source table |
| 3 | PostgreSQL | MongoDB | JSONB support, field mapping, Neon cloud compatible |
| 4 | PostgreSQL | CouchDB | TIMESTAMP handling, upsert modes, bulk document inserts |

**How SQL-to-NoSQL pipelines work:**
```
SQL Table ──> SELECT * FROM table
                  |
          Apply field_mappings from AI plan
                  |
          Type conversions (datetime→ISO, Decimal→float, bytes→UTF-8)
                  |
          Build documents [{field: value}, ...]
                  |
          Insert into MongoDB collection / CouchDB database
```

### NoSQL to SQL

| # | Source | Target | Key Features |
|---|--------|--------|-------------|
| 5 | MongoDB | MySQL | ObjectId to string, nested object flattening, JSON column auto-detection, batch inserts with row-by-row fallback |
| 6 | MongoDB | PostgreSQL | Same as above plus JSONB native support |
| 7 | CouchDB | MySQL | Auto MySQL version detection, VARCHAR safety guardrails (max 255), JSON column support |
| 8 | CouchDB | PostgreSQL | JSONB for complex fields, BYTEA for binary, TIMESTAMP for dates, ON CONFLICT upsert |

**How NoSQL-to-SQL pipelines work:**
```
MongoDB Collection / CouchDB Database
          |
    Fetch all documents
          |
    Infer SQL column types from document values
          |
    Flatten nested objects (address.city → address_city)
          |
    CREATE TABLE with inferred schema
          |
    Batch INSERT (500 docs/batch, fallback to row-by-row on error)
          |
    SQL Table
```

NoSQL-to-SQL pipelines use a shared `MongoToSqlETLEngine` that supports three migration modes:
- **REPLACE** — Drop and recreate the target table
- **APPEND** — Insert new rows only
- **UPSERT** — Insert or update on primary key conflict (`ON DUPLICATE KEY UPDATE` for MySQL, `ON CONFLICT DO UPDATE` for PostgreSQL)

### NoSQL to NoSQL

| # | Source | Target | Key Features |
|---|--------|--------|-------------|
| - | MongoDB | CouchDB | `ObjectId` to string, `_rev` handling for upserts, `doc_type` tagging, REST API bulk inserts |

---

## System Architecture

```
+------------------------------------------------------------------+
|                         USER INTERFACES                           |
|                                                                   |
|   +----------------+    +----------------+    +----------------+  |
|   |    Web UI      |    |     CLI        |    | SQLAI Dashboard|  |
|   | (5-step wizard)|    | (terminal)     |    | (query analyzer|  |
|   |  HTML/CSS/JS   |    |  Python        |    |  + viz engine) |  |
|   +-------+--------+    +-------+--------+    +-------+--------+  |
|           |                      |                     |          |
+-----------+----------------------+---------------------+----------+
            |                      |                     |
            v                      v                     v
+------------------------------------------------------------------+
|                     FASTAPI BACKEND                               |
|                                                                   |
|   /api/test-connection    /api/extract-schema                     |
|   /api/generate-plan      /api/update-plan                        |
|   /api/approve-plan       /api/execute-migration                  |
|   /api/migration-status   /api/migration-history                  |
|                                                                   |
|   /connect-dual           /generate-dual                          |
|   /table-details-dual     /table-data-dual                        |
|   /optimize               /gen-dashboard                          |
+------------------------------------------------------------------+
            |                                    |
            v                                    v
+-------------------------+      +-----------------------------+
|    PIPELINE ENGINE       |      |     AI SERVICE              |
|                          |      |                             |
|  BasePipeline (abstract) |      |  Azure OpenAI GPT-4o       |
|  8 pipeline classes      |      |  - Schema analysis          |
|  MongoToSqlETLEngine     |      |  - Migration plan gen       |
|  Schema extractors       |      |  - NL → SQL/NoSQL query     |
|  Type converters         |      |  - Query optimization       |
|  Batch processors        |      |  - Viz code generation      |
+-----------+--------------+      +-----------------------------+
            |
            v
+------------------------------------------------------------------+
|                       DATABASES                                   |
|                                                                   |
|  +----------+  +-----------+  +----------+  +----------+         |
|  |  MySQL   |  | PostgreSQL|  | MongoDB  |  | CouchDB  |         |
|  | (pymysql)|  | (psycopg2)|  | (pymongo)|  | (httpx)  |         |
|  +----------+  +-----------+  +----------+  +----------+         |
+------------------------------------------------------------------+
```

### How Components Connect

1. **Web UI / CLI** calls the FastAPI backend REST endpoints
2. **Backend** uses **Pipeline Engine** to extract schemas, test connections, and execute migrations
3. **Pipeline Engine** calls **AI Service** (Azure GPT-4o) for plan generation and query translation
4. **Pipeline Engine** reads from source database, transforms data, writes to target database
5. **SQLAI Dashboard** connects to both databases post-migration for dual-query analysis

---

## SQLAI — Post-Migration Dual-DB Intelligence

After migration completes, SQLAI is the analytics and validation layer. It connects to both the original (source) and migrated (target) databases simultaneously.

### What SQLAI Does

**1. Natural Language Querying**
- Type a question in plain English: *"Show me total active users who made a purchase over $500"*
- AI generates the appropriate query for BOTH databases (SQL for the source, MongoDB aggregation pipeline for the target)
- Executes both queries and returns results side-by-side for comparison
- Validates that the migration preserved data integrity

**2. Query Optimization**
- Paste any slow SQL or MongoDB query
- AI analyzes it and returns an optimized version
- Includes indexing recommendations and explains the reasoning

**3. Data Visualization**
- AI generates Matplotlib/Seaborn charts based on your data
- Ask *"Show me a bar chart of orders per month"* and get a rendered PNG
- Charts are generated dynamically using AI-written visualization code

**4. Dashboard Generation**
- AI analyzes your schema and suggests relevant charts and analytics
- Auto-generates a dashboard layout based on what business stakeholders would want to see

**5. Schema Explorer**
- Browse tables/collections from both databases
- View columns, types, primary keys, foreign keys, indexes
- Paginated data viewer (20 rows per page)

**6. Schema Caching**
- Caches schema metadata in a separate PostgreSQL database
- Avoids re-extracting schema on every query, improving response time

### SQLAI Data Flow

```
User asks: "How many orders were placed last month?"
         |
         v
    Azure GPT-4o receives both schemas + question
         |
         +---> Generates SQL: SELECT COUNT(*) FROM orders WHERE ...
         |
         +---> Generates MongoDB: db.orders.aggregate([{$match: ...}])
         |
         v
    Execute SQL against source PostgreSQL/MySQL
    Execute aggregation against target MongoDB/CouchDB
         |
         v
    Return both results side-by-side + AI explanation
```

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| AI Engine | Azure OpenAI GPT-4o | Schema analysis, migration plan generation, NL-to-query, optimization, visualization |
| Backend | FastAPI (Python) | REST API, migration orchestration, session management |
| Frontend | HTML5, CSS3, Vanilla JavaScript | 5-step migration wizard UI |
| SQL Access | SQLAlchemy + pymysql + psycopg2 | MySQL and PostgreSQL operations |
| MongoDB Access | PyMongo | MongoDB Atlas operations |
| CouchDB Access | httpx (REST API) | CouchDB operations |
| Data Processing | Pandas | DataFrame operations, CSV export |
| Visualization | Matplotlib, Seaborn | Chart generation in SQLAI |
| Containerization | Docker Compose | Local MySQL + CouchDB + phpMyAdmin |
| Deployment | Render | Cloud hosting (render.yaml config) |

---

## Project Structure

```
QueryVista/
|
|-- cli.py                             # Interactive CLI migration wizard
|-- docker-compose.yml                 # Docker: MySQL 8.0 + CouchDB 3 + phpMyAdmin
|-- render.yaml                        # Render cloud deployment config
|-- .env.example                       # Environment variable template
|-- .env                               # Your actual credentials (gitignored)
|
|-- backend/                           # FastAPI backend + pipeline engine
|   |-- main.py                        # FastAPI app — all REST endpoints
|   |-- requirements.txt               # Python dependencies
|   |-- pipelines/
|       |-- __init__.py
|       |-- base.py                    # BasePipeline class, schema extractors, AI plan generator
|       |-- mysql_to_mongo.py          # MySQL -> MongoDB pipeline
|       |-- mysql_to_couchdb.py        # MySQL -> CouchDB pipeline
|       |-- postgres_to_mongo.py       # PostgreSQL -> MongoDB pipeline
|       |-- postgres_to_couchdb.py     # PostgreSQL -> CouchDB pipeline
|       |-- mongo_to_mysql.py          # MongoDB -> MySQL pipeline
|       |-- mongo_to_couchdb.py        # MongoDB -> CouchDB pipeline
|       |-- couchdb_to_mysql.py        # CouchDB -> MySQL pipeline
|       |-- couchdb_to_postgres.py     # CouchDB -> PostgreSQL pipeline
|       |-- mongo_sql_etl.py           # Shared ETL engine for NoSQL-to-SQL migrations
|       |-- dynamic_executor.py        # Utility module
|
|-- frontend/                          # Web UI — migration wizard
|   |-- index.html                     # 5-step stepper interface
|   |-- style.css                      # Animated gradient UI, cards, progress bars
|   |-- app.js                         # State management, API calls, polling
|
|-- SQLAI/                             # Post-migration dual-DB AI query analyzer
|   |-- app2.py                        # Main FastAPI app (primary entry point)
|   |-- ai_service.py                  # Azure OpenAI wrapper — query gen, optimization, safety
|   |-- database_manager.py            # SQL database connection, schema extraction
|   |-- nosql_manager.py               # MongoDB + CouchDB managers, schema profiling
|   |-- cache_manager.py               # Query result caching (PostgreSQL-backed)
|   |-- viz_service.py                 # Matplotlib/Seaborn visualization generation
|   |-- models.py                      # Pydantic request/response models
|   |-- config.py                      # Settings loaded from .env
|   |-- utils.py                       # Helper functions
|   |-- frontend/                      # SQLAI dashboard UI
|       |-- index.html
|       |-- app.js
|       |-- styles.css
|
|-- all_pipelinee/                     # Jupyter notebook prototypes of each pipeline
|   |-- mysql_to_mongo.ipynb
|   |-- mysql_to_couchdb_pipeline (3) (1).ipynb
|   |-- Query_vista.ipynb              # PostgreSQL -> MongoDB
|   |-- postgres_to_couchdb_pipeline (1).ipynb
|   |-- CouchDB-MySQL (1).ipynb
|   |-- CouchDB-PostgreSQL.ipynb
|   |-- Mongo-Sql (1).ipynb
|   |-- Mongodb-COuchdb.ipynb
|   |-- query_vista_postgrestomongo.py
|
|-- initial_migration/                 # Sample CSV data + loader script
|   |-- main.py
|   |-- csvs/
|       |-- customers.csv
|       |-- orders.csv
|       |-- products.csv
|
|-- mysql_couch/                       # Docker config for MySQL + CouchDB
|   |-- docker-compose.yml
|
|-- postgres_couch/                    # PostgreSQL -> CouchDB notebook copy
|
|-- sql/                               # Sample SQL scripts
```

---

## Setup & Installation

### Prerequisites

- Python 3.10+
- Docker & Docker Compose (for local MySQL + CouchDB)
- Azure OpenAI API key (GPT-4o deployment)
- MongoDB Atlas account (free tier works)
- PostgreSQL / Neon account (free tier works)

### 1. Clone the Repository

```bash
git clone https://github.com/VedanshGupta750/Query-vista-SQL-AI.git
cd Query-vista-SQL-AI
```

### 2. Create Virtual Environment & Install Dependencies

```bash
python -m venv venv

# Windows
venv\Scripts\activate

# Linux/Mac
source venv/bin/activate

pip install -r backend/requirements.txt
pip install -r SQLAI/requirements.txt
```

### 3. Configure Environment Variables

Copy the example and fill in your credentials:

```bash
cp .env.example .env
```

Required variables:
```
AZURE_ENDPOINT=https://your-endpoint.openai.azure.com/
AZURE_API_KEY=your-api-key
AZURE_API_VERSION=2024-12-01-preview
DEPLOYMENT_NAME=gpt-4o
MONGO_URL=mongodb+srv://user:pass@cluster.mongodb.net/
SQL_URL=postgresql+psycopg2://user:pass@host/dbname
```

### 4. Start Docker Services (for local MySQL + CouchDB)

```bash
docker-compose up -d
```

This starts:
- **MySQL 8.0** on port `3310`
- **CouchDB 3** on port `5984` (Fauxton UI: http://localhost:5984/_utils)
- **phpMyAdmin** on port `8081` (http://localhost:8081)

### 5. (Optional) Load Sample Data

```bash
cd initial_migration
python main.py
```

This loads `customers.csv`, `orders.csv`, and `products.csv` into MySQL.

### 6. Run the Backend API

```bash
cd backend
uvicorn main:app --reload --port 8000
```

Backend available at: http://localhost:8000

### 7. Open the Frontend

Serve the frontend directory:
```bash
cd frontend
python -m http.server 3000
```

Open http://localhost:3000 in your browser.

### 8. Run SQLAI (Post-Migration Query Analyzer)

```bash
cd SQLAI
uvicorn app2:app --reload --port 8000
```

SQLAI dashboard available at: http://localhost:8000

### 9. Or Use the CLI

```bash
python cli.py
```

---

## API Reference

### Domain 1: Migration Pipeline Endpoints

Base URL: `http://localhost:8000`

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Health check |
| `GET` | `/api/databases` | List supported database types (MySQL, PostgreSQL, MongoDB, CouchDB) |
| `GET` | `/api/pipelines` | List all 8 migration pipeline directions |
| `POST` | `/api/test-connection` | Test database connection with provided credentials |
| `POST` | `/api/extract-schema` | Extract schema/metadata from source DB, returns `session_id` |
| `POST` | `/api/generate-plan` | Send schema to Azure GPT-4o, get migration plan JSON |
| `POST` | `/api/update-plan` | Submit human feedback, AI regenerates the plan |
| `POST` | `/api/approve-plan` | Mark migration plan as approved |
| `POST` | `/api/execute-migration` | Execute the approved migration (background task) |
| `GET` | `/api/migration-status/{session_id}` | Poll migration progress (percentage, current table) |
| `GET` | `/api/migration-history` | List all completed/failed migrations |

### Domain 2: SQLAI Dual-Database Intelligence Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/connect-dual` | Connect to both source and target databases, fetch schemas |
| `GET` | `/schema/source` | Get source database schema details |
| `GET` | `/schema/target` | Get target database schema details |
| `POST` | `/generate-dual` | Natural language query — executes against both DBs, returns side-by-side results |
| `POST` | `/table-details-dual` | Get column details, PKs, FKs, indexes for a table/collection |
| `POST` | `/table-data-dual` | Fetch paginated data (20 rows/page) from either DB |
| `POST` | `/optimize` | AI-powered query optimization with indexing recommendations |
| `POST` | `/gen-dashboard` | AI suggests analytics charts based on schema |
| `POST` | `/migrate` | Run full end-to-end migration from SQLAI interface |

### Example: Test a Connection

```bash
curl -X POST http://localhost:8000/api/test-connection \
  -H "Content-Type: application/json" \
  -d '{
    "db_type": "mysql",
    "host": "localhost",
    "port": 3310,
    "user": "user1",
    "password": "pass123",
    "database": "testdb"
  }'
```

### Example: Generate a Migration Plan

```bash
curl -X POST http://localhost:8000/api/generate-plan \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "your-session-id",
    "source_type": "mysql",
    "target_type": "mongodb"
  }'
```

### Example: Natural Language Dual Query

```bash
curl -X POST http://localhost:8000/generate-dual \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Show me total active users who made a purchase over $500",
    "source_url": "postgresql://...",
    "target_url": "mongodb://...",
    "source_db_name": "sales_db",
    "target_db_name": "sales_mongo",
    "safe_mode": true
  }'
```

Response:
```json
{
  "source_result": {
    "query_text": "SELECT COUNT(*) FROM users JOIN orders ON ...",
    "data_preview": [{"count": 142}]
  },
  "target_result": {
    "query_text": "db.users.aggregate([{$lookup: ...}])",
    "data_preview": [{"count": 142}]
  },
  "explanation": "Joined users with orders filtering purchases above $500..."
}
```

---

## Frontend — Migration Wizard UI

The web frontend is a single-page application with a 5-step stepper:

**Step 1 — Select Databases:** Choose source and target from MySQL, PostgreSQL, MongoDB, CouchDB. The UI shows which pipeline will be used.

**Step 2 — Connect:** Enter connection credentials (host, port, username, password, database name or connection URL). Test both connections before proceeding.

**Step 3 — Schema Review:** Displays extracted schema in a structured view — table/collection names, column types, primary keys, foreign keys, indexes, and row counts.

**Step 4 — AI Plan Review:** Shows the AI-generated migration plan in an editable JSON view. You can modify field mappings directly or type natural language feedback and have the AI regenerate.

**Step 5 — Execute:** Runs the migration with a live progress bar. Shows results table with rows migrated per table and any errors.

---

## CLI — Command Line Interface

The CLI (`cli.py`) provides the same functionality as the web UI in a terminal:

```
$ python cli.py

=== QueryVista Migration CLI ===

Select a pipeline:
  1. MySQL → MongoDB
  2. MySQL → CouchDB
  3. PostgreSQL → MongoDB
  4. PostgreSQL → CouchDB
  5. MongoDB → MySQL
  6. MongoDB → CouchDB
  7. CouchDB → MySQL
  8. CouchDB → PostgreSQL
  9. Launch SQLAI (standalone)

Enter choice:
```

The CLI walks you through:
1. Pipeline selection
2. Source and target credential entry (with sensible defaults)
3. Connection testing
4. Schema extraction and display
5. AI plan generation and review
6. Feedback loop (provide feedback or approve)
7. Migration execution with progress output
8. Optional SQLAI launch for post-migration exploration

---

## End-to-End User Journey

Here is a complete example: migrating a MySQL database to MongoDB.

```
1. User selects MySQL as source, MongoDB as target
   
2. User enters MySQL credentials:
   Host: localhost, Port: 3310, User: user1, Password: pass123, DB: testdb
   
3. User enters MongoDB Atlas URL:
   mongodb+srv://user:pass@cluster.mongodb.net/migrated_db

4. System tests both connections                              --> Both OK

5. System extracts MySQL schema:
   - users: 5 columns (id, name, email, created_at, status), 2,500 rows
   - orders: 7 columns (id, user_id, product_id, ...), 15,000 rows
   - products: 4 columns (id, name, price, category), 500 rows

6. Schema sent to Azure GPT-4o
   AI returns migration plan:
   {
     "collections": [
       {
         "source": "users",
         "target": "users", 
         "strategy": "flat",
         "field_mappings": [
           {"source_field": "id", "target_field": "_id", "type": "string"},
           {"source_field": "created_at", "target_field": "created_at", "type": "datetime"}
         ]
       },
       ...
     ]
   }

7. User reviews the plan
   Feedback: "Embed orders inside the users collection instead of separate"
   AI regenerates plan with embedded strategy

8. User approves the plan

9. ETL executes:
   - Reading users table... 2,500 rows
   - Reading orders table... 15,000 rows  
   - Embedding orders into user documents...
   - Inserting into MongoDB... done
   - Reading products table... 500 rows
   - Inserting into MongoDB... done
   Result: 3 tables migrated, 18,000 total rows, 0 errors

10. User launches SQLAI
    Asks: "How many users have more than 5 orders?"
    
    Source (MySQL):  SELECT COUNT(*) FROM users u JOIN orders o ON ...  --> 342
    Target (MongoDB): db.users.aggregate([{$match: {orders: {$gt: 5}}}]) --> 342
    
    Data integrity confirmed.
```

---

## Security Considerations

- **SQL Injection Protection:** All SQL queries use SQLAlchemy parameterized queries
- **NoSQL Injection Protection:** PyMongo driver auto-escapes queries; CouchDB uses JSON Mango queries
- **Human-in-the-Loop:** AI-generated plans require explicit human approval before any data moves
- **Safe Mode:** SQLAI blocks dangerous keywords (INSERT, DELETE, UPDATE, DROP, ALTER, TRUNCATE) in query mode
- **Credential Storage:** All secrets stored in `.env` file which is gitignored
- **Visualization Sandboxing:** AI-generated chart code runs with restricted globals (no `exit()`, `quit()`, or system access)

---

## License

This project is for educational and portfolio purposes.

---

Built by Vedansh Gupta
