# 5. Procesamiento Batch y Streaming

---

## Batch Processing (Procesamiento por Lotes)

Procesar **grandes volúmenes** de datos en **intervalos definidos** (horario, diario, semanal).

### Características

- **Alta latencia:** Minutos a horas
- **Alto throughput:** Procesa millones/billones de registros
- **Datos completos:** Todo el conjunto de datos disponible
- **Recursos predecibles:** Se puede programar en horarios específicos
- **Más simple:** Lógica menos compleja que streaming

### Casos de Uso

- Reportes diarios/mensuales
- ETL nocturno
- Análisis histórico
- Agregaciones complejas
- Machine learning training

---

## Apache Spark

Framework de **procesamiento distribuido** para big data.

### Arquitectura

```
┌─────────────────┐
│     Driver      │  ← Coordina el trabajo
│   (SparkContext)│
└────────┬────────┘
         │
    ┌────┴─────┬─────────┬─────────┐
    │          │         │         │
┌───▼───┐  ┌───▼───┐  ┌──▼───┐  ┌──▼───┐
│Executor│  │Executor│  │Executor│ │Executor│
│ (Worker)  │(Worker)│  │(Worker)│ │(Worker)│
└────────┘  └────────┘  └───────┘  └───────┘
```

### Conceptos Básicos

#### RDD (Resilient Distributed Dataset)

Colección **inmutable y distribuida** de datos.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("MyApp").getOrCreate()

# Crear RDD
rdd = spark.sparkContext.parallelize([1, 2, 3, 4, 5])

# Transformaciones (lazy)
rdd2 = rdd.map(lambda x: x * 2)  # [2, 4, 6, 8, 10]
rdd3 = rdd2.filter(lambda x: x > 5)  # [6, 8, 10]

# Acción (ejecuta las transformaciones)
result = rdd3.collect()  # [6, 8, 10]
print(result)
```

#### DataFrame (Más común)

API de alto nivel similar a pandas pero distribuida.

```python
# Leer CSV
df = spark.read.csv("s3://bucket/data.csv", header=True, inferSchema=True)

# Ver schema
df.printSchema()
# root
#  |-- customer_id: integer
#  |-- name: string
#  |-- amount: double

# Transformaciones
result = (df
    .filter(df.amount > 100)
    .groupBy("customer_id")
    .agg({"amount": "sum"})
    .orderBy("sum(amount)", ascending=False)
)

# Mostrar
result.show(10)

# Escribir
result.write.parquet("s3://bucket/output/")
```

### Transformations vs Actions

#### Transformations (Lazy - no ejecutan inmediatamente)

```python
# Transformations
df2 = df.select("customer_id", "amount")  # Lazy
df3 = df2.filter(df2.amount > 100)        # Lazy
df4 = df3.groupBy("customer_id").sum()    # Lazy

# Nada se ejecuta hasta aquí ↑
```

#### Actions (Trigger execution)

```python
# Actions - ejecutan el plan de transformaciones
df4.show()           # Muestra resultados
df4.count()          # Cuenta filas
df4.collect()        # Trae todo a memoria (¡cuidado!)
df4.write.parquet()  # Escribe a disco
```

**Ventaja de lazy evaluation:**
- Spark optimiza el plan completo antes de ejecutar
- Evita cálculos innecesarios

### Ejemplo Completo: ETL en Spark

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, avg, max, min, count, when

# Inicializar Spark
spark = SparkSession.builder \
    .appName("Sales ETL") \
    .config("spark.sql.adaptive.enabled", "true") \
    .getOrCreate()

# Extract
orders = spark.read.parquet("s3://raw/orders/")
customers = spark.read.parquet("s3://raw/customers/")
products = spark.read.parquet("s3://raw/products/")

# Transform

# 1. Limpieza
orders_clean = orders \
    .filter(col("order_date").isNotNull()) \
    .filter(col("amount") > 0) \
    .dropDuplicates(["order_id"])

# 2. Enriquecimiento (joins)
enriched = orders_clean \
    .join(customers, "customer_id", "left") \
    .join(products, "product_id", "left")

# 3. Agregaciones
daily_summary = enriched \
    .groupBy("order_date", "product_category") \
    .agg(
        count("order_id").alias("num_orders"),
        sum("amount").alias("total_revenue"),
        avg("amount").alias("avg_order_value"),
        count("customer_id").alias("unique_customers")
    )

# 4. Business logic
result = daily_summary.withColumn(
    "revenue_tier",
    when(col("total_revenue") > 10000, "High")
    .when(col("total_revenue") > 5000, "Medium")
    .otherwise("Low")
)

# Load
result.write \
    .mode("overwrite") \
    .partitionBy("order_date") \
    .parquet("s3://processed/daily_sales/")

spark.stop()
```

