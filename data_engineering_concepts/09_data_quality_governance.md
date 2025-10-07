# 9. Data Quality y Governance

## 📋 Tabla de Contenidos
- [Fundamentos de Data Quality](#fundamentos-de-data-quality)
- [Data Validation y Testing](#data-validation-y-testing)
- [Data Governance](#data-governance)
- [Data Lineage](#data-lineage)
- [Metadata Management](#metadata-management)
- [Data Catalogs](#data-catalogs)
- [Compliance y Regulaciones](#compliance-y-regulaciones)
- [Herramientas y Frameworks](#herramientas-y-frameworks)

---

## Fundamentos de Data Quality

### Dimensiones de Data Quality

```
┌─────────────────────────────────────────────────────┐
│          6 DIMENSIONES DE DATA QUALITY              │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. ACCURACY (Precisión)                            │
│     ¿Los datos son correctos?                       │
│     Ejemplo: email='user@example..com' ❌           │
│                                                     │
│  2. COMPLETENESS (Completitud)                      │
│     ¿Hay valores faltantes?                         │
│     Ejemplo: customer_name=NULL ❌                  │
│                                                     │
│  3. CONSISTENCY (Consistencia)                      │
│     ¿Datos contradictorios entre sistemas?          │
│     Ejemplo: age=25 pero birthdate=1980 ❌          │
│                                                     │
│  4. TIMELINESS (Puntualidad)                        │
│     ¿Datos actualizados?                            │
│     Ejemplo: dashboard muestra datos de ayer ❌     │
│                                                     │
│  5. VALIDITY (Validez)                              │
│     ¿Cumplen reglas de negocio?                     │
│     Ejemplo: amount=-100 para una venta ❌          │
│                                                     │
│  6. UNIQUENESS (Unicidad)                           │
│     ¿Hay duplicados?                                │
│     Ejemplo: mismo customer_id 3 veces ❌           │
└─────────────────────────────────────────────────────┘
```

### Data Quality Metrics

```python
import pandas as pd
import numpy as np

def calculate_dq_metrics(df: pd.DataFrame, unique_cols: list = None) -> dict:
    """
    Calcular métricas de data quality para un DataFrame
    """
    metrics = {}

    # 1. COMPLETENESS
    total_cells = df.size
    missing_cells = df.isnull().sum().sum()
    metrics['completeness_pct'] = 100 * (1 - missing_cells / total_cells)

    # Por columna
    metrics['completeness_by_col'] = {}
    for col in df.columns:
        non_null = df[col].notna().sum()
        metrics['completeness_by_col'][col] = 100 * non_null / len(df)

    # 2. UNIQUENESS
    if unique_cols:
        duplicates = df.duplicated(subset=unique_cols).sum()
        metrics['uniqueness_pct'] = 100 * (1 - duplicates / len(df))
        metrics['duplicate_count'] = duplicates

    # 3. VALIDITY (ejemplo: rangos numéricos)
    metrics['validity_checks'] = {}
    for col in df.select_dtypes(include=[np.number]).columns:
        # Valores negativos en columnas que deberían ser positivas
        if col in ['amount', 'price', 'quantity']:
            invalid_count = (df[col] < 0).sum()
            metrics['validity_checks'][f'{col}_negative'] = {
                'invalid_count': invalid_count,
                'invalid_pct': 100 * invalid_count / len(df)
            }

    # 4. TIMELINESS
    if 'updated_at' in df.columns:
        from datetime import datetime, timedelta
        now = datetime.now()
        stale_threshold = now - timedelta(days=7)

        df['updated_at'] = pd.to_datetime(df['updated_at'])
        stale_count = (df['updated_at'] < stale_threshold).sum()
        metrics['timeliness'] = {
            'stale_count': stale_count,
            'stale_pct': 100 * stale_count / len(df)
        }

    return metrics

# Uso
df = pd.read_csv('customers.csv')
metrics = calculate_dq_metrics(df, unique_cols=['customer_id'])

print(f"Completeness: {metrics['completeness_pct']:.2f}%")
print(f"Uniqueness: {metrics['uniqueness_pct']:.2f}%")
print(f"Duplicates: {metrics['duplicate_count']}")
```

---

## Data Validation y Testing

### 1. Great Expectations

**Características:**
- Framework de validación de datos en Python
- Expectativas reutilizables
- Generación automática de documentación
- Integración con Airflow, dbt, Spark

**Instalación y Setup:**

```bash
pip install great-expectations
great_expectations init  # Crear proyecto
```

**Ejemplo Completo:**

```python
import great_expectations as ge
from great_expectations.dataset import PandasDataset
import pandas as pd

# ===== 1. CREAR EXPECTATIONS =====
df = pd.read_csv('sales.csv')
ge_df = ge.from_pandas(df)

# Expectativas básicas
expectations = [
    # Columnas requeridas
    ge_df.expect_table_columns_to_match_ordered_list([
        'order_id', 'customer_id', 'amount', 'order_date'
    ]),

    # No nulls en columnas críticas
    ge_df.expect_column_values_to_not_be_null('order_id'),
    ge_df.expect_column_values_to_not_be_null('customer_id'),

    # Unicidad
    ge_df.expect_column_values_to_be_unique('order_id'),

    # Rangos válidos
    ge_df.expect_column_values_to_be_between('amount', min_value=0, max_value=100000),

    # Valores en set
    ge_df.expect_column_values_to_be_in_set('status', ['pending', 'completed', 'cancelled']),

    # Formato de email
    ge_df.expect_column_values_to_match_regex('email', r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'),

    # Fecha en rango
    ge_df.expect_column_values_to_be_between(
        'order_date',
        min_value='2020-01-01',
        max_value='2025-12-31'
    ),

    # Suma de columna
    ge_df.expect_column_sum_to_be_between('amount', min_value=1000000),

    # Conteo de filas
    ge_df.expect_table_row_count_to_be_between(min_value=100, max_value=1000000)
]

# ===== 2. GUARDAR EXPECTATIONS SUITE =====
from great_expectations.core.batch import RuntimeBatchRequest

context = ge.get_context()

# Crear expectation suite
suite = context.create_expectation_suite(
    expectation_suite_name="sales_validation",
    overwrite_existing=True
)

# Agregar expectations
for expectation in expectations:
    suite.add_expectation(expectation)

context.save_expectation_suite(suite)

# ===== 3. VALIDAR DATOS =====
def validate_sales_data(df: pd.DataFrame) -> dict:
    """
    Validar DataFrame contra expectation suite
    """
    context = ge.get_context()

    # Crear batch
    batch = context.get_validator(
        batch_request=RuntimeBatchRequest(
            datasource_name="pandas_datasource",
            data_connector_name="default_runtime_data_connector_name",
            data_asset_name="sales",
            runtime_parameters={"batch_data": df},
            batch_identifiers={"default_identifier_name": "sales_batch"}
        ),
        expectation_suite_name="sales_validation"
    )

    # Validar
    results = batch.validate()

    return {
        'success': results.success,
        'total_expectations': len(results.results),
        'successful_expectations': sum([r.success for r in results.results]),
        'failed_expectations': [
            {
                'expectation': r.expectation_config.expectation_type,
                'column': r.expectation_config.kwargs.get('column'),
                'details': r.result
            }
            for r in results.results if not r.success
        ]
    }

# Validar nuevo batch
new_df = pd.read_csv('sales_today.csv')
validation_results = validate_sales_data(new_df)

if not validation_results['success']:
    print("❌ Validación falló:")
    for failure in validation_results['failed_expectations']:
        print(f"  - {failure['expectation']} en columna {failure['column']}")
        print(f"    {failure['details']}")
else:
    print("✅ Validación exitosa")

# ===== 4. INTEGRACIÓN CON AIRFLOW =====
from airflow import DAG
from airflow.operators.python import PythonOperator
from great_expectations_provider.operators.great_expectations import GreatExpectationsOperator

dag = DAG('sales_etl_with_validation', ...)

validate_task = GreatExpectationsOperator(
    task_id='validate_sales',
    expectation_suite_name='sales_validation',
    batch_kwargs={
        'path': 's3://data/sales/{{ ds }}.csv',
        'datasource': 's3_datasource'
    },
    fail_task_on_validation_failure=True,  # Fallar task si validación falla
    dag=dag
)

load_task = PythonOperator(...)

validate_task >> load_task  # Solo carga si validación pasa
```

### 2. Custom Validation Functions

```python
from typing import List, Dict, Any
import pandas as pd

class DataValidator:
    """
    Validador custom de datos con reglas de negocio
    """

    def __init__(self):
        self.errors = []
        self.warnings = []

    def validate_sales_data(self, df: pd.DataFrame) -> bool:
        """
        Validaciones custom para datos de ventas
        """
        self.errors = []
        self.warnings = []

        # 1. Schema validation
        required_columns = ['order_id', 'customer_id', 'amount', 'order_date']
        missing_cols = set(required_columns) - set(df.columns)
        if missing_cols:
            self.errors.append(f"Columnas faltantes: {missing_cols}")
            return False

        # 2. Business rules
        # Regla: amount debe ser positivo
        negative_amounts = df[df['amount'] <= 0]
        if len(negative_amounts) > 0:
            self.errors.append(
                f"{len(negative_amounts)} órdenes con amount <= 0: {negative_amounts['order_id'].tolist()[:5]}"
            )

        # Regla: order_date no puede ser futuro
        df['order_date'] = pd.to_datetime(df['order_date'])
        future_dates = df[df['order_date'] > pd.Timestamp.now()]
        if len(future_dates) > 0:
            self.errors.append(
                f"{len(future_dates)} órdenes con fecha futura: {future_dates['order_id'].tolist()[:5]}"
            )

        # 3. Referential integrity
        # Regla: customer_id debe existir en tabla customers
        # (asumiendo que tenemos acceso a tabla customers)
        # invalid_customers = df[~df['customer_id'].isin(valid_customer_ids)]

        # 4. Data consistency
        # Regla: customer_email debe matchear formato
        if 'email' in df.columns:
            invalid_emails = df[~df['email'].str.match(r'^[^@]+@[^@]+\.[^@]+$')]
            if len(invalid_emails) > 0:
                self.warnings.append(
                    f"{len(invalid_emails)} emails con formato inválido"
                )

        # 5. Statistical validation
        # Regla: amount no debe ser outlier extremo (> 5 std)
        mean_amount = df['amount'].mean()
        std_amount = df['amount'].std()
        outliers = df[df['amount'] > mean_amount + 5 * std_amount]
        if len(outliers) > 0:
            self.warnings.append(
                f"{len(outliers)} órdenes con amount outlier: max={outliers['amount'].max()}"
            )

        # 6. Duplicate check
        duplicates = df[df.duplicated(subset=['order_id'])]
        if len(duplicates) > 0:
            self.errors.append(
                f"{len(duplicates)} order_id duplicados"
            )

        return len(self.errors) == 0

    def get_report(self) -> Dict[str, Any]:
        """
        Generar reporte de validación
        """
        return {
            'is_valid': len(self.errors) == 0,
            'error_count': len(self.errors),
            'warning_count': len(self.warnings),
            'errors': self.errors,
            'warnings': self.warnings
        }

# Uso
validator = DataValidator()
df = pd.read_csv('sales.csv')

if validator.validate_sales_data(df):
    print("✅ Datos válidos")
    # Proceder con ETL
else:
    print("❌ Datos inválidos:")
    report = validator.get_report()
    for error in report['errors']:
        print(f"  - {error}")
```

### 3. SQL Data Quality Checks

```sql
-- ===== VALIDACIONES EN SQL =====

-- 1. COMPLETENESS: Detectar nulls
SELECT
    'customer_id' as column_name,
    COUNT(*) as total_rows,
    COUNT(customer_id) as non_null_rows,
    COUNT(*) - COUNT(customer_id) as null_count,
    100.0 * COUNT(customer_id) / COUNT(*) as completeness_pct
FROM customers
UNION ALL
SELECT
    'email',
    COUNT(*),
    COUNT(email),
    COUNT(*) - COUNT(email),
    100.0 * COUNT(email) / COUNT(*)
FROM customers;

-- 2. UNIQUENESS: Detectar duplicados
SELECT
    customer_id,
    COUNT(*) as duplicate_count
FROM customers
GROUP BY customer_id
HAVING COUNT(*) > 1;

-- 3. VALIDITY: Detectar valores fuera de rango
SELECT
    COUNT(*) as invalid_amount_count
FROM orders
WHERE amount < 0 OR amount > 1000000;

-- 4. REFERENTIAL INTEGRITY
-- Órdenes con customer_id que no existe
SELECT o.order_id, o.customer_id
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL;

-- 5. CONSISTENCY: Cross-table checks
-- Verificar que total en orders_summary = SUM(orders)
SELECT
    os.summary_date,
    os.total_amount as summary_total,
    COALESCE(SUM(o.amount), 0) as actual_total,
    os.total_amount - COALESCE(SUM(o.amount), 0) as difference
FROM orders_summary os
LEFT JOIN orders o ON DATE(o.order_date) = os.summary_date
GROUP BY os.summary_date, os.total_amount
HAVING ABS(os.total_amount - COALESCE(SUM(o.amount), 0)) > 0.01;

-- 6. TIMELINESS: Detectar datos stale
SELECT
    COUNT(*) as stale_records
FROM customers
WHERE updated_at < CURRENT_DATE - INTERVAL '30 days';

-- ===== CREAR TABLA DE MÉTRICAS DQ =====
CREATE TABLE data_quality_metrics (
    metric_date DATE,
    table_name VARCHAR(100),
    metric_name VARCHAR(100),
    metric_value DECIMAL(10,2),
    status VARCHAR(20),  -- 'PASS' or 'FAIL'
    details TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insertar métricas diarias
INSERT INTO data_quality_metrics
SELECT
    CURRENT_DATE as metric_date,
    'customers' as table_name,
    'completeness_email' as metric_name,
    100.0 * COUNT(email) / COUNT(*) as metric_value,
    CASE
        WHEN 100.0 * COUNT(email) / COUNT(*) >= 95 THEN 'PASS'
        ELSE 'FAIL'
    END as status,
    CONCAT(COUNT(*) - COUNT(email), ' null values') as details,
    CURRENT_TIMESTAMP
FROM customers;

-- Dashboard de DQ metrics
SELECT
    metric_date,
    table_name,
    metric_name,
    metric_value,
    status
FROM data_quality_metrics
WHERE metric_date >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY metric_date DESC, table_name, metric_name;
```

---

## Data Governance

### Principios de Data Governance

```
┌────────────────────────────────────────────────────┐
│           DATA GOVERNANCE FRAMEWORK                │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. DATA OWNERSHIP                                 │
│     • Data Stewards por domain                     │
│     • Responsabilidad clara                        │
│                                                    │
│  2. DATA POLICIES                                  │
│     • Retention policies (cuánto guardar)          │
│     • Access policies (quién puede acceder)        │
│     • Privacy policies (PII, GDPR)                 │
│                                                    │
│  3. DATA STANDARDS                                 │
│     • Naming conventions                           │
│     • Data types consistency                       │
│     • Documentation requirements                   │
│                                                    │
│  4. DATA QUALITY RULES                             │
│     • Validation requirements                      │
│     • Quality thresholds                           │
│     • Monitoring and alerting                      │
│                                                    │
│  5. DATA LIFECYCLE                                 │
│     • Creation → Usage → Archival → Deletion       │
│     • Compliance en cada fase                      │
└────────────────────────────────────────────────────┘
```

### Data Classification

```python
from enum import Enum
from dataclasses import dataclass
from typing import List

class DataSensitivity(Enum):
    PUBLIC = "PUBLIC"           # Datos públicos
    INTERNAL = "INTERNAL"       # Datos internos
    CONFIDENTIAL = "CONFIDENTIAL"  # Datos confidenciales
    RESTRICTED = "RESTRICTED"   # Datos altamente sensibles (PII, PHI)

class DataRetention(Enum):
    SHORT = 90    # 90 días
    MEDIUM = 365  # 1 año
    LONG = 2555   # 7 años
    PERMANENT = -1  # Permanente

@dataclass
class DataAsset:
    """
    Metadata de un asset de datos
    """
    name: str
    database: str
    schema: str
    table: str
    owner: str
    sensitivity: DataSensitivity
    retention_days: DataRetention
    pii_columns: List[str]
    description: str
    tags: List[str]

    def is_pii_data(self) -> bool:
        return len(self.pii_columns) > 0

    def requires_encryption(self) -> bool:
        return self.sensitivity in [DataSensitivity.CONFIDENTIAL, DataSensitivity.RESTRICTED]

    def requires_audit_log(self) -> bool:
        return self.sensitivity == DataSensitivity.RESTRICTED

# Ejemplo: Definir assets
customers_table = DataAsset(
    name="customers",
    database="analytics",
    schema="public",
    table="customers",
    owner="data-team@company.com",
    sensitivity=DataSensitivity.RESTRICTED,
    retention_days=DataRetention.LONG,
    pii_columns=["email", "phone", "ssn", "address"],
    description="Customer master data including PII",
    tags=["pii", "gdpr", "critical"]
)

sales_table = DataAsset(
    name="sales",
    database="analytics",
    schema="public",
    table="sales",
    owner="data-team@company.com",
    sensitivity=DataSensitivity.INTERNAL,
    retention_days=DataRetention.LONG,
    pii_columns=[],
    description="Sales transactions (no PII)",
    tags=["financial", "core"]
)

# ===== APLICAR POLÍTICAS =====
def apply_data_policies(asset: DataAsset):
    """
    Aplicar políticas de governance a un asset
    """
    policies = []

    # Encryption
    if asset.requires_encryption():
        policies.append({
            'policy': 'ENCRYPTION',
            'action': f"Enable encryption for {asset.name}"
        })

    # Access control
    if asset.is_pii_data():
        policies.append({
            'policy': 'ACCESS_CONTROL',
            'action': f"Restrict access to {asset.name}, require approval"
        })

    # Audit logging
    if asset.requires_audit_log():
        policies.append({
            'policy': 'AUDIT_LOG',
            'action': f"Enable query audit log for {asset.name}"
        })

    # Retention
    if asset.retention_days != DataRetention.PERMANENT:
        policies.append({
            'policy': 'RETENTION',
            'action': f"Delete records older than {asset.retention_days.value} days"
        })

    # Masking PII in non-prod
    if asset.is_pii_data():
        policies.append({
            'policy': 'PII_MASKING',
            'action': f"Mask columns {asset.pii_columns} in dev/test environments"
        })

    return policies

# Aplicar políticas
policies = apply_data_policies(customers_table)
for policy in policies:
    print(f"[{policy['policy']}] {policy['action']}")
```

### Access Control (RBAC)

```sql
-- ===== SNOWFLAKE: Role-Based Access Control =====

-- 1. Crear roles
CREATE ROLE data_analyst;
CREATE ROLE data_engineer;
CREATE ROLE data_scientist;
CREATE ROLE pii_viewer;  -- Acceso especial a PII

-- 2. Asignar permisos a roles
-- Data Analyst: Solo lectura en analytics
GRANT USAGE ON DATABASE analytics TO ROLE data_analyst;
GRANT USAGE ON SCHEMA analytics.public TO ROLE data_analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.public TO ROLE data_analyst;

-- Data Engineer: Lectura/escritura en staging y analytics
GRANT USAGE ON DATABASE staging TO ROLE data_engineer;
GRANT USAGE ON SCHEMA staging.raw TO ROLE data_engineer;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA staging.raw TO ROLE data_engineer;

-- PII Viewer: Acceso a columnas PII (solo usuarios aprobados)
GRANT SELECT ON analytics.public.customers TO ROLE pii_viewer;

-- 3. Asignar roles a usuarios
GRANT ROLE data_analyst TO USER john@company.com;
GRANT ROLE data_engineer TO USER jane@company.com;
GRANT ROLE pii_viewer TO USER compliance_officer@company.com;

-- 4. Row-Level Security
-- Solo ver clientes de tu región
CREATE ROW ACCESS POLICY customer_region_policy AS (region STRING)
    RETURNS BOOLEAN ->
        CURRENT_ROLE() = 'ADMIN' OR
        region = CURRENT_USER_REGION();

ALTER TABLE customers ADD ROW ACCESS POLICY customer_region_policy ON (region);

-- 5. Column-Level Security (Masking)
-- Ocultar email para usuarios sin permiso PII
CREATE MASKING POLICY email_mask AS (val STRING)
    RETURNS STRING ->
        CASE
            WHEN CURRENT_ROLE() IN ('pii_viewer', 'ADMIN') THEN val
            ELSE '***@***.com'
        END;

ALTER TABLE customers MODIFY COLUMN email SET MASKING POLICY email_mask;

-- Ahora queries muestran email masked para usuarios sin permiso:
-- data_analyst: SELECT email FROM customers → '***@***.com'
-- pii_viewer:   SELECT email FROM customers → 'user@example.com'
```

---

## Data Lineage

### ¿Qué es Data Lineage?

**Data Lineage** es el rastreo de datos desde su origen hasta su destino, mostrando todas las transformaciones en el camino.

```
┌─────────────────────────────────────────────────────┐
│              DATA LINEAGE EXAMPLE                   │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Source                Transform         Destination│
│  ┌──────────┐         ┌─────────┐      ┌─────────┐  │
│  │ Shopify  │────────▶│  Glue   │─────▶│  S3     │  │
│  │   API    │         │  ETL    │      │  Raw    │  │
│  └──────────┘         └─────────┘      └─────────┘  │
│                             │                │      │
│                             │                │      │
│                             ▼                ▼      │
│                       ┌─────────┐      ┌─────────┐  │
│                       │  Spark  │─────▶│  S3     │  │
│                       │Transform│      │Processed│  │
│                       └─────────┘      └─────────┘  │
│                             │                       │
│                             ▼                       │
│                       ┌─────────┐                   │
│                       │Redshift │                   │
│                       │Analytics│                   │
│                       └─────────┘                   │
│                             │                       │
│                             ▼                       │
│                       ┌─────────┐                   │
│                       │Tableau  │                   │
│                       │Dashboard│                   │
│                       └─────────┘                   │
└─────────────────────────────────────────────────────┘
```

### Capturar Lineage Automáticamente

```python
import json
from datetime import datetime
from typing import List, Dict

class LineageTracker:
    """
    Rastrear lineage de transformaciones
    """

    def __init__(self):
        self.lineage_events = []

    def track_transformation(
        self,
        source_tables: List[str],
        target_table: str,
        transformation_type: str,
        transformation_logic: str,
        metadata: Dict = None
    ):
        """
        Registrar un evento de transformación
        """
        event = {
            'timestamp': datetime.utcnow().isoformat(),
            'source_tables': source_tables,
            'target_table': target_table,
            'transformation_type': transformation_type,
            'transformation_logic': transformation_logic,
            'metadata': metadata or {}
        }

        self.lineage_events.append(event)

        # Persistir a storage (S3, DB, etc.)
        self._persist_lineage(event)

    def _persist_lineage(self, event: dict):
        """
        Guardar evento de lineage
        """
        # Ejemplo: Escribir a S3
        import boto3
        s3 = boto3.client('s3')

        key = f"lineage/{event['target_table']}/{event['timestamp']}.json"
        s3.put_object(
            Bucket='data-lineage',
            Key=key,
            Body=json.dumps(event)
        )

# ===== USO EN ETL =====
lineage = LineageTracker()

# Ejemplo: ETL con PySpark
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("ETL with Lineage").getOrCreate()

# Leer fuentes
customers = spark.read.table("raw.customers")
orders = spark.read.table("raw.orders")

# Track lineage
lineage.track_transformation(
    source_tables=["raw.customers", "raw.orders"],
    target_table="analytics.customer_summary",
    transformation_type="aggregation_join",
    transformation_logic="""
        SELECT
            c.customer_id,
            c.customer_name,
            COUNT(o.order_id) as order_count,
            SUM(o.amount) as total_spent
        FROM customers c
        LEFT JOIN orders o ON c.customer_id = o.customer_id
        GROUP BY c.customer_id, c.customer_name
    """,
    metadata={
        'spark_version': spark.version,
        'executor_count': spark.sparkContext.defaultParallelism,
        'row_count': customers.count() + orders.count()
    }
)

# Transformación
result = customers.join(orders, "customer_id") \
    .groupBy("customer_id", "customer_name") \
    .agg(...)

# Escribir resultado
result.write.saveAsTable("analytics.customer_summary")
```

### Herramientas de Lineage

**1. Apache Atlas**
- Open-source metadata and governance
- Integración con Hadoop ecosystem

**2. Marquez (OpenLineage)**
- Open-source lineage collection
- Integración con Airflow, dbt, Spark

**3. Alation**
- Data catalog comercial con lineage
- ML-powered automation

**4. Collibra**
- Enterprise data governance platform
- End-to-end lineage

---

## Metadata Management

### Tipos de Metadata

| Tipo | Descripción | Ejemplo |
|------|-------------|---------|
| **Technical** | Estructura de datos | Schema, types, constraints |
| **Business** | Significado de negocio | Definiciones, ownership |
| **Operational** | Datos de operación | Size, row count, last updated |
| **Lineage** | Origen y transformaciones | Source → Transform → Target |

### Metadata Store

```python
from dataclasses import dataclass, asdict
from typing import List, Optional
from datetime import datetime
import json

@dataclass
class TableMetadata:
    """
    Metadata completo de una tabla
    """
    # Technical metadata
    database: str
    schema: str
    table_name: str
    columns: List[dict]  # [{'name': 'col1', 'type': 'string', ...}]
    primary_key: Optional[List[str]]
    foreign_keys: List[dict]

    # Business metadata
    description: str
    owner: str
    steward: str
    domain: str  # finance, marketing, sales, etc.

    # Operational metadata
    row_count: int
    size_bytes: int
    last_updated: datetime
    update_frequency: str  # daily, hourly, etc.

    # Governance metadata
    sensitivity: str  # public, internal, confidential, restricted
    retention_days: int
    pii_columns: List[str]
    tags: List[str]

    # Lineage metadata
    source_tables: List[str]
    downstream_tables: List[str]

    def to_json(self) -> str:
        data = asdict(self)
        data['last_updated'] = self.last_updated.isoformat()
        return json.dumps(data, indent=2)

# ===== EJEMPLO: Metadata de tabla customers =====
customers_metadata = TableMetadata(
    database="analytics",
    schema="public",
    table_name="customers",
    columns=[
        {'name': 'customer_id', 'type': 'VARCHAR(50)', 'nullable': False},
        {'name': 'email', 'type': 'VARCHAR(255)', 'nullable': False},
        {'name': 'name', 'type': 'VARCHAR(100)', 'nullable': True},
        {'name': 'created_at', 'type': 'TIMESTAMP', 'nullable': False},
    ],
    primary_key=['customer_id'],
    foreign_keys=[],
    description="Customer master data including contact information",
    owner="data-team@company.com",
    steward="john.doe@company.com",
    domain="sales",
    row_count=1_500_000,
    size_bytes=350_000_000,
    last_updated=datetime.utcnow(),
    update_frequency="daily",
    sensitivity="restricted",
    retention_days=2555,  # 7 years
    pii_columns=['email', 'name', 'phone'],
    tags=['core', 'pii', 'gdpr'],
    source_tables=['raw.shopify_customers', 'raw.salesforce_contacts'],
    downstream_tables=['analytics.customer_segments', 'ml.churn_features']
)

print(customers_metadata.to_json())

# ===== METADATA CATALOG (centralizado) =====
class MetadataCatalog:
    """
    Catálogo centralizado de metadata
    """

    def __init__(self, storage_backend='dynamodb'):
        self.backend = storage_backend

    def register_table(self, metadata: TableMetadata):
        """
        Registrar metadata de tabla en catálogo
        """
        if self.backend == 'dynamodb':
            import boto3
            dynamodb = boto3.resource('dynamodb')
            table = dynamodb.Table('metadata_catalog')

            table.put_item(Item={
                'table_id': f"{metadata.database}.{metadata.schema}.{metadata.table_name}",
                'metadata': metadata.to_json(),
                'last_updated': metadata.last_updated.isoformat()
            })

    def get_table_metadata(self, database: str, schema: str, table_name: str) -> Optional[TableMetadata]:
        """
        Obtener metadata de tabla
        """
        # Implementación depende del backend
        pass

    def search_tables(self, tags: List[str] = None, domain: str = None) -> List[TableMetadata]:
        """
        Buscar tablas por tags o domain
        """
        pass

# Uso
catalog = MetadataCatalog()
catalog.register_table(customers_metadata)
```

---

## Data Catalogs

### AWS Glue Data Catalog

```python
import boto3

glue = boto3.client('glue')

# ===== BUSCAR TABLAS =====
def search_tables(database: str, keyword: str = None):
    """
    Buscar tablas en Glue Catalog
    """
    paginator = glue.get_paginator('get_tables')

    for page in paginator.paginate(DatabaseName=database):
        for table in page['TableList']:
            if keyword is None or keyword.lower() in table['Name'].lower():
                print(f"Table: {table['Name']}")
                print(f"  Location: {table['StorageDescriptor']['Location']}")
                print(f"  Columns: {len(table['StorageDescriptor']['Columns'])}")
                print()

# ===== AGREGAR METADATA CUSTOM =====
def add_table_metadata(database: str, table: str, metadata: dict):
    """
    Agregar metadata custom a tabla en Glue Catalog
    """
    response = glue.get_table(DatabaseName=database, Name=table)
    table_input = response['Table']

    # Agregar parameters (metadata custom)
    if 'Parameters' not in table_input:
        table_input['Parameters'] = {}

    table_input['Parameters'].update({
        'owner': metadata.get('owner', ''),
        'pii_columns': ','.join(metadata.get('pii_columns', [])),
        'sensitivity': metadata.get('sensitivity', ''),
        'retention_days': str(metadata.get('retention_days', 0))
    })

    # Actualizar tabla
    glue.update_table(
        DatabaseName=database,
        TableInput=table_input
    )

# Uso
add_table_metadata('analytics', 'customers', {
    'owner': 'data-team@company.com',
    'pii_columns': ['email', 'phone'],
    'sensitivity': 'restricted',
    'retention_days': 2555
})

# ===== BUSCAR POR METADATA =====
def find_pii_tables(database: str):
    """
    Encontrar todas las tablas que contienen PII
    """
    response = glue.get_tables(DatabaseName=database)

    pii_tables = []
    for table in response['TableList']:
        params = table.get('Parameters', {})
        if 'pii_columns' in params and params['pii_columns']:
            pii_tables.append({
                'table': table['Name'],
                'pii_columns': params['pii_columns'].split(',')
            })

    return pii_tables

pii_tables = find_pii_tables('analytics')
for table in pii_tables:
    print(f"{table['table']}: {table['pii_columns']}")
```

---

## Compliance y Regulaciones

### GDPR (General Data Protection Regulation)

**Principios clave:**

1. **Right to Access** - Usuario puede solicitar sus datos
2. **Right to Erasure** (Right to be Forgotten) - Eliminar datos de usuario
3. **Data Portability** - Exportar datos en formato portable
4. **Data Minimization** - Solo recolectar datos necesarios
5. **Privacy by Design** - Privacy integrado desde diseño

**Implementación:**

```python
# ===== RIGHT TO ACCESS: Exportar datos de usuario =====
def export_user_data(user_id: str) -> dict:
    """
    Exportar todos los datos de un usuario (GDPR Art. 15)
    """
    from snowflake.connector import connect

    conn = connect(...)
    cursor = conn.cursor()

    user_data = {}

    # Buscar en todas las tablas con PII
    pii_tables = [
        'customers',
        'orders',
        'support_tickets',
        'marketing_preferences'
    ]

    for table in pii_tables:
        cursor.execute(f"""
            SELECT *
            FROM {table}
            WHERE user_id = %s
        """, (user_id,))

        columns = [desc[0] for desc in cursor.description]
        rows = cursor.fetchall()

        user_data[table] = [
            dict(zip(columns, row))
            for row in rows
        ]

    return user_data

# ===== RIGHT TO ERASURE: Eliminar datos de usuario =====
def delete_user_data(user_id: str, reason: str):
    """
    Eliminar todos los datos de un usuario (GDPR Art. 17)
    """
    from snowflake.connector import connect
    import logging

    conn = connect(...)
    cursor = conn.cursor()

    # Log de auditoría
    logging.info(f"Deleting data for user {user_id}, reason: {reason}")

    try:
        cursor.execute("BEGIN")

        # Eliminar o anonimizar en cada tabla
        tables_to_delete = [
            'customers',
            'orders',
            'support_tickets'
        ]

        for table in tables_to_delete:
            cursor.execute(f"""
                DELETE FROM {table}
                WHERE user_id = %s
            """, (user_id,))

        # Anonimizar (en vez de eliminar) en tablas de analytics
        cursor.execute("""
            UPDATE analytics.user_behavior
            SET user_id = 'DELETED_USER',
                email = 'deleted@deleted.com'
            WHERE user_id = %s
        """, (user_id,))

        # Guardar registro en tabla de auditoría
        cursor.execute("""
            INSERT INTO gdpr_deletion_log (user_id, deleted_at, reason)
            VALUES (%s, CURRENT_TIMESTAMP, %s)
        """, (user_id, reason))

        cursor.execute("COMMIT")
        logging.info(f"Successfully deleted data for user {user_id}")

    except Exception as e:
        cursor.execute("ROLLBACK")
        logging.error(f"Failed to delete data for user {user_id}: {e}")
        raise

# ===== DATA MINIMIZATION: Retención automática =====
def apply_retention_policy():
    """
    Aplicar políticas de retención (eliminar datos viejos)
    """
    from snowflake.connector import connect

    conn = connect(...)
    cursor = conn.cursor()

    # Eliminar datos más viejos que retention period
    cursor.execute("""
        DELETE FROM customers
        WHERE deleted_at IS NOT NULL
            AND deleted_at < DATEADD(day, -30, CURRENT_DATE)
    """)

    cursor.execute("""
        DELETE FROM orders
        WHERE order_date < DATEADD(year, -7, CURRENT_DATE)
    """)

    conn.commit()
```

### CCPA (California Consumer Privacy Act)

Similar a GDPR, con énfasis en:
- Right to know what data is collected
- Right to delete
- Right to opt-out of sale of data

### HIPAA (Health Insurance Portability and Accountability Act)

Para datos de salud (PHI - Protected Health Information):
- Encriptación en tránsito y en reposo
- Access logs y audit trails
- Backup y disaster recovery
- Business Associate Agreements (BAA)

---

## Herramientas y Frameworks

### Comparación de Herramientas

| Herramienta | Tipo | Características | Costo |
|-------------|------|-----------------|-------|
| **Great Expectations** | Data validation | Open-source, Python, extensible | Free |
| **dbt** | Transformation + testing | SQL-based, CI/CD friendly | Free (core) |
| **Apache Atlas** | Metadata + lineage | Hadoop ecosystem | Free |
| **Collibra** | Enterprise governance | End-to-end platform | $$$ |
| **Alation** | Data catalog | ML-powered discovery | $$$ |
| **Monte Carlo** | Data observability | Anomaly detection | $$ |
| **Datadog** | Monitoring | Full-stack observability | $$ |

---

## 🎯 Preguntas de Entrevista

**P: ¿Cómo implementarías data quality checks en un pipeline ETL?**

R:
1. **Pre-validación**: Verificar schema y formato antes de procesar
2. **Great Expectations**: Definir expectation suites para cada dataset
3. **Airflow integration**: Task de validación antes de load
4. **Post-validación**: Checks en datos cargados (counts, aggregations)
5. **Alerting**: Slack/PagerDuty si validación falla
6. **Quarantine**: Mover registros inválidos a tabla separada para revisión

**P: ¿Qué harías si un dashboard muestra métricas incorrectas?**

R:
1. **Data lineage**: Rastrear de dónde viene el dato
2. **Validation**: Verificar cada paso del pipeline (source → transform → dashboard)
3. **Compare with source**: Query directo a fuente vs dashboard
4. **Check transformations**: Revisar lógica de agregaciones/joins
5. **Temporal analysis**: ¿Cuándo empezó a fallar? Cambios recientes?
6. **Root cause**: Identificar si es código, datos, o configuración

**P: Explica cómo cumplirías con GDPR en un data warehouse**

R:
1. **Data classification**: Identificar tablas con PII
2. **Encryption**: At-rest y in-transit para PII
3. **Access control**: RBAC, columnas PII solo accesibles con permiso
4. **Audit logging**: Log de accesos a PII
5. **Right to access**: API para exportar datos de usuario
6. **Right to erasure**: Proceso automatizado para eliminar/anonimizar
7. **Retention policies**: Auto-delete después de período requerido
8. **Consent management**: Tracking de consentimiento de usuario

---

**Siguiente:** [10. Seguridad y Cumplimiento](10_seguridad_cumplimiento.md)
