# 3. Procesos de Ingesta y Transformación (ETL / ELT)

---

## Definición de ETL y ELT

### ETL (Extract, Transform, Load)

**Proceso tradicional:** Extrae → Transforma → Carga

```
┌─────────┐     ┌───────────┐     ┌──────────┐     ┌───────────┐     ┌───────────┐
│ Sources │ --> │  Extract  │ --> │Transform │ --> │   Load    │ --> │ Warehouse │
└─────────┘     └───────────┘     └──────────┘     └───────────┘     └───────────┘
                                   (Servidor        (Datos limpios)
                                   intermedio)
```

**Características:**
- Transformación **fuera** del data warehouse
- Datos llegan **limpios y procesados**
- Requiere servidor intermedio (ETL engine)
- Más control sobre calidad antes de cargar

### ELT (Extract, Load, Transform)

**Proceso moderno:** Extrae → Carga → Transforma

```
┌─────────┐     ┌──────────┐     ┌──────────────────────────┐
│ Sources │ --> │ Extract  │ --> │         Load             │
└─────────┘     └──────────┘     │    (Raw Data Lake)       │
                                 └──────────┬───────────────┘
                                            │
                                            ▼
                                 ┌──────────────────────────┐
                                 │      Transform           │
                                 │  (Dentro del Warehouse)  │
                                 │      usando dbt/SQL      │
                                 └──────────────────────────┘
```

**Características:**
- Transformación **dentro** del data warehouse
- Datos raw disponibles
- Aprovecha poder de cloud warehouses
- Más flexible y rápido

---

## Herramientas Comunes

### Ingesta (Extract + Load)

#### Herramientas Managed (SaaS)

**Fivetran**
```yaml
# Configuración declarativa
connector:
  type: postgres
  host: mydb.example.com
  database: production
  schema: public

destination:
  type: snowflake
  database: raw_data

sync_frequency: every_5_minutes
```

**Características:**
- 🟢 Conectores pre-construidos (500+)
- 🟢 CDC automático
- 🟢 Schema drift handling
- 🔴 Costo alto
- 🔴 Menos control

**Airbyte**
```yaml
# Open source alternative a Fivetran
source:
  name: Postgres
  config:
    host: localhost
    port: 5432
    database: mydb

destination:
  name: BigQuery
  config:
    project_id: my-project
    dataset: raw_data
```

**Características:**
- 🟢 Open source
- 🟢 Self-hosted o cloud
- 🟢 Customizable
- 🔴 Más mantenimiento
- 🔴 Menos conectores que Fivetran

**Stitch**
- Similar a Fivetran
- Propiedad de Talend
- Más barato que Fivetran

#### Herramientas Cloud-Native

**AWS Glue**
```python
# Glue PySpark job
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

# Leer de S3
datasource = glueContext.create_dynamic_frame.from_catalog(
    database = "raw_db",
    table_name = "customers"
)

# Transformar
transformed = datasource.apply_mapping([
    ("customer_id", "long", "customer_id", "long"),
    ("name", "string", "customer_name", "string"),
    ("email", "string", "email", "string")
])

# Escribir a S3
glueContext.write_dynamic_frame.from_options(
    frame = transformed,
    connection_type = "s3",
    connection_options = {"path": "s3://my-bucket/processed/"},
    format = "parquet"
)
```

**Azure Data Factory**
```json
{
  "name": "CopyFromSQLToBlob",
  "type": "Copy",
  "source": {
    "type": "SqlSource",
    "sqlReaderQuery": "SELECT * FROM dbo.customers"
  },
  "sink": {
    "type": "BlobSink",
    "writeBatchSize": 10000
  }
}
```

**Google Cloud Dataflow**
```python
# Apache Beam pipeline
import apache_beam as beam

with beam.Pipeline() as pipeline:
    (pipeline
     | 'Read from BigQuery' >> beam.io.ReadFromBigQuery(
         query='SELECT * FROM `project.dataset.table`')
     | 'Transform' >> beam.Map(transform_function)
     | 'Write to GCS' >> beam.io.WriteToText('gs://bucket/output')
    )
```

### Transformación

**dbt (Data Build Tool)**
```sql
-- models/staging/stg_customers.sql
{{ config(materialized='view') }}

SELECT
    customer_id,
    UPPER(TRIM(name)) AS customer_name,
    LOWER(email) AS email,
    created_at
FROM {{ source('raw', 'customers') }}
WHERE deleted_at IS NULL
```

