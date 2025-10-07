# 6. Orquestación y Automatización de Pipelines

## 📋 Tabla de Contenidos
- [¿Qué es la Orquestación?](#qué-es-la-orquestación)
- [Apache Airflow](#apache-airflow)
- [Alternativas Modernas](#alternativas-modernas)
- [Cloud-Native Solutions](#cloud-native-solutions)
- [Retry Policies y Error Handling](#retry-policies-y-error-handling)
- [Monitoring y Alerting](#monitoring-y-alerting)
- [Backfill Strategies](#backfill-strategies)
- [Best Practices](#best-practices)

---

## ¿Qué es la Orquestación?

### Definición
La **orquestación de pipelines** es el proceso de coordinar, programar y monitorear flujos de trabajo complejos de datos, asegurando que las tareas se ejecuten en el orden correcto, con las dependencias adecuadas y con manejo de errores.

### Características Principales

| Característica | Descripción | Ejemplo |
|---------------|-------------|---------|
| **Scheduling** | Programación temporal | Ejecutar ETL diario a las 2 AM |
| **Dependency Management** | Gestión de dependencias | Task B espera a Task A |
| **Retry Logic** | Reintentos automáticos | 3 intentos con backoff exponencial |
| **Monitoring** | Monitoreo en tiempo real | Alertas si pipeline falla |
| **Idempotencia** | Ejecuciones repetibles | Re-run sin duplicar datos |
| **Paralelización** | Ejecución concurrente | Procesar múltiples tablas a la vez |

### Componentes de un Workflow

```
┌─────────────────────────────────────────┐
│    WORKFLOW / DAG (Directed Acyclic     │
│            Graph)                       │
│                                         │
│ ┌────────┐    ┌─────────┐    ┌────────┐ │
│ │ Task 1 │───▶│ Task 2  │───▶│ Task 3 │ │
│ │Extract │    │Transform│    │  Load  │ │
│ └────────┘    └─────────┘    └────────┘ │
│      │             │             │      │
│      └─────────────┴┬────────────┘      │
│                     ▼                   │
│              ┌────────────┐             │
│              │  Task 4    │             │
│              │ Validate   │             │
│              └────────────┘             │
└─────────────────────────────────────────┘
```

---

## Apache Airflow

### Arquitectura

```
┌──────────────────────────────────────────────┐
│              APACHE AIRFLOW                  │
├──────────────────────────────────────────────┤
│                                              │
│  ┌──────────────┐      ┌──────────────┐      │
│  │  Web Server  │◀────▶│  Scheduler   │      │
│  │   (Flask)    │      │  (Executor)  │      │
│  └──────────────┘      └──────────────┘      │
│         │                      │             │
│         ▼                      ▼             │
│  ┌─────────────────────────────────────┐     │
│  │      Metadata Database              │     │
│  │      (PostgreSQL/MySQL)             │     │
│  └─────────────────────────────────────┘     │
│                      │                       │
│                      ▼                       │
│         ┌────────────────────────┐           │
│         │   Workers (Executors)  │           │
│         │  - Local / Sequential  │           │
│         │  - Celery / K8s        │           │
│         └────────────────────────┘           │
└──────────────────────────────────────────────┘
```

### Conceptos Fundamentales

#### 1. DAG (Directed Acyclic Graph)

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from datetime import datetime, timedelta

# Argumentos por defecto para todas las tareas
default_args = {
    'owner': 'data_team',
    'depends_on_past': False,  # No depende de ejecuciones anteriores
    'email': ['alerts@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,  # Reintentar 3 veces
    'retry_delay': timedelta(minutes=5),  # Esperar 5 min entre reintentos
    'execution_timeout': timedelta(hours=2),  # Timeout de 2 horas
}

# Definir DAG
dag = DAG(
    'ecommerce_daily_etl',
    default_args=default_args,
    description='ETL diario de ventas e-commerce',
    schedule_interval='0 2 * * *',  # Cron: 2 AM diario
    start_date=datetime(2025, 1, 1),
    catchup=False,  # No ejecutar fechas pasadas
    max_active_runs=1,  # Solo 1 instancia a la vez
    tags=['production', 'sales', 'daily'],
)
```

#### 2. Operators (Tipos de Tareas)

```python
# ===== PYTHON OPERATOR =====
def extract_from_api(**context):
    """Extraer datos de API externa"""
    import requests
    execution_date = context['execution_date']

    response = requests.get(
        'https://api.ecommerce.com/sales',
        params={'date': execution_date.strftime('%Y-%m-%d')}
    )

    data = response.json()
    # Guardar en XCom para compartir entre tareas
    context['task_instance'].xcom_push(key='raw_sales_count', value=len(data))

    # Guardar en S3
    s3_path = f's3://raw-data/sales/{execution_date.strftime("%Y/%m/%d")}/sales.json'
    # ... código para subir a S3 ...

    return s3_path

extract_task = PythonOperator(
    task_id='extract_sales_data',
    python_callable=extract_from_api,
    provide_context=True,  # Pasar contexto de Airflow
    dag=dag,
)

# ===== BASH OPERATOR =====
validate_s3_file = BashOperator(
    task_id='validate_s3_upload',
    bash_command='''
        aws s3 ls {{ ti.xcom_pull(task_ids='extract_sales_data') }} && \
        echo "File exists in S3" || exit 1
    ''',
    dag=dag,
)

# ===== SNOWFLAKE OPERATOR =====
load_to_snowflake = SnowflakeOperator(
    task_id='load_to_warehouse',
    snowflake_conn_id='snowflake_prod',
    sql='''
        COPY INTO raw.sales
        FROM 's3://raw-data/sales/{{ ds_nodash }}/'
        CREDENTIALS = (AWS_KEY_ID='{{ var.value.aws_key }}'
                       AWS_SECRET_KEY='{{ var.value.aws_secret }}')
        FILE_FORMAT = (TYPE = 'JSON');

        -- Transformar e insertar en tabla limpia
        INSERT INTO analytics.sales_daily
        SELECT
            order_id,
            customer_id,
            product_id,
            amount,
            order_date,
            CURRENT_TIMESTAMP() as loaded_at
        FROM raw.sales
        WHERE DATE(order_date) = '{{ ds }}'
        ON CONFLICT (order_id) DO NOTHING;
    ''',
    dag=dag,
)

# ===== SENSOR (Espera Condicional) =====
from airflow.sensors.external_task import ExternalTaskSensor

wait_for_customer_data = ExternalTaskSensor(
    task_id='wait_customer_etl',
    external_dag_id='customer_daily_etl',
    external_task_id='load_customers',
    execution_delta=timedelta(hours=1),  # Ejecuta 1h antes
    timeout=3600,  # Esperar máximo 1 hora
    mode='poke',  # Revisar cada X segundos
    poke_interval=60,  # Cada 60 segundos
    dag=dag,
)
```

#### 3. Dependencias y Task Flow

```python
# ===== MÉTODO 1: Bitshift Operators =====
extract_task >> validate_s3_file >> load_to_snowflake

# ===== MÉTODO 2: Set Dependencies =====
wait_for_customer_data.set_downstream(load_to_snowflake)

# ===== MÉTODO 3: TaskFlow API (Airflow 2.0+) =====
from airflow.decorators import task

@task
def transform_sales_data(s3_path: str) -> dict:
    """Transformar datos usando Pandas"""
    import pandas as pd

    df = pd.read_json(s3_path)

    # Limpiar datos
    df['amount'] = df['amount'].astype(float)
    df = df[df['amount'] > 0]

    # Agregar métricas
    metrics = {
        'total_sales': df['amount'].sum(),
        'order_count': len(df),
        'avg_order_value': df['amount'].mean()
    }

    return metrics

@task
def send_metrics_to_slack(metrics: dict):
    """Enviar métricas a Slack"""
    from slack_sdk import WebClient

    client = WebClient(token="xoxb-your-token")
    client.chat_postMessage(
        channel="#data-alerts",
        text=f"📊 Ventas diarias:\n"
             f"• Total: ${metrics['total_sales']:,.2f}\n"
             f"• Órdenes: {metrics['order_count']}\n"
             f"• Promedio: ${metrics['avg_order_value']:.2f}"
    )

# Definir flujo
s3_path = extract_task
metrics = transform_sales_data(s3_path)
send_metrics_to_slack(metrics)
```

#### 4. Templating con Jinja

```python
# Airflow provee variables de template
templated_command = BashOperator(
    task_id='templated_task',
    bash_command='''
        # Variables de ejecución
        echo "Execution Date: {{ ds }}"  # 2025-01-15
        echo "Previous Date: {{ prev_ds }}"  # 2025-01-14
        echo "Next Date: {{ next_ds }}"  # 2025-01-16
        echo "Year: {{ macros.ds_format(ds, "%Y-%m-%d", "%Y") }}"

        # Acceder a XCom
        echo "Sales count: {{ ti.xcom_pull(task_ids='extract_sales_data', key='raw_sales_count') }}"

        # Variables de Airflow
        echo "DAG: {{ dag.dag_id }}"
        echo "Task: {{ task.task_id }}"
        echo "Run ID: {{ run_id }}"
    ''',
    dag=dag,
)
```

#### 5. Dynamic Task Mapping (Airflow 2.3+)

```python
from airflow.decorators import task

@task
def get_tables_to_process() -> list:
    """Obtener dinámicamente lista de tablas"""
    return ['customers', 'orders', 'products', 'inventory']

@task
def process_table(table_name: str):
    """Procesar una tabla específica"""
    from snowflake.connector import connect

    conn = connect(
        user='etl_user',
        password='{{ var.value.snowflake_password }}',
        account='myaccount',
        warehouse='ETL_WH'
    )

    query = f"SELECT COUNT(*) FROM {table_name}"
    result = conn.cursor().execute(query).fetchone()
    print(f"Table {table_name} has {result[0]} rows")

# Mapear dinámicamente
tables = get_tables_to_process()
process_table.expand(table_name=tables)  # Crea N tareas dinámicamente
```

### Ejecutores en Airflow

| Executor | Uso | Pros | Contras |
|----------|-----|------|---------|
| **SequentialExecutor** | Desarrollo local | Simple, fácil debug | 1 tarea a la vez |
| **LocalExecutor** | Small production | Paralelización local | Limitado a 1 máquina |
| **CeleryExecutor** | Production (multi-node) | Escalable, robusto | Requiere Redis/RabbitMQ |
| **KubernetesExecutor** | Cloud-native | Auto-scaling, aislamiento | Complejidad de K8s |
| **DaskExecutor** | Data science | Integración con Dask | Menos maduro |

---

## Alternativas Modernas

### Prefect

```python
from prefect import flow, task
from prefect.task_runners import ConcurrentTaskRunner
import httpx

@task(retries=3, retry_delay_seconds=60)
def extract_sales(date: str) -> list:
    """Extraer datos de API"""
    response = httpx.get(f'https://api.ecommerce.com/sales?date={date}')
    response.raise_for_status()
    return response.json()

@task
def transform_sales(raw_data: list) -> dict:
    """Transformar datos"""
    import pandas as pd
    df = pd.DataFrame(raw_data)

    return {
        'total': df['amount'].sum(),
        'count': len(df)
    }

@task(log_prints=True)
def load_to_warehouse(data: dict):
    """Cargar a warehouse"""
    from snowflake.connector import connect

    conn = connect(user='user', password='pass', account='account')
    cursor = conn.cursor()

    cursor.execute(f"""
        INSERT INTO metrics (date, total_sales, order_count)
        VALUES (CURRENT_DATE, {data['total']}, {data['count']})
    """)

    print(f"Loaded {data['count']} orders totaling ${data['total']}")

@flow(
    name="Sales ETL",
    task_runner=ConcurrentTaskRunner(),  # Ejecución concurrente
)
def sales_etl_flow(date: str):
    """Pipeline completo de ventas"""
    raw_data = extract_sales(date)
    transformed = transform_sales(raw_data)
    load_to_warehouse(transformed)

# Ejecutar
if __name__ == "__main__":
    from datetime import datetime
    sales_etl_flow(date=datetime.today().strftime('%Y-%m-%d'))
```

**Ventajas de Prefect sobre Airflow:**
- Sintaxis más pythónica (decoradores)
- Ejecución local sin setup complejo
- Hybrid execution (cloud + on-premise)
- Mejor UI y observabilidad
- Dinámico (DAGs se generan en runtime)

### Dagster

```python
from dagster import asset, OpExecutionContext, Definitions
import pandas as pd

@asset(
    group_name="raw",
    compute_kind="python"
)
def raw_sales(context: OpExecutionContext) -> pd.DataFrame:
    """Asset de datos crudos"""
    context.log.info("Extracting sales from API")

    # Simulación de extracción
    df = pd.DataFrame({
        'order_id': [1, 2, 3],
        'amount': [100, 200, 150]
    })

    return df

@asset(
    group_name="cleaned",
    deps=[raw_sales]  # Depende de raw_sales
)
def cleaned_sales(raw_sales: pd.DataFrame) -> pd.DataFrame:
    """Asset de datos limpios"""
    df = raw_sales.copy()
    df['amount'] = df['amount'].astype(float)
    return df[df['amount'] > 0]

@asset(
    group_name="analytics",
    deps=[cleaned_sales]
)
def sales_metrics(cleaned_sales: pd.DataFrame) -> dict:
    """Asset de métricas"""
    return {
        'total': cleaned_sales['amount'].sum(),
        'avg': cleaned_sales['amount'].mean(),
        'count': len(cleaned_sales)
    }

# Definir materialización completa
defs = Definitions(
    assets=[raw_sales, cleaned_sales, sales_metrics]
)
```

**Ventajas de Dagster:**
- **Asset-centric** (enfoque en datos, no tareas)
- Type checking nativo
- Testing integrado
- Mejor para data lakehouse architectures
- Software-defined assets (SDA)

---

## Cloud-Native Solutions

### AWS Step Functions

```json
{
  "Comment": "ETL Pipeline for E-commerce Sales",
  "StartAt": "ExtractData",
  "States": {
    "ExtractData": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "extract-sales-data",
        "Payload": {
          "date.$": "$.execution_date"
        }
      },
      "ResultPath": "$.s3_path",
      "Next": "ValidateExtraction",
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 60,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "NotifyFailure"
        }
      ]
    },

    "ValidateExtraction": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "validate-s3-file",
        "Payload": {
          "s3_path.$": "$.s3_path"
        }
      },
      "Next": "ParallelTransform"
    },

    "ParallelTransform": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "TransformSales",
          "States": {
            "TransformSales": {
              "Type": "Task",
              "Resource": "arn:aws:states:::glue:startJobRun.sync",
              "Parameters": {
                "JobName": "transform-sales",
                "Arguments": {
                  "--S3_PATH.$": "$.s3_path"
                }
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "CalculateMetrics",
          "States": {
            "CalculateMetrics": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:function:calculate-metrics",
              "End": true
            }
          }
        }
      ],
      "Next": "LoadToRedshift"
    },

    "LoadToRedshift": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "load-to-redshift"
      },
      "Next": "Success"
    },

    "Success": {
      "Type": "Succeed"
    },

    "NotifyFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789:etl-failures",
        "Message": "ETL pipeline failed"
      },
      "End": true
    }
  }
}
```

### Google Cloud Composer (Managed Airflow)

```python
# Mismo código de Airflow, pero en Cloud Composer
from airflow import DAG
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
from airflow.providers.google.cloud.transfers.gcs_to_bigquery import GCSToBigQueryOperator

