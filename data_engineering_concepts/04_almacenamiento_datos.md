# 4. Almacenamiento de Datos

---

## Data Warehouse (Almacén de Datos)

Sistema optimizado para **análisis y reportes** con datos estructurados y procesados.

### Arquitectura Columnar

A diferencia de bases de datos tradicionales (row-based), los warehouses usan **almacenamiento columnar**.

**Row-based (OLTP):**
```
Row 1: [1, "Ana", "ana@email.com", 50000]
Row 2: [2, "Carlos", "carlos@email.com", 60000]
Row 3: [3, "María", "maria@email.com", 55000]
```

**Column-based (OLAP):**
```
ID:     [1, 2, 3]
Nombre: ["Ana", "Carlos", "María"]
Email:  ["ana@email.com", "carlos@email.com", "maria@email.com"]
Salario:[50000, 60000, 55000]
```

**Ventajas columnar:**
- ✅ Lee solo columnas necesarias
- ✅ Mejor compresión (valores similares juntos)
- ✅ Queries analíticas más rápidas
- ✅ Agregaciones eficientes

### Principales Data Warehouses

#### Snowflake

**Arquitectura:** Almacenamiento y cómputo separados.

```
┌─────────────────────────────────────┐
│      Servicios Cloud (Metadata)    │
└─────────────────────────────────────┘
            │
    ┌───────┴────────┐
    │                │
┌───▼────┐    ┌──────▼──┐
│Compute │    │ Compute │  ← Warehouses (escalables independientemente)
│ Layer  │    │ Layer   │
└───┬────┘    └────┬────┘
    │              │
    └──────┬───────┘
           │
    ┌──────▼──────┐
    │   Storage   │  ← S3 (un solo lugar)
    │   Layer     │
    └─────────────┘
```

**Características:**
- Auto-scaling de compute
- Zero-copy cloning
- Time Travel (hasta 90 días)
- Multi-cloud (AWS, Azure, GCP)

**Ejemplo:**
```sql
-- Crear warehouse (compute)
CREATE WAREHOUSE analytics_wh
WITH WAREHOUSE_SIZE = 'MEDIUM'
AUTO_SUSPEND = 300
AUTO_RESUME = TRUE;

-- Usar warehouse
USE WAREHOUSE analytics_wh;

-- Query
SELECT
    DATE_TRUNC('month', order_date) AS month,
    SUM(amount) AS revenue
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY month;

-- Time Travel
SELECT * FROM customers
AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);

-- Clone (instantáneo, sin copiar datos)
CREATE TABLE customers_dev CLONE customers;
```

**Pricing:** Almacenamiento + Compute (créditos)

#### Amazon Redshift

**Arquitectura:** MPP (Massively Parallel Processing) con nodos.

```
┌────────────┐
│   Leader   │  ← Coordina queries
│    Node    │
└─────┬──────┘
      │
  ┌───┴────┬────────┬────────┐
  │        │        │        │
┌─▼──┐  ┌──▼─┐  ┌──▼─┐  ┌──▼─┐
│Comp│  │Comp│  │Comp│  │Comp│  ← Compute nodes (almacenan y procesan)
│ute │  │ute │  │ute │  │ute │
└────┘  └────┘  └────┘  └────┘
```

**Características:**
- Integración nativa con AWS
- Distribution keys y sort keys
- Spectrum (query sobre S3 sin cargar)
- Concurrency scaling

**Ejemplo:**
```sql
-- Crear tabla con distribution y sort keys
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    order_date DATE,
    amount DECIMAL(10,2)
)
DISTKEY(customer_id)    -- Distribuir por customer_id
SORTKEY(order_date);    -- Ordenar por fecha

-- COPY desde S3 (forma óptima de cargar datos)
COPY orders
FROM 's3://my-bucket/orders/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftRole'
FORMAT AS PARQUET;

-- Spectrum query (sin cargar a Redshift)
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'my_db'
IAM_ROLE 'arn:aws:iam::123456789012:role/SpectrumRole';

SELECT * FROM spectrum_schema.external_table
WHERE date >= '2024-01-01';
```

**Pricing:** Nodos por hora

#### Google BigQuery