### Partitioning en Spark

**Particiones** = divisiones lógicas de datos para procesamiento paralelo.

```python
# Ver número de particiones
df.rdd.getNumPartitions()  # 200 (default)

# Repartition (shuffle completo)
df_repartitioned = df.repartition(100)

# Coalesce (reduce sin shuffle)
df_coalesced = df.coalesce(10)

# Partition por columna (para writes)
df.write.partitionBy("year", "month").parquet("output/")
```

**Resultado:**
```
output/
├── year=2024/
│   ├── month=01/
│   │   └── part-00000.parquet
│   └── month=02/
│       └── part-00000.parquet
```

### Optimización de Spark

#### 1. Broadcast Join

Para **joins con tabla pequeña** (< 10 MB).

```python
from pyspark.sql.functions import broadcast

# Tabla grande JOIN tabla pequeña
result = large_table.join(
    broadcast(small_table),  # Envía a todos los workers
    "customer_id"
)
```

#### 2. Cache/Persist

Guardar **datos en memoria** para reutilizar.

```python
# Leer una vez
df = spark.read.parquet("s3://data/")

# Cachear si se usa múltiples veces
df.cache()  # O df.persist()

# Usar múltiples veces (sin re-leer)
df.filter(col("amount") > 100).count()
df.groupBy("category").sum().show()

# Liberar memoria
df.unpersist()
```

#### 3. Configuración

```python
spark = SparkSession.builder \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
    .config("spark.sql.shuffle.partitions", "200") \
    .config("spark.executor.memory", "4g") \
    .config("spark.executor.cores", "4") \
    .getOrCreate()
```

### AWS Glue

**Servicio managed** de Spark en AWS.

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

# Inicializar
args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Leer desde Catalog
datasource = glueContext.create_dynamic_frame.from_catalog(
    database = "my_database",
    table_name = "orders"
)

# Transformar
transformed = ApplyMapping.apply(
    frame = datasource,
    mappings = [
        ("order_id", "long", "order_id", "long"),
        ("customer_id", "long", "customer_id", "long"),
        ("amount", "double", "amount", "double")
    ]
)

# Escribir
glueContext.write_dynamic_frame.from_options(
    frame = transformed,
    connection_type = "s3",
    connection_options = {"path": "s3://bucket/output/"},
    format = "parquet"
)

job.commit()
```

### Google Cloud Dataproc

**Cluster managed** de Spark en GCP.

```bash
# Crear cluster
gcloud dataproc clusters create my-cluster \
    --region us-central1 \
    --num-workers 4

# Submit job
gcloud dataproc jobs submit pyspark \
    --cluster my-cluster \
    --region us-central1 \
    gs://my-bucket/spark_job.py
```

---

## Streaming Processing (Procesamiento en Tiempo Real)

Procesar datos **continuamente** a medida que llegan.

### Características

- **Baja latencia:** Milisegundos a segundos
- **Procesamiento continuo:** 24/7
- **Datos incrementales:** Procesa eventos uno por uno o en micro-batches
- **Complejidad mayor:** Manejo de estado, ventanas, late data
- **Casos de uso:** Alertas, fraud detection, dashboards real-time

---

## Apache Kafka

**Plataforma de streaming distribuida** para publicar/consumir eventos.

### Arquitectura

```
┌──────────────┐
│  Producers   │  ← Publican mensajes
└──────┬───────┘
       │
       ▼
┌─────────────────────────────────┐
│         Kafka Cluster           │
│  ┌─────────┐  ┌─────────┐       │
│  │ Broker 1│  │ Broker 2│  ...  │
│  └─────────┘  └─────────┘       │
│                                 │
│  Topic: orders                  │
│  ├── Partition 0 [===msgs===]   │
│  ├── Partition 1 [===msgs===]   │
│  └── Partition 2 [===msgs===]   │
└──────────┬──────────────────────┘
           │
           ▼
    ┌──────────────┐
    │  Consumers   │  ← Leen mensajes
    └──────────────┘