```sql
-- models/marts/fct_orders.sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),

customers AS (
    SELECT * FROM {{ ref('stg_customers') }}
)

SELECT
    o.order_id,
    o.customer_id,
    c.customer_name,
    o.order_date,
    o.total_amount
FROM orders o
LEFT JOIN customers c USING(customer_id)

{% if is_incremental() %}
WHERE o.order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

**Apache NiFi**
- GUI drag-and-drop para pipelines
- Buen para transformaciones complejas en tiempo real
- Menos común en modern data stack

**Talend**
- ETL tradicional (GUI-based)
- Enterprise-focused
- Menos usado en cloud-first companies

---

## Extracción de Datos

### Desde APIs

```python
import requests
import pandas as pd

# REST API
def extract_from_api():
    url = "https://api.example.com/v1/orders"
    headers = {"Authorization": f"Bearer {API_KEY}"}

    all_data = []
    page = 1

    while True:
        response = requests.get(
            url,
            headers=headers,
            params={"page": page, "per_page": 100}
        )

        data = response.json()
        if not data:
            break

        all_data.extend(data)
        page += 1

    return pd.DataFrame(all_data)

# GraphQL
def extract_from_graphql():
    query = """
    query {
      orders(first: 100) {
        edges {
          node {
            id
            customer { name email }
            items { product quantity }
          }
        }
      }
    }
    """

    response = requests.post(
        "https://api.example.com/graphql",
        json={"query": query},
        headers={"Authorization": f"Bearer {API_KEY}"}
    )

    return response.json()
```

### Desde Bases de Datos

```python
import psycopg2
import pandas as pd

# Full extract
def extract_from_postgres():
    conn = psycopg2.connect(
        host="localhost",
        database="mydb",
        user="user",
        password="password"
    )

    query = "SELECT * FROM orders WHERE created_at >= CURRENT_DATE - 1"
    df = pd.read_sql(query, conn)

    conn.close()
    return df

# Incremental extract (CDC simulation)
def extract_incremental():
    last_sync = get_last_sync_timestamp()  # From metadata table

    query = f"""
    SELECT * FROM orders
    WHERE updated_at > '{last_sync}'
    ORDER BY updated_at
    """

    df = pd.read_sql(query, conn)

    # Save new timestamp
    save_last_sync_timestamp(df['updated_at'].max())

    return df
```

### Change Data Capture (CDC)

**Captura solo cambios** en la base de datos.

```python
# Usando Debezium (CDC tool)
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "localhost",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "password",
    "database.dbname": "mydb",
    "table.include.list": "public.orders,public.customers",
    "plugin.name": "pgoutput"
  }
}
```

**Eventos capturados:**
```json
{
  "op": "u",  // update
  "before": {
    "order_id": 123,
    "status": "pending",
    "total": 100.00
  },
  "after": {
    "order_id": 123,
    "status": "completed",
    "total": 100.00
  },
  "ts_ms": 1642531200000
}
```

### Desde Archivos

```python
import pandas as pd
import boto3

# CSV desde S3
def extract_from_s3_csv():
    s3 = boto3.client('s3')
    obj = s3.get_object(Bucket='my-bucket', Key='data/orders.csv')
    df = pd.read_csv(obj['Body'])
    return df

# Parquet desde S3
def extract_from_s3_parquet():
    df = pd.read_parquet('s3://my-bucket/data/orders.parquet')
    return df

# Multiple files
def extract_from_s3_folder():
    import glob
    files = glob.glob('s3://my-bucket/data/2024-01-*/*.parquet')
    dfs = [pd.read_parquet(f) for f in files]
    return pd.concat(dfs, ignore_index=True)
```

### Desde Logs

```python
import re
from datetime import datetime

# Parse Apache logs
def extract_from_logs():
    log_pattern = r'(\S+) - - \[(.*?)\] "(.*?)" (\d+) (\d+)'

    logs = []
    with open('/var/log/apache/access.log') as f:
        for line in f:
            match = re.match(log_pattern, line)
            if match:
                logs.append({
                    'ip': match.group(1),
                    'timestamp': match.group(2),
                    'request': match.group(3),
                    'status': int(match.group(4)),
                    'size': int(match.group(5))
                })

    return pd.DataFrame(logs)
```

---

## Transformaciones Comunes

### Limpieza y Validación

```sql
-- Eliminar duplicados
WITH deduped AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id, order_date
            ORDER BY created_at DESC
        ) AS rn
    FROM raw_orders
)
SELECT * FROM deduped WHERE rn = 1;