dag = DAG('gcp_etl', ...)

load_to_bq = GCSToBigQueryOperator(
    task_id='load_csv_to_bigquery',
    bucket='my-bucket',
    source_objects=['sales/{{ ds }}/data.csv'],
    destination_project_dataset_table='project.dataset.sales',
    schema_fields=[
        {'name': 'order_id', 'type': 'INTEGER', 'mode': 'REQUIRED'},
        {'name': 'amount', 'type': 'FLOAT', 'mode': 'REQUIRED'},
    ],
    write_disposition='WRITE_APPEND',
    dag=dag,
)
```

---

## Retry Policies y Error Handling

### Estrategias de Retry

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import timedelta
import time

# ===== ESTRATEGIA 1: Retry Lineal =====
def flaky_task():
    """Tarea que puede fallar intermitentemente"""
    import random
    if random.random() < 0.3:  # 30% probabilidad de fallo
        raise Exception("Random failure")
    return "Success"

linear_retry_task = PythonOperator(
    task_id='linear_retry',
    python_callable=flaky_task,
    retries=5,
    retry_delay=timedelta(minutes=2),  # Esperar 2 min entre cada intento
    dag=dag,
)

# ===== ESTRATEGIA 2: Exponential Backoff =====
from airflow.utils.trigger_rule import TriggerRule

exponential_retry_task = PythonOperator(
    task_id='exponential_retry',
    python_callable=flaky_task,
    retries=5,
    retry_delay=timedelta(seconds=30),
    retry_exponential_backoff=True,  # 30s, 1m, 2m, 4m, 8m
    max_retry_delay=timedelta(minutes=10),
    dag=dag,
)

# ===== ESTRATEGIA 3: Custom Retry Logic =====
from airflow.exceptions import AirflowException

def smart_retry(**context):
    """Retry con lógica personalizada"""
    task_instance = context['task_instance']
    try_number = task_instance.try_number

    # Aumentar timeout con cada intento
    timeout = 30 * try_number

    try:
        import requests
        response = requests.get('https://api.example.com/data', timeout=timeout)
        response.raise_for_status()
        return response.json()

    except requests.exceptions.Timeout:
        if try_number < 3:
            raise AirflowException(f"Timeout on attempt {try_number}, retrying...")
        else:
            # Después de 3 intentos, usar datos en cache
            return get_cached_data()

    except requests.exceptions.HTTPError as e:
        if e.response.status_code == 429:  # Rate limit
            time.sleep(60 * try_number)  # Esperar más tiempo
            raise AirflowException("Rate limited, retrying...")
        elif e.response.status_code >= 500:
            raise AirflowException("Server error, retrying...")
        else:
            # 4xx errors no se deberían reintentar
            raise AirflowException(f"Client error {e.response.status_code}, not retrying")

smart_task = PythonOperator(
    task_id='smart_retry',
    python_callable=smart_retry,
    retries=5,
    provide_context=True,
    dag=dag,
)
```