```

### Conceptos Básicos

**Topic:** Categoría de mensajes (ej: "orders", "user-clicks")
**Partition:** División de un topic para paralelismo
**Producer:** Publica mensajes a topics
**Consumer:** Lee mensajes de topics
**Consumer Group:** Grupo de consumers que se dividen el trabajo

### Ejemplo: Producer

```python
from kafka import KafkaProducer
import json

# Crear producer
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Enviar mensaje
order = {
    "order_id": 123,
    "customer_id": 456,
    "amount": 99.99,
    "timestamp": "2024-01-15T10:30:00Z"
}

producer.send('orders', value=order)
producer.flush()
```

### Ejemplo: Consumer

```python
from kafka import KafkaConsumer
import json

# Crear consumer
consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    group_id='order-processor',
    auto_offset_reset='earliest'
)

# Procesar mensajes
for message in consumer:
    order = message.value
    print(f"Processing order {order['order_id']}")

    # Lógica de negocio
    if order['amount'] > 1000:
        send_alert(f"Large order: {order['order_id']}")
```

### Kafka Streams

**Biblioteca** para procesamiento de streams en Kafka.

```java
StreamsBuilder builder = new StreamsBuilder();

// Leer stream
KStream<String, Order> orders = builder.stream("orders");

// Filtrar órdenes grandes
KStream<String, Order> largeOrders = orders.filter(
    (key, order) -> order.getAmount() > 1000
);

// Escribir a otro topic
largeOrders.to("large-orders");
```

### AWS Kinesis

**Servicio managed** de streaming en AWS (similar a Kafka).

```python
import boto3
import json

kinesis = boto3.client('kinesis')

# Put record
response = kinesis.put_record(
    StreamName='orders',
    Data=json.dumps({
        "order_id": 123,
        "amount": 99.99
    }),
    PartitionKey='customer_456'
)

# Get records
response = kinesis.get_records(
    ShardIterator=shard_iterator,
    Limit=100
)

for record in response['Records']:
    data = json.loads(record['Data'])
    print(data)
```

---

## Apache Flink

Framework de **streaming real** (no micro-batches).

### Características

- **True streaming:** Procesamiento continuo (no batches)
- **Stateful:** Mantiene estado entre eventos
- **Exactly-once semantics:** Garantías fuertes
- **Low latency:** Milisegundos
- **Fault tolerance:** Checkpoints automáticos

### Ejemplo

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors import FlinkKafkaConsumer

env = StreamExecutionEnvironment.get_execution_environment()

# Leer desde Kafka
kafka_consumer = FlinkKafkaConsumer(
    topics='orders',
    deserialization_schema=...,
    properties={'bootstrap.servers': 'localhost:9092'}
)

stream = env.add_source(kafka_consumer)

# Procesar
stream \
    .filter(lambda order: order['amount'] > 100) \
    .key_by(lambda order: order['customer_id']) \
    .sum('amount') \
    .print()

env.execute("Order Processing")
```

---

## Spark Streaming

**Micro-batch streaming** con Spark.

### Structured Streaming

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, window, avg

spark = SparkSession.builder.appName("StreamApp").getOrCreate()

# Leer stream desde Kafka
orders_stream = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "orders") \
    .load()

# Parsear JSON
from pyspark.sql.functions import from_json
from pyspark.sql.types import StructType, StructField, IntegerType, DoubleType

schema = StructType([
    StructField("order_id", IntegerType()),
    StructField("customer_id", IntegerType()),
    StructField("amount", DoubleType())
])

