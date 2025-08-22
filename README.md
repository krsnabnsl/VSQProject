# VSQProject

# **DDL Notebook Summary**

The DDL notebook defines the Bronze layer schema by creating raw Delta tables (raw_products, raw_sales, raw_customers, stores) with explicit storage paths in S3. This ensures consistent structure, governed storage, and idempotent table creation (IF NOT EXISTS) to support reliable ingestion pipelines.

# **Bronze Layer Summary
# **
The Bronze Layer acts as the raw ingestion zone of the data lakehouse, capturing data from multiple heterogeneous sources in its original, unprocessed form.

In this project, the following ingestion pipelines were implemented:

**1.Environment Setup & Governance**
- Defined working catalog and schema (vsqproject.bronze) to ensure data is organized by layer.

- Credentials (Postgres, API keys, JDBC URL) securely retrieved via Databricks Secrets, avoiding hardcoding sensitive information.

**2. Batch Ingestion (Static Data Sources) **
- Stores Data: Ingested from CSV files in S3.

- Products Data: Pulled directly from PostgreSQL via JDBC connection.

- Customers Data: Retrieved from a Mock API (Mockaroo), stored in S3 as JSON, then loaded into Delta.

**3. Streaming Ingestion (Dynamic Data Source)**
**Sales Data**: Ingested continuously using Auto Loader (cloudFiles) from S3, enabling schema evolution, checkpointing, and incremental processing into Delta tables.

**Data Persistence**

- All ingested datasets were stored as **Delta Tables** in the Bronze Layer, ensuring:

- Raw data fidelity (no transformation applied).

- Schema flexibility for evolving upstream sources.

- Foundation for downstream Silver (cleansing & standardization) and Gold (business-ready) layers.


# **Silver Layer Summary
# **

The Silver Layer refines the raw Bronze data into clean, business-ready datasets by applying data quality checks, standardization, and integrity enforcement. Each domain has its own transformation logic, ensuring high-quality inputs for downstream Gold analytics.

**Customers (silver_customers)**

- Removes nulls and duplicates based on customer_id.

- Standardizes text formatting (names, city, state, gender).

- Converts dates to proper DATE format.

- Ensures a reliable, clean customer dimension for analysis.


**Products (silver_products)**

Cleans and validates product_id and prices.

Handles nulls with business defaults (average price, “UnknownBrand”, etc.).

Removes invalid or duplicate products.

Adds audit column (last_updated) and partitions by category for efficiency.


**Stores (silver_stores) **

Deduplicates on store_id.

Standardizes store attributes (trimmed names, initcapped city, uppercased state).

Produces a consistent and trusted store dimension.


**Sales (silver_sales)**

Cleans and validates transactional records.

Robustly parses timestamps and derives sale_date.

Enforces referential integrity by joining with Silver dimensions (customers, stores, products).

Computes derived metrics like sale_amount.

Writes partitioned Delta tables for optimized query performance.


**Gold Layer - Star Schema Modeling**

The Gold Layer organizes curated Silver data into fact and dimension tables, following a star schema design for analytics and reporting. This layer is optimized for BI tools, dashboards, and downstream machine learning.


**Fact Table**

fact_sales

Central transactional fact table capturing all sales activity.

Keys link to dimensions: DateKey, ProductKey, CustomerKey, StoreKey.

Measures include:

**Quantity** - number of items sold

**NetSales** - total sales amount

**DiscountAmount** - placeholder for future discount logic

**Profit** - derived profit metric (sales vs. cost assumption)

Partitioned by date for query optimization.


**Dimension Tables**

dim_product (SCD Type 2)

- Tracks product attributes (ProductName, Category, Brand, UnitPrice).

- Implements slowly changing dimension logic:

    Marks historical records as inactive (IsCurrent=0, EndDate).

    Inserts new versions of changed products with updated effective dates.

  Ensures accurate historical reporting when product details change.

**dim_customer (SCD Type 2)**

- Stores enriched customer attributes (FullName, Gender, AgeGroup, City, State).

- Dynamically calculates AgeGroup buckets.

- Maintains history of changes (e.g., relocation, name updates) using SCD Type 2 strategy.

**dim_store (Type 1 overwrite)**

Standardized store information (StoreName, City, State).

Overwrites records on refresh (no history tracking needed).


**Key Highlights**

SCD Type 2 implemented for dim_product and dim_customer to preserve history.

Fact table integrates all Silver dimensions for referential integrity.

Derived metrics and attributes (Profit, AgeGroup) added for richer insights.

Optimized star schema ensures scalable analytics and dashboard performance.