### Error Handling Patterns

```python
# ===== PATRÓN 1: Cleanup on Failure =====
from airflow.operators.python import PythonOperator

def cleanup_on_failure(**context):
    """Ejecutar limpieza si tarea falla"""
    import boto3

    # Borrar archivos temporales de S3
    s3 = boto3.client('s3')
    execution_date = context['ds']

    s3.delete_object(
        Bucket='temp-data',
        Key=f'processing/{execution_date}/'
    )

    # Enviar notificación
    send_slack_alert(f"Pipeline failed on {execution_date}")

risky_task = PythonOperator(
    task_id='risky_operation',
    python_callable=process_data,
    on_failure_callback=cleanup_on_failure,
    dag=dag,
)

# ===== PATRÓN 2: Fallback Task =====
from airflow.utils.trigger_rule import TriggerRule

primary_data_source = PythonOperator(
    task_id='fetch_from_primary_api',
    python_callable=fetch_from_primary,
    dag=dag,
)

fallback_data_source = PythonOperator(
    task_id='fetch_from_backup_api',
    python_callable=fetch_from_backup,
    trigger_rule=TriggerRule.ONE_FAILED,  # Solo ejecuta si primary falla
    dag=dag,
)

process_data_task = PythonOperator(
    task_id='process_data',
    python_callable=process,
    trigger_rule=TriggerRule.ONE_SUCCESS,  # Ejecuta si al menos uno tuvo éxito
    dag=dag,
)

[primary_data_source, fallback_data_source] >> process_data_task

# ===== PATRÓN 3: Circuit Breaker =====
from airflow.models import Variable

def circuit_breaker_task(**context):
    """Detener ejecuciones si hay muchos fallos consecutivos"""
    dag_id = context['dag'].dag_id
    failure_count_key = f'{dag_id}_failure_count'

    failure_count = int(Variable.get(failure_count_key, default_var=0))

    if failure_count >= 5:
        # Pausar DAG automáticamente
        from airflow.models import DagModel
        dag_model = DagModel.get_dagmodel(dag_id)
        dag_model.set_is_paused(is_paused=True)

        send_critical_alert(f"DAG {dag_id} has been paused due to 5+ consecutive failures")
        raise AirflowException("Circuit breaker activated")

    try:
        # Ejecutar lógica principal
        result = execute_main_logic()

        # Reset counter on success
        Variable.set(failure_count_key, 0)
        return result

    except Exception as e:
        # Incrementar contador
        Variable.set(failure_count_key, failure_count + 1)
        raise

circuit_task = PythonOperator(
    task_id='circuit_breaker',
    python_callable=circuit_breaker_task,
    provide_context=True,
    dag=dag,
)
```

