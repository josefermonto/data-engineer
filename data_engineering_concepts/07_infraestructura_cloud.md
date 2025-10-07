# 7. Infraestructura Cloud para Data Engineering

## 📋 Tabla de Contenidos
- [Cloud Providers Overview](#cloud-providers-overview)
- [AWS Data Services](#aws-data-services)
- [Google Cloud Platform (GCP)](#google-cloud-platform-gcp)
- [Azure Data Platform](#azure-data-platform)
- [Comparación de Servicios](#comparación-de-servicios)
- [Arquitecturas de Referencia](#arquitecturas-de-referencia)
- [Cost Optimization](#cost-optimization)
- [Multi-Cloud y Hybrid](#multi-cloud-y-hybrid)

---

## Cloud Providers Overview

### Comparación General

| Aspecto | AWS | GCP | Azure |
|---------|-----|-----|-------|
| **Cuota de mercado** | ~32% | ~10% | ~23% |
| **Fortaleza** | Madurez, servicios | BigQuery, ML | Enterprise, Microsoft stack |
| **Curva aprendizaje** | Media-Alta | Media | Media |
| **Pricing** | Complejo | Simple | Media complejidad |
| **Data services** | Más opciones | Integrado | Synapse (todo-en-uno) |
| **Mejor para** | Empresas grandes | Analytics, ML | Corporativos, .NET |

---

## AWS Data Services

### Arquitectura de Servicios AWS

```
┌────────────────────────────────────────────────┐
│                    AWS DATA PLATFORM           │
├────────────────────────────────────────────────┤
│                                                │
│  INGESTION                    PROCESSING       │
│  ┌──────────────┐            ┌──────────────┐  │
│  │   Kinesis    │            │   EMR        │  │
│  │   Data       │───────────▶│   (Spark)    │  │
│  │   Streams    │            └──────────────┘  │
│  └──────────────┘                   │          │
│         │                           │          │
│         ▼                           ▼          │
│  ┌──────────────┐            ┌──────────────┐  │
│  │   Kinesis    │            │    Glue      │  │
│  │   Firehose   │───────────▶│    ETL       │  │
│  └──────────────┘            └──────────────┘  │
│         │                            │         │
│         ▼                            ▼         │
│      STORAGE                  DATA WAREHOUSE   │
│  ┌──────────────┐            ┌──────────────┐  │
│  │      S3      │◀───────────│   Redshift   │  │
│  │   Data Lake  │            │   Cluster    │  │
│  └──────────────┘            └──────────────┘  │
│         │                            │         │
│         ▼                            ▼         │
│      CATALOG                     ANALYTICS     │
│  ┌──────────────┐            ┌──────────────┐  │
│  │     Glue     │            │   Athena     │  │
│  │   Catalog    │            │   (Query)    │  │
│  └──────────────┘            └──────────────┘  │
│                                      │         │
│   ORCHESTRATION                      ▼         │
│  ┌──────────────┐            ┌──────────────┐  │
│  │     MWAA     │            │  QuickSight  │  │
│  │  (Airflow)   │            │      BI      │  │
│  └──────────────┘            └──────────────┘  │
└────────────────────────────────────────────────┘
```

### 1. Amazon S3 (Simple Storage Service)

**Características:**
- Object storage infinitamente escalable
- 99.999999999% durabilidad (11 nines)
- Múltiples storage classes para optimizar costos
- Soporte para data lakes y archival

**Storage Classes:**

| Storage Class | Uso | Costo (GB/mes) | Retrieval |
|--------------|-----|----------------|-----------|
| **S3 Standard** | Datos frecuentes | $0.023 | Inmediato |
| **S3 Intelligent-Tiering** | Acceso variable | $0.023-$0.0125 | Automático |
| **S3 Standard-IA** | Acceso infrecuente | $0.0125 | Inmediato |
| **S3 Glacier Instant** | Archivo + query | $0.004 | Milisegundos |
| **S3 Glacier Flexible** | Backup | $0.0036 | 1-5 minutos |
| **S3 Glacier Deep Archive** | Compliance | $0.00099 | 12 horas |

**Ejemplo: Organización de Data Lake**

```bash
s3://my-datalake/
├── raw/                          # Bronze layer
│   ├── sales/
│   │   └── year=2025/
│   │       └── month=01/
│   │           └── day=15/
│   │               └── data.parquet
│   ├── customers/
│   └── products/
├── processed/                    # Silver layer
│   ├── sales_cleaned/
│   └── customer_enriched/
├── analytics/                    # Gold layer
│   ├── sales_aggregated/
│   └── customer_segments/
└── archive/                      # Glacier
    └── historical_data/
```

**Python: Trabajar con S3**

```python
import boto3
from botocore.exceptions import ClientError
import pandas as pd
from io import BytesIO

# Cliente S3
s3_client = boto3.client('s3')

# ===== SUBIR ARCHIVO =====
def upload_to_s3(local_file: str, bucket: str, s3_key: str):
    """Subir archivo a S3"""
    try:
        s3_client.upload_file(
            Filename=local_file,
            Bucket=bucket,
            Key=s3_key,
            ExtraArgs={
                'ServerSideEncryption': 'AES256',  # Encriptar
                'StorageClass': 'INTELLIGENT_TIERING'  # Auto-tiering
            }
        )
        print(f"Uploaded {local_file} to s3://{bucket}/{s3_key}")
    except ClientError as e:
        print(f"Error: {e}")

# ===== LEER PARQUET DESDE S3 =====
def read_parquet_from_s3(bucket: str, key: str) -> pd.DataFrame:
    """Leer Parquet directamente desde S3"""
    obj = s3_client.get_object(Bucket=bucket, Key=key)
    df = pd.read_parquet(BytesIO(obj['Body'].read()))
    return df

# Uso
df = read_parquet_from_s3('my-datalake', 'raw/sales/2025/01/15/data.parquet')

# ===== ESCRIBIR DATAFRAME A S3 =====
def write_parquet_to_s3(df: pd.DataFrame, bucket: str, key: str):
    """Escribir DataFrame como Parquet en S3"""
    buffer = BytesIO()
    df.to_parquet(buffer, engine='pyarrow', compression='snappy')
    buffer.seek(0)

    s3_client.put_object(
        Bucket=bucket,
        Key=key,
        Body=buffer.getvalue(),
        ServerSideEncryption='AES256'
    )

# ===== LIFECYCLE POLICIES (Boto3) =====
lifecycle_config = {
    'Rules': [
        {
            'Id': 'Move raw to Glacier after 90 days',
            'Status': 'Enabled',
            'Prefix': 'raw/',
            'Transitions': [
                {
                    'Days': 90,
                    'StorageClass': 'GLACIER'
                }
            ]
        },
        {
            'Id': 'Delete temp files after 7 days',
            'Status': 'Enabled',
            'Prefix': 'temp/',
            'Expiration': {
                'Days': 7
            }
        }
    ]
}

s3_client.put_bucket_lifecycle_configuration(
    Bucket='my-datalake',
    LifecycleConfiguration=lifecycle_config
)

# ===== PARTITIONED WRITES (PySpark) =====
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("S3Write").getOrCreate()

df = spark.read.parquet("s3://raw-data/sales/")

# Escribir particionado por fecha
df.write.mode("overwrite") \
    .partitionBy("year", "month", "day") \
    .parquet("s3://processed-data/sales/")

# Resultado:
# s3://processed-data/sales/
#   year=2025/
#     month=01/
#       day=15/
#         part-00000.snappy.parquet
```

### 2. AWS Glue

**Características:**
- Servicio ETL serverless
- Glue Catalog (metastore compatible con Hive)
- Glue Crawlers (auto-discovery de esquemas)
- Glue DataBrew (transformaciones no-code)

**Componentes:**

```
┌────────────────────────────────────┐
│          AWS GLUE                  │
├────────────────────────────────────┤
│                                    │
│  1. Crawlers                       │
│     └─ Escanean S3/RDS/Redshift    │
│     └─ Infieren schema             │
│     └─ Populan Catalog             │
│                                    │
│  2. Glue Catalog                   │
│     └─ Metadata store              │
│     └─ Compatible Hive metastore   │
│     └─ Usado por Athena, EMR       │
│                                    │
│  3. Glue ETL Jobs                  │
│     └─ Spark jobs (Python/Scala)   │
│     └─ Serverless                  │
│     └─ Auto-scaling                │
│                                    │
│  4. Glue DataBrew                  │
│     └─ Visual data prep            │
│     └─ Data quality rules          │
└────────────────────────────────────┘
```

**Glue ETL Job (PySpark)**

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

# Inicializar contexto
args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# ===== LEER DESDE GLUE CATALOG =====
# El Crawler ya descubrió la tabla
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="ecommerce",
    table_name="raw_sales"
)

# ===== TRANSFORMACIONES =====
# Filtrar registros inválidos
filtered = Filter.apply(
    frame=datasource,
    f=lambda x: x["amount"] > 0 and x["customer_id"] is not None
)

# Mapear campos (renombrar, cambiar tipos)
mapped = ApplyMapping.apply(
    frame=filtered,
    mappings=[
        ("order_id", "string", "order_id", "string"),
        ("customer_id", "string", "customer_id", "string"),
        ("amount", "double", "amount", "decimal(10,2)"),
        ("order_date", "string", "order_date", "date"),
    ]
)

# Resolver choice types (cuando Glue detecta múltiples tipos)
resolved = ResolveChoice.apply(
    frame=mapped,
    choice="make_struct"
)

# Drop nulls
clean = DropNullFields.apply(frame=resolved)

# ===== ESCRIBIR A S3 + CATALOG =====
glueContext.write_dynamic_frame.from_catalog(
    frame=clean,
    database="ecommerce",
    table_name="processed_sales",
    transformation_ctx="datasink"
)

# O escribir directamente a S3 como Parquet
glueContext.write_dynamic_frame.from_options(
    frame=clean,
    connection_type="s3",
    connection_options={
        "path": "s3://processed-data/sales/",
        "partitionKeys": ["year", "month", "day"]
    },
    format="parquet",
    format_options={
        "compression": "snappy"
    }
)

job.commit()
```

**Glue Crawler (Terraform)**

```hcl
# Crear base de datos en Glue Catalog
resource "aws_glue_catalog_database" "ecommerce" {
  name = "ecommerce"
}

# Crawler para descubrir tablas en S3
resource "aws_glue_crawler" "sales_crawler" {
  name          = "sales-crawler"
  role          = aws_iam_role.glue_role.arn
  database_name = aws_glue_catalog_database.ecommerce.name

  s3_target {
    path = "s3://my-datalake/raw/sales/"
  }

  schedule = "cron(0 2 * * ? *)"  # Diario a las 2 AM

  schema_change_policy {
    delete_behavior = "LOG"
    update_behavior = "UPDATE_IN_DATABASE"
  }
}

# Glue Job
resource "aws_glue_job" "etl_sales" {
  name     = "etl-sales-job"
  role_arn = aws_iam_role.glue_role.arn

  command {
    name            = "glueetl"
    script_location = "s3://scripts/etl_sales.py"
    python_version  = "3"
  }

  glue_version = "4.0"

  max_capacity = 10  # DPU (Data Processing Units)

  default_arguments = {
    "--job-language"        = "python"
    "--job-bookmark-option" = "job-bookmark-enable"  # Procesar solo nuevos datos
    "--enable-metrics"      = "true"
  }
}
```

### 3. Amazon Redshift

**Características:**
- Data warehouse columnar
- Escala hasta PB de datos
- Integración con S3 (Redshift Spectrum)
- Serverless option disponible

**Arquitectura:**

```
┌─────────────────────────────────────┐
│       REDSHIFT CLUSTER              │
├─────────────────────────────────────┤
│                                     │
│  Leader Node                        │
│  └─ Query planning                  │
│  └─ Coordina compute nodes          │
│                                     │
│  Compute Nodes (2-128)              │
│  └─ Almacenamiento columnar         │
│  └─ Procesamiento paralelo          │
│  └─ Slices (2-32 por node)          │
│                                     │
│  Redshift Spectrum                  │
│  └─ Query S3 sin cargar datos       │
│  └─ Extiende storage infinitamente  │
└─────────────────────────────────────┘
```

**SQL Examples:**

```sql
-- ===== CREAR TABLA OPTIMIZADA =====
CREATE TABLE sales (
    order_id        VARCHAR(50) ENCODE LZO,
    customer_id     VARCHAR(50) ENCODE LZO,
    product_id      VARCHAR(50) ENCODE LZO,
    amount          DECIMAL(10,2) ENCODE AZ64,
    order_date      DATE ENCODE AZ64,
    created_at      TIMESTAMP ENCODE AZ64
)
DISTSTYLE KEY  -- Distribuir por key
DISTKEY (customer_id)  -- Particionar por customer_id
SORTKEY (order_date);  -- Ordenar por fecha

-- ===== COPIAR DESDE S3 =====
COPY sales
FROM 's3://my-datalake/raw/sales/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftRole'
FORMAT AS PARQUET
DATEFORMAT 'auto';

-- Con particiones específicas
COPY sales
FROM 's3://my-datalake/raw/sales/year=2025/month=01/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftRole'
FORMAT AS PARQUET;

-- ===== UNLOAD A S3 (Export) =====
UNLOAD ('SELECT * FROM sales WHERE order_date >= ''2025-01-01''')
TO 's3://exports/sales/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftRole'
PARQUET
PARTITION BY (order_date)
PARALLEL ON;

-- ===== REDSHIFT SPECTRUM (Query S3 directamente) =====
-- Crear schema externo
CREATE EXTERNAL SCHEMA spectrum
FROM DATA CATALOG
DATABASE 'ecommerce'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftRole';

-- Query tabla externa en S3
SELECT
    customer_id,
    SUM(amount) as total_spent
FROM spectrum.raw_sales  -- Tabla en S3, no en Redshift
WHERE order_date >= '2025-01-01'
GROUP BY customer_id;

-- Join entre Redshift (local) y S3 (spectrum)
SELECT
    c.customer_name,
    SUM(s.amount) as total
FROM customers c  -- Tabla local en Redshift
JOIN spectrum.raw_sales s  -- Tabla en S3
    ON c.customer_id = s.customer_id
GROUP BY c.customer_name;

-- ===== VACUUM Y ANALYZE =====
-- Recuperar espacio y reordenar
VACUUM sales;

-- Actualizar estadísticas para query planner
ANALYZE sales;

-- ===== MATERIALIZAR VISTA =====
CREATE MATERIALIZED VIEW sales_daily AS
SELECT
    order_date,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount
FROM sales
GROUP BY order_date;

-- Refrescar vista materializada
REFRESH MATERIALIZED VIEW sales_daily;
```

**Python: Conexión a Redshift**

```python
import psycopg2
import pandas as pd

# Conexión
conn = psycopg2.connect(
    host='my-cluster.abc123.us-east-1.redshift.amazonaws.com',
    port=5439,
    dbname='analytics',
    user='admin',
    password='SecurePassword123'
)

# Query con Pandas
df = pd.read_sql("""
    SELECT
        order_date,
        SUM(amount) as daily_sales
    FROM sales
    WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY order_date
    ORDER BY order_date
""", conn)

# Bulk insert con COPY (más eficiente que INSERT)
cursor = conn.cursor()

# 1. Subir CSV a S3
df.to_csv('s3://temp-data/sales.csv', index=False)

# 2. COPY desde S3
cursor.execute("""
    COPY sales
    FROM 's3://temp-data/sales.csv'
    IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftRole'
    CSV
    IGNOREHEADER 1
""")

conn.commit()
conn.close()
```

### 4. Amazon Kinesis

**Servicios:**

| Servicio | Propósito | Uso |
|----------|-----------|-----|
| **Kinesis Data Streams** | Streaming de datos en tiempo real | Logs, clickstream, IoT |
| **Kinesis Data Firehose** | Carga a destinos (S3, Redshift) | ETL streaming simple |
| **Kinesis Data Analytics** | SQL sobre streams | Agregaciones real-time |

**Kinesis Data Streams (Python)**

```python
import boto3
import json
from datetime import datetime

kinesis = boto3.client('kinesis', region_name='us-east-1')

# ===== PRODUCER: Enviar eventos =====
def send_event_to_kinesis(stream_name: str, event: dict):
    """Enviar evento individual"""
    response = kinesis.put_record(
        StreamName=stream_name,
        Data=json.dumps(event),
        PartitionKey=event['user_id']  # Determina shard
    )
    print(f"Sent to shard {response['ShardId']}, sequence {response['SequenceNumber']}")

# Enviar evento
event = {
    'event_type': 'page_view',
    'user_id': 'user_12345',
    'page': '/products/laptop',
    'timestamp': datetime.utcnow().isoformat()
}

send_event_to_kinesis('clickstream', event)

# ===== BATCH PRODUCER =====
def send_batch_to_kinesis(stream_name: str, events: list):
    """Enviar múltiples eventos (hasta 500)"""
    records = [
        {
            'Data': json.dumps(event),
            'PartitionKey': event['user_id']
        }
        for event in events
    ]

    response = kinesis.put_records(
        StreamName=stream_name,
        Records=records
    )

    print(f"Sent {len(records)} events, {response['FailedRecordCount']} failed")

# ===== CONSUMER: Leer eventos =====
def consume_from_kinesis(stream_name: str):
    """Consumir eventos de stream"""
    # Obtener shard iterator
    shards = kinesis.describe_stream(StreamName=stream_name)['StreamDescription']['Shards']

    for shard in shards:
        shard_id = shard['ShardId']

        # Obtener iterator para leer desde el inicio
        shard_iterator_response = kinesis.get_shard_iterator(
            StreamName=stream_name,
            ShardId=shard_id,
            ShardIteratorType='TRIM_HORIZON'  # Desde el principio
        )

        shard_iterator = shard_iterator_response['ShardIterator']

        # Leer registros
        while shard_iterator:
            records_response = kinesis.get_records(
                ShardIterator=shard_iterator,
                Limit=100
            )

            records = records_response['Records']

            for record in records:
                data = json.loads(record['Data'])
                print(f"Received event: {data}")

                # Procesar evento
                process_event(data)

            # Siguiente batch
            shard_iterator = records_response.get('NextShardIterator')

            if not records:
                break  # No más datos

def process_event(event: dict):
    """Procesar evento individual"""
    if event['event_type'] == 'purchase':
        # Enviar a sistema de recomendaciones
        update_recommendations(event['user_id'], event['product_id'])
```

**Kinesis Firehose (Terraform)**

```hcl
# Delivery stream de Kinesis a S3
resource "aws_kinesis_firehose_delivery_stream" "extended_s3_stream" {
  name        = "clickstream-to-s3"
  destination = "extended_s3"

  extended_s3_configuration {
    role_arn   = aws_iam_role.firehose_role.arn
    bucket_arn = aws_s3_bucket.datalake.arn
    prefix     = "clickstream/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/"

    # Buffer settings
    buffer_size     = 5    # MB
    buffer_interval = 300  # segundos

    # Compresión
    compression_format = "GZIP"

    # Transformación con Lambda (opcional)
    processing_configuration {
      enabled = true

      processors {
        type = "Lambda"

        parameters {
          parameter_name  = "LambdaArn"
          parameter_value = aws_lambda_function.transform.arn
        }
      }
    }

    # Convertir a Parquet
    data_format_conversion_configuration {
      enabled = true

      input_format_configuration {
        deserializer {
          open_x_json_ser_de {}
        }
      }

      output_format_configuration {
        serializer {
          parquet_ser_de {}
        }
      }

      schema_configuration {
        database_name = "ecommerce"
        table_name    = "clickstream"
        role_arn      = aws_iam_role.firehose_role.arn
      }
    }
  }
}
```

### 5. Amazon Athena

**Características:**
- Query S3 usando SQL estándar
- Serverless (paga por query)
- Integrado con Glue Catalog
- Soporta Parquet, ORC, JSON, CSV

**SQL Examples:**

```sql
-- ===== CREAR TABLA EXTERNA =====
CREATE EXTERNAL TABLE IF NOT EXISTS sales (
    order_id STRING,
    customer_id STRING,
    amount DECIMAL(10,2),
    order_date DATE
)
PARTITIONED BY (
    year INT,
    month INT,
    day INT
)
STORED AS PARQUET
LOCATION 's3://my-datalake/processed/sales/';

-- Descubrir particiones automáticamente
MSCK REPAIR TABLE sales;

-- ===== QUERIES OPTIMIZADAS =====
-- Query particionada (rápido y barato)
SELECT
    customer_id,
    SUM(amount) as total
FROM sales
WHERE year = 2025
  AND month = 1
  AND day = 15
GROUP BY customer_id;

-- ===== CREATE TABLE AS SELECT (CTAS) =====
-- Crear tabla optimizada desde query
CREATE TABLE sales_aggregated
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    partitioned_by = ARRAY['year', 'month'],
    external_location = 's3://my-datalake/analytics/sales_agg/'
) AS
SELECT
    customer_id,
    DATE_TRUNC('month', order_date) as month_date,
    YEAR(order_date) as year,
    MONTH(order_date) as month,
    SUM(amount) as total_amount,
    COUNT(*) as order_count
FROM sales
GROUP BY customer_id, DATE_TRUNC('month', order_date), YEAR(order_date), MONTH(order_date);

-- ===== WINDOW FUNCTIONS =====
SELECT
    customer_id,
    order_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as cumulative_amount,
    RANK() OVER (
        PARTITION BY YEAR(order_date)
        ORDER BY amount DESC
    ) as yearly_rank
FROM sales
WHERE year = 2025;
```

**Python: Ejecutar queries de Athena**

```python
import boto3
import time
import pandas as pd

athena = boto3.client('athena', region_name='us-east-1')
s3 = boto3.client('s3')

def execute_athena_query(query: str, database: str = 'ecommerce') -> str:
    """Ejecutar query y retornar query execution ID"""
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={'Database': database},
        ResultConfiguration={
            'OutputLocation': 's3://athena-results/',
            'EncryptionConfiguration': {
                'EncryptionOption': 'SSE_S3'
            }
        }
    )
    return response['QueryExecutionId']

def wait_for_query_completion(query_execution_id: str) -> dict:
    """Esperar a que query termine"""
    while True:
        response = athena.get_query_execution(
            QueryExecutionId=query_execution_id
        )
        status = response['QueryExecution']['Status']['State']

        if status in ['SUCCEEDED', 'FAILED', 'CANCELLED']:
            return response

        time.sleep(1)

def get_query_results_as_df(query_execution_id: str) -> pd.DataFrame:
    """Obtener resultados como DataFrame"""
    result = athena.get_query_results(QueryExecutionId=query_execution_id)

    # Extraer columnas
    columns = [col['Label'] for col in result['ResultSet']['ResultSetMetadata']['ColumnInfo']]

    # Extraer filas (skip header)
    rows = []
    for row in result['ResultSet']['Rows'][1:]:
        rows.append([field.get('VarCharValue', None) for field in row['Data']])

    return pd.DataFrame(rows, columns=columns)

# Uso completo
query = """
SELECT
    customer_id,
    SUM(amount) as total
FROM sales
WHERE year = 2025 AND month = 1
GROUP BY customer_id
ORDER BY total DESC
LIMIT 100
"""

execution_id = execute_athena_query(query)
result = wait_for_query_completion(execution_id)

if result['QueryExecution']['Status']['State'] == 'SUCCEEDED':
    df = get_query_results_as_df(execution_id)
    print(df.head())
else:
    error = result['QueryExecution']['Status']['StateChangeReason']
    print(f"Query failed: {error}")
```

### 6. Amazon EMR (Elastic MapReduce)

**Características:**
- Managed Hadoop/Spark cluster
- Auto-scaling
- Spot instances para reducir costos
- Integración con S3, Glue, Redshift

**Arquitectura EMR:**

```
┌──────────────────────────────────────┐
│          EMR CLUSTER                 │
├──────────────────────────────────────┤
│                                      │
│  Master Node                         │
│  └─ Resource Manager (YARN)          │
│  └─ NameNode (HDFS)                  │
│  └─ Spark Master                     │
│                                      │
│  Core Nodes (2-20+)                  │
│  └─ HDFS storage                     │
│  └─ Task execution                   │
│                                      │
│  Task Nodes (optional, 0-100+)       │
│  └─ Solo task execution              │
│  └─ Puede usar Spot instances        │
│                                      │
│  Applications:                       │
│  - Spark, Hive, Presto, Flink        │
│  - JupyterHub, Zeppelin              │
└──────────────────────────────────────┘
```

**Lanzar EMR Cluster (AWS CLI)**

```bash
aws emr create-cluster \
  --name "Spark ETL Cluster" \
  --release-label emr-6.15.0 \
  --applications Name=Spark Name=Hive \
  --ec2-attributes KeyName=my-key,SubnetId=subnet-12345 \
  --instance-groups \
    InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m5.xlarge \
    InstanceGroupType=CORE,InstanceCount=2,InstanceType=m5.2xlarge \
    InstanceGroupType=TASK,InstanceCount=4,InstanceType=m5.xlarge,BidPrice=0.10 \
  --use-default-roles \
  --log-uri s3://emr-logs/ \
  --bootstrap-actions Path=s3://my-scripts/bootstrap.sh \
  --steps Type=Spark,Name="ETL Job",ActionOnFailure=CONTINUE,Args=[--deploy-mode,cluster,--master,yarn,s3://scripts/etl.py] \
  --auto-terminate
```

**PySpark Job en EMR:**

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, avg, count, window

# Inicializar Spark (EMR configura automáticamente)
spark = SparkSession.builder \
    .appName("Sales ETL") \
    .getOrCreate()

# Configuraciones para S3 optimizado
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")

# ===== LEER DESDE S3 =====
df = spark.read.parquet("s3://my-datalake/raw/sales/")

# ===== TRANSFORMACIONES =====
# Filtrar y limpiar
cleaned = df.filter(
    (col("amount") > 0) &
    (col("customer_id").isNotNull())
)

# Agregaciones
daily_sales = cleaned.groupBy("order_date").agg(
    sum("amount").alias("total_sales"),
    count("*").alias("order_count"),
    avg("amount").alias("avg_order_value")
)

# ===== ESCRIBIR A S3 PARTICIONADO =====
daily_sales.write.mode("overwrite") \
    .partitionBy("order_date") \
    .parquet("s3://my-datalake/analytics/daily_sales/")

# ===== ESCRIBIR A REDSHIFT =====
daily_sales.write \
    .format("io.github.spark_redshift_community.spark.redshift") \
    .option("url", "jdbc:redshift://cluster:5439/analytics?user=admin&password=pass") \
    .option("dbtable", "daily_sales") \
    .option("tempdir", "s3://temp-data/") \
    .option("aws_iam_role", "arn:aws:iam::123456789:role/RedshiftRole") \
    .mode("overwrite") \
    .save()

spark.stop()
```

---

## Google Cloud Platform (GCP)

### Arquitectura GCP Data Platform

```
┌────────────────────────────────────────────────────────┐
│              GCP DATA PLATFORM                         │
├────────────────────────────────────────────────────────┤
│                                                        │
│  INGESTION           PROCESSING           STORAGE      │
│  ┌─────────┐        ┌─────────┐         ┌──────────┐   │
│  │  Pub/   │───────▶│ Data    │────────▶│ BigQuery │   │
│  │  Sub    │        │  flow   │         │  (DWH)   │   │
│  └─────────┘        └─────────┘         └──────────┘   │
│       │                  │                     │       │
│       │                  │                     │       │
│       ▼                  ▼                     ▼       │
│  ┌─────────┐        ┌─────────┐         ┌──────────┐   │
│  │ Cloud   │        │ Data    │         │  Cloud   │   │
│  │ Funct   │        │  proc   │         │ Storage  │   │
│  └─────────┘        └─────────┘         └──────────┘   │
│                                                │       │
│  ORCHESTRATION                                 │       │
│  ┌─────────┐                                   │       │
│  │ Cloud   │                                   │       │
│  │Composer │◀──────────────────────────────────┘       │
│  │(Airflow)│                                           │
│  └─────────┘                                           │
└────────────────────────────────────────────────────────┘
```

### BigQuery

**Características:**
- Data warehouse serverless
- Separa storage y compute
- Pricing: $5/TB scanned, $20/TB storage
- ML integrado (BQML)

**SQL Examples:**

```sql
-- ===== CREAR TABLA PARTICIONADA =====
CREATE TABLE `project.dataset.sales`
(
    order_id STRING,
    customer_id STRING,
    amount NUMERIC(10,2),
    order_date DATE
)
PARTITION BY order_date
CLUSTER BY customer_id
OPTIONS(
    description="Sales transactions",
    require_partition_filter=true  -- Fuerza filtrar por partition
);

-- ===== CARGAR DESDE GCS =====
LOAD DATA INTO `project.dataset.sales`
FROM FILES (
    format = 'PARQUET',
    uris = ['gs://my-bucket/sales/*.parquet']
);

-- ===== QUERY CON CACHE =====
SELECT
    customer_id,
    SUM(amount) as total
FROM `project.dataset.sales`
WHERE order_date >= '2025-01-01'  -- Usa partición
GROUP BY customer_id;

-- ===== EXPORTAR A GCS =====
EXPORT DATA OPTIONS(
    uri='gs://exports/sales/*.parquet',
    format='PARQUET',
    overwrite=true
) AS
SELECT * FROM `project.dataset.sales`
WHERE order_date >= '2025-01-01';

-- ===== BIGQUERY ML (Machine Learning) =====
-- Crear modelo de predicción
CREATE OR REPLACE MODEL `project.dataset.churn_model`
OPTIONS(
    model_type='LOGISTIC_REG',
    input_label_cols=['churned']
) AS
SELECT
    customer_age,
    total_purchases,
    avg_order_value,
    days_since_last_order,
    churned
FROM `project.dataset.customer_features`;

-- Predecir con el modelo
SELECT
    customer_id,
    predicted_churned,
    predicted_churned_probs[OFFSET(0)].prob as churn_probability
FROM ML.PREDICT(
    MODEL `project.dataset.churn_model`,
    (SELECT * FROM `project.dataset.new_customers`)
);

-- ===== SCHEDULED QUERIES =====
-- BigQuery puede ejecutar queries automáticamente
CREATE OR REPLACE TABLE `project.dataset.daily_summary`
PARTITION BY summary_date
AS
SELECT
    CURRENT_DATE() as summary_date,
    COUNT(*) as order_count,
    SUM(amount) as total_sales
FROM `project.dataset.sales`
WHERE order_date = CURRENT_DATE();
-- Programar desde UI: daily @ 2 AM
```

**Python: BigQuery Client**

```python
from google.cloud import bigquery
import pandas as pd

# Cliente
client = bigquery.Client(project='my-project')

# ===== QUERY A DATAFRAME =====
query = """
SELECT
    customer_id,
    SUM(amount) as total
FROM `project.dataset.sales`
WHERE order_date >= '2025-01-01'
GROUP BY customer_id
ORDER BY total DESC
LIMIT 1000
"""

df = client.query(query).to_dataframe()

# ===== ESCRIBIR DATAFRAME A BIGQUERY =====
table_id = 'project.dataset.customers'

job_config = bigquery.LoadJobConfig(
    write_disposition="WRITE_TRUNCATE",  # Sobreescribir
    schema=[
        bigquery.SchemaField("customer_id", "STRING"),
        bigquery.SchemaField("name", "STRING"),
        bigquery.SchemaField("email", "STRING"),
    ]
)

job = client.load_table_from_dataframe(df, table_id, job_config=job_config)
job.result()  # Esperar a que termine

# ===== QUERY GRANDE (usa Cloud Storage como staging) =====
job_config = bigquery.QueryJobConfig(
    destination='project.dataset.temp_results',
    write_disposition='WRITE_TRUNCATE',
    use_query_cache=False
)

query_job = client.query(large_query, job_config=job_config)
query_job.result()

# Leer resultados por chunks
for row in client.list_rows('project.dataset.temp_results', max_results=100000):
    process_row(row)
```

### Cloud Dataflow (Apache Beam)

**Características:**
- Procesamiento batch y streaming unificado
- Auto-scaling
- Apache Beam SDK (Python, Java)

**Beam Pipeline (Python)**

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
from apache_beam.io.gcp.bigquery import WriteToBigQuery

# Opciones de Dataflow
options = PipelineOptions(
    project='my-project',
    runner='DataflowRunner',  # 'DirectRunner' para local
    region='us-central1',
    temp_location='gs://temp-bucket/temp',
    staging_location='gs://temp-bucket/staging',
    num_workers=5,
    max_num_workers=10,
    autoscaling_algorithm='THROUGHPUT_BASED'
)

# Pipeline
with beam.Pipeline(options=options) as pipeline:

    # ===== LEER DESDE CLOUD STORAGE =====
    sales = (
        pipeline
        | 'Read from GCS' >> beam.io.ReadFromText('gs://data/sales/*.json')
        | 'Parse JSON' >> beam.Map(json.loads)
    )

    # ===== TRANSFORMACIONES =====
    # Filtrar
    valid_sales = (
        sales
        | 'Filter valid' >> beam.Filter(lambda x: x['amount'] > 0)
    )

    # Map (transformar cada elemento)
    enriched = (
        valid_sales
        | 'Add fields' >> beam.Map(lambda x: {
            **x,
            'revenue': x['amount'] * x['quantity'],
            'processed_at': datetime.utcnow().isoformat()
        })
    )

    # GroupBy y Aggregate
    daily_totals = (
        enriched
        | 'Extract date' >> beam.Map(lambda x: (x['order_date'], x['revenue']))
        | 'Group by date' >> beam.GroupByKey()
        | 'Sum revenue' >> beam.CombineValues(sum)
        | 'Format output' >> beam.Map(lambda x: {'date': x[0], 'total': x[1]})
    )

    # ===== ESCRIBIR A BIGQUERY =====
    daily_totals | 'Write to BQ' >> WriteToBigQuery(
        table='project:dataset.daily_sales',
        schema='date:DATE,total:NUMERIC',
        write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
        create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED
    )

    # ===== ESCRIBIR A CLOUD STORAGE =====
    enriched | 'Write to GCS' >> beam.io.WriteToParquet(
        'gs://processed/sales',
        schema=sales_schema,
        file_name_suffix='.parquet'
    )
```

**Streaming Pipeline (Pub/Sub → BigQuery)**

```python
import apache_beam as beam
from apache_beam.transforms import window

# Pipeline streaming
with beam.Pipeline(options=streaming_options) as pipeline:

    # Leer desde Pub/Sub
    events = (
        pipeline
        | 'Read from Pub/Sub' >> beam.io.ReadFromPubSub(
            subscription='projects/my-project/subscriptions/events-sub'
        )
        | 'Decode' >> beam.Map(lambda x: json.loads(x.decode('utf-8')))
    )

    # Window de 1 minuto
    windowed = (
        events
        | 'Window' >> beam.WindowInto(window.FixedWindows(60))  # 60 segundos
    )

    # Agregaciones por ventana
    aggregated = (
        windowed
        | 'Extract user' >> beam.Map(lambda x: (x['user_id'], 1))
        | 'Count per user' >> beam.CombinePerKey(sum)
        | 'Format' >> beam.Map(lambda x: {
            'user_id': x[0],
            'event_count': x[1],
            'window_start': window.start.to_utc_datetime().isoformat()
        })
    )

    # Escribir a BigQuery
    aggregated | 'Write to BQ' >> WriteToBigQuery(
        'project:dataset.user_activity',
        schema='user_id:STRING,event_count:INTEGER,window_start:TIMESTAMP',
        write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND
    )
```

---

## Azure Data Platform

### Arquitectura Azure

```
┌────────────────────────────────────────────────────┐
│           AZURE DATA PLATFORM                      │
├────────────────────────────────────────────────────┤
│                                                    │
│  INGESTION          PROCESSING         STORAGE     │
│  ┌──────────┐      ┌──────────┐      ┌─────────┐   │
│  │  Event   │─────▶│  Azure   │─────▶│ Synapse │   │
│  │   Hubs   │      │Databricks│      │  (DWH)  │   │
│  └──────────┘      └──────────┘      └─────────┘   │
│       │                 │                   │      │
│       │                 │                   │      │
│       ▼                 ▼                   ▼      │
│  ┌──────────┐      ┌──────────┐      ┌─────────┐   │
│  │  Azure   │      │   Data   │      │  ADLS   │   │
│  │Functions │      │ Factory  │      │  Gen2   │   │
│  └──────────┘      └──────────┘      └─────────┘   │
│                                            │       │
│  ORCHESTRATION                             │       │
│  ┌──────────┐                              │       │
│  │   Data   │◀─────────────────────────────┘       │
│  │ Factory  │                                      │
│  └──────────┘                                      │
└────────────────────────────────────────────────────┘
```

### Azure Synapse Analytics

**Características:**
- Unified analytics (DWH + Spark + Pipelines)
- Serverless SQL pool
- Dedicated SQL pool (antes SQL Data Warehouse)
- Integración con Power BI

```sql
-- ===== SYNAPSE SQL (Serverless) =====
-- Query ADLS directamente
SELECT
    customer_id,
    SUM(CAST(JSON_VALUE(data, '$.amount') AS DECIMAL)) as total
FROM OPENROWSET(
    BULK 'https://mystorageaccount.dfs.core.windows.net/sales/**/*.parquet',
    FORMAT = 'PARQUET'
) AS sales
CROSS APPLY OPENJSON(sales.jsonColumn) WITH (
    customer_id VARCHAR(50),
    amount DECIMAL(10,2)
) AS data
GROUP BY customer_id;

-- ===== DEDICATED SQL POOL =====
-- Similar a Redshift
CREATE TABLE sales (
    order_id VARCHAR(50),
    customer_id VARCHAR(50),
    amount DECIMAL(10,2),
    order_date DATE
)
WITH (
    DISTRIBUTION = HASH(customer_id),
    CLUSTERED COLUMNSTORE INDEX
);

-- COPY desde ADLS
COPY INTO sales
FROM 'https://mystorageaccount.dfs.core.windows.net/sales/*.parquet'
WITH (
    FILE_TYPE = 'PARQUET',
    CREDENTIAL = (IDENTITY = 'Managed Identity')
);
```

### Azure Data Factory

**Características:**
- Servicio ETL visual
- 90+ conectores
- Data flows (transformaciones sin código)
- Integration con Synapse Pipelines

**Pipeline JSON Example:**

```json
{
  "name": "CopySalesToSynapse",
  "properties": {
    "activities": [
      {
        "name": "CopyFromBlobToSynapse",
        "type": "Copy",
        "inputs": [
          {
            "referenceName": "SourceBlobDataset",
            "type": "DatasetReference"
          }
        ],
        "outputs": [
          {
            "referenceName": "SynapseSalesTable",
            "type": "DatasetReference"
          }
        ],
        "typeProperties": {
          "source": {
            "type": "ParquetSource"
          },
          "sink": {
            "type": "SqlDWSink",
            "preCopyScript": "TRUNCATE TABLE sales"
          },
          "enableStaging": true,
          "stagingSettings": {
            "linkedServiceName": "AzureBlobStorage"
          }
        }
      }
    ],
    "triggers": [
      {
        "name": "DailyTrigger",
        "type": "ScheduleTrigger",
        "typeProperties": {
          "recurrence": {
            "frequency": "Day",
            "interval": 1,
            "startTime": "2025-01-01T02:00:00Z",
            "timeZone": "UTC"
          }
        }
      }
    ]
  }
}
```

---

## Comparación de Servicios

### Data Warehouses

| Feature | Redshift | BigQuery | Synapse (Dedicated) |
|---------|----------|----------|---------------------|
| **Pricing model** | Por hora de cluster | Por TB scanned | Por DWU (compute) |
| **Scaling** | Manual resize | Auto (serverless) | Manual |
| **Storage/Compute** | Acoplados | Separados | Separados |
| **Performance** | Excelente | Excelente | Muy bueno |
| **ML integrado** | Redshift ML | BQML | Synapse ML |
| **Costo (TB/mes)** | ~$1,000 (dc2.large) | $20 storage + query | ~$1,200 (DW500c) |
| **Mejor para** | AWS ecosystem | Analytics ad-hoc | Microsoft stack |

### ETL Services

| Feature | Glue | Dataflow | Data Factory |
|---------|------|----------|--------------|
| **Pricing** | Por DPU-hour | Por vCPU-hour | Por activity run |
| **Learning curve** | Media (Spark) | Alta (Beam) | Baja (visual) |
| **Streaming** | Limited | Excelente | Limited |
| **Transformations** | Spark (código) | Beam (código) | Visual + código |
| **Mejor para** | AWS batch ETL | Streaming complex | ETL visual |

---

## Cost Optimization

### Estrategias Generales

```python
# ===== 1. PARTITIONING =====
# Mal: Escanea 1 TB
SELECT * FROM sales WHERE order_date = '2025-01-15'

# Bien: Escanea solo 1 GB (partición)
# Tabla particionada por order_date
SELECT * FROM sales WHERE partition_date = '2025-01-15'

# ===== 2. COLUMNAR FORMATS =====
# CSV: 10 GB
# Parquet (comprimido): 1 GB → 90% ahorro

# ===== 3. LIFECYCLE POLICIES =====
# Mover a storage barato después de 90 días
# S3: Standard → Glacier
# GCS: Standard → Nearline → Coldline
# ADLS: Hot → Cool → Archive

# ===== 4. SPOT/PREEMPTIBLE INSTANCES =====
# EMR Task nodes: hasta 90% descuento
# Dataflow: Preemptible workers
# Azure: Spot VMs

# ===== 5. RESERVED CAPACITY =====
# Redshift: 1-year RI → 40% descuento
# BigQuery: Flat-rate pricing (> 100 TB/mes)
# Synapse: Reserved capacity

# ===== 6. QUERY OPTIMIZATION =====
# Limitar columnas
SELECT customer_id, amount  -- No SELECT *
FROM sales

# Filtrar temprano
WHERE order_date >= '2025-01-01'  -- Antes de JOINs

# Usar aproximaciones
SELECT APPROX_COUNT_DISTINCT(customer_id)  -- Más rápido que COUNT(DISTINCT)

# ===== 7. AUTO-PAUSE =====
# Redshift Serverless: pausa después de 5 min inactividad
# Synapse Serverless: solo paga cuando ejecuta
# BigQuery: sin clusters, siempre "pausado"
```

### Cost Monitoring

```python
# AWS Cost Explorer API
import boto3

ce = boto3.client('ce')

# Obtener costos de servicios de datos
response = ce.get_cost_and_usage(
    TimePeriod={
        'Start': '2025-01-01',
        'End': '2025-01-31'
    },
    Granularity='DAILY',
    Metrics=['UnblendedCost'],
    GroupBy=[
        {
            'Type': 'SERVICE',
            'Key': 'SERVICE'
        }
    ],
    Filter={
        'Dimensions': {
            'Key': 'SERVICE',
            'Values': ['Amazon Redshift', 'AWS Glue', 'Amazon EMR', 'Amazon S3']
        }
    }
)

# Alertas de presupuesto
budgets = boto3.client('budgets')

budgets.create_budget(
    AccountId='123456789',
    Budget={
        'BudgetName': 'Data Platform Monthly',
        'BudgetLimit': {
            'Amount': '10000',
            'Unit': 'USD'
        },
        'TimeUnit': 'MONTHLY',
        'BudgetType': 'COST'
    },
    NotificationsWithSubscribers=[
        {
            'Notification': {
                'NotificationType': 'ACTUAL',
                'ComparisonOperator': 'GREATER_THAN',
                'Threshold': 80.0,  # 80% del presupuesto
                'ThresholdType': 'PERCENTAGE'
            },
            'Subscribers': [
                {
                    'SubscriptionType': 'EMAIL',
                    'Address': 'data-team@company.com'
                }
            ]
        }
    ]
)
```

---

## 🎯 Preguntas de Entrevista

**P: ¿Cuándo usarías Redshift vs Athena?**

R:
- **Redshift**: Queries frecuentes, baja latencia (<1s), joins complejos, datasets <10 TB
- **Athena**: Queries ad-hoc, puede tolerar latencia (5-30s), datos en S3, costo por query

**P: Diseña una arquitectura data lake en AWS para 100 TB de datos**

R:
```
1. Ingestion: Kinesis Firehose → S3 (raw/)
2. Cataloging: Glue Crawlers → Glue Catalog
3. Processing: EMR Spark jobs → S3 (processed/)
4. Serving:
   - Ad-hoc queries: Athena
   - BI dashboards: Redshift Spectrum + QuickSight
5. Governance: Lake Formation para permisos
6. Cost: Lifecycle policy (raw → Glacier después de 90d)
```

**P: ¿Cómo optimizarías costos de BigQuery si escaneas 500 TB/mes?**

R:
1. **Partitioning**: Reducir scan con filtros de partición
2. **Clustering**: Ordenar datos para filtros frecuentes
3. **Materialized views**: Pre-computar queries caras
4. **Flat-rate pricing**: $40k/mes fijo (vs $2.5M pay-per-query)
5. **BI Engine**: Cachear queries de dashboards

---

**Siguiente:** [08. Optimización y Performance](08_optimizacion_performance.md)
