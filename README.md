# 🛒 E-Commerce Medallion Pipeline (Databricks)

An enterprise-grade, end-to-end data engineering pipeline built on Databricks using the Medallion Architecture (Bronze → Silver → Gold). This project processes daily e-commerce transactional data for a PC accessories company, standardizing messy child-company records and aggregating them into a conformed, monthly-grain parent company reporting model. 

Building on previous experience developing containerized Airflow and dbt pipelines for hardware sales data, this repository adapts robust orchestration and data quality principles directly into the Databricks Lakehouse ecosystem.

## 🏗️ Architecture & Pipeline Flow

<div align="center">
  <img src="job_pipeline_dag.png" alt="Databricks Workflow DAG" width="800"/>
  <p><em>Automated Databricks Workflow orchestrating the Medallion pipeline</em></p>
</div>
The pipeline handles raw CSV ingestion from AWS S3, cleanses data anomalies (e.g., messy string dates, null quantities), and performs complex incremental recalculations.

*   **🥉 Bronze Layer (Landing & Staging):** Ingests raw wildcard CSVs from the AWS S3 landing zone. Appends full history to a master table while isolating daily incremental files into a `staging_orders` table to prevent full-table rescans. 
*   **🥈 Silver Layer (Cleansing & Conforming):** Reads strictly from Bronze staging. Enforces strict schema typing, drops invalid null quantities, standardizes alphanumeric customer IDs, and safely parses multiple date formats (`try_to_date`). Cleaned records are upserted via Delta `MERGE`.
*   **🥇 Gold Layer (Business Aggregation):** 
    *   **Child Company Fact:** Daily incremental data is merged into the child fact table (`sb_fact_orders`).
    *   **Parent Company Fact (Recalculation):** The pipeline dynamically identifies which months received new data, truncates dates to the month start (`F.trunc`), and recalculates total `sold_quantity` from scratch to prevent double-counting before merging into the central `fact_orders` table.
*   **💎 BI & Consumption:** A denormalized SQL view (`vw_fact_orders_enriched`) joins fact tables with dimension tables (Customers, Products, Gross Price). This view powers an interactive Databricks Dashboard and is fully integrated with Databricks Genie for natural language querying.

## 🛠️ Tech Stack
*   **Compute & Storage:** Databricks, Apache Spark (PySpark), Delta Lake, AWS S3
*   **Transformations & Orchestration:** SQL, Databricks Workflows (Job Tasks)
*   **Visualization:** Databricks Dashboards, Databricks Genie

## 📁 Repository Structure
*   `2_dimension_data_processing/`
    *   `1_customer_data_processing.ipynb` - Cleanses and conforms customer attributes.
    *   `2_products_data_processing.ipynb` - Structures the product hierarchy (division, category, variant).
    *   `3_pricing_data_processing.ipynb` - Processes historical gross pricing data.
*   `3_fact_data_processing/`
    *   `4_full_load_fact.ipynb` - Generates the initial historical fact table and monthly roll-ups.
    *   `5_incremental_fact_orders.ipynb` - Handles isolated staging and daily incremental upserts.
*   `setup_1/` - Contains schema definitions and initial path configurations.
*   `PC_Hardware_insights 2026-09-1...` - Visual export of the Databricks dashboard.
*   `PC_Hardware_insights.lvdash.json` - Exported dashboard configuration for version control.
*   `README.md` - Project documentation.
*   `job_pipeline_dag.png` - Automated Databricks Workflow DAG execution graph.

## 🚀 How to Run the Pipeline
1.  Upload the extracted historical CSV files to your designated `s3://.../orders/landing/` path.
2.  Execute the notebooks in the `2_dimension_data_processing/` folder to build the foundational dimension tables.
3.  Run `4_full_load_fact.ipynb` to establish the baseline Gold fact tables.
4.  Once new extracted daily data is deposited into the S3 landing zone, trigger the automated **Databricks Workflow Job**. This orchestrates the dependencies, runs `5_incremental_fact_orders.ipynb` to process the new records, updates the Gold facts, and automatically moves the raw CSVs to the `processed/` directory.

---
**Author:** Syed Junaid 
*Based in Bengaluru, India | Specializing in Data Engineering, AI/ML Integrations, and Analytics.*