-- Manejar NULLs
SELECT
    customer_id,
    COALESCE(email, 'unknown@example.com') AS email,
    COALESCE(phone, 'N/A') AS phone,
    NULLIF(status, '') AS status  -- Convert empty string to NULL
FROM raw_customers;

-- Validar formatos
SELECT *
FROM raw_customers
WHERE email LIKE '%@%.%'  -- Basic email validation
  AND phone ~ '^[0-9]{10}$'  -- 10 digit phone
  AND created_at <= CURRENT_DATE;  -- No future dates
```

### Enriquecimiento

```sql
-- Join para agregar información
SELECT
    o.order_id,
    o.customer_id,
    c.customer_name,
    c.segment,
    o.order_date,
    o.total_amount,
    -- Calcular campos derivados
    DATE_PART('year', o.order_date) AS order_year,
    DATE_PART('month', o.order_date) AS order_month,
    DATE_PART('dow', o.order_date) AS day_of_week,
    -- Categorizar
    CASE
        WHEN o.total_amount < 50 THEN 'Small'
        WHEN o.total_amount < 200 THEN 'Medium'
        ELSE 'Large'
    END AS order_size
FROM orders o
LEFT JOIN customers c USING(customer_id);
```

### Agregaciones

```sql
-- Resumen por cliente
SELECT
    customer_id,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS lifetime_value,
    AVG(total_amount) AS avg_order_value,
    MIN(order_date) AS first_order_date,
    MAX(order_date) AS last_order_date,
    MAX(order_date) - MIN(order_date) AS customer_tenure_days
FROM orders
GROUP BY customer_id;
```

### Pivoting

```sql
-- De filas a columnas
SELECT
    product_id,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 1 THEN quantity ELSE 0 END) AS jan,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 2 THEN quantity ELSE 0 END) AS feb,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 3 THEN quantity ELSE 0 END) AS mar
FROM order_items
WHERE EXTRACT(YEAR FROM order_date) = 2024
GROUP BY product_id;
```

### Deduplicación

```sql
-- Mantener registro más reciente
DELETE FROM customers
WHERE id IN (
    SELECT id
    FROM (
        SELECT id,
            ROW_NUMBER() OVER (
                PARTITION BY email
                ORDER BY updated_at DESC
            ) AS rn
        FROM customers
    ) t
    WHERE rn > 1
);
```

---

## Data Lineage y Trazabilidad

**Data Lineage:** Rastrear el **origen** y **transformaciones** de los datos.

### Implementación Básica

```sql
-- Metadata table
CREATE TABLE data_lineage (
    id SERIAL PRIMARY KEY,
    source_table VARCHAR(100),
    source_query TEXT,
    target_table VARCHAR(100),
    transformation_type VARCHAR(50),
    executed_at TIMESTAMP,
    row_count INT,
    execution_time_seconds INT
);

-- Log transformations
INSERT INTO data_lineage VALUES (
    DEFAULT,
    'raw.orders',
    'SELECT * FROM raw.orders WHERE date >= ...',
    'staging.orders',
    'deduplicate_and_clean',
    CURRENT_TIMESTAMP,
    15234,
    12
);
```

### Con dbt

dbt automáticamente genera lineage:

```yaml
# schema.yml
version: 2

models:
  - name: fct_orders
    description: "Fact table de órdenes procesadas"
    columns:
      - name: order_id
        description: "ID único del pedido"
        tests:
          - unique
          - not_null
```

**dbt genera:**
```
raw.orders
    ↓
stg_orders (limpieza)
    ↓
int_orders_enriched (join con customers)
    ↓
fct_orders (fact table final)
```

### Herramientas de Lineage

**Comerciales:**
- Alation
- Collibra
- Atlan

**Open Source:**
- Amundsen (Lyft)
- DataHub (LinkedIn)
- OpenMetadata

---

## Idempotencia e Incrementalidad

### Idempotencia

**Pipeline que puede ejecutarse múltiples veces** con el mismo resultado.

```python
# ❌ NO idempotente
def process_orders():
    # Siempre inserta, creará duplicados
    new_orders = extract_from_api()
    insert_into_db(new_orders)

# ✅ Idempotente
def process_orders_idempotent():
    new_orders = extract_from_api()

    # Opción 1: UPSERT (INSERT or UPDATE)
    upsert_into_db(new_orders, key='order_id')

    # Opción 2: DELETE + INSERT
    delete_existing(new_orders['order_id'])
    insert_into_db(new_orders)

    # Opción 3: MERGE (SQL)
    merge_into_db(new_orders)