**Arquitectura:** Serverless (sin gestión de infraestructura).

```
┌─────────────────────────────────────┐
│         Dremel Engine               │  ← Query execution
│    (Columnar, distributed)          │
└──────────────┬──────────────────────┘
               │
        ┌──────▼──────┐
        │  Colossus   │  ← Storage (columnar)
        │  (Storage)  │
        └─────────────┘
```

**Características:**
- Serverless (no infraestructura)
- Escala automática
- Pago por query (bytes escaneados)
- ML integrado (BigQuery ML)

**Ejemplo:**
```sql
-- Crear tabla particionada y clustered
CREATE TABLE `project.dataset.orders`
PARTITION BY DATE(order_date)
CLUSTER BY customer_id, product_id
AS
SELECT * FROM `project.dataset.raw_orders`;

-- Query (partition pruning)
SELECT
    customer_id,
    SUM(amount) AS total_spent
FROM `project.dataset.orders`
WHERE DATE(order_date) = '2024-01-15'  -- Solo escanea 1 partición
GROUP BY customer_id;

-- BigQuery ML
CREATE MODEL `project.dataset.churn_model`
OPTIONS(model_type='logistic_reg') AS
SELECT
    customer_id,
    total_orders,
    avg_order_value,
    churned AS label
FROM `project.dataset.customer_features`;

-- Predicción
SELECT * FROM ML.PREDICT(
    MODEL `project.dataset.churn_model`,
    TABLE `project.dataset.new_customers`
);
```

**Pricing:** Almacenamiento + Queries (por TB escaneado)

#### Azure Synapse Analytics

**Arquitectura:** Integración de data warehouse + big data.

**Características:**
- Integración con Azure ecosystem
- Serverless o dedicated pools
- Spark integrado
- Power BI integration

**Ejemplo:**
```sql
-- Dedicated SQL pool
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    order_date DATE,
    amount DECIMAL(10,2)
)
WITH (
    DISTRIBUTION = HASH(customer_id),
    CLUSTERED COLUMNSTORE INDEX
);

-- Serverless SQL
SELECT * FROM
OPENROWSET(
    BULK 'https://myaccount.blob.core.windows.net/data/orders/*.parquet',
    FORMAT = 'PARQUET'
) AS orders;
```

### Comparación Data Warehouses

| Característica | Snowflake | Redshift | BigQuery | Synapse |
|----------------|-----------|----------|----------|---------|
| **Cloud** | Multi-cloud | AWS only | GCP only | Azure only |
| **Escalabilidad** | Automática | Manual | Automática | Híbrida |
| **Pricing** | Storage + Compute | Nodos/hora | Storage + Query | Flexible |
| **Administración** | Baja | Media | Muy baja | Media |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **ML integrado** | ❌ | ❌ | ✅ | ✅ |

---

## Data Lake (Lago de Datos)

Repositorio centralizado para **datos raw** en cualquier formato.

### Características

- **Schema-on-read:** Define estructura al leer
- **Formatos múltiples:** CSV, JSON, Parquet, Avro, logs, imágenes
- **Bajo costo:** Almacenamiento object storage (S3, GCS)
- **Escalable:** Petabytes de datos
- **Flexible:** Cualquier tipo de análisis

### Principales Plataformas

#### Amazon S3

**Estructura típica:**
```
s3://my-data-lake/
├── raw/                    # Datos crudos
│   ├── orders/
│   │   └── year=2024/
│   │       └── month=01/
│   │           └── day=15/
│   │               └── orders.parquet
│   ├── customers/
│   └── products/
├── processed/              # Datos procesados
│   ├── fact_orders/
│   └── dim_customers/
└── curated/                # Datos business-ready
    └── dashboards/
```

**Características:**
- Durabilidad 99.999999999% (11 nines)
- Storage classes (S3 Standard, IA, Glacier)
- Lifecycle policies
- Versioning

**Ejemplo:**
```python
import boto3

s3 = boto3.client('s3')

# Upload
s3.upload_file(
    'local_file.csv',
    'my-data-lake',
    'raw/orders/2024/01/15/orders.csv'
)

# List objects
response = s3.list_objects_v2(
    Bucket='my-data-lake',
    Prefix='raw/orders/2024/01/'
)

for obj in response['Contents']:
    print(obj['Key'])
```