---

## Monitoring y Alerting

### Airflow Monitoring

```python
# ===== 1. SLA Monitoring =====
from datetime import timedelta

dag = DAG(
    'critical_etl',
    default_args={
        'sla': timedelta(hours=2),  # Debe completarse en 2 horas
        'email': ['oncall@company.com'],
        'email_on_failure': True,
        'email_on_retry': False,
        'sla_miss_callback': handle_sla_miss,
    },
    ...
)

def handle_sla_miss(dag, task_list, blocking_task_list, slas, blocking_tis):
    """Callback cuando se incumple SLA"""
    message = f"""
    ⚠️ SLA MISS ALERT

    DAG: {dag.dag_id}
    Tasks: {[task.task_id for task in task_list]}
    Blocking: {[task.task_id for task in blocking_task_list]}
    """

    send_pagerduty_alert(message)
    send_slack_alert(message, channel='#critical-alerts')

# ===== 2. Custom Metrics =====
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.hooks.cloudwatch import CloudWatchHook

def publish_metrics(**context):
    """Publicar métricas custom a CloudWatch"""
    ti = context['task_instance']

    # Obtener métricas de XCom
    row_count = ti.xcom_pull(task_ids='load_data', key='rows_processed')
    duration = ti.xcom_pull(task_ids='load_data', key='duration_seconds')

    cloudwatch = CloudWatchHook()

    cloudwatch.put_metric_data(
        namespace='DataPipelines',
        metric_data=[
            {
                'MetricName': 'RowsProcessed',
                'Value': row_count,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'DAG', 'Value': context['dag'].dag_id}
                ]
            },
            {
                'MetricName': 'PipelineDuration',
                'Value': duration,
                'Unit': 'Seconds',
                'Dimensions': [
                    {'Name': 'DAG', 'Value': context['dag'].dag_id}
                ]
            }
        ]
    )

metrics_task = PythonOperator(
    task_id='publish_metrics',
    python_callable=publish_metrics,
    provide_context=True,
    dag=dag,
)

# ===== 3. Health Checks =====
from airflow.sensors.sql import SqlSensor

data_quality_check = SqlSensor(
    task_id='validate_data_quality',
    conn_id='snowflake_prod',
    sql='''
        SELECT COUNT(*)
        FROM analytics.sales_daily
        WHERE date = '{{ ds }}'
        AND amount > 0
        HAVING COUNT(*) > 100  -- Al menos 100 registros válidos
    ''',
    timeout=600,
    poke_interval=60,
    dag=dag,
)

# ===== 4. Alerting con Slack =====
from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator

def format_failure_message(context):
    """Formatear mensaje de fallo para Slack"""
    task_instance = context['task_instance']

    return f"""
:x: *Task Failed*

*DAG:* {context['dag'].dag_id}
*Task:* {task_instance.task_id}
*Execution Date:* {context['execution_date']}
*Log URL:* {task_instance.log_url}
*Error:* {context.get('exception', 'Unknown')}
"""

alert_task = SlackWebhookOperator(
    task_id='send_failure_alert',
    http_conn_id='slack_webhook',
    message=format_failure_message,
    trigger_rule=TriggerRule.ONE_FAILED,
    dag=dag,
)
```