```

```sql
-- MERGE (idempotente)
MERGE INTO target_orders t
USING source_orders s
ON t.order_id = s.order_id
WHEN MATCHED THEN
    UPDATE SET
        status = s.status,
        updated_at = s.updated_at
WHEN NOT MATCHED THEN
    INSERT (order_id, customer_id, status, created_at)
    VALUES (s.order_id, s.customer_id, s.status, s.created_at);
```

### Incrementalidad

**Procesar solo datos nuevos/modificados**, no todo desde cero.

```python
# Full load (ineficiente)
def full_load():
    all_orders = extract_all_orders()  # Millones de filas
    process(all_orders)

# Incremental load (eficiente)
def incremental_load():
    last_sync = get_last_sync_timestamp()

    new_orders = extract_orders_since(last_sync)  # Solo últimas 24h
    process(new_orders)

    update_last_sync_timestamp()
```

```sql
-- dbt incremental model
{{ config(materialized='incremental') }}

SELECT
    order_id,
    customer_id,
    order_date,
    total_amount
FROM {{ source('raw', 'orders') }}

{% if is_incremental() %}
-- Solo procesar nuevos datos
WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

---

## Estrategias de Carga

### Full Refresh

Recargar **todo** desde cero.

```python
# Truncate + Insert
def full_refresh():
    truncate_table('target_table')
    all_data = extract_all()
    insert_into_table(all_data)
```

**Cuándo usar:**
- Tabla pequeña
- Datos cambian frecuentemente
- Simple de implementar

### Incremental Append

**Agregar solo nuevos registros**.

```python
def incremental_append():
    last_id = get_max_id('target_table')
    new_data = extract_where(f"id > {last_id}")
    insert_into_table(new_data)
```

**Cuándo usar:**
- Datos inmutables (logs, eventos)
- Solo hay INSERT, no UPDATE/DELETE

### Incremental Upsert

**Insertar nuevos, actualizar existentes**.

```python
def incremental_upsert():
    last_sync = get_last_sync()
    changed_data = extract_where(f"updated_at > '{last_sync}'")
    upsert_into_table(changed_data, key='id')
```

**Cuándo usar:**
- Datos pueden cambiar (UPDATE)
- Necesitas estado más reciente

---

## Mejores Prácticas

### 1. Separar Raw de Processed

```
s3://bucket/
  ├── raw/              # Datos tal cual llegan
  │   └── orders/
  │       └── 2024-01-15/
  ├── staging/          # Limpieza básica
  │   └── orders/
  └── processed/        # Transformaciones finales
      └── fct_orders/
```

### 2. Versionamiento de Pipelines

```python
# etl/v1/extract_orders.py
def extract_orders_v1():
    # Lógica antigua
    pass

# etl/v2/extract_orders.py
def extract_orders_v2():
    # Nueva lógica con mejoras
    # Permite rollback a v1 si falla
    pass
```

### 3. Testing

```python
# tests/test_transformations.py
def test_deduplicate():
    input_data = [
        {'id': 1, 'name': 'Ana'},
        {'id': 1, 'name': 'Ana'},  # Duplicado
        {'id': 2, 'name': 'Carlos'}
    ]

    result = deduplicate(input_data)

    assert len(result) == 2
    assert result[0]['id'] == 1
```

### 4. Monitoring

```python
import logging

def extract_with_monitoring():
    logger = logging.getLogger(__name__)

    try:
        start_time = time.time()
        data = extract_from_api()

        logger.info(f"Extracted {len(data)} rows in {time.time() - start_time}s")

        return data
    except Exception as e:
        logger.error(f"Extraction failed: {e}")
        send_alert(f"ETL failed: {e}")
        raise
```

---

## Resumen

✅ **ETL** (transforma antes) vs **ELT** (transforma después) - ELT es moderno
✅ **Herramientas:** Fivetran/Airbyte (ingesta), dbt (transformación)
✅ **Extracción:** APIs, DBs, archivos, logs, CDC
✅ **Transformaciones:** Limpieza, joins, agregaciones, pivoting
✅ **Data Lineage:** Rastrear origen y transformaciones
✅ **Idempotencia:** Pipelines ejecutables múltiples veces
✅ **Incrementalidad:** Procesar solo datos nuevos

**Siguiente:** [04. Almacenamiento de Datos](04_almacenamiento_datos.md)
