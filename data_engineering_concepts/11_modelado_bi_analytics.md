# 11. Modelado para BI y Analytics

## 📋 Tabla de Contenidos
- [Fundamentos de Data Modeling](#fundamentos-de-data-modeling)
- [Star Schema](#star-schema)
- [Snowflake Schema](#snowflake-schema)
- [Data Vault 2.0](#data-vault-20)
- [Dimensional Modeling](#dimensional-modeling)
- [OLAP vs OLTP](#olap-vs-oltp)
- [Slowly Changing Dimensions (SCD)](#slowly-changing-dimensions-scd)
- [Aggregate Tables](#aggregate-tables)

---

## Fundamentos de Data Modeling

### ¿Por qué modelar datos?

```
┌────────────────────────────────────────────────┐
│         BENEFICIOS DE DATA MODELING            │
├────────────────────────────────────────────────┤
│                                                 │
│  ✅ PERFORMANCE                                │
│     • Queries más rápidos                      │
│     • Menos JOINs                              │
│                                                 │
│  ✅ SIMPLICIDAD                                │
│     • Fácil de entender para analistas         │
│     • Self-service BI                          │
│                                                 │
│  ✅ CONSISTENCY                                │
│     • Single source of truth                   │
│     • Métricas consistentes                    │
│                                                 │
│  ✅ MAINTAINABILITY                            │
│     • Cambios centralizados                    │
│     • Menos duplicación                        │
└────────────────────────────────────────────────┘
```

### Tipos de Modelos

| Modelo | Uso | Complejidad | Performance |
|--------|-----|-------------|-------------|
| **Star Schema** | BI, dashboards | Baja | Excelente |
| **Snowflake Schema** | Data warehouse normalizado | Media | Buena |
| **Data Vault** | Enterprise DWH, auditoría | Alta | Media |
| **One Big Table (OBT)** | Analytics simples | Muy baja | Variable |
| **3NF (Third Normal Form)** | OLTP, transaccional | Media | Buena para writes |

---

## Star Schema

### Estructura

```
┌──────────────────────────────────────────────────┐
│              STAR SCHEMA                         │
├──────────────────────────────────────────────────┤
│                                                   │
│         ┌─────────────┐                          │
│         │   DIM_DATE  │                          │
│         │  date_key   │                          │
│         │  date       │                          │
│         │  year       │                          │
│         │  quarter    │                          │
│         │  month      │                          │
│         └─────────────┘                          │
│                │                                  │
│                ▼                                  │
│  ┌────────────┐        ┌────────────────┐       │
│  │DIM_CUSTOMER│◀──────▶│  FACT_SALES    │       │
│  │customer_key│        │  sale_id       │       │
│  │name        │        │  date_key   ◀──┼──┐    │
│  │email       │        │  customer_key  │  │    │
│  │segment     │        │  product_key   │  │    │
│  └────────────┘        │  store_key     │  │    │
│                        │  quantity      │  │    │
│                        │  amount        │  │    │
│                        │  discount      │  │    │
│                        └────────────────┘  │    │
│                                │            │    │
│                                ▼            │    │
│         ┌─────────────┐  ┌────────────┐   │    │
│         │ DIM_PRODUCT │  │ DIM_STORE  │   │    │
│         │ product_key │  │ store_key  │   │    │
│         │ name        │  │ name       │   │    │
│         │ category    │  │ city       │   │    │
│         │ brand       │  │ region     │   │    │
│         └─────────────┘  └────────────┘   │    │
└────────────────────────────────────────────┼────┘
                                             │
                                             └────
                                    (todas las dimensiones
                                     conectan directamente
                                     a fact table)
```

### Implementación en SQL

```sql
-- ===== DIMENSION TABLES =====

-- Dimensión: Date
CREATE TABLE dim_date (
    date_key        INTEGER PRIMARY KEY,  -- YYYYMMDD (ej: 20250115)
    date            DATE NOT NULL,
    year            INTEGER NOT NULL,
    quarter         INTEGER NOT NULL,
    month           INTEGER NOT NULL,
    month_name      VARCHAR(20),
    day_of_month    INTEGER,
    day_of_week     INTEGER,
    day_name        VARCHAR(20),
    week_of_year    INTEGER,
    is_weekend      BOOLEAN,
    is_holiday      BOOLEAN,
    holiday_name    VARCHAR(100)
);

-- Poblar dim_date con generador de series
INSERT INTO dim_date
SELECT
    TO_CHAR(d, 'YYYYMMDD')::INTEGER as date_key,
    d as date,
    EXTRACT(YEAR FROM d) as year,
    EXTRACT(QUARTER FROM d) as quarter,
    EXTRACT(MONTH FROM d) as month,
    TO_CHAR(d, 'Month') as month_name,
    EXTRACT(DAY FROM d) as day_of_month,
    EXTRACT(DOW FROM d) as day_of_week,
    TO_CHAR(d, 'Day') as day_name,
    EXTRACT(WEEK FROM d) as week_of_year,
    EXTRACT(DOW FROM d) IN (0, 6) as is_weekend,
    FALSE as is_holiday,
    NULL as holiday_name
FROM generate_series(
    '2020-01-01'::DATE,
    '2030-12-31'::DATE,
    '1 day'::INTERVAL
) d;

-- Dimensión: Customer (SCD Type 2 - con historial)
CREATE TABLE dim_customer (
    customer_key        SERIAL PRIMARY KEY,      -- Surrogate key
    customer_id         VARCHAR(50) NOT NULL,    -- Natural key (business key)
    customer_name       VARCHAR(255),
    email               VARCHAR(255),
    phone               VARCHAR(50),
    address             TEXT,
    city                VARCHAR(100),
    state               VARCHAR(50),
    country             VARCHAR(50),
    customer_segment    VARCHAR(50),             -- 'VIP', 'Regular', 'New'

    -- SCD Type 2 fields
    valid_from          DATE NOT NULL,
    valid_to            DATE,                    -- NULL = current version
    is_current          BOOLEAN DEFAULT TRUE,

    -- Metadata
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_customer_natural_key ON dim_customer(customer_id, is_current);

-- Dimensión: Product
CREATE TABLE dim_product (
    product_key         SERIAL PRIMARY KEY,
    product_id          VARCHAR(50) NOT NULL UNIQUE,
    product_name        VARCHAR(255),
    brand               VARCHAR(100),
    category            VARCHAR(100),
    subcategory         VARCHAR(100),
    unit_cost           DECIMAL(10,2),
    unit_price          DECIMAL(10,2),

    -- Product attributes para filtering
    color               VARCHAR(50),
    size                VARCHAR(20),
    weight_kg           DECIMAL(10,2),

    valid_from          DATE NOT NULL,
    valid_to            DATE,
    is_current          BOOLEAN DEFAULT TRUE
);

-- Dimensión: Store
CREATE TABLE dim_store (
    store_key           SERIAL PRIMARY KEY,
    store_id            VARCHAR(50) NOT NULL UNIQUE,
    store_name          VARCHAR(255),
    store_type          VARCHAR(50),  -- 'Physical', 'Online'
    city                VARCHAR(100),
    state               VARCHAR(50),
    country             VARCHAR(50),
    region              VARCHAR(50),  -- 'North', 'South', 'East', 'West'
    manager_name        VARCHAR(255),
    opening_date        DATE,
    square_meters       INTEGER
);

-- ===== FACT TABLE =====
CREATE TABLE fact_sales (
    sale_id             SERIAL PRIMARY KEY,

    -- Foreign keys (dimensional keys)
    date_key            INTEGER NOT NULL REFERENCES dim_date(date_key),
    customer_key        INTEGER NOT NULL REFERENCES dim_customer(customer_key),
    product_key         INTEGER NOT NULL REFERENCES dim_product(product_key),
    store_key           INTEGER NOT NULL REFERENCES dim_store(store_key),

    -- Measures (métricas/hechos)
    quantity            INTEGER NOT NULL,
    unit_price          DECIMAL(10,2) NOT NULL,
    discount_amount     DECIMAL(10,2) DEFAULT 0,
    tax_amount          DECIMAL(10,2) DEFAULT 0,
    total_amount        DECIMAL(10,2) NOT NULL,
    cost_amount         DECIMAL(10,2),
    profit_amount       DECIMAL(10,2),

    -- Degenerate dimension (atributo de fact sin dimensión propia)
    order_number        VARCHAR(50),
    line_number         INTEGER,

    -- Audit
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índices para performance
CREATE INDEX idx_fact_sales_date ON fact_sales(date_key);
CREATE INDEX idx_fact_sales_customer ON fact_sales(customer_key);
CREATE INDEX idx_fact_sales_product ON fact_sales(product_key);
CREATE INDEX idx_fact_sales_store ON fact_sales(store_key);
CREATE INDEX idx_fact_sales_composite ON fact_sales(date_key, customer_key, product_key);

-- ===== QUERIES TÍPICAS =====

-- 1. Ventas por mes y categoría
SELECT
    d.year,
    d.month_name,
    p.category,
    SUM(f.total_amount) as total_sales,
    SUM(f.quantity) as total_quantity,
    COUNT(DISTINCT f.customer_key) as unique_customers
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
WHERE d.year = 2025
GROUP BY d.year, d.month_name, p.category
ORDER BY d.year, d.month, total_sales DESC;

-- 2. Top 10 clientes por región
SELECT
    s.region,
    c.customer_name,
    SUM(f.total_amount) as total_spent,
    COUNT(*) as order_count,
    AVG(f.total_amount) as avg_order_value
FROM fact_sales f
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_store s ON f.store_key = s.store_key
WHERE c.is_current = TRUE
GROUP BY s.region, c.customer_name
ORDER BY s.region, total_spent DESC
LIMIT 10;

-- 3. Performance de productos por tienda
SELECT
    s.store_name,
    p.product_name,
    SUM(f.quantity) as units_sold,
    SUM(f.total_amount) as revenue,
    SUM(f.profit_amount) as profit,
    SUM(f.profit_amount) / NULLIF(SUM(f.total_amount), 0) * 100 as profit_margin_pct
FROM fact_sales f
JOIN dim_store s ON f.store_key = s.store_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.year = 2025 AND d.quarter = 1
GROUP BY s.store_name, p.product_name
HAVING SUM(f.total_amount) > 10000
ORDER BY profit DESC;
```

### ETL para Star Schema

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, count, avg, max as spark_max

spark = SparkSession.builder.appName("Star Schema ETL").getOrCreate()

# ===== CARGAR DATOS RAW =====
raw_sales = spark.read.parquet("s3://raw-data/sales/")
raw_customers = spark.read.parquet("s3://raw-data/customers/")
raw_products = spark.read.parquet("s3://raw-data/products/")

# ===== POBLAR DIMENSIONES =====

# 1. dim_customer (SCD Type 2)
from pyspark.sql.functions import current_date, lit

# Detectar cambios en clientes
new_customers = raw_customers.alias("new")
existing_customers = spark.read.table("analytics.dim_customer") \
    .filter(col("is_current") == True).alias("existing")

# Clientes que cambiaron
changed_customers = new_customers.join(
    existing_customers,
    col("new.customer_id") == col("existing.customer_id"),
    "inner"
).where(
    (col("new.email") != col("existing.email")) |
    (col("new.customer_segment") != col("existing.customer_segment"))
)

# Cerrar versión anterior (set valid_to, is_current=False)
updates = changed_customers.select(
    col("existing.customer_key"),
    current_date().alias("valid_to"),
    lit(False).alias("is_current")
)

# Insertar nueva versión
inserts = changed_customers.select(
    col("new.customer_id"),
    col("new.customer_name"),
    col("new.email"),
    # ... otros campos
    current_date().alias("valid_from"),
    lit(None).alias("valid_to"),
    lit(True).alias("is_current")
)

# Ejecutar SCD Type 2 update
# (código simplificado, usar MERGE en producción)
updates.write.mode("append").saveAsTable("analytics.dim_customer_updates")
inserts.write.mode("append").saveAsTable("analytics.dim_customer")

# 2. dim_product (simpler - SCD Type 1, sobrescribir)
raw_products.write.mode("overwrite").saveAsTable("analytics.dim_product")

# ===== POBLAR FACT TABLE =====

# Join raw sales con dimensions para obtener surrogate keys
fact_sales = raw_sales \
    .join(
        spark.read.table("analytics.dim_date"),
        raw_sales.sale_date == col("date"),
        "inner"
    ) \
    .join(
        spark.read.table("analytics.dim_customer").filter(col("is_current") == True),
        "customer_id",
        "inner"
    ) \
    .join(
        spark.read.table("analytics.dim_product").filter(col("is_current") == True),
        "product_id",
        "inner"
    ) \
    .join(
        spark.read.table("analytics.dim_store"),
        "store_id",
        "inner"
    ) \
    .select(
        col("date_key"),
        col("customer_key"),
        col("product_key"),
        col("store_key"),
        col("quantity"),
        col("unit_price"),
        col("discount_amount"),
        col("tax_amount"),
        (col("quantity") * col("unit_price") - col("discount_amount") + col("tax_amount")).alias("total_amount"),
        col("order_number")
    )

# Escribir fact table (append incremental)
fact_sales.write.mode("append") \
    .partitionBy("date_key") \
    .saveAsTable("analytics.fact_sales")
```

---

## Snowflake Schema

### Diferencia con Star Schema

```
┌───────────────────────────────────────────────────┐
│           SNOWFLAKE SCHEMA                        │
│  (Dimensiones normalizadas en sub-dimensiones)   │
├───────────────────────────────────────────────────┤
│                                                    │
│                ┌────────────────┐                 │
│                │  FACT_SALES    │                 │
│                │  sale_id       │                 │
│                │  date_key      │                 │
│                │  customer_key  │                 │
│                │  product_key   │                 │
│                │  quantity      │                 │
│                │  amount        │                 │
│                └────────────────┘                 │
│                        │                           │
│                        ▼                           │
│                ┌────────────────┐                 │
│                │  DIM_PRODUCT   │                 │
│                │  product_key   │                 │
│                │  product_name  │                 │
│                │  category_key  │─────┐           │
│                │  brand_key     │──┐  │           │
│                └────────────────┘  │  │           │
│                                    │  │           │
│                    ┌───────────────┘  │           │
│                    ▼                  ▼           │
│          ┌────────────────┐  ┌────────────────┐ │
│          │  DIM_BRAND     │  │ DIM_CATEGORY   │ │
│          │  brand_key     │  │ category_key   │ │
│          │  brand_name    │  │ category_name  │ │
│          │  manufacturer  │  │ department     │ │
│          └────────────────┘  └────────────────┘ │
│                                                   │
│  Ventajas:                                        │
│  • Menos redundancia (más normalizado)           │
│  • Integridad referencial                        │
│                                                   │
│  Desventajas:                                     │
│  • Más JOINs (performance)                       │
│  • Más complejo para analistas                   │
└───────────────────────────────────────────────────┘
```

### Cuándo usar cada uno

| Escenario | Star | Snowflake |
|-----------|------|-----------|
| BI/Dashboards | ✅ Mejor | ❌ Más lento |
| Data warehouse grande | ❌ Más storage | ✅ Menos redundancia |
| Self-service analytics | ✅ Más simple | ❌ Más complejo |
| Dimensiones grandes (millones) | ✅ OK | ✅ Mejor normalización |
| Performance crítico | ✅ Menos JOINs | ❌ Más JOINs |

---

## Data Vault 2.0

### Componentes

```
┌──────────────────────────────────────────────────┐
│             DATA VAULT 2.0                       │
├──────────────────────────────────────────────────┤
│                                                   │
│  1. HUBS (Business keys)                         │
│     ┌────────────────┐                           │
│     │  HUB_CUSTOMER  │                           │
│     │  customer_hk   │ (hash key)                │
│     │  customer_id   │ (business key)            │
│     │  load_date     │                           │
│     │  record_source │                           │
│     └────────────────┘                           │
│                                                   │
│  2. LINKS (Relationships)                        │
│     ┌────────────────┐                           │
│     │  LINK_SALE     │                           │
│     │  sale_hk       │                           │
│     │  customer_hk   │ (FK to HUB_CUSTOMER)      │
│     │  product_hk    │ (FK to HUB_PRODUCT)       │
│     │  load_date     │                           │
│     │  record_source │                           │
│     └────────────────┘                           │
│                                                   │
│  3. SATELLITES (Descriptive data)                │
│     ┌────────────────┐                           │
│     │  SAT_CUSTOMER  │                           │
│     │  customer_hk   │ (FK to HUB)               │
│     │  load_date     │                           │
│     │  customer_name │                           │
│     │  email         │                           │
│     │  phone         │                           │
│     │  hash_diff     │ (detect changes)          │
│     └────────────────┘                           │
│                                                   │
│  Ventajas:                                        │
│  • Auditoría completa (histórico)                │
│  • Flexible (fácil agregar fuentes)              │
│  • Paralelizable (carga independiente)           │
│                                                   │
│  Desventajas:                                     │
│  • Complejo de implementar                       │
│  • No optimizado para queries (necesita views)   │
│  • Más storage que star schema                   │
└──────────────────────────────────────────────────┘
```

### Implementación básica

```sql
-- ===== HUB: Business key =====
CREATE TABLE hub_customer (
    customer_hk     VARCHAR(64) PRIMARY KEY,  -- SHA-256 hash de customer_id
    customer_id     VARCHAR(50) NOT NULL UNIQUE,
    load_date       TIMESTAMP NOT NULL,
    record_source   VARCHAR(50) NOT NULL
);

-- ===== SATELLITE: Descriptive attributes =====
CREATE TABLE sat_customer (
    customer_hk     VARCHAR(64) NOT NULL REFERENCES hub_customer(customer_hk),
    load_date       TIMESTAMP NOT NULL,
    load_end_date   TIMESTAMP,  -- NULL = current version
    customer_name   VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    address         TEXT,
    hash_diff       VARCHAR(64),  -- Hash de todos los atributos (detect changes)
    record_source   VARCHAR(50),

    PRIMARY KEY (customer_hk, load_date)
);

-- ===== LINK: Relationship =====
CREATE TABLE link_sale (
    sale_hk         VARCHAR(64) PRIMARY KEY,
    customer_hk     VARCHAR(64) NOT NULL REFERENCES hub_customer(customer_hk),
    product_hk      VARCHAR(64) NOT NULL REFERENCES hub_product(product_hk),
    store_hk        VARCHAR(64) NOT NULL REFERENCES hub_store(store_hk),
    load_date       TIMESTAMP NOT NULL,
    record_source   VARCHAR(50) NOT NULL
);

-- ===== SATELLITE del LINK (métricas) =====
CREATE TABLE sat_sale (
    sale_hk         VARCHAR(64) NOT NULL REFERENCES link_sale(sale_hk),
    load_date       TIMESTAMP NOT NULL,
    quantity        INTEGER,
    unit_price      DECIMAL(10,2),
    total_amount    DECIMAL(10,2),
    hash_diff       VARCHAR(64),
    record_source   VARCHAR(50),

    PRIMARY KEY (sale_hk, load_date)
);

-- ===== CARGAR DATOS (con hashing) =====
INSERT INTO hub_customer (customer_hk, customer_id, load_date, record_source)
SELECT
    SHA2(customer_id, 256) as customer_hk,
    customer_id,
    CURRENT_TIMESTAMP,
    'SHOPIFY_API'
FROM staging.customers
WHERE NOT EXISTS (
    SELECT 1 FROM hub_customer h
    WHERE h.customer_id = staging.customers.customer_id
);

-- ===== VIEW para analytics (presenta como star schema) =====
CREATE VIEW analytics.dim_customer AS
SELECT
    h.customer_hk,
    h.customer_id,
    s.customer_name,
    s.email,
    s.phone
FROM hub_customer h
JOIN sat_customer s ON h.customer_hk = s.customer_hk
WHERE s.load_end_date IS NULL;  -- Solo versión actual
```

---

## Dimensional Modeling

### Tipos de Facts

```sql
-- ===== 1. TRANSACTION FACT (granularidad fina) =====
-- Cada fila = 1 transacción
CREATE TABLE fact_sales_transaction (
    transaction_id      SERIAL PRIMARY KEY,
    date_key            INTEGER,
    customer_key        INTEGER,
    product_key         INTEGER,
    quantity            INTEGER,
    amount              DECIMAL(10,2)
);

-- ===== 2. PERIODIC SNAPSHOT FACT (agregado periódico) =====
-- Cada fila = estado en un punto en el tiempo
CREATE TABLE fact_account_snapshot (
    snapshot_date_key   INTEGER,
    account_key         INTEGER,
    balance             DECIMAL(15,2),
    transaction_count   INTEGER,
    PRIMARY KEY (snapshot_date_key, account_key)
);

-- ===== 3. ACCUMULATING SNAPSHOT FACT (pipeline/proceso) =====
-- Cada fila = ciclo de vida completo de un proceso
CREATE TABLE fact_order_fulfillment (
    order_key           INTEGER PRIMARY KEY,
    order_date_key      INTEGER,
    payment_date_key    INTEGER,
    shipped_date_key    INTEGER,
    delivered_date_key  INTEGER,
    order_amount        DECIMAL(10,2),
    days_to_ship        INTEGER,
    days_to_deliver     INTEGER
);

-- Se actualiza conforme avanza el proceso
-- Ejemplo: Orden creada → payment_date_key NULL
--          Pago recibido → payment_date_key populated
--          Enviado       → shipped_date_key populated
```

### Tipos de Dimensions

```sql
-- ===== 1. CONFORMED DIMENSION (compartida entre facts) =====
-- dim_date usada por TODOS los fact tables
CREATE TABLE dim_date (...);  -- Una sola tabla date

-- Múltiples facts la referencian
fact_sales.date_key → dim_date.date_key
fact_inventory.date_key → dim_date.date_key
fact_hr.date_key → dim_date.date_key

-- ===== 2. JUNK DIMENSION (flags/indicators) =====
-- Combinar múltiples flags en una dimensión
CREATE TABLE dim_sales_flags (
    sales_flag_key      SERIAL PRIMARY KEY,
    is_promotion        BOOLEAN,
    is_return           BOOLEAN,
    is_warranty         BOOLEAN,
    payment_method      VARCHAR(20),  -- 'Cash', 'Credit', 'Debit'
    shipment_type       VARCHAR(20)   -- 'Standard', 'Express', 'Overnight'
);

-- En vez de tener 5 columnas en fact, solo 1 FK
-- fact_sales.sales_flag_key → dim_sales_flags

-- ===== 3. DEGENERATE DIMENSION (sin tabla propia) =====
-- Atributos que viven en fact table
CREATE TABLE fact_sales (
    sale_id             SERIAL PRIMARY KEY,
    date_key            INTEGER,
    customer_key        INTEGER,
    order_number        VARCHAR(50),  -- Degenerate dimension (no tiene dim_order)
    invoice_number      VARCHAR(50),  -- Degenerate dimension
    amount              DECIMAL(10,2)
);

-- ===== 4. ROLE-PLAYING DIMENSION (misma dim, múltiples roles) =====
-- dim_date usada múltiples veces con distintos roles
CREATE TABLE fact_orders (
    order_id            SERIAL PRIMARY KEY,
    order_date_key      INTEGER REFERENCES dim_date(date_key),
    ship_date_key       INTEGER REFERENCES dim_date(date_key),
    delivery_date_key   INTEGER REFERENCES dim_date(date_key),
    amount              DECIMAL(10,2)
);

-- Misma tabla dim_date, 3 roles distintos:
-- order_date_key    = "Fecha de orden"
-- ship_date_key     = "Fecha de envío"
-- delivery_date_key = "Fecha de entrega"
```

---

## OLAP vs OLTP

### Comparación

| Aspecto | OLTP | OLAP |
|---------|------|------|
| **Propósito** | Transactions (día a día) | Analytics (reporting) |
| **Operaciones** | INSERT, UPDATE, DELETE | SELECT (complex queries) |
| **Normalización** | Alta (3NF) | Baja (star/snowflake) |
| **Tamaño queries** | Pequeños, rápidos | Grandes, complejos |
| **Volumen datos** | GB-TB | TB-PB |
| **Usuarios** | Muchos (app users) | Pocos (analistas) |
| **Ejemplo** | PostgreSQL, MySQL | Redshift, Snowflake, BigQuery |

### Ejemplo práctico

```sql
-- ===== OLTP: Insertar orden =====
-- Base normalizada (3NF)
BEGIN;

INSERT INTO customers (customer_id, name, email)
VALUES ('C123', 'John Doe', 'john@example.com')
ON CONFLICT (customer_id) DO NOTHING;

INSERT INTO orders (order_id, customer_id, order_date, status)
VALUES ('O456', 'C123', CURRENT_DATE, 'pending');

INSERT INTO order_lines (order_id, product_id, quantity, price)
VALUES
    ('O456', 'P001', 2, 29.99),
    ('O456', 'P002', 1, 49.99);

COMMIT;

-- Query rápido, pocos registros, ACID compliant

-- ===== OLAP: Análisis de ventas =====
-- Base dimensional (star schema)
SELECT
    d.year,
    d.quarter,
    p.category,
    c.customer_segment,
    COUNT(DISTINCT f.customer_key) as customer_count,
    SUM(f.quantity) as total_units,
    SUM(f.total_amount) as total_revenue,
    AVG(f.total_amount) as avg_order_value,
    SUM(f.profit_amount) / NULLIF(SUM(f.total_amount), 0) * 100 as profit_margin_pct
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_customer c ON f.customer_key = c.customer_key
WHERE d.year IN (2024, 2025)
GROUP BY d.year, d.quarter, p.category, c.customer_segment
ORDER BY total_revenue DESC;

-- Query complejo, millones de registros, read-only
```

---

## Slowly Changing Dimensions (SCD)

### Type 0: No cambios

```sql
-- Los valores NUNCA cambian (ej: fecha de nacimiento)
CREATE TABLE dim_customer_type0 (
    customer_key    SERIAL PRIMARY KEY,
    customer_id     VARCHAR(50),
    birth_date      DATE  -- NUNCA cambia
);
```

### Type 1: Sobrescribir

```sql
-- Solo mantener valor actual (sin historial)
CREATE TABLE dim_customer_type1 (
    customer_key    SERIAL PRIMARY KEY,
    customer_id     VARCHAR(50),
    email           VARCHAR(255),  -- Se sobrescribe
    phone           VARCHAR(50)    -- Se sobrescribe
);

-- Update sobrescribe valor anterior
UPDATE dim_customer_type1
SET email = 'new_email@example.com'
WHERE customer_id = 'C123';

-- Historial perdido (no sabemos email anterior)
```

### Type 2: Mantener historial (más común)

```sql
-- Mantener versiones históricas
CREATE TABLE dim_customer_type2 (
    customer_key    SERIAL PRIMARY KEY,      -- Surrogate key (nuevo por versión)
    customer_id     VARCHAR(50),             -- Natural key (mismo por versión)
    customer_name   VARCHAR(255),
    email           VARCHAR(255),
    customer_segment VARCHAR(50),

    -- SCD Type 2 fields
    valid_from      DATE NOT NULL,
    valid_to        DATE,                    -- NULL = current
    is_current      BOOLEAN DEFAULT TRUE,
    version         INTEGER DEFAULT 1
);

-- ===== INSERTAR NUEVO CLIENTE =====
INSERT INTO dim_customer_type2 (customer_id, customer_name, email, customer_segment, valid_from, is_current)
VALUES ('C123', 'John Doe', 'john@example.com', 'Regular', CURRENT_DATE, TRUE);

/*
customer_key | customer_id | email              | customer_segment | valid_from | valid_to | is_current
-------------|-------------|--------------------|------------------|------------|----------|------------
1            | C123        | john@example.com   | Regular          | 2025-01-01 | NULL     | TRUE
*/

-- ===== CLIENTE CAMBIA A SEGMENTO VIP =====
-- 1. Cerrar versión anterior
UPDATE dim_customer_type2
SET valid_to = CURRENT_DATE - INTERVAL '1 day',
    is_current = FALSE
WHERE customer_id = 'C123' AND is_current = TRUE;

-- 2. Insertar nueva versión
INSERT INTO dim_customer_type2 (customer_id, customer_name, email, customer_segment, valid_from, is_current)
VALUES ('C123', 'John Doe', 'john@example.com', 'VIP', CURRENT_DATE, TRUE);

/*
customer_key | customer_id | email              | customer_segment | valid_from | valid_to   | is_current
-------------|-------------|--------------------|------------------|------------|------------|------------
1            | C123        | john@example.com   | Regular          | 2025-01-01 | 2025-06-30 | FALSE
2            | C123        | john@example.com   | VIP              | 2025-07-01 | NULL       | TRUE
*/

-- ===== QUERY HISTÓRICO =====
-- Ventas cuando era Regular vs VIP
SELECT
    c.customer_segment,
    SUM(f.total_amount) as total_revenue
FROM fact_sales f
JOIN dim_customer_type2 c ON f.customer_key = c.customer_key
WHERE c.customer_id = 'C123'
GROUP BY c.customer_segment;

/*
customer_segment | total_revenue
-----------------|---------------
Regular          | 5,000
VIP              | 15,000
*/
```

### Type 3: Mantener valores anterior y actual

```sql
-- Solo guardar valor anterior y actual (no historial completo)
CREATE TABLE dim_customer_type3 (
    customer_key            SERIAL PRIMARY KEY,
    customer_id             VARCHAR(50),
    current_segment         VARCHAR(50),
    previous_segment        VARCHAR(50),
    segment_change_date     DATE
);

-- Cliente cambia de Regular a VIP
UPDATE dim_customer_type3
SET previous_segment = current_segment,
    current_segment = 'VIP',
    segment_change_date = CURRENT_DATE
WHERE customer_id = 'C123';

/*
customer_key | customer_id | current_segment | previous_segment | segment_change_date
-------------|-------------|-----------------|------------------|--------------------
1            | C123        | VIP             | Regular          | 2025-07-01
*/

-- Útil cuando solo necesitas comparar "antes vs ahora"
-- No mantiene historial completo (si cambia 3 veces, pierdes la primera)
```

---

## Aggregate Tables

### Por qué agregar

- Queries más rápidos (pre-computado)
- Menos procesamiento en tiempo de query
- Mejor UX en dashboards

### Ejemplo

```sql
-- ===== FACT TABLE DETALLADO (granularidad transaction) =====
-- 100M registros
CREATE TABLE fact_sales (
    sale_id         SERIAL PRIMARY KEY,
    date_key        INTEGER,
    customer_key    INTEGER,
    product_key     INTEGER,
    store_key       INTEGER,
    quantity        INTEGER,
    amount          DECIMAL(10,2)
);

-- Query sobre 100M registros = lento
SELECT
    d.year,
    d.month,
    SUM(f.amount) as total_sales
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
GROUP BY d.year, d.month;
-- Execution time: 45 seconds

-- ===== AGGREGATE TABLE (pre-computado) =====
-- Solo 1000 registros (agregado por mes)
CREATE TABLE fact_sales_monthly (
    year            INTEGER,
    month           INTEGER,
    product_key     INTEGER,
    store_key       INTEGER,
    total_quantity  BIGINT,
    total_amount    DECIMAL(15,2),
    order_count     INTEGER,
    unique_customers INTEGER,

    PRIMARY KEY (year, month, product_key, store_key)
);

-- Poblar aggregate (incremental nightly)
INSERT INTO fact_sales_monthly
SELECT
    d.year,
    d.month,
    f.product_key,
    f.store_key,
    SUM(f.quantity) as total_quantity,
    SUM(f.amount) as total_amount,
    COUNT(*) as order_count,
    COUNT(DISTINCT f.customer_key) as unique_customers
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.date >= CURRENT_DATE - INTERVAL '1 day'  -- Solo ayer
GROUP BY d.year, d.month, f.product_key, f.store_key
ON CONFLICT (year, month, product_key, store_key) DO UPDATE SET
    total_quantity = fact_sales_monthly.total_quantity + EXCLUDED.total_quantity,
    total_amount = fact_sales_monthly.total_amount + EXCLUDED.total_amount,
    order_count = fact_sales_monthly.order_count + EXCLUDED.order_count;

-- Query sobre 1000 registros = rápido
SELECT
    year,
    month,
    SUM(total_amount) as total_sales
FROM fact_sales_monthly
GROUP BY year, month;
-- Execution time: 0.2 seconds (225x más rápido!)

-- ===== MÚLTIPLES NIVELES DE AGREGACIÓN =====
-- Daily aggregate
CREATE TABLE fact_sales_daily AS
SELECT date_key, product_key, SUM(amount) as daily_total
FROM fact_sales
GROUP BY date_key, product_key;

-- Monthly aggregate (de daily)
CREATE TABLE fact_sales_monthly AS
SELECT year, month, product_key, SUM(daily_total) as monthly_total
FROM fact_sales_daily
JOIN dim_date ON date_key = dim_date.date_key
GROUP BY year, month, product_key;

-- Yearly aggregate (de monthly)
CREATE TABLE fact_sales_yearly AS
SELECT year, product_key, SUM(monthly_total) as yearly_total
FROM fact_sales_monthly
GROUP BY year, product_key;
```

---

## 🎯 Preguntas de Entrevista

**P: ¿Cuándo usarías Star Schema vs Snowflake Schema?**

R:
- **Star Schema**: BI/dashboards, performance crítico, analistas necesitan simplicidad
- **Snowflake Schema**: DWH grande con dimensiones enormes, necesitas normalización para reducir storage

**P: Explica SCD Type 2 y cuándo usarlo**

R:
SCD Type 2 mantiene historial completo de cambios en dimensiones.

Cada cambio crea nueva fila con:
- `valid_from` / `valid_to` dates
- `is_current` flag
- Surrogate key único por versión

Usar cuando:
- Necesitas análisis histórico ("¿qué segmento era el cliente cuando compró?")
- Compliance requiere audit trail
- Dimensiones cambian frecuentemente

NO usar si:
- No necesitas historial (Type 1 más simple)
- Dimensión cambia muy frecuentemente (explosión de registros)

**P: Diseña un modelo dimensional para e-commerce con estos requerimientos:**
- Dashboard de ventas por producto, tienda, tiempo
- Análisis de clientes (segmentación, LTV)
- Historial de cambios de precio de producto

R:
```
DIMENSIONS:
- dim_date (granularidad día, con atributos year/quarter/month)
- dim_customer (SCD Type 2 para tracking de segment changes)
- dim_product (SCD Type 2 para price history)
- dim_store (SCD Type 1, stores rara vez cambian)

FACTS:
- fact_sales (transaction grain, una fila por item vendido)
  - FKs: date_key, customer_key, product_key, store_key
  - Measures: quantity, unit_price, discount, total_amount, cost, profit

AGGREGATES:
- fact_sales_daily (pre-agregado para dashboards)
- fact_sales_monthly (para trending análisis)
```

---

**Siguiente:** [12. DataOps y CI/CD](12_dataops_cicd.md)