parsed = orders_stream.select(
    from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")

# Agregación por ventana de tiempo
windowed = parsed \
    .withWatermark("timestamp", "10 minutes") \
    .groupBy(
        window(col("timestamp"), "5 minutes"),
        col("customer_id")
    ) \
    .agg(avg("amount").alias("avg_amount"))

# Escribir stream
query = windowed \
    .writeStream \
    .outputMode("update") \
    .format("console") \
    .start()

query.awaitTermination()
```

---

## Window Functions en Streams

Agrupar eventos por **ventanas de tiempo**.

### Tipos de Ventanas

#### 1. Tumbling Window (No overlap)

```
[0-5min] [5-10min] [10-15min]
   ▓▓▓      ▓▓▓        ▓▓▓
```

```python
# Spark Streaming
windowed = stream.groupBy(
    window(col("timestamp"), "5 minutes")  # Ventana de 5 min
).count()
```

#### 2. Sliding Window (Overlap)

```
[0-5min]
  [2-7min]
    [4-9min]
```

```python
windowed = stream.groupBy(
    window(col("timestamp"), "5 minutes", "2 minutes")  # 5 min window, cada 2 min
).count()
```

#### 3. Session Window

Ventana basada en **gaps de inactividad**.

```
Usuario activo → 10 min inactivo → Nueva sesión
[====eventos====]   (gap)   [====eventos====]
    Sesión 1                   Sesión 2
```

### Watermarks

**Manejo de eventos tardíos** (late data).

```python
windowed = stream \
    .withWatermark("timestamp", "10 minutes") \  # Espera hasta 10 min de retraso
    .groupBy(window(col("timestamp"), "5 minutes")) \
    .count()
```

**Ejemplo:**
```
Watermark = Max Event Time - 10 minutes

Evento llega con timestamp 10:00 → Watermark = 9:50
Evento llega con timestamp 9:45 → Descartado (muy tarde)
```

---

## Event-Driven Architecture

Arquitectura basada en **eventos** y **reacciones**.

```
┌──────────┐
│   User   │
│  Action  │
└────┬─────┘
     │ event
     ▼
┌─────────────┐
│   Kafka     │
│   Topic     │
└──┬──────┬───┘
   │      │
   ▼      ▼
┌────┐  ┌────────┐
│Svc1│  │  Svc2  │  ← Consumers reaccionan a eventos
└────┘  └────────┘
```

### Ejemplo: E-commerce

```python
# Event: Order Created
{
    "event_type": "order.created",
    "order_id": 123,
    "customer_id": 456,
    "amount": 99.99,
    "timestamp": "2024-01-15T10:30:00Z"
}

# Consumers
# 1. Payment Service → Procesa pago
# 2. Inventory Service → Reduce stock
# 3. Notification Service → Envía email
# 4. Analytics Service → Actualiza métricas
```

**Ventajas:**
- ✅ Desacoplamiento (servicios independientes)
- ✅ Escalabilidad (cada servicio escala por separado)
- ✅ Resilience (si un servicio falla, otros siguen)

---

## Lambda vs Kappa Architecture

### Lambda Architecture

**Dos capas:** Batch (histórico) + Streaming (real-time)

```
           ┌──────────┐
           │  Source  │
           └────┬─────┘
                │
        ┌───────┴────────┐
        │                │
        ▼                ▼
┌───────────┐    ┌──────────────┐
│  Batch    │    │  Streaming   │
│  Layer    │    │    Layer     │
│ (Spark)   │    │   (Kafka)    │
└─────┬─────┘    └──────┬───────┘
      │                 │
      └────────┬────────┘
               │
               ▼
       ┌───────────────┐
       │ Serving Layer │
       │   (Query)     │
       └───────────────┘
```

**Ventajas:**
- ✅ Corrección garantizada (batch recomputa todo)
- ✅ Baja latencia (streaming para datos recientes)

**Desventajas:**
- ❌ Complejidad (dos pipelines diferentes)
- ❌ Duplicación de lógica

### Kappa Architecture

**Solo streaming** (simplificación de Lambda).

```
┌──────────┐
│  Source  │
└────┬─────┘
     │
     ▼
┌──────────────┐
│  Streaming   │
│   Layer      │
│  (Kafka +    │
│   Flink)     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Serving Layer│
└──────────────┘
```

**Ventajas:**
- ✅ Más simple (un solo pipeline)
- ✅ Una sola lógica de negocio

**Desventajas:**
- ❌ Reprocessing más complejo

---

## Casos de Uso por Tipo

### Batch
- ✅ Reportes diarios/mensuales
- ✅ ETL histórico
- ✅ ML training
- ✅ Data warehouse loads
- ✅ Analytics complejos

### Streaming
- ✅ Fraud detection
- ✅ Real-time dashboards
- ✅ Alertas inmediatas
- ✅ Recommendations en vivo
- ✅ IoT data processing
- ✅ Log monitoring

---

## Resumen

✅ **Batch** (Spark) para grandes volúmenes históricos con alta latencia tolerada
✅ **Streaming** (Kafka, Flink) para datos en tiempo real con baja latencia
✅ **Spark** es estándar para batch processing distribuido
✅ **Kafka** es estándar para message streaming
✅ **Flink** para true streaming, **Spark Streaming** para micro-batches
✅ **Window functions** para agregaciones temporales en streams
✅ **Lambda** (batch + streaming) vs **Kappa** (solo streaming)

**Siguiente:** [06. Orquestación y Automatización](06_orquestacion_automatizacion.md)