### Prometheus + Grafana para Airflow

```yaml
# docker-compose.yml para monitoring stack
version: '3'
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  statsd-exporter:
    image: prom/statsd-exporter
    ports:
      - "9102:9102"
      - "9125:9125/udp"
```

```python
# Airflow configuration
# airflow.cfg
[metrics]
statsd_on = True
statsd_host = statsd-exporter
statsd_port = 9125
statsd_prefix = airflow

# Métricas automáticas disponibles:
# - airflow.dag.<dag_id>.duration
# - airflow.dag_processing.import_errors
# - airflow.executor.open_slots
# - airflow.task.<task_id>.duration
# - airflow.scheduler.heartbeat
```

---

## Backfill Strategies

### ¿Qué es Backfill?

**Backfill** es el proceso de ejecutar un pipeline para fechas pasadas, útil cuando:
- Se crea un nuevo pipeline y se necesitan datos históricos
- Un bug afectó datos pasados y hay que reprocesarlos
- Se agrega una nueva transformación que requiere datos históricos

### Estrategias de Backfill

```python
# ===== ESTRATEGIA 1: Incremental Backfill =====
from airflow import DAG
from datetime import datetime, timedelta

dag = DAG(
    'incremental_backfill',
    start_date=datetime(2024, 1, 1),  # Fecha de inicio
    schedule_interval='@daily',
    catchup=True,  # IMPORTANTE: Ejecutar fechas pasadas
    max_active_runs=3,  # Limitar ejecuciones concurrentes
    dagrun_timeout=timedelta(hours=4),
)

# Airflow ejecutará automáticamente desde start_date hasta hoy
# 2024-01-01, 2024-01-02, ..., 2025-01-15 (max 3 a la vez)

# ===== ESTRATEGIA 2: Manual Backfill (CLI) =====
# Backfill de un rango específico
# airflow dags backfill \
#     --start-date 2024-01-01 \
#     --end-date 2024-12-31 \
#     --rerun-failed-tasks \
#     my_dag_id

# ===== ESTRATEGIA 3: Chunked Backfill =====
def process_date_range(start_date: str, end_date: str):
    """Procesar rango de fechas en chunks"""
    from datetime import datetime, timedelta
    import pendulum

    start = pendulum.parse(start_date)
    end = pendulum.parse(end_date)

    chunk_size = 7  # Procesar semanas
    current = start

    while current <= end:
        chunk_end = min(current.add(days=chunk_size), end)

        print(f"Processing {current} to {chunk_end}")

        # Procesar chunk
        process_chunk(current, chunk_end)

        current = chunk_end.add(days=1)

backfill_task = PythonOperator(
    task_id='chunked_backfill',
    python_callable=process_date_range,
    op_kwargs={
        'start_date': '2024-01-01',
        'end_date': '2024-12-31'
    },
    dag=dag,
)

# ===== ESTRATEGIA 4: Idempotent Backfill =====
def idempotent_load(**context):
    """Carga idempotente que puede re-ejecutarse sin duplicar datos"""
    from snowflake.connector import connect

    execution_date = context['ds']

    conn = connect(...)
    cursor = conn.cursor()

    # DELETE + INSERT (idempotente)
    cursor.execute(f"""
        DELETE FROM analytics.sales_daily
        WHERE date = '{execution_date}';

        INSERT INTO analytics.sales_daily
        SELECT * FROM staging.sales
        WHERE date = '{execution_date}';
    """)

    # Alternativa: MERGE (upsert)
    cursor.execute(f"""
        MERGE INTO analytics.sales_daily target
        USING (SELECT * FROM staging.sales WHERE date = '{execution_date}') source
        ON target.order_id = source.order_id
        WHEN MATCHED THEN UPDATE SET target.amount = source.amount
        WHEN NOT MATCHED THEN INSERT VALUES (source.*);
    """)

# ===== ESTRATEGIA 5: Parallel Backfill con Dynamic Tasks =====
from airflow.decorators import task

@task
def generate_date_ranges(start: str, end: str, chunks: int = 10) -> list:
    """Dividir rango en N chunks para paralelizar"""
    from datetime import datetime, timedelta
    import pendulum

    start_dt = pendulum.parse(start)
    end_dt = pendulum.parse(end)

    total_days = (end_dt - start_dt).days
    chunk_size = total_days // chunks

    ranges = []
    for i in range(chunks):
        chunk_start = start_dt.add(days=i * chunk_size)
        chunk_end = start_dt.add(days=(i + 1) * chunk_size - 1)
        if i == chunks - 1:
            chunk_end = end_dt

        ranges.append({
            'start': chunk_start.to_date_string(),
            'end': chunk_end.to_date_string()
        })

    return ranges

@task
def process_chunk(date_range: dict):
    """Procesar un chunk específico"""
    print(f"Processing {date_range['start']} to {date_range['end']}")
    # ... lógica de procesamiento ...

# Ejecutar en paralelo
date_ranges = generate_date_ranges('2024-01-01', '2024-12-31', chunks=10)
process_chunk.expand(date_range=date_ranges)  # 10 tareas en paralelo
```

