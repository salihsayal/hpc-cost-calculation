# HPC Resource & Cost Aggregator

A containerized 3-stage ETL (Extract, Transform, Load) and reporting pipeline designed to automate accounting and cost distribution for heterogeneous compute environments (Bare-Metal HPC, VDI, and Cloud environments).

> **Technical Showcase:** This repository serves as a read-only architectural showcase. The raw source code, proprietary business logic, and internal configurations have been removed to comply with enterprise confidentiality.

## 🏗️ Architecture & Orchestration

The system is built as a strictly sequential pipeline to prevent race conditions and ensure data integrity. Instead of running continuously via `docker compose up`, the pipeline consists of three independent Python programs encapsulated in separate Docker containers. They are executed in a logical chain (ingestion ➔ automatisation ➔ notification) via shell scripts.

Data exchange between containers is handled via shared Docker volumes (`shared_files/`), while the central state is managed in a persistent PostgreSQL database utilizing a local bind mount.

---

## 🧩 Pipeline Modules

### 1. Ingestion (`HPC_COST_INGESTION`)

Acts as the automated data collector and parser.

* **Graph API Integration:** Connects to a dedicated Microsoft 365 service mailbox via OAuth2 (Client Credentials Flow) using the Microsoft Graph API.


* **Smart Filtering:** Only processes unread emails, extracting data based on specific case-sensitive subject keywords and dates (e.g., "July 2026").


* **Type A (Invoice):** Downloads attachments to generate delimiter-separated invoice and VDI text files.


* **Type B (PLM):** Parses the email body directly to generate storage usage files (`sfs` and `obs`).


* **Type C (Azure):** Processes both the email body and attachments to extract cloud usage data (e.g., `jobs`, `sub`, `acc`).


* **Type D (CSV):** Identifies raw HPC export CSV files and saves them unaltered to the shared directory.


* **Parsing & Normalization:** Extracts payloads using Regex and BeautifulSoup, normalizing them into standardized DSV (Delimiter-Separated Values) files.



### 2. Automatisation (`HPC_COST_AUTOMATISATION`)

The core processing and calculation engine.

* **Data Ingestion:** Reads the generated CSV and DSV files directly from the shared volume, automatically detecting the newest `.csv` file containing granular HPC metrics (e.g., `jobid`, `runningtime`, `flavor`).


* **Active Directory (LDAP) Sync:** Connects to the corporate AD to synchronize user metadata and map compute resources to specific departments.


* **Self-Healing Mappings:** Users without an assigned department are automatically parked in an `unknown_departments.txt` file. Once manually mapped by an admin, the system dynamically updates the database on the next run.


* **Cost Calculation:** Transforms raw hours, node flavors, and storage consumption into actual financial metrics based on predefined tariffs, enforcing strict data formatting rules (e.g., comma-separated SFS shares).


* **Data Persistence:** Upserts all aggregated data into schema-separated PostgreSQL tables.



### 3. Notification (`HPC_COST_NOTIFICATION`)

The automated reporting and distribution service.

* **Timeframe Logic:** Calculates data for a specific date passed via CLI argument, or automatically defaults to processing data from two months prior.


* **Dynamic Distribution:** Reads mapping configurations from `monate.txt` and `departments.txt`. If a new department is detected in the database, it is automatically appended to the list and skipped until a valid email address is manually configured.


* **Multi-Threaded Plotting:** Uses `Plotly` and `kaleido` to concurrently generate absolute cost pie charts and 3-month stacked bar trend charts for both PLM Cloud and Azure.


* **Rich Emails:** Compiles HTML reports and injects the generated PNG plots directly into the email body as inline attachments using Content-ID (`cid:`). Interactive `.html` versions of the plots are also included as file attachments for deep dives.



---

## 📂 Project Structure

```text
.
├── docker-compose.yml              # Central orchestration file
├── HPC_COST_INGESTION/             # Stage 1: Mail retrieval & parsing
│   ├── m365_connector.py           # MS Graph API OAuth2 handler
│   └── parser_type_[a-d].py        # Specific regex and attachment parsers
├── HPC_COST_AUTOMATISATION/        # Stage 2: DB, LDAP, and Calculation
│   ├── import_to_db/               # LDAP sync and CSV/DSV importers
│   ├── querys/                     # SQL aggregations and self-healing logic
│   └── database/                   # PostgreSQL setup & bind mounts
├── HPC_COST_NOTIFICATION/          # Stage 3: Visuals & Dispatch
│   ├── plot.py                     # Plotly chart generation (HTML & PNG)
│   └── email_handler.py            # Graph API mail dispatcher (Inline CID)
├── shared_files/                   # Shared volume for ETL handovers (MOCK DATA)
│   ├── csv-file/                   # Raw HPC exports
│   ├── dsv-file/                   # Parsed intermediary files
│   └── txt-files/                  # Mapping configs (departments, months)
├── plots/                          # Output directory for generated graphs
└── run_pipeline.sh                 # Sequential execution wrapper

```

*Developed for automated infrastructure accounting and cloud cost observability.*