#### Google Cloud Storage (GCS)

Similar a S3, con integración nativa a BigQuery.

```bash
# Upload
gsutil cp local_file.parquet gs://my-data-lake/raw/orders/

# Query directo desde BigQuery
SELECT * FROM `my-project.my_dataset.EXTERNAL_TABLE`
```

#### Azure Blob Storage / ADLS Gen2

**ADLS Gen2:** Azure Data Lake Storage (optimizado para analytics).

```python
from azure.storage.blob import BlobServiceClient

blob_service = BlobServiceClient.from_connection_string(conn_str)
blob_client = blob_service.get_blob_client(
    container="data-lake",
    blob="raw/orders/2024/01/15/orders.parquet"
)

with open("orders.parquet", "rb") as data:
    blob_client.upload_blob(data)
```

---

## Formatos de Almacenamiento

### CSV (Comma-Separated Values)

**Ventajas:**
- ✅ Legible por humanos
- ✅ Universal
- ✅ Fácil de generar

**Desventajas:**
- ❌ No tiene schema
- ❌ No soporta tipos de datos complejos
- ❌ Baja compresión
- ❌ Lento para queries

```csv
order_id,customer_id,amount,order_date
1,101,99.99,2024-01-15
2,102,149.50,2024-01-15
```

**Cuándo usar:** Intercambio simple de datos, exports

### Parquet (Formato Columnar)

**Ventajas:**
- ✅ Compresión excelente (10x vs CSV)
- ✅ Columnar (lee solo columnas necesarias)
- ✅ Schema incluido
- ✅ Soporta tipos complejos (nested structures)
- ✅ Splitting para procesamiento paralelo

**Desventajas:**
- ❌ No legible por humanos
- ❌ Requiere herramientas especiales

```python
import pandas as pd

# Escribir
df = pd.DataFrame({
    'order_id': [1, 2, 3],
    'amount': [99.99, 149.50, 75.00]
})
df.to_parquet('orders.parquet', compression='snappy')

# Leer
df = pd.read_parquet('orders.parquet')

# Leer columnas específicas
df = pd.read_parquet('orders.parquet', columns=['order_id', 'amount'])
```

**Cuándo usar:** Data lakes, analytics, Spark

### ORC (Optimized Row Columnar)

Similar a Parquet, optimizado para Hive/Hadoop.

**Ventajas:**
- ✅ Compresión excelente
- ✅ ACID transactions
- ✅ Índices integrados

**Cuándo usar:** Hadoop ecosystem, Hive

### Avro (Formato de Fila)

**Ventajas:**
- ✅ Schema evolution
- ✅ Compacto
- ✅ Eficiente para streaming (Kafka)

**Schema:**
```json
{
  "type": "record",
  "name": "Order",
  "fields": [
    {"name": "order_id", "type": "int"},
    {"name": "customer_id", "type": "int"},
    {"name": "amount", "type": "double"}
  ]
}
```

**Cuándo usar:** Kafka, event sourcing, streaming

### JSON

**Ventajas:**
- ✅ Legible por humanos
- ✅ Flexible (schema-less)
- ✅ Estructuras anidadas

**Desventajas:**
- ❌ Tamaño grande
- ❌ Lento para queries
- ❌ Poca compresión

**Cuándo usar:** APIs, configuración, datos semi-estructurados pequeños

### Comparación de Formatos

| Formato | Compresión | Velocidad | Schema | Uso Principal |
|---------|-----------|-----------|--------|---------------|
| **CSV** | ❌ Baja | ❌ Lento | ❌ No | Intercambio simple |
| **Parquet** | ✅ Alta | ✅ Rápido | ✅ Sí | Analytics, Data Lake |
| **ORC** | ✅ Alta | ✅ Rápido | ✅ Sí | Hadoop/Hive |
| **Avro** | ✅ Media | ✅ Medio | ✅ Sí | Streaming, Kafka |
| **JSON** | ❌ Baja | ❌ Lento | ❌ No | APIs, configs |