### Best Practices para Backfill

```python
# ===== 1. Rate Limiting =====
from airflow.models import Variable
import time

def rate_limited_api_call(**context):
    """Limitar llamadas a API externa"""
    # Obtener contador global
    call_count = int(Variable.get('api_call_count_today', default_var=0))

    # Límite de 10,000 llamadas/día
    if call_count >= 10000:
        raise AirflowException("Daily API limit reached")

    # Hacer llamada
    result = requests.get('https://api.example.com/data')

    # Incrementar contador
    Variable.set('api_call_count_today', call_count + 1)

    # Rate limit: 10 req/segundo
    time.sleep(0.1)

    return result.json()

# ===== 2. Checkpoint/Resume =====
def checkpoint_backfill(**context):
    """Guardar progreso para poder resumir si falla"""
    from airflow.models import Variable
    import json

    checkpoint_key = f"{context['dag'].dag_id}_checkpoint"

    # Cargar checkpoint anterior
    checkpoint = json.loads(Variable.get(checkpoint_key, default_var='{}'))
    last_processed = checkpoint.get('last_date', '2024-01-01')

    # Procesar desde último checkpoint
    dates_to_process = get_dates_after(last_processed)

    for date in dates_to_process:
        process_date(date)

        # Actualizar checkpoint cada 10 fechas
        if dates_to_process.index(date) % 10 == 0:
            checkpoint['last_date'] = date
            Variable.set(checkpoint_key, json.dumps(checkpoint))

# ===== 3. Data Validation Post-Backfill =====
def validate_backfill(**context):
    """Validar que backfill fue exitoso"""
    from snowflake.connector import connect

    conn = connect(...)

    # Verificar que no hay gaps
    result = conn.cursor().execute("""
        WITH date_series AS (
            SELECT DATEADD(day, seq4(), '2024-01-01')::DATE as expected_date
            FROM TABLE(GENERATOR(ROWCOUNT => 365))
        )
        SELECT expected_date
        FROM date_series
        WHERE expected_date NOT IN (
            SELECT DISTINCT date FROM analytics.sales_daily
        )
    """).fetchall()

    if result:
        missing_dates = [row[0] for row in result]
        raise AirflowException(f"Missing dates after backfill: {missing_dates}")

    # Verificar conteos esperados
    conn.cursor().execute("""
        SELECT date, COUNT(*) as cnt
        FROM analytics.sales_daily
        GROUP BY date
        HAVING cnt < 10  -- Esperamos al menos 10 registros/día
    """)

    print("Backfill validation passed!")
```

