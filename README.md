\# Marketing Analytics Platform



An internal-style analytics platform for marketing performance measurement, built on

Google BigQuery, dbt, and Python.



This repository is maintained as a long-term engineering project. It is structured,

documented, and tested the way a small analytics engineering team would maintain an

internal platform — not as a collection of exploratory notebooks.



\---



\## Table of Contents



\- \[Project Vision](#project-vision)

\- \[Objectives](#objectives)

\- \[Architecture Overview](#architecture-overview)

\- \[Technology Stack](#technology-stack)

\- \[Repository Structure](#repository-structure)

\- \[Data Sources](#data-sources)

\- \[Data Model](#data-model)

\- \[Development Standards](#development-standards)

\- \[Documentation Standards](#documentation-standards)

\- \[Setup Instructions](#setup-instructions)

\- \[Common Commands](#common-commands)

\- \[Cost Controls](#cost-controls)

\- \[Learning Roadmap](#learning-roadmap)

\- \[Contribution Guidelines](#contribution-guidelines)

\- \[Future Roadmap](#future-roadmap)



\---



\## Project Vision



Marketing teams routinely operate on numbers that nobody can trace. A "conversion rate"

appears in three dashboards with three different values, because the logic behind it was

written three times, in three places, by three people.



This platform exists to make marketing measurement \*\*reproducible, tested, and

explainable\*\*. Every metric has exactly one definition, that definition lives in version

control, and it is validated on every run.



The platform ingests behavioural event data and marketing spend, models it into a

dimensional warehouse, exposes a governed metrics layer, and serves both scheduled

reporting and ad-hoc statistical analysis including experiment evaluation.



\### Design Principles



1\. \*\*One definition per metric.\*\* Business logic is defined once and referenced

&#x20;  everywhere. Duplicated logic is treated as a defect.

2\. \*\*Everything in version control.\*\* No transformation logic exists only in the

&#x20;  BigQuery console. If it is not in a file, it does not exist.

3\. \*\*Tested data, not just tested code.\*\* Data quality assertions run alongside unit

&#x20;  tests. A pipeline that completes successfully but produces wrong numbers has failed.

4\. \*\*Decisions are recorded.\*\* Significant architectural choices are captured as

&#x20;  Architecture Decision Records, with the alternatives that were rejected and why.

5\. \*\*Cost is a first-class constraint.\*\* Query cost is treated as a performance metric,

&#x20;  not an afterthought.



\---



\## Objectives



\*\*Engineering objectives\*\*



\- Build a modular, importable Python package with real separation of concerns

\- Implement an ELT pipeline with idempotent, incrementally-processed transformations

\- Establish a dimensional data model that survives changing business questions

\- Achieve meaningful test coverage across both code and data

\- Operate a reproducible local development environment



\*\*Analytical objectives\*\*



\- Produce a governed set of marketing KPIs with documented definitions

\- Implement multi-touch attribution and compare it against last-click

\- Build cohort retention and customer lifetime value models

\- Design and evaluate A/B tests with correct statistical methodology

\- Deliver a self-service dashboard over the metrics layer



\---



\## Architecture Overview



```

┌─────────────────────┐     ┌─────────────────────┐

│  BigQuery Public    │     │  Marketing Spend    │

│  GA4 Sample Events  │     │  (generated source) │

└──────────┬──────────┘     └──────────┬──────────┘

&#x20;          │                           │

&#x20;          │       EXTRACT / LOAD      │

&#x20;          │        (Python)           │

&#x20;          ▼                           ▼

&#x20;    ┌───────────────────────────────────────┐

&#x20;    │            raw  (BigQuery)            │   append-only, untransformed

&#x20;    └───────────────────┬───────────────────┘

&#x20;                        │

&#x20;                        │   TRANSFORM (dbt)

&#x20;                        ▼

&#x20;    ┌───────────────────────────────────────┐

&#x20;    │  staging   — typed, renamed, 1:1       │

&#x20;    ├───────────────────────────────────────┤

&#x20;    │  intermediate — joins, sessionisation  │

&#x20;    ├───────────────────────────────────────┤

&#x20;    │  marts  — facts \& dimensions           │

&#x20;    └───────────────────┬───────────────────┘

&#x20;                        │

&#x20;       ┌────────────────┼────────────────┐

&#x20;       ▼                ▼                ▼

&#x20; ┌───────────┐   ┌────────────┐   ┌────────────┐

&#x20; │ Streamlit │   │ Statistical│   │  Scheduled │

&#x20; │ Dashboard │   │  Analysis  │   │  Reporting │

&#x20; └───────────┘   └────────────┘   └────────────┘

```



\### Why ELT rather than ETL



Transformation happens \*\*inside\*\* BigQuery, after loading, rather than in Python before

loading.



The reason is where the compute lives. BigQuery is a distributed columnar engine that

parallelises aggregation across thousands of workers. A Python process transforming the

same data runs on one machine and must first pull the data across the network. For any

non-trivial volume, in-warehouse transformation is faster by orders of magnitude.



The secondary benefit matters more for maintainability: because raw data lands untouched,

transformation logic can be rewritten and replayed without re-extracting from source. If

a business rule was wrong six months ago, the fix is a code change and a rebuild — not a

data recovery exercise.



\*\*When ETL is still correct:\*\* when data must be masked or dropped before it touches the

warehouse for compliance reasons, or when the source is so large that loading raw is

itself prohibitive.



\### Why layered models rather than direct queries



Each dbt layer has one job:



\- \*\*staging\*\* — one model per source table. Cast types, rename to project conventions,

&#x20; no joins and no business logic. This is the isolation boundary: if a source schema

&#x20; changes, exactly one file changes.

\- \*\*intermediate\*\* — the genuinely complex logic. Sessionisation, attribution windows,

&#x20; deduplication. Not exposed to end users.

\- \*\*marts\*\* — dimensional facts and dimensions, named in business language. This is the

&#x20; contract with consumers.



The cost is more objects to maintain. The benefit is that a source schema change, a

business logic change, and a presentation change are three different files, touched by

three different pull requests.



\---



\## Technology Stack



| Layer | Choice | Rationale |

|---|---|---|

| Warehouse | BigQuery | Serverless, generous free tier, native public datasets, columnar with partition/cluster pruning |

| Transformation | dbt-core | Industry standard for analytics engineering; dependency resolution, testing, lineage, and docs in one tool |

| Language | Python 3.11+ | Ingestion, orchestration, statistics, dashboard |

| Dependency management | uv | Fast, lockfile-based, reproducible environments |

| Testing | pytest | Unit and integration tests for Python |

| Data testing | dbt tests + dbt-expectations | Assertions on the data itself, not just the code |

| Linting / formatting | ruff, sqlfluff | Enforced in pre-commit and CI |

| Type checking | mypy | Catches interface errors before runtime |

| Dashboard | Streamlit | Python-native, keeps the whole stack in one language |

| Orchestration | Python CLI + Cloud Scheduler | Deliberately simple to start; see ADR-0004 |

| CI | GitHub Actions | Lint, type-check, test, and dbt build on every pull request |

| Infrastructure | Terraform | Datasets and service accounts defined as code |



\### Notable rejections



\*\*Stored procedures\*\* are not used for business logic. They version poorly (the diff is

opaque), cannot be unit tested in isolation, and are invisible to lineage tooling —

which defeats the purpose of a governed metrics layer. dbt macros provide the same reuse

with none of these costs.



\*\*Persistent UDFs\*\* are used, but sparingly, and only for genuinely scalar reusable logic

(channel classification, currency normalisation). They are deployed from version-controlled

files, never authored in the console.



\*\*Airflow\*\* is not used initially. It is the correct answer at a certain scale, but

adopting it before there is a real DAG-shaped dependency problem converts an analytics

project into an infrastructure project. See ADR-0004.



\---



\## Repository Structure



```

marketing-analytics-platform/

│

├── README.md

├── pyproject.toml                 # Package metadata, dependencies, tool config

├── uv.lock                        # Pinned dependency graph

├── Makefile                       # Canonical entry point for every task

├── .env.example                   # Required environment variables (no secrets)

├── .pre-commit-config.yaml

├── .gitignore

│

├── docs/

│   ├── adr/                       # Architecture Decision Records

│   │   ├── 0001-elt-over-etl.md

│   │   ├── 0002-dbt-for-transformation.md

│   │   ├── 0003-dimensional-model.md

│   │   └── 0004-defer-orchestration.md

│   ├── data\_model.md              # ER diagram, grain of each fact table

│   ├── metrics.md                 # Canonical metric definitions

│   ├── runbooks/                  # What to do when something breaks

│   └── learning\_journal/          # Per-milestone write-ups

│

├── src/

│   └── marketing\_analytics/

│       ├── \_\_init\_\_.py

│       ├── cli.py                 # Command-line entry point

│       ├── config/

│       │   ├── settings.py        # Pydantic settings, env-driven

│       │   └── logging.py         # Structured logging configuration

│       ├── warehouse/

│       │   ├── client.py          # BigQuery wrapper: cost caps, retries, logging

│       │   └── schemas.py         # Explicit table schemas

│       ├── ingestion/

│       │   ├── base.py            # Abstract extractor interface

│       │   ├── ga4.py

│       │   └── spend.py

│       ├── analytics/

│       │   ├── experiments/       # A/B test design and evaluation

│       │   ├── attribution/

│       │   └── cohorts/

│       └── utils/

│           ├── exceptions.py      # Project exception hierarchy

│           └── validation.py

│

├── transform/                     # dbt project

│   ├── dbt\_project.yml

│   ├── packages.yml

│   ├── models/

│   │   ├── staging/

│   │   │   ├── \_sources.yml       # Source declarations + freshness

│   │   │   ├── \_staging.yml       # Tests and column documentation

│   │   │   └── stg\_\*.sql

│   │   ├── intermediate/

│   │   └── marts/

│   │       ├── core/              # Shared dims and facts

│   │       └── marketing/         # Domain-specific marts

│   ├── macros/                    # Reusable SQL — the anti-duplication layer

│   ├── seeds/                     # Small static reference data

│   ├── snapshots/                 # Slowly changing dimension capture

│   └── tests/                     # Singular (custom) data tests

│

├── dashboards/

│   └── app/                       # Streamlit application

│

├── tests/

│   ├── unit/

│   ├── integration/

│   └── conftest.py

│

├── infra/

│   └── terraform/

│

├── notebooks/                     # Exploration only. Never a dependency.

│

└── .github/

&#x20;   └── workflows/

&#x20;       └── ci.yml

```



\### Why `src/` layout



The package lives under `src/` rather than at the repository root. This forces the test

suite to import the \*installed\* package rather than accidentally picking up local files

from the working directory. Without it, tests can pass locally and fail on any other

machine — a class of bug that is genuinely painful to diagnose.



\### Why dbt is a subdirectory, not a sibling repository



Splitting Python and dbt into separate repositories is common at larger organisations,

where different teams own each. At this scale it would mean two pull requests for one

logical change, and no way to run the full pipeline in a single CI job. Keeping them

together is the right trade-off until team boundaries force otherwise.



\### On `notebooks/`



Notebooks are for exploration and are never imported by production code. Anything that

proves useful gets rewritten as a tested module. This boundary is enforced by convention

and by review, and it is the main thing separating this repository from a portfolio

project.



\---



\## Data Sources



\### Primary: GA4 obfuscated sample e-commerce dataset



`bigquery-public-data.ga4\_obfuscated\_sample\_ecommerce.events\_\*`



Real event-level GA4 export data from the Google Merchandise Store. This is the exact

schema that production GA4 → BigQuery exports produce, which means the modelling work

here transfers directly to a real job.



Key characteristics that drive the design:



\- \*\*Sharded by date\*\* (`events\_YYYYMMDD`), not partitioned. Queries must use wildcard

&#x20; tables with `\_TABLE\_SUFFIX` filters or they will scan the entire history.

\- \*\*Deeply nested.\*\* `event\_params` is an array of key-value structs where the value

&#x20; itself is a struct with one populated type field. Extracting a single parameter

&#x20; requires `UNNEST` and type-aware selection — this is the defining skill of GA4

&#x20; modelling.

\- \*\*Event-grained, not session-grained.\*\* Sessions must be reconstructed. This is

&#x20; intermediate-layer work.



\### Secondary: marketing spend



No suitable public dataset exists for ad platform spend, so this source is generated to

represent realistic campaign-level cost data, joined to GA4 on campaign identifiers.



This is not a compromise — it is the realistic case. Real marketing analytics almost

always involves reconciling behavioural data with a spend system that shares no clean

primary key. The imperfect join is the interesting engineering problem: surrogate keys,

conformed dimensions, and fan-out traps all appear here.



\---



\## Data Model



Dimensional (Kimball-style), not Data Vault and not one wide table.



\*\*Why dimensional.\*\* Marketing questions are overwhelmingly "measure X, sliced by Y" —

which is precisely the shape a star schema is optimised for. It is also the model that

every BI tool expects, and the one most analysts can read without a tutorial.



\*\*Why not one wide denormalised table.\*\* It is tempting on a columnar engine, and it does

perform well. But it duplicates dimension attributes across every row, makes

slowly-changing attributes nearly impossible to handle correctly, and requires a full

rebuild whenever a dimension changes.



\*\*Why not Data Vault.\*\* It solves auditability and multi-source integration problems that

this project does not have. The overhead would be substantial and the benefit theoretical.



Every fact table declares its \*\*grain\*\* explicitly in `docs/data\_model.md`. Grain

ambiguity is the single most common cause of double-counted metrics, and it is worth

being pedantic about.



\---



\## Development Standards



\### Python



\- Type hints on all public functions; `mypy` enforced in CI

\- `ruff` for linting and formatting, configured in `pyproject.toml`

\- Google-style docstrings on public interfaces

\- Project-specific exception hierarchy rooted in a single base exception —

&#x20; never bare `except:`

\- Structured logging via the standard `logging` module. No `print()` outside the CLI.

\- Configuration is environment-driven and validated with Pydantic at startup, so a

&#x20; misconfiguration fails immediately with a clear message rather than midway through

&#x20; a pipeline run



\### SQL



\- `sqlfluff` enforced, BigQuery dialect

\- CTEs over subqueries; import CTEs at the top of every model

\- Explicit column lists. `SELECT \*` is permitted only in staging passthroughs.

\- Every model declares partitioning and clustering where the data volume warrants it

\- Any expression written twice becomes a macro



\### Testing



Three distinct layers, each catching a different failure mode:



| Layer | Tool | Catches |

|---|---|---|

| Unit | pytest | Logic errors in Python functions |

| Data | dbt tests | Wrong or unexpected data reaching the marts |

| Integration | pytest + BigQuery sandbox | Broken assumptions about the warehouse itself |



Every mart model carries at minimum: a uniqueness test on its key, a not-null test on

its key, and referential integrity tests on its foreign keys.



\### Git workflow



\- Trunk-based. Short-lived branches off `main`.

\- Branch naming: `feat/`, `fix/`, `docs/`, `refactor/`, `chore/`

\- Conventional Commits

\- All merges via pull request, even solo. The PR description explains \*why\*, not what —

&#x20; the diff already shows what.

\- CI must pass before merge



\---



\## Documentation Standards



Documentation lives as close to the thing it describes as possible, because documentation

that lives elsewhere goes stale silently.



| What | Where | Why there |

|---|---|---|

| Function behaviour | Docstrings | Cannot drift far from the code |

| Column meaning | dbt `schema.yml` | Validated by dbt; rendered into docs site |

| Model purpose | dbt `schema.yml` description | Same |

| Metric definition | `docs/metrics.md` | Single canonical reference |

| Why a decision was made | `docs/adr/` | Immutable record; never edited, only superseded |

| How to fix a failure | `docs/runbooks/` | Read under pressure; needs to be findable |



Top-level directory READMEs exist only where the \*why\* is not inferable from the code.



\### Architecture Decision Records



Every significant technical decision gets a numbered ADR containing: the context, the

decision, the alternatives considered, and the consequences — including negative ones.



ADRs are never edited after acceptance. When a decision is reversed, a new ADR supersedes

the old one and both remain. The history is the point.



\---



\## Setup Instructions



\### Prerequisites



\- Python 3.11 or later

\- A Google Cloud project with billing enabled (the free tier is sufficient)

\- `gcloud` CLI

\- `uv`

\- `make`



\### 1. Clone and install



```bash

git clone <repository-url>

cd marketing-analytics-platform

uv sync

uv run pre-commit install

```



\### 2. Authenticate to Google Cloud



```bash

gcloud auth application-default login

gcloud config set project YOUR\_PROJECT\_ID

```



Application Default Credentials are used deliberately in local development. Service

account \*\*key files are never created or committed\*\* — they are the most common source

of credential leaks in public repositories. Production authentication uses Workload

Identity.



\### 3. Configure environment



```bash

cp .env.example .env

```



Then edit `.env`:



```

GCP\_PROJECT\_ID=your-project-id

BQ\_LOCATION=US

BQ\_DATASET\_RAW=raw

BQ\_DATASET\_STAGING=staging

BQ\_DATASET\_MARTS=marts

BQ\_MAX\_BYTES\_BILLED=10000000000

LOG\_LEVEL=INFO

```



`BQ\_LOCATION` must be `US` — the GA4 public dataset lives there, and BigQuery cannot

join across locations.



\### 4. Create datasets and verify



```bash

make setup

make verify

```



\---



\## Common Commands



All routine work goes through the `Makefile`, so the canonical command for any task is

discoverable in one place.



```bash

make install       # Sync dependencies and install hooks

make setup         # Create BigQuery datasets

make verify        # Confirm connectivity and permissions

make ingest        # Run the ingestion pipeline

make build         # dbt build (models + tests)

make test          # Python test suite

make lint          # ruff + sqlfluff + mypy

make docs          # Generate and serve dbt documentation

make dashboard     # Launch the Streamlit app

make ci            # Everything CI runs, locally

```



\---



\## Cost Controls



BigQuery bills on bytes scanned, and the GA4 sample dataset is large enough that a

careless query is genuinely expensive. These controls are structural, not advisory:



1\. \*\*`maximum\_bytes\_billed` is set on every query\*\* by default in the BigQuery client

&#x20;  wrapper. A query that would exceed the cap fails before executing rather than

&#x20;  completing and generating a bill.

2\. \*\*Wildcard queries must filter `\_TABLE\_SUFFIX`.\*\* Without it, every historical shard

&#x20;  is scanned. This is enforced by review and by a sqlfluff rule.

3\. \*\*Marts are partitioned and clustered\*\* on the columns that filters actually use, so

&#x20;  pruning eliminates most scanning.

4\. \*\*Dry runs in CI.\*\* Pull requests report the estimated bytes each changed model will

&#x20;  scan, making cost regressions visible at review time rather than at month end.

5\. \*\*A budget alert\*\* is configured on the GCP project as a backstop.



\---



\## Learning Roadmap



Each milestone ends with the repository in a working, documented, tested state, and a

write-up in `docs/learning\_journal/`.



\*\*Milestone 0 — Foundations\*\*

Repository scaffolding, tooling, CI, BigQuery connectivity, ADRs 0001–0004.

\*Concepts: packaging, dependency resolution, pre-commit, GCP authentication.\*



\*\*Milestone 1 — Ingestion\*\*

Extract from the public dataset into `raw`. Idempotent, restartable, logged.

\*Concepts: abstract base classes, retries and backoff, idempotency, schema evolution.\*



\*\*Milestone 2 — Staging\*\*

dbt project initialised. Staging models over every source. Nested GA4 structures

flattened.

\*Concepts: `UNNEST`, arrays and structs, dbt sources, `ref()` and DAG resolution.\*



\*\*Milestone 3 — Dimensional model\*\*

Sessionisation, conformed dimensions, fact tables with declared grain.

\*Concepts: surrogate keys, slowly changing dimensions, fan-out traps, incremental models.\*



\*\*Milestone 4 — Metrics layer\*\*

Canonical KPI definitions. Macros for every reused expression.

\*Concepts: metric governance, DRY in SQL, semantic layers.\*



\*\*Milestone 5 — Attribution\*\*

Last-click, first-click, linear, and time-decay attribution, compared.

\*Concepts: window functions, attribution windows, why models disagree.\*



\*\*Milestone 6 — Experimentation\*\*

A/B test design and evaluation: power analysis, sequential testing pitfalls,

multiple comparisons.

\*Concepts: hypothesis testing, CUPED, peeking, effect size vs significance.\*



\*\*Milestone 7 — Cohorts and lifetime value\*\*

Retention curves, cohort analysis, predictive LTV.

\*Concepts: survival analysis, cohort grain, probabilistic models.\*



\*\*Milestone 8 — Dashboard\*\*

Streamlit application over the marts, with query caching.

\*Concepts: caching strategy, cost-aware UI, dashboard-driven query patterns.\*



\*\*Milestone 9 — Optimisation\*\*

Partitioning, clustering, materialisation strategy, query plan analysis.

\*Concepts: reading execution details, slot contention, shuffle.\*



\*\*Milestone 10 — Operations\*\*

Scheduling, freshness monitoring, alerting, runbooks.

\*Concepts: SLAs, observability, on-call thinking.\*



\---



\## Contribution Guidelines



Written for a future contributor — including future-me, who will not remember any of this.



1\. Open an issue describing the problem before writing code.

2\. Branch from `main` using the naming convention above.

3\. For any architecturally significant change, write the ADR \*\*first\*\*. If the decision

&#x20;  cannot be explained in writing, it is not ready to implement.

4\. Add tests. New models require data tests; new Python requires unit tests.

5\. Update documentation in the same commit as the change.

6\. Run `make ci` locally before pushing.

7\. Open a pull request explaining the reasoning and the alternatives considered.



\### Definition of done



A change is complete when: CI passes, tests cover the new behaviour, documentation

reflects reality, no business logic was duplicated, and the reasoning is recorded

somewhere a future reader will find it.



\---



\## Future Roadmap



Deliberately deferred, with the trigger that would justify each:



| Item | Adopt when |

|---|---|

| Dagster orchestration | Dependencies become genuinely DAG-shaped and scheduling is non-trivial |

| dbt Semantic Layer | More than one consumer needs the same metrics and they begin to drift |

| Reverse ETL | Marts need to reach operational tools, not just dashboards |

| Streaming ingestion | A business question requires latency below one hour |

| ML models | Descriptive analytics is exhausted and prediction adds real value |

| Data contracts | An upstream schema change breaks production more than once |



The ordering principle throughout: adopt tooling in response to a problem that actually

exists, not in anticipation of one. Premature infrastructure is the most common way

analytics projects stall.

