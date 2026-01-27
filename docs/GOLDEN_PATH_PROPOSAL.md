# Golden Path Proposal: Standardizing Legacy-to-GCP Data Migrations

## 1. Executive Summary
This proposal outlines the transition of our existing data migration framework into an enterprise **Golden Path** for the Enterprise Data Platform (EDP). By leveraging a **Library-First, 3-Unit Architecture**, we provide a standardized, governed, and low-friction path for migrating legacy mainframe data to GCP BigQuery.

## 2. What We Have Built (Current State)
We have moved away from monolithic "one-off" pipelines to a production-grade framework consisting of:

### 2.1. The Shared Library Foundation
Abstracted enterprise logic into versioned, reusable libraries:
*   `gcp-pipeline-core`: Centralized Audit, Job Control, and FinOps (Cost Tracking).
*   `gcp-pipeline-beam`: Standardized Ingestion logic (HDR/TRL validation, 25MB+ split-file handling).
*   `gcp-pipeline-orchestration`: Airflow DAG Factories for rapid, consistent scheduling.
*   `gcp-pipeline-transform`: dbt Macros for automated PII masking and audit injection.
*   `gcp-pipeline-tester`: Mocks, fixtures, and base test classes for consistent pipeline testing.

### 2.2. The 3-Unit Deployment Model
A decoupled architecture that minimizes blast radius and optimizes cloud costs:
1.  **Ingestion Unit:** Dataflow (Flex Templates) for high-scale processing.
2.  **Transformation Unit:** dbt for business-ready Data Products (ODP/FDP).
3.  **Orchestration Unit:** Cloud Composer (Airflow) for event-driven coordination.

### 2.3. Reference Implementations
*   **EM (Excess Management):** Handles complex multi-entity joins and multi-target transformation.
*   **LOA (Loan Origination):** A high-speed, 1:1 mapping pattern for simpler entities.
*   **Spanner Transformation:** A reference implementation for modern cloud sources using dbt Federated Queries.

## 3. Why This Should Be the Golden Path
*   **Multi-Source Flexibility:** Proven patterns for both legacy **Teradata-to-GCP** (via GCS/Beam) and **Spanner-to-BigQuery**. We support two distinct Spanner paths: **Low-Friction Federated Queries** for medium data and **High-Volume Beam Ingestion** for enterprise-scale migrations, ensuring stability regardless of data size.
*   **Production Stability:** Over **1,000+ unit tests** ensuring the reliability of core migration logic.
*   **Observability & Monitoring:** Built-in integration with Cloud Monitoring and Dynatrace via `gcp-pipeline-core`, providing real-time visibility into pipeline health and performance.
*   **Compliance by Default:** Every row in BigQuery is automatically tagged with `_run_id` and `_source_file` for 100% lineage.
*   **Future-Proof Modularization:** A clear roadmap to move from system-specific code to a **"Generic Engine" model**, enabling "Zero-Code" onboarding via YAML manifests (see [Modularization Roadmap](./MODULARIZATION_AND_CONFIG_ROADMAP.md)).
*   **Rapid Onboarding:** We have established **standardized templates** and a **'Creating New Deployment' guide** that allows a new team to deploy a governed pipeline in days.

---

## 4. Proposed Message to EDP Owner
**Subject:** Proposal: Standardizing Legacy-to-GCP Migrations via [Framework Name] Golden Path

"Hi [EDP Owner Name],

Following up on your suggestion to take our migration framework to a **Golden Path**—I’ve put together a summary of our current production-ready assets and our vision for a **"Generic Engine" platform**.

We have successfully abstracted the 'heavy lifting' of legacy migrations—including HDR/TRL validation, split-file handling, **real-time monitoring**, and FinOps tracking—into a **shared-library foundation**. This framework has already been proven through our **EM, LOA, and new Spanner implementations**.

The key differentiator for this Golden Path is our **Modularization Roadmap**. We are moving toward a **Manifest-Driven model** where onboarding a new system requires zero new Python code—only a YAML configuration. This will significantly lower the barrier for other teams while ensuring 100% consistent audit and observability standards across the bank.

I’d love to show you a 15-minute walkthrough of our **'Job Control' metadata layer**, the **Universal Engine vision**, and our **Spanner-to-FDP reference implementation**. Do you have any time next week for a brief demo?

Best regards,

[Your Name]"