**Recomendación:** **Parquet** para la mayoría de casos en data lakes modernos.

---

## Lakehouse Architecture

Combina **Data Lake** (flexibilidad, bajo costo) con **Data Warehouse** (performance, ACID).

### Tecnologías

#### Delta Lake (Databricks)

**Características:**
- ACID transactions en data lake
- Schema enforcement
- Time travel
- Merge, update, delete operations

```python
from delta.tables import DeltaTable

# Escribir Delta
df.write.format("delta").save("/path/to/delta-table")

# Update
deltaTable = DeltaTable.forPath(spark, "/path/to/delta-table")
deltaTable.update(
    condition = "customer_id = 123",
    set = {"status": "'inactive'"}
)

# Time Travel
df = spark.read.format("delta").option("versionAsOf", 0).load("/path")

# MERGE (upsert)
deltaTable.alias("target").merge(
    updates.alias("source"),
    "target.id = source.id"
).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()
```

#### Apache Iceberg

**Características:**
- Schema evolution
- Hidden partitioning
- Time travel
- Multiple engine support (Spark, Flink, Presto)

```python
# Crear tabla Iceberg
spark.sql("""
    CREATE TABLE catalog.db.table (
        id INT,
        data STRING,
        category STRING
    )
    USING iceberg
    PARTITIONED BY (category)
""")

# Time travel
spark.read.option("snapshot-id", 10963874102873L).table("catalog.db.table")
```

#### Apache Hudi

**Características:**
- Upserts eficientes
- Incremental processing
- Record-level updates

**Tipos de tablas:**
- **Copy on Write (CoW):** Optimizado para lectura
- **Merge on Read (MoR):** Optimizado para escritura

```python
# Escribir Hudi
df.write.format("hudi").options(**hudi_options).mode("append").save(path)
```

### Comparación Lakehouse Technologies

| Característica | Delta Lake | Iceberg | Hudi |
|----------------|------------|---------|------|
| **Creador** | Databricks | Netflix | Uber |
| **ACID** | ✅ | ✅ | ✅ |
| **Time Travel** | ✅ | ✅ | ✅ |
| **Engine Support** | Spark | Multi-engine | Spark, Flink |
| **Performance** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Adopción** | Alta | Media-Alta | Media |

---

## Medallion Architecture (Bronze/Silver/Gold)

Organización en **capas** de refinamiento de datos.

```
┌──────────────┐
│   BRONZE     │  ← Raw data (tal cual llega)
│ (Raw Layer)  │
└──────┬───────┘
       │
       ▼
┌───────────────┐
│   SILVER      │  ← Limpieza, validación, deduplicación
│(Refined Layer)│
└──────┬────────┘
       │
       ▼
┌───────────────┐
│    GOLD       │  ← Business logic, agregaciones, features
│(Curated Layer)│
└───────────────┘
```

### Bronze Layer (Raw)

**Propósito:** Almacenar datos **tal cual llegan** de la fuente.

**Características:**
- Inmutable (append-only)
- Mínima transformación
- Todos los campos originales
- Metadata de ingesta

```python
# Bronze: guardar raw data
raw_data = extract_from_api()

bronze_df = raw_data.assign(
    ingested_at=datetime.now(),
    source='api_orders',
    ingestion_id=uuid.uuid4()
)

bronze_df.write.format("delta").mode("append").save("s3://lake/bronze/orders/")
```

### Silver Layer (Refined)

**Propósito:** Datos **limpios y validados**.

**Transformaciones:**
- Deduplicación
- Validación de tipos
- Estandarización
- Eliminar columnas innecesarias

```sql
-- Silver: limpieza y validación
CREATE TABLE silver.orders AS
SELECT DISTINCT
    order_id,
    customer_id,
    CAST(order_date AS DATE) AS order_date,
    CAST(amount AS DECIMAL(10,2)) AS amount,
    UPPER(TRIM(status)) AS status
FROM bronze.orders
WHERE order_id IS NOT NULL
  AND amount > 0
  AND order_date >= '2020-01-01';
```

### Gold Layer (Curated)

**Propósito:** Datos **business-ready** para analytics/ML.

