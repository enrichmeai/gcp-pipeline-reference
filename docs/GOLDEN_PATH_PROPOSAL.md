# Golden Path Proposal: Standardising Legacy-to-GCP Data Migrations

## 1. Executive Summary

This proposal outlines the transition of our existing data migration framework into an enterprise **Golden Path** for the Credit Platform. By leveraging a **Library-First, 3-Unit Architecture**, we provide a standardised, governed, and low-friction path for migrating legacy mainframe data to GCP BigQuery.

The **Generic reference system** is the centrepiece of this proposal. It is a deliberate combination of what were previously two separate applications — Excess Management and Loan Origination — into a single unified deployment. This design choice is intentional: it demonstrates two distinct pipeline patterns simultaneously, proving that a single shared framework can handle both a complex multi-entity join dependency and a simple single-entity mapping within the same codebase and infrastructure.

---

## 2. What We Have Built (Current State)

### 2.1 The Shared Library Foundation

Enterprise logic has been abstracted into five versioned, reusable libraries published to PyPI under the umbrella package `gcp-pipeline-framework==1.0.11`:

- `gcp-pipeline-core`: Centralised Audit, Job Control, and FinOps (Cost Tracking). Zero dependencies on Beam or Airflow.
- `gcp-pipeline-beam`: Standardised Ingestion logic — HDR/TRL envelope validation, 25MB+ split-file handling, Dead Letter Queue side outputs.
- `gcp-pipeline-orchestration`: Airflow DAG Factories, `PubSubPullSensor`, `EntityDependencyChecker` for rapid and consistent scheduling.
- `gcp-pipeline-transform`: dbt Macros for automated PII masking and audit column injection.
- `gcp-pipeline-tester`: Mocks, fixtures, and base test classes for consistent pipeline testing.

### 2.2 The 3-Unit Deployment Model

A decoupled architecture that minimises blast radius and optimises cloud costs:

1. **Ingestion Unit** (`original-data-to-bigqueryload`): Dataflow Flex Templates for high-scale processing.
2. **Transformation Unit** (`bigquery-to-mapped-product`): dbt for business-ready Data Products (ODP → FDP).
3. **Orchestration Unit** (`data-pipeline-orchestrator`): Cloud Composer (managed Airflow) for event-driven coordination.

### 2.3 The Generic Reference System: Two Patterns in One

The Generic system consolidates two previously separate applications into a single deployment, demonstrating both patterns the Credit Platform needs to support:

#### JOIN Pattern (from Excess Management)

Multiple entities with staggered extract schedules must all complete before transformation can begin:

- 3 source entities: Customers (4 PM extract), Accounts (4 PM extract), Decision (5 AM extract)
- 3 ODP tables: `odp_generic.customers`, `odp_generic.accounts`, `odp_generic.decision`
- 2 FDP tables: `fdp_generic.event_transaction_excess`, `fdp_generic.portfolio_account_excess`
- Dependency: all 3 entities must reach `SUCCESS` in `job_control` before FDP transformation triggers

This pattern is common across the Credit Platform for any system where a business-ready data product requires joining data from multiple source entities that arrive independently.

#### MAP Pattern (from Loan Origination)

A single entity with a straightforward 1:1 mapping:

- 1 source entity: Applications (daily extract)
- 1 ODP table: `odp_generic.applications`
- 1 FDP table: `fdp_generic.portfolio_account_facility`
- Dependency: transformation triggers immediately after ODP load — no wait required

This pattern covers the majority of simpler migrations where a single mainframe entity maps to a single data product.

### 2.4 Additional Reference Patterns

- **Spanner Transformation (Federated)**: A reference implementation for modern cloud sources using dbt Federated Queries (`EXTERNAL_QUERY`), demonstrating that the 3-Unit model extends beyond mainframe origins.

---

## 3. Why This Should Be the Enterprise Golden Path

### 3.1 Multi-Pattern Coverage

The Generic system proves that a single shared framework handles both the JOIN and MAP patterns — the two most common pipeline shapes on the Credit Platform. Teams do not need to build separate frameworks for simple vs. complex migrations.

### 3.2 Multi-Source Flexibility

The framework supports:
- **Mainframe/Teradata to GCP** via GCS landing + Beam ingestion (proven in production).
- **Cloud Spanner to BigQuery** via two paths: low-friction federated queries for medium data volumes, and high-volume Beam ingestion for enterprise scale.

### 3.3 Production Stability

- Over **780 unit tests** across 5 libraries ensuring the reliability of core migration logic.
- Idempotent pipelines: every run can be safely re-executed with the same `run_id`.
- Dead Letter Queue pattern ensures invalid records never block a pipeline run.

### 3.4 Observability & Compliance by Default

- Built-in integration with Cloud Monitoring and Dynatrace via `gcp-pipeline-core`.
- Every BigQuery row is automatically tagged with `_run_id` and `_source_file` for 100% lineage from mainframe extract to FDP.
- PII masking macros in `gcp-pipeline-transform` applied at the FDP layer.

### 3.5 Rapid Onboarding — Config-Driven Pipelines

The entire pipeline is now driven by a single `config/system.yaml` file. A new team onboards by:

1. **Copy** the project structure
2. **Edit** `config/system.yaml` — define system_id, entities (with full schemas), FDP models (with dependency type: `join` or `map`), and infrastructure naming
3. **Write** dbt SQL models for their FDP transformations (the only business logic that can't be generated)
4. **Deploy** — Terraform + workflow handle the rest

**No Python code editing. No DAG editing. No constants/settings file editing.**

The config defines:

| Section | What It Controls |
|---------|-----------------|
| `system_id` / `system_name` | System identification, file prefixes, resource naming |
| `entities` | ODP table schemas, CSV headers, field validation rules, PII flags, foreign keys |
| `fdp_models` | FDP transformation targets, dependency type (`join`/`map`), required ODP entities |
| `infrastructure` | Dataset names, bucket naming conventions, Pub/Sub topics |

Under the hood:
- **Ingestion**: `schema/registry.py` loads YAML → builds `EntitySchema` objects (same API, config-driven backend)
- **Orchestration**: `generate_dags.py` reads YAML → produces 5 static DAG files at build time (trigger, ingestion, transformation, status, error handling) with full observability stack
- **Settings**: `config/settings.py` loads infrastructure section from YAML

Full specification: [CONFIG_DRIVEN_PIPELINE_SPEC.md](./CONFIG_DRIVEN_PIPELINE_SPEC.md)

### 3.6 Granular FDP Dependency Management

The transformation layer uses per-model dependency checking, not all-or-nothing:

| FDP Model | Type | ODP Dependencies | Trigger Behaviour |
|-----------|------|------------------|-------------------|
| `event_transaction_excess` | JOIN | customers + accounts | Triggers only when **both** are loaded |
| `portfolio_account_excess` | MAP | decision | Triggers **immediately** when decision loads |
| `portfolio_account_facility` | MAP | applications | Triggers **immediately** when applications loads |

This is defined in `system.yaml` under `fdp_models` and consumed by the DAG factory — no DAG code changes needed when adding new FDP models.

### 3.7 Config-Driven dbt Model Generation

The transformation layer (FDP and CDP) is now config-driven via a generator script:

```bash
# Generate FDP models (ODP → FDP)
python generate_dbt_models.py --layer fdp --config config/system.yaml

# Generate CDP scaffolding (FDP → CDP)
python generate_dbt_models.py --layer cdp --config config/system.yaml
```

**FDP layer** — MAP and JOIN models are 100% auto-generated from `system.yaml`:
- ODP staging views with code maps and renames
- FDP SQL models with surrogate keys, column mappings, PII masking, incremental filters
- Source definitions with column-level tests
- Model metadata YAML

**CDP layer** — scaffolding is auto-generated, but CDP SQL is hand-written:
- FDP staging views (thin passthroughs from FDP tables) — auto-generated
- FDP source definitions with tests — auto-generated
- CDP model metadata — auto-generated
- **CDP SQL with complex business logic** — hand-written by the team

This reflects reality: FDP is standardised data transformation (YAML-configurable), while CDP involves complex business logic (aggregations, scoring, segmentation) that requires hand-written SQL.

Full specification: [DBT_CONFIG_DRIVEN_SPEC.md](./DBT_CONFIG_DRIVEN_SPEC.md)

### 3.8 No Library Changes Required

All config-driven logic lives at the deployment level. The published `gcp-pipeline-framework` libraries remain unchanged:

| What | Where | Team Edits? |
|------|-------|------------|
| `system.yaml` | Per-deployment config | Yes — define entities, models, infrastructure |
| `generate_dbt_models.py` | Deployment-level script | No — copy as-is |
| `gcp-pipeline-framework` | PyPI (shared libraries) | **No** — provides the engine |

### 3.9 Future Modularisation

- **Auto-generate Terraform** from `system.yaml` infrastructure section
- **Auto-generate test data** from entity schema definitions

---

## 4. Proposed Message to EDP Owner

**Subject:** Proposal: Standardising Legacy-to-GCP Migrations via GCP Pipeline Reference — Golden Path

"Hi [EDP Owner Name],

Following up on your suggestion to take our migration framework to a Golden Path — I have put together a summary of our current production-ready assets and our vision for a 'Generic Engine' platform.

We have successfully abstracted the heavy lifting of legacy migrations — including HDR/TRL validation, split-file handling, real-time monitoring, and FinOps tracking — into a shared-library foundation (published to PyPI as `gcp-pipeline-framework`). This framework has been proven across two distinct pipeline patterns, which we have consolidated into our **Generic reference system**: the JOIN pattern from Excess Management and the MAP pattern from Loan Origination.

The key differentiator for this Golden Path is our **Modularisation Roadmap**. We are moving towards a Manifest-Driven model where onboarding a new system requires zero new Python code — only a YAML configuration. This will significantly lower the barrier for other Credit Platform teams while ensuring 100% consistent audit and observability standards across the bank.

I would love to show you a 15-minute walkthrough of our **Job Control metadata layer**, the **two-pattern Generic reference system**, and our **Spanner-to-FDP federated implementation**. Do you have any time next week for a brief demo?

Best regards,

[Your Name]"
