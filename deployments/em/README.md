# EM Deployment

**Excess Management (EM)** data migration pipeline.

**Status:** ⚠️ Partially Complete | 152 tests passing

---

## Overview

| Attribute | Value |
|-----------|-------|
| **System ID** | EM |
| **Source Entities** | 3 (Customers, Accounts, Decision) |
| **ODP Tables** | 3 (`odp_em.customers`, `odp_em.accounts`, `odp_em.decision`) |
| **FDP Tables** | 1 (`fdp_em.em_attributes`) |
| **Transformation** | JOIN 3 sources → 1 target |
| **Dependency** | Wait for all 3 entities before FDP |

---

## File Format

```
HDR|EM|{Entity}|{YYYYMMDD}
{csv_header_row}
{data_rows...}
TRL|RecordCount={n}|Checksum={hash}
```

**Example (Customers):**
```
HDR|EM|Customers|20260101
customer_id,name,email,status,created_date
CUST001,John Doe,john@example.com,ACTIVE,2025-01-15
CUST002,Jane Smith,jane@example.com,ACTIVE,2025-01-14
TRL|RecordCount=2|Checksum=abc123
```

---

## Data Flow

```
                         ┌─────────────────────┐
                         │ EntityDependency    │
                         │ Checker (Wait)      │
                         └──────────┬──────────┘
                                    │
    ┌───────────────────────────────┼───────────────────────────────┐
    │                               │                               │
    ▼                               ▼                               ▼
┌─────────┐                   ┌─────────┐                   ┌─────────┐
│Customers│                   │Accounts │                   │Decision │
│  (ODP)  │                   │  (ODP)  │                   │  (ODP)  │
└────┬────┘                   └────┬────┘                   └────┬────┘
     │                              │                              │
     └──────────────────────────────┼──────────────────────────────┘
                                    │
                                    ▼
                            ┌───────────────┐
                            │  dbt JOIN     │
                            │ Transformation│
                            └───────┬───────┘
                                    │
                                    ▼
                            ┌───────────────┐
                            │ em_attributes │
                            │    (FDP)      │
                            └───────────────┘
```

---

## Directory Structure

```
deployments/em/
├── config/
│   ├── __init__.py
│   ├── settings.py          # SYSTEM_ID="EM", datasets
│   └── constants.py         # Headers, allowed values
│
├── schema/
│   ├── __init__.py
│   ├── customers.py         # CustomerSchema
│   ├── accounts.py          # AccountSchema
│   ├── decision.py          # DecisionSchema
│   └── registry.py          # EM_SCHEMAS
│
├── domain/
│   ├── __init__.py
│   └── schema.py            # BigQuery schemas
│
├── validation/
│   ├── __init__.py
│   ├── types.py             # ValidationResult
│   ├── file_validator.py    # HDR/TRL validation
│   ├── record_validator.py  # Field validation
│   └── validator.py         # EMValidator
│
├── pipeline/
│   ├── __init__.py
│   ├── em_pipeline.py       # Main Beam pipeline
│   ├── dag_template.py      # create_em_dag()
│   └── transforms.py        # Beam DoFns
│
├── orchestration/
│   └── airflow/
│       ├── dags/            # Airflow DAGs
│       ├── sensors/         # PubSub sensors
│       └── callbacks/       # Error handlers
│
├── transformations/
│   └── dbt/
│       └── models/
│           ├── staging/em/  # stg_em_customers, etc.
│           └── fdp/         # em_attributes (JOIN)
│
├── schemas/                 # BigQuery JSON schemas
│   ├── odp_em_customers.json
│   ├── odp_em_accounts.json
│   ├── odp_em_decision.json
│   └── fdp_em_attributes.json
│
└── tests/
    ├── unit/
    └── integration/
```

---

## Quick Start

```bash
# Run unit tests (excluding orchestration which needs Airflow)
PYTHONPATH=. pytest deployments/em/tests/unit/ -v --ignore=deployments/em/tests/unit/orchestration/

# Validate imports
python -c "
from deployments.em.config import SYSTEM_ID, REQUIRED_ENTITIES
from deployments.em.validation import EMValidator
print(f'SYSTEM_ID: {SYSTEM_ID}')
print(f'ENTITIES: {REQUIRED_ENTITIES}')
"
```

---

## Key Components

### Configuration

```python
from deployments.em.config import (
    SYSTEM_ID,           # "EM"
    REQUIRED_ENTITIES,   # ["customers", "accounts", "decision"]
    ODP_DATASET,         # "odp_em"
    FDP_DATASET,         # "fdp_em"
)
```

### Validation

```python
from deployments.em.validation import EMValidator

validator = EMValidator()

# Validate file structure
result = validator.validate_file(file_lines, "customers")

# Validate records
valid, errors = validator.validate_records(records, "customers")
```

### Entity Dependency

EM uses `EntityDependencyChecker` to wait for all 3 entities:

```python
from gdw_data_core.orchestration.dependency import EntityDependencyChecker

checker = EntityDependencyChecker(
    project_id="my-project",
    system_id="EM",
    required_entities=["customers", "accounts", "decision"],
)

if checker.all_entities_loaded(extract_date):
    # Trigger FDP transformation
    run_dbt_transformation()
```

---

## dbt Transformation

EM uses a **JOIN** transformation to combine 3 ODP tables into 1 FDP table:

```sql
-- models/fdp/em_attributes.sql
SELECT
    c.customer_id,
    c.name,
    c.status,
    a.account_id,
    a.account_type,
    a.balance,
    d.decision_id,
    d.decision_code,
    d.score
FROM {{ ref('stg_em_customers') }} c
LEFT JOIN {{ ref('stg_em_accounts') }} a
    ON c.customer_id = a.customer_id
LEFT JOIN {{ ref('stg_em_decision') }} d
    ON c.customer_id = d.customer_id
```

---

## Key Difference from LOA

| Aspect | EM | LOA |
|--------|-----|-----|
| Entities | 3 | 1 |
| Dependency | Wait for all | Immediate |
| FDP Transformation | JOIN (3→1) | SPLIT (1→2) |
| EntityDependencyChecker | Required | Not needed |

---

## Known Issues

1. **Orchestration tests require Airflow** - Skip with `--ignore=tests/unit/orchestration/`
2. **Some validation tests have API mismatches** - Need method updates
3. **Infrastructure tests reference old paths** - Need path updates

See [Project Status Analysis](../../docs/PROJECT_STATUS.md) for full details.