---

## Best Practices

### 1. DAG Design Principles

```python
# ❌ MAL: DAG muy complejo
dag = DAG('monolith_pipeline', ...)

extract_customers = PythonOperator(...)
extract_orders = PythonOperator(...)
extract_products = PythonOperator(...)
transform_all = PythonOperator(...)  # Hace todo
load_all = PythonOperator(...)

extract_customers >> transform_all
extract_orders >> transform_all
extract_products >> transform_all
transform_all >> load_all

# ✅ BIEN: DAGs modulares
# customers_dag.py
customers_dag = DAG('customers_etl', ...)
# Solo se enfoca en customers

# orders_dag.py
orders_dag = DAG('orders_etl', ...)
wait_for_customers = ExternalTaskSensor(
    external_dag_id='customers_etl',
    external_task_id='load_customers'
)
```

### 2. Idempotencia

```python
# ❌ MAL: No idempotente
def load_data():
    cursor.execute("INSERT INTO table VALUES (...)")  # Duplica en re-run

# ✅ BIEN: Idempotente
def load_data(**context):
    execution_date = context['ds']

    cursor.execute(f"""
        DELETE FROM table WHERE date = '{execution_date}';
        INSERT INTO table SELECT * FROM staging WHERE date = '{execution_date}';
    """)

    # O usar MERGE/UPSERT
```

### 3. Configuración Dinámica

```python
# ❌ MAL: Hardcoded
def extract():
    conn = connect(user='admin', password='pass123', host='prod-db.com')

# ✅ BIEN: Usar Airflow Connections y Variables
from airflow.hooks.base import BaseHook

def extract():
    connection = BaseHook.get_connection('snowflake_prod')
    conn = connect(
        user=connection.login,
        password=connection.password,
        account=connection.host
    )
```

### 4. Testing

```python
# test_dags.py
import pytest
from airflow.models import DagBag

def test_no_import_errors():
    """Verificar que todos los DAGs se importan sin errores"""
    dag_bag = DagBag(include_examples=False)
    assert len(dag_bag.import_errors) == 0, f"Errors: {dag_bag.import_errors}"

def test_dag_structure():
    """Verificar estructura de DAG específico"""
    dag_bag = DagBag(include_examples=False)
    dag = dag_bag.get_dag('sales_etl')

    assert dag is not None
    assert len(dag.tasks) == 5
    assert dag.schedule_interval == '@daily'

def test_task_dependencies():
    """Verificar dependencias correctas"""
    dag_bag = DagBag()
    dag = dag_bag.get_dag('sales_etl')

    extract_task = dag.get_task('extract_data')
    assert extract_task.downstream_task_ids == {'transform_data'}
```

### 5. Documentation

```python
# ✅ DAG bien documentado
dag = DAG(
    'sales_etl_v2',
    description='''
        ETL diario para procesar ventas de e-commerce.

        Extrae datos de Shopify API, transforma con Pandas,
        y carga a Snowflake warehouse.

        Dependencias:
        - Requiere que customers_etl se ejecute primero
        - Usa S3 como staging area

        Alertas: #data-alerts en Slack
        On-call: data-team@company.com

        Última modificación: 2025-01-15 por @jose
    ''',
    tags=['production', 'sales', 'critical'],
    doc_md='''
    ## Sales ETL Pipeline

    ### Architecture
    ```
    Shopify API → S3 (staging) → Snowflake (raw) → Snowflake (analytics)
    ```

    ### SLA
    - Debe completarse antes de las 6 AM
    - Procesa ~1M registros/día

    ### Runbook
    Si falla:
    1. Revisar logs en CloudWatch
    2. Verificar credenciales API en Secrets Manager
    3. Escalar a @data-lead si persiste
    ''',
    dag=dag,
)

# Documentar tareas
extract_task = PythonOperator(
    task_id='extract_from_shopify',
    doc_md='''
    ### Extract Task

    Extrae órdenes del día anterior de Shopify API.

    **Parámetros:**
    - `date`: Fecha de ejecución ({{ ds }})
    - `endpoint`: /admin/api/2024-01/orders.json

    **Output:**
    - S3 path en XCom key 'raw_data_path'
    ''',
    dag=dag,
)
```