**Transformaciones:**
- Joins entre tablas
- Agregaciones
- Business logic
- Features para ML

```sql
-- Gold: business metrics
CREATE TABLE gold.customer_metrics AS
SELECT
    c.customer_id,
    c.customer_name,
    c.segment,
    COUNT(o.order_id) AS total_orders,
    SUM(o.amount) AS lifetime_value,
    AVG(o.amount) AS avg_order_value,
    MIN(o.order_date) AS first_order_date,
    MAX(o.order_date) AS last_order_date,
    DATEDIFF(day, MIN(o.order_date), MAX(o.order_date)) AS customer_tenure_days
FROM silver.customers c
LEFT JOIN silver.orders o USING(customer_id)
GROUP BY c.customer_id, c.customer_name, c.segment;
```

---

## Data Retention Policies

Políticas para **retener** o **eliminar** datos antiguos.

### Razones

- **Costo:** Reducir almacenamiento
- **Compliance:** GDPR, HIPAA (derecho al olvido)
- **Performance:** Menos datos = queries más rápidas

### Estrategias

#### 1. Time-based Retention

```sql
-- Eliminar datos > 2 años
DELETE FROM orders
WHERE order_date < CURRENT_DATE - INTERVAL '2 years';

-- Snowflake: Table retention
ALTER TABLE orders SET DATA_RETENTION_TIME_IN_DAYS = 7;
```

#### 2. Lifecycle Policies (S3)

```json
{
  "Rules": [{
    "Id": "Archive old data",
    "Status": "Enabled",
    "Transitions": [
      {
        "Days": 90,
        "StorageClass": "STANDARD_IA"  // Infrequent Access
      },
      {
        "Days": 365,
        "StorageClass": "GLACIER"  // Archive
      }
    ],
    "Expiration": {
      "Days": 2555  // 7 años, luego delete
    }
  }]
}
```

#### 3. Partitioning + Drop

```sql
-- Drop partición completa (rápido)
ALTER TABLE orders DROP PARTITION (year=2020, month=01);
```

---

## Cost Optimization

### Strategies

#### 1. Compresión

```python
# Parquet con compresión
df.to_parquet('data.parquet', compression='snappy')  # Rápido
df.to_parquet('data.parquet', compression='gzip')    # Más compresión
```

#### 2. Particionamiento

```sql
-- BigQuery: particionar reduce bytes escaneados
CREATE TABLE orders
PARTITION BY DATE(order_date)
AS SELECT * FROM raw_orders;

-- Query solo escanea partición necesaria
SELECT * FROM orders
WHERE DATE(order_date) = '2024-01-15';  -- Solo 1 día, no todo
```

#### 3. Clustering

```sql
-- Snowflake
ALTER TABLE orders CLUSTER BY (customer_id, order_date);

-- BigQuery
CREATE TABLE orders
PARTITION BY DATE(order_date)
CLUSTER BY customer_id, product_id;
```

#### 4. Lifecycle Policies

Mover datos fríos a storage más barato.

#### 5. Query Optimization

```sql
-- ❌ Malo: escanea todas las columnas
SELECT * FROM large_table;

-- ✅ Bueno: solo columnas necesarias
SELECT id, amount FROM large_table;
```

#### 6. Materialized Views

```sql
-- Pre-calcular agregaciones costosas
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT
    DATE(order_date) AS day,
    SUM(amount) AS daily_revenue
FROM orders
GROUP BY day;

-- Query rápida sobre MV
SELECT * FROM mv_daily_sales WHERE day >= '2024-01-01';
```

---

## Resumen

✅ **Data Warehouse** (Snowflake, BigQuery, Redshift) para analytics estructurado
✅ **Data Lake** (S3, GCS) para datos raw de cualquier tipo
✅ **Parquet** es el formato estándar para data lakes
✅ **Lakehouse** (Delta Lake, Iceberg) combina lo mejor de ambos
✅ **Medallion** (Bronze/Silver/Gold) para organizar data lake
✅ **Particionamiento y Clustering** para reducir costos
✅ **Retention policies** para compliance y cost optimization

**Siguiente:** [05. Procesamiento Batch y Streaming](05_procesamiento_batch_streaming.md)