---

## Comparación de Herramientas

| Feature | Airflow | Prefect | Dagster | Step Functions |
|---------|---------|---------|---------|----------------|
| **Curva de aprendizaje** | Alta | Media | Media | Baja |
| **Deployment** | Complejo | Simple | Media | Muy simple (AWS) |
| **Dynamic pipelines** | Limitado | Excelente | Excelente | Limitado |
| **Testing** | Manual | Built-in | Excelente | Difícil |
| **Observabilidad** | Buena | Excelente | Excelente | Buena |
| **Escalabilidad** | Excelente | Buena | Buena | Excelente |
| **Costo** | Open source | Freemium | Open source | Pay-per-use |
| **Mejor para** | Batch ETL enterprise | Hybrid cloud/local | Data lakehouse | AWS ecosystems |

---

## 🎯 Preguntas Comunes de Entrevista

### Nivel Mid

**P: ¿Qué es un DAG y por qué debe ser acíclico?**

R: Un DAG (Directed Acyclic Graph) es un grafo dirigido sin ciclos. En orquestación, representa un workflow donde:
- **Directed**: Las tareas tienen orden (A → B → C)
- **Acyclic**: No hay loops infinitos (C no puede volver a A)

Debe ser acíclico para garantizar que el pipeline termine y para poder determinar el orden de ejecución.

**P: ¿Cómo manejarías una tarea que falla intermitentemente?**

R: Implementaría:
1. **Retry con exponential backoff**: 30s, 1m, 2m, 4m
2. **Idempotencia**: La tarea puede re-ejecutarse sin efectos secundarios
3. **Circuit breaker**: Pausar DAG después de N fallos consecutivos
4. **Fallback**: Tarea alternativa si la principal falla consistentemente

**P: ¿Cuál es la diferencia entre `catchup=True` y `catchup=False`?**

R:
- `catchup=True`: Ejecuta todas las fechas desde `start_date` hasta hoy (backfill automático)
- `catchup=False`: Solo ejecuta desde hoy en adelante, ignora fechas pasadas

Usa `catchup=False` para nuevos DAGs si no necesitas datos históricos.

### Nivel Senior

**P: Diseña una estrategia de backfill para 5 años de datos históricos (1.8 TB)**

R:
```
1. Chunked backfill: Dividir en chunks mensuales (60 chunks)
2. Parallel execution: Procesar 5 chunks a la vez (max_active_runs=5)
3. Checkpoint/resume: Guardar progreso cada chunk
4. Rate limiting: 10 req/s para no sobrecargar API
5. Validation: Query post-backfill para detectar gaps
6. Incremental loads: Usar MERGE en vez de full refresh
```

**P: ¿Cómo optimizarías un DAG que tarda 6 horas cuando el SLA es 2 horas?**

R:
1. **Paralelización**: Identificar tareas independientes y ejecutar concurrentemente
2. **Incrementalidad**: Procesar solo datos nuevos en vez de full refresh
3. **Caching**: Guardar resultados intermedios en XCom o S3
4. **Broadcast joins**: En Spark, hacer broadcast de tablas pequeñas
5. **Partitioning**: Particionar datos por fecha para filtrar eficientemente
6. **Executor upgrade**: Cambiar de LocalExecutor a CeleryExecutor con más workers

**P: Explica cómo implementarías exactly-once semantics en un pipeline streaming con Kafka + Snowflake**

R:
```python
# 1. Kafka: Usar transactional producer
producer = KafkaProducer(
    transactional_id='my-transactional-id',
    enable_idempotence=True
)

# 2. Consumer: Guardar offset en Snowflake transaccionalmente
def consume_and_load():
    for message in consumer:
        conn.cursor().execute("BEGIN")

        # Insertar data
        conn.cursor().execute("INSERT INTO table VALUES (...)")

        # Guardar offset en misma transacción
        conn.cursor().execute(f"""
            MERGE INTO kafka_offsets
            USING (SELECT {message.partition}, {message.offset}) source
            ON target.partition = source.partition
            WHEN MATCHED THEN UPDATE SET offset = source.offset
            WHEN NOT MATCHED THEN INSERT VALUES (source.*)
        """)

        conn.cursor().execute("COMMIT")

# 3. Restart: Leer offset desde Snowflake
last_offset = conn.cursor().execute(
    "SELECT offset FROM kafka_offsets WHERE partition = 0"
).fetchone()[0]

consumer.seek(TopicPartition('topic', 0), last_offset + 1)
```

---

## 📚 Recursos Adicionales

- [Airflow Documentation](https://airflow.apache.org/docs/)
- [Prefect Docs](https://docs.prefect.io/)
- [Dagster University](https://dagster.io/learn)
- [AWS Step Functions Workshop](https://catalog.workshops.aws/stepfunctions/)

---

**Siguiente:** [07. Infraestructura Cloud](07_infraestructura_cloud.md)
