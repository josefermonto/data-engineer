# 8. Optimización y Performance en Data Engineering

## 📋 Tabla de Contenidos
- [Fundamentos de Performance](#fundamentos-de-performance)
- [Optimización de Queries SQL](#optimización-de-queries-sql)
- [Optimización de Spark](#optimización-de-spark)
- [Partitioning Strategies](#partitioning-strategies)
- [Indexing y Clustering](#indexing-y-clustering)
- [Caching y Materialized Views](#caching-y-materialized-views)
- [Data Compression](#data-compression)
- [Profiling y Monitoring](#profiling-y-monitoring)

---

## Fundamentos de Performance

### Métricas Clave

| Métrica | Descripción | Objetivo |
|---------|-------------|----------|
| **Latency** | Tiempo de respuesta | <1s para dashboards, <5min para ETL |
| **Throughput** | Datos procesados/segundo | Maximizar (GB/s, registros/s) |
| **Resource Utilization** | CPU, memoria, I/O | 60-80% (evitar saturación) |
| **Cost per Query** | $ por query/job | Minimizar |
| **Data Freshness** | Retraso de datos | <5min para real-time, <1h para batch |

### Ley de Amdahl

```
Speedup total = 1 / ((1 - P) + P/S)

P = Porción paralelizable del código
S = Speedup de esa porción

Ejemplo:
- 90% del código es paralelizable (P=0.9)
- Se paraleliza con 10 cores (S=10)
- Speedup = 1 / (0.1 + 0.9/10) = 5.26x

Conclusión: Enfócate en optimizar el cuello de botella
```

### Bottlenecks Comunes

```
┌────────────────────────────────────────────┐
│         BOTTLENECK ANALYSIS                │
├────────────────────────────────────────────┤
│                                            │
│  1. CPU-bound                              │
│     • Transformaciones complejas           │
│     • Agregaciones en memoria              │
│     • Solución: Más workers, vectorización │
│                                            │
│  2. I/O-bound                              │
│     • Lectura/escritura disco              │
│     • Network transfer                     │
│     • Solución: Compresión, caching        │
│                                            │
│  3. Memory-bound                           │
│     • Joins de tablas grandes              │
│     • Group by con alta cardinalidad       │
│     • Solución: Spill to disk, partitioning│
│                                            │
│  4. Network-bound                          │
│     • Shuffles en Spark                    │
│     • Data transfer cross-region           │
│     • Solución: Broadcast, co-location     │
└────────────────────────────────────────────┘
```

---

## Optimización de Queries SQL

### 1. Query Planning y Execution

```sql
-- ===== EXPLAIN PLAN =====
-- Ver plan de ejecución antes de ejecutar

-- PostgreSQL/Redshift
EXPLAIN ANALYZE
SELECT
    c.customer_name,
    SUM(o.amount) as total
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2025-01-01'
GROUP BY c.customer_name;

/*
Output:
HashAggregate  (cost=1000..1500 rows=100)
  Group Key: c.customer_name
  ->  Hash Join  (cost=500..800 rows=10000)
        Hash Cond: (o.customer_id = c.customer_id)
        ->  Seq Scan on orders o  (cost=0..300 rows=10000)
              Filter: (order_date >= '2025-01-01')
        ->  Hash  (cost=200..200 rows=5000)
              ->  Seq Scan on customers c  (cost=0..200 rows=5000)
*/

-- BigQuery
-- Ve plan en UI después de ejecutar query

-- Snowflake
EXPLAIN USING TEXT
SELECT ...;
```

### 2. JOIN Optimization

```sql
-- ===== MAL: Join sin filtros =====
SELECT
    c.customer_name,
    o.order_id,
    o.amount
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2025-01-01';  -- Filtro DESPUÉS del join

-- Problema: Join procesa TODAS las órdenes, luego filtra
-- Si orders tiene 100M registros, join procesa 100M

-- ===== BIEN: Filtro antes del join =====
SELECT
    c.customer_name,
    o.order_id,
    o.amount
FROM customers c
JOIN (
    SELECT *
    FROM orders
    WHERE order_date >= '2025-01-01'  -- Filtro ANTES del join
) o ON c.customer_id = o.customer_id;

-- Join solo procesa registros filtrados (ej: 1M)
-- 100x más rápido

-- ===== MEJOR: CTEs legibles =====
WITH recent_orders AS (
    SELECT
        customer_id,
        order_id,
        amount
    FROM orders
    WHERE order_date >= '2025-01-01'
        AND amount > 0  -- Filtros adicionales
)
SELECT
    c.customer_name,
    o.order_id,
    o.amount
FROM customers c
JOIN recent_orders o ON c.customer_id = o.customer_id;

-- ===== JOIN ORDER importa =====
-- Regla: Tabla pequeña primero (drive table)

-- Mal (tabla grande primero)
SELECT *
FROM orders o  -- 100M rows
JOIN customers c ON o.customer_id = c.customer_id;  -- 1M rows

-- Bien (tabla pequeña primero)
SELECT *
FROM customers c  -- 1M rows
JOIN orders o ON c.customer_id = o.customer_id;  -- 100M rows

-- En muchos sistemas modernos, el optimizer hace esto automáticamente
-- Pero en queries complejos, el orden manual puede ayudar
```

### 3. Agregaciones Eficientes

```sql
-- ===== MAL: Agregaciones anidadas =====
SELECT
    customer_id,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) as order_count,
    (SELECT SUM(amount) FROM orders WHERE customer_id = c.customer_id) as total_spent
FROM customers c;

-- Problema: Subconsulta por cada customer → N queries
-- 1M customers = 2M subqueries adicionales

-- ===== BIEN: Una agregación =====
SELECT
    c.customer_id,
    COUNT(o.order_id) as order_count,
    COALESCE(SUM(o.amount), 0) as total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id;

-- 1 query vs 2M queries → 1000x+ más rápido

-- ===== DISTINCT vs GROUP BY =====
-- Mal (puede ser lento)
SELECT DISTINCT customer_id FROM orders;

-- Mejor (optimizer puede usar índice)
SELECT customer_id FROM orders GROUP BY customer_id;

-- Aún mejor (si solo necesitas contar)
SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id;

-- ===== APROXIMACIONES para datasets grandes =====
-- Exacto (lento en 100M+ registros)
SELECT COUNT(DISTINCT customer_id) FROM orders;

-- Aproximado (error <2%, 10x más rápido)
-- BigQuery
SELECT APPROX_COUNT_DISTINCT(customer_id) FROM orders;

-- PostgreSQL (extensión)
SELECT hll_cardinality(hll_add_agg(hll_hash_text(customer_id))) FROM orders;

-- Redshift
SELECT APPROXIMATE COUNT(DISTINCT customer_id) FROM orders;
```

### 4. Window Functions Optimization

```sql
-- ===== MAL: Múltiples window functions con distintas particiones =====
SELECT
    order_id,
    customer_id,
    order_date,
    amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as customer_order_num,
    SUM(amount) OVER (PARTITION BY YEAR(order_date)) as yearly_total,
    AVG(amount) OVER (PARTITION BY product_id ORDER BY order_date) as product_avg
FROM orders;

-- Problema: 3 diferentes particiones → 3 sorts/shuffles

-- ===== MEJOR: Agrupar window functions con misma partición =====
WITH customer_windows AS (
    SELECT
        order_id,
        customer_id,
        order_date,
        amount,
        product_id,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as customer_order_num
    FROM orders
),
yearly_totals AS (
    SELECT
        YEAR(order_date) as year,
        SUM(amount) as yearly_total
    FROM orders
    GROUP BY YEAR(order_date)
),
product_avgs AS (
    SELECT
        product_id,
        order_date,
        AVG(amount) OVER (PARTITION BY product_id ORDER BY order_date) as product_avg
    FROM orders
)
SELECT
    cw.*,
    yt.yearly_total,
    pa.product_avg
FROM customer_windows cw
JOIN yearly_totals yt ON YEAR(cw.order_date) = yt.year
JOIN product_avgs pa ON cw.product_id = pa.product_id AND cw.order_date = pa.order_date;

-- ===== FRAME CLAUSES para optimizar =====
-- Mal (default frame: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
SELECT
    order_date,
    amount,
    AVG(amount) OVER (ORDER BY order_date) as running_avg
FROM orders;

-- Mejor (especificar frame exacto)
SELECT
    order_date,
    amount,
    AVG(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW  -- Solo últimas 7 filas
    ) as weekly_avg
FROM orders;
```

### 5. Subquery Optimization

```sql
-- ===== MAL: Subquery en WHERE (ejecuta por cada fila) =====
SELECT *
FROM orders o
WHERE o.customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE country = 'US'
);

-- Problema: Subquery puede ejecutarse múltiples veces

-- ===== MEJOR: JOIN =====
SELECT o.*
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.country = 'US';

-- ===== O usar SEMI JOIN explícito =====
SELECT o.*
FROM orders o
WHERE EXISTS (
    SELECT 1
    FROM customers c
    WHERE c.customer_id = o.customer_id
        AND c.country = 'US'
);

-- EXISTS es más eficiente que IN cuando hay duplicados
-- porque EXISTS puede terminar en el primer match

-- ===== MAL: Correlated subquery =====
SELECT
    c.customer_id,
    c.customer_name,
    (
        SELECT MAX(order_date)
        FROM orders o
        WHERE o.customer_id = c.customer_id  -- Correlacionada
    ) as last_order_date
FROM customers c;

-- Ejecuta subquery por cada customer → N queries

-- ===== BIEN: LEFT JOIN con agregación =====
SELECT
    c.customer_id,
    c.customer_name,
    MAX(o.order_date) as last_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name;

-- 1 query, 100x+ más rápido
```

---

## Optimización de Spark

### 1. Partitioning en Spark

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, avg, count

spark = SparkSession.builder \
    .appName("Optimization") \
    .config("spark.sql.adaptive.enabled", "true") \
    .getOrCreate()

# ===== PROBLEMA: Default partitions inadecuado =====
df = spark.read.parquet("s3://data/sales/")
print(f"Partitions: {df.rdd.getNumPartitions()}")  # Ej: 2000 partitions

# Si cada partition tiene 1 KB → overhead de 2000 tasks
# Si cada partition tiene 10 GB → OOM (out of memory)

# ===== SOLUCIÓN: Repartition adecuado =====
# Regla: 100-200 MB por partition

# Si dataset es 100 GB:
# 100 GB / 128 MB = ~800 partitions ideal
df = df.repartition(800)

# O calcular dinámicamente
import math
size_gb = 100
target_partition_mb = 128
num_partitions = math.ceil((size_gb * 1024) / target_partition_mb)
df = df.repartition(num_partitions)

# ===== COALESCE vs REPARTITION =====
# COALESCE: Reduce partitions sin shuffle (más rápido)
df = df.coalesce(100)  # De 2000 a 100, sin shuffle

# REPARTITION: Full shuffle (más lento pero balancea mejor)
df = df.repartition(800)  # Balancea datos uniformemente

# Regla: Usa coalesce cuando reduces, repartition cuando aumentas

# ===== REPARTITION por columna (para joins/groups) =====
# Si harás join por customer_id, reparticiona por esa key
df_customers = spark.read.parquet("s3://data/customers/")
df_orders = spark.read.parquet("s3://data/orders/")

# Repartition ambas por join key
df_customers = df_customers.repartition(200, "customer_id")
df_orders = df_orders.repartition(200, "customer_id")

# Join sin shuffle adicional (co-located)
result = df_customers.join(df_orders, "customer_id")
```

### 2. Broadcast Joins

```python
from pyspark.sql.functions import broadcast

# ===== PROBLEMA: Shuffle join =====
large_df = spark.read.parquet("s3://data/orders/")  # 100 GB
small_df = spark.read.parquet("s3://data/products/")  # 10 MB

result = large_df.join(small_df, "product_id")

# Problema: Spark shuffle ambas tablas (lento, costoso)
# Shuffle de 100 GB por la red

# ===== SOLUCIÓN: Broadcast join =====
result = large_df.join(
    broadcast(small_df),  # Envía tabla pequeña a todos los executors
    "product_id"
)

# No shuffle de large_df → 100x más rápido
# Límite: Tabla broadcast < 10 GB (configurable)

# Configurar threshold automático
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100MB")

# Spark auto-detecta tablas <100MB y hace broadcast
result = large_df.join(small_df, "product_id")  # Auto broadcast

# ===== VERIFICAR: Broadcast en query plan =====
result.explain()
"""
== Physical Plan ==
...
BroadcastHashJoin [product_id]  ← Broadcast join
...
"""
```

### 3. Caching Strategies

```python
# ===== CUÁNDO usar cache/persist =====
# 1. DataFrame usado múltiples veces
# 2. Operaciones costosas (shuffle, join)
# 3. Tienes suficiente memoria

df = spark.read.parquet("s3://data/sales/") \
    .filter(col("order_date") >= "2025-01-01") \
    .join(customers, "customer_id")  # Join costoso

# ===== MAL: Sin cache =====
result1 = df.groupBy("country").sum("amount")  # Ejecuta todo
result2 = df.groupBy("product").count()        # Ejecuta todo DE NUEVO
result3 = df.filter(col("amount") > 100)       # Ejecuta todo OTRA VEZ

# ===== BIEN: Cache después de operaciones costosas =====
df.cache()  # O df.persist()

result1 = df.groupBy("country").sum("amount")  # Ejecuta y cachea
result2 = df.groupBy("product").count()        # Lee de cache (rápido)
result3 = df.filter(col("amount") > 100)       # Lee de cache

# Liberar cache cuando ya no se necesita
df.unpersist()

# ===== STORAGE LEVELS =====
from pyspark import StorageLevel

# MEMORY_ONLY (default de .cache())
df.persist(StorageLevel.MEMORY_ONLY)
# Pros: Más rápido
# Contras: Si no cabe, evict partitions (re-computa)

# MEMORY_AND_DISK
df.persist(StorageLevel.MEMORY_AND_DISK)
# Pros: No re-computa (spill to disk)
# Contras: Disk I/O más lento

# MEMORY_AND_DISK_SER (serialized)
df.persist(StorageLevel.MEMORY_AND_DISK_SER)
# Pros: Usa menos memoria (comprimido)
# Contras: Overhead de deserialización

# OFF_HEAP (para large datasets)
df.persist(StorageLevel.OFF_HEAP)
# Pros: No afecta GC de JVM
# Contras: Requiere configuración extra

# Regla general:
# - Dataset < 50% memoria → MEMORY_ONLY
# - Dataset > 50% memoria → MEMORY_AND_DISK_SER
```

### 4. Evitar Shuffles

```python
# ===== SHUFFLE es la operación más costosa =====
# Operaciones que causan shuffle:
# - repartition(), coalesce(increase)
# - join(), groupBy(), distinct()
# - sortBy(), orderBy()

# ===== ESTRATEGIA 1: Pre-partition antes de múltiples operaciones =====
df = spark.read.parquet("s3://data/orders/")

# Mal: Shuffle en cada operación
df.groupBy("customer_id").sum("amount")  # Shuffle 1
df.groupBy("customer_id").avg("amount")  # Shuffle 2
df.groupBy("customer_id").count()        # Shuffle 3

# Bien: Un shuffle, múltiples agregaciones
from pyspark.sql.functions import sum, avg, count

df.groupBy("customer_id").agg(
    sum("amount").alias("total"),
    avg("amount").alias("average"),
    count("*").alias("count")
)  # 1 shuffle para todo

# ===== ESTRATEGIA 2: Usar mapPartitions para operaciones complejas =====
# Mal: collect() trae todo a driver
data = df.collect()  # OOM si dataset es grande
for row in data:
    process(row)

# Bien: mapPartitions procesa en executors
def process_partition(partition):
    # Procesar batch completo de una partition
    results = []
    for row in partition:
        result = expensive_operation(row)
        results.append(result)
    return iter(results)

df.rdd.mapPartitions(process_partition).toDF()

# ===== ESTRATEGIA 3: Broadcast variables =====
# Mal: Closure captura large object, se serializa por task
lookup_dict = load_large_lookup()  # 1 GB

df.rdd.map(lambda x: lookup_dict.get(x.key))  # 1 GB × N tasks

# Bien: Broadcast una vez
lookup_broadcast = spark.sparkContext.broadcast(lookup_dict)

df.rdd.map(lambda x: lookup_broadcast.value.get(x.key))  # 1 GB total
```

### 5. Configuraciones Críticas

```python
# ===== MEMORY CONFIGURATION =====
spark = SparkSession.builder \
    .appName("Optimized Job") \
    .config("spark.executor.memory", "16g") \
    .config("spark.executor.cores", "4") \
    .config("spark.executor.instances", "10") \
    .config("spark.driver.memory", "8g") \
    \
    .config("spark.memory.fraction", "0.8") \
    .config("spark.memory.storageFraction", "0.3") \
    \
    .config("spark.sql.shuffle.partitions", "200") \
    .config("spark.default.parallelism", "200") \
    \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
    .config("spark.sql.adaptive.skewJoin.enabled", "true") \
    \
    .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
    .config("spark.kryo.registrationRequired", "false") \
    \
    .config("spark.sql.files.maxPartitionBytes", "128MB") \
    .config("spark.sql.autoBroadcastJoinThreshold", "10MB") \
    .getOrCreate()

# Explicación:
# executor.memory: 16 GB por executor
# executor.cores: 4 cores por executor (balance CPU/memory)
# memory.fraction: 80% de heap para Spark (20% para user code)
# storageFraction: 30% del Spark memory para cache (70% para execution)
# shuffle.partitions: 200 partitions después de shuffle
# adaptive.enabled: Ajusta partitions automáticamente
# skewJoin.enabled: Maneja data skew en joins

# ===== CÁLCULO DE RECURSOS =====
# Regla: executor.memory * executor.cores * executor.instances
# = 16 GB * 4 cores * 10 executors
# = 640 GB total memory, 40 cores total

# Partitions ideales = 2-3 × total cores
# = 2.5 × 40 = 100 partitions
```

### 6. Data Skew Handling

```python
# ===== PROBLEMA: Data skew =====
# Algunos customer_id tienen 1M órdenes, otros tienen 10
# Resultado: Algunas tasks tardan 1 hora, otras 1 segundo

# ===== DETECTAR SKEW =====
from pyspark.sql.functions import count

skew_check = df.groupBy("customer_id").agg(
    count("*").alias("count")
).orderBy(col("count").desc())

skew_check.show(20)
"""
+-----------+--------+
|customer_id|count   |
+-----------+--------+
|CUST_123   |1000000 | ← Skew!
|CUST_456   |500000  |
|CUST_789   |10      |
+-----------+--------+
"""

# ===== SOLUCIÓN 1: Salting =====
from pyspark.sql.functions import rand, expr

# Agregar salt random a keys con skew
df_salted = df.withColumn(
    "customer_id_salted",
    expr("CONCAT(customer_id, '_', CAST(FLOOR(RAND() * 10) AS STRING))")
)

# Join/group por key salted
result = df_salted.groupBy("customer_id_salted").agg(...)

# Remover salt
result = result.withColumn(
    "customer_id",
    expr("SPLIT(customer_id_salted, '_')[0]")
).drop("customer_id_salted")

# Agregar final
final = result.groupBy("customer_id").agg(...)

# ===== SOLUCIÓN 2: Adaptive Query Execution (Spark 3.0+) =====
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")

# Spark detecta y divide automáticamente partitions con skew

# ===== SOLUCIÓN 3: Separate and Union =====
# Procesar clientes grandes por separado
large_customers = ["CUST_123", "CUST_456"]

df_large = df.filter(col("customer_id").isin(large_customers)) \
    .repartition(100, "customer_id")  # Más partitions

df_small = df.filter(~col("customer_id").isin(large_customers)) \
    .repartition(50, "customer_id")

result_large = df_large.groupBy("customer_id").agg(...)
result_small = df_small.groupBy("customer_id").agg(...)

result = result_large.union(result_small)
```

---

## Partitioning Strategies

### 1. Partitioning en Storage

```python
# ===== ESCRIBIR PARTICIONADO =====
df.write.mode("overwrite") \
    .partitionBy("year", "month", "day") \
    .parquet("s3://data/sales/")

# Resultado:
# s3://data/sales/
#   year=2025/
#     month=01/
#       day=01/
#         part-00000.snappy.parquet
#       day=02/
#         part-00000.snappy.parquet

# ===== BENEFICIOS =====
# 1. Partition pruning: Solo lee particiones relevantes
spark.read.parquet("s3://data/sales/") \
    .filter(col("year") == 2025)  # Solo lee year=2025/

# 2. Parallel writes: Cada partition se escribe independientemente

# ===== ELEGIR PARTITION KEY =====
# ❌ MAL: Alta cardinalidad
.partitionBy("order_id")  # 100M partitions → overhead

# ❌ MAL: Baja cardinalidad
.partitionBy("country")  # 10 partitions → partitions muy grandes

# ✅ BIEN: Cardinalidad media + usado en queries
.partitionBy("order_date")  # 365 partitions/año, filtrado frecuente

# ✅ BIEN: Particionar jerárquicamente
.partitionBy("year", "month", "day")  # Flexible

# ===== BUCKETING (alternativa a partitioning) =====
df.write.bucketBy(100, "customer_id") \
    .sortBy("order_date") \
    .saveAsTable("orders_bucketed")

# Beneficios:
# - Join sin shuffle si ambas tablas usan mismo bucketing
# - Útil cuando partition key tiene alta cardinalidad
```

### 2. Partitioning en Databases

```sql
-- ===== POSTGRESQL: Partitioning por rango =====
CREATE TABLE sales (
    order_id BIGINT,
    order_date DATE,
    amount DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

-- Crear particiones
CREATE TABLE sales_2024 PARTITION OF sales
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE sales_2025 PARTITION OF sales
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Queries automáticamente usan partition pruning
SELECT * FROM sales WHERE order_date >= '2025-01-01';
-- Solo escanea sales_2025

-- ===== SNOWFLAKE: Clustering =====
CREATE TABLE sales (
    order_id VARCHAR,
    customer_id VARCHAR,
    order_date DATE,
    amount NUMBER
)
CLUSTER BY (order_date, customer_id);

-- Snowflake mantiene automáticamente micro-partitions clustered

-- ===== BIGQUERY: Partitioning + Clustering =====
CREATE TABLE `project.dataset.sales`
(
    order_id STRING,
    customer_id STRING,
    order_date DATE,
    amount NUMERIC
)
PARTITION BY order_date
CLUSTER BY customer_id;

-- Partition pruning automático
SELECT * FROM sales
WHERE order_date BETWEEN '2025-01-01' AND '2025-01-31';
-- Solo escanea partición de enero

-- ===== REDSHIFT: Distribution + Sort Keys =====
CREATE TABLE sales (
    order_id VARCHAR(50),
    customer_id VARCHAR(50),
    order_date DATE,
    amount DECIMAL(10,2)
)
DISTSTYLE KEY
DISTKEY (customer_id)  -- Distribuir por customer_id
SORTKEY (order_date);  -- Ordenar por fecha

-- DISTKEY: Controla distribución entre nodes
-- SORTKEY: Mejora queries con filtros/joins por esa columna
```

---

## Indexing y Clustering

### 1. Índices en SQL

```sql
-- ===== TIPOS DE ÍNDICES =====

-- 1. B-Tree Index (default, para equality/range queries)
CREATE INDEX idx_customer_id ON orders(customer_id);

-- Mejora:
SELECT * FROM orders WHERE customer_id = 'CUST_123';
SELECT * FROM orders WHERE customer_id > 'CUST_100';

-- 2. Hash Index (solo equality)
CREATE INDEX idx_email_hash ON customers USING HASH (email);

-- Mejora:
SELECT * FROM customers WHERE email = 'user@example.com';

-- 3. Partial Index (solo subset de datos)
CREATE INDEX idx_active_customers
    ON customers(customer_id)
    WHERE status = 'ACTIVE';

-- Mejora queries solo sobre active customers

-- 4. Composite Index (múltiples columnas)
CREATE INDEX idx_customer_date ON orders(customer_id, order_date);

-- Orden importa: customer_id primero, luego order_date
-- Mejora:
SELECT * FROM orders WHERE customer_id = 'CUST_123' AND order_date >= '2025-01-01';

-- NO mejora (no usa primera columna del índice):
SELECT * FROM orders WHERE order_date >= '2025-01-01';

-- 5. Covering Index (include extra columns)
CREATE INDEX idx_customer_covering
    ON orders(customer_id)
    INCLUDE (amount, order_date);

-- Query puede satisfacerse solo con índice (index-only scan)
SELECT customer_id, amount, order_date
FROM orders
WHERE customer_id = 'CUST_123';

-- ===== CUÁNDO NO crear índice =====
-- ❌ Tablas pequeñas (<1000 rows) → Full scan es más rápido
-- ❌ Columnas con baja cardinalidad (ej: boolean) → No selectivo
-- ❌ Tablas con writes frecuentes → Overhead de mantener índice
-- ❌ Columnas nunca usadas en WHERE/JOIN

-- ===== MONITOREAR USO DE ÍNDICES =====
-- PostgreSQL: Índices no usados
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Nunca usado
    AND indexname NOT LIKE 'pk_%';  -- Ignorar primary keys

-- Redshift: Query monitoring
SELECT
    query,
    is_diskbased,
    workmem,
    query_execution_time
FROM svl_query_summary
WHERE is_diskbased = 't';  -- Queries que spill to disk
```

### 2. Clustering

```sql
-- ===== SNOWFLAKE: Automatic clustering =====
CREATE TABLE sales (
    order_id VARCHAR,
    customer_id VARCHAR,
    order_date DATE,
    amount NUMBER
)
CLUSTER BY (order_date);

-- Snowflake re-organiza micro-partitions automáticamente
-- Queries con filter por order_date → pruning eficiente

-- Verificar clustering
SELECT SYSTEM$CLUSTERING_INFORMATION('sales');

-- ===== BIGQUERY: Clustering (max 4 columnas) =====
CREATE TABLE `project.dataset.sales`
PARTITION BY order_date
CLUSTER BY customer_id, product_id;

-- Orden de clustering importa: customer_id más selectivo primero

-- ===== REDSHIFT: Zone maps automáticos =====
-- Redshift mantiene automáticamente min/max por block (1 MB)
-- Mejora queries con WHERE en SORTKEY

CREATE TABLE sales (...)
SORTKEY (order_date);

-- Query usa zone maps para skip blocks
SELECT * FROM sales WHERE order_date = '2025-01-15';
-- Solo lee blocks con order_date in range [2025-01-15, 2025-01-15]
```

---

## Caching y Materialized Views

### 1. Result Caching

```sql
-- ===== SNOWFLAKE: Result cache automático =====
-- Primera ejecución: ~30 segundos
SELECT customer_id, SUM(amount)
FROM sales
GROUP BY customer_id;

-- Segunda ejecución (misma query): ~0.5 segundos
-- Lee de result cache (válido por 24 horas)

-- Cache se invalida si datos cambian

-- ===== BIGQUERY: Cache automático =====
-- Similar a Snowflake, pero solo si tabla no cambió
-- Cache válido por 24 horas
-- No usa cache si query tiene CURRENT_DATE(), RAND(), etc.

-- Forzar no-cache
SELECT customer_id, SUM(amount)
FROM sales
WHERE order_date = CURRENT_DATE()  -- No cacheable
GROUP BY customer_id;

-- ===== REDSHIFT: Result caching =====
-- Cache de resultados si:
-- - Query idéntica (exact match)
-- - Datos no cambiaron
-- - Usuario tiene permisos

-- Ver cache hits
SELECT
    userid,
    query,
    cache_hit_ratio
FROM svl_qlog
WHERE cache_hit_ratio > 0;
```

### 2. Materialized Views

```sql
-- ===== CREAR MATERIALIZED VIEW =====
-- PostgreSQL
CREATE MATERIALIZED VIEW sales_daily AS
SELECT
    order_date,
    COUNT(*) as order_count,
    SUM(amount) as total_sales,
    AVG(amount) as avg_order_value
FROM sales
GROUP BY order_date;

-- Crear índice en MV
CREATE INDEX idx_sales_daily_date ON sales_daily(order_date);

-- Refrescar manualmente
REFRESH MATERIALIZED VIEW sales_daily;

-- Refrescar concurrentemente (sin lock)
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_daily;

-- ===== SNOWFLAKE: Materialized Views automáticas =====
CREATE MATERIALIZED VIEW sales_daily AS
SELECT
    order_date,
    COUNT(*) as order_count,
    SUM(amount) as total_sales
FROM sales
GROUP BY order_date;

-- Snowflake mantiene automáticamente (background refresh)
-- No necesitas REFRESH manual

-- ===== BIGQUERY: Materialized Views =====
CREATE MATERIALIZED VIEW `project.dataset.sales_daily`
AS
SELECT
    order_date,
    COUNT(*) as order_count,
    SUM(amount) as total_sales
FROM `project.dataset.sales`
GROUP BY order_date;

-- Auto-refresh incremental
-- BigQuery usa MV automáticamente cuando hace sentido

-- ===== REDSHIFT: Materialized Views auto-refresh =====
CREATE MATERIALIZED VIEW sales_daily
AUTO REFRESH YES
AS
SELECT
    order_date,
    COUNT(*) as order_count,
    SUM(amount) as total_sales
FROM sales
GROUP BY order_date;

-- Refresh manual
REFRESH MATERIALIZED VIEW sales_daily;

-- ===== CUÁNDO usar MV =====
-- ✅ Queries costosas ejecutadas frecuentemente
-- ✅ Agregaciones complejas
-- ✅ Datos cambian lentamente (no real-time)
-- ❌ Datos cambian constantemente
-- ❌ Query usa filtros muy variados
```

---

## Data Compression

### 1. Formatos y Compresión

| Formato | Compresión | Ratio | Lectura | Escritura | Splittable |
|---------|-----------|-------|---------|-----------|------------|
| **CSV** | gzip | 10:1 | Lento | Rápido | No (con gzip) |
| **JSON** | gzip | 5:1 | Muy lento | Rápido | No (con gzip) |
| **Avro** | Snappy | 3:1 | Rápido | Rápido | Sí |
| **Parquet** | Snappy | 5:1 | Muy rápido | Medio | Sí |
| **Parquet** | gzip | 8:1 | Rápido | Lento | Sí |
| **ORC** | Zlib | 7:1 | Muy rápido | Medio | Sí |

```python
# ===== ESCRIBIR PARQUET con compresión =====
# Snappy (default): Balance velocidad/compresión
df.write.mode("overwrite") \
    .option("compression", "snappy") \
    .parquet("s3://data/sales/")

# gzip: Mejor compresión, lectura más lenta
df.write.mode("overwrite") \
    .option("compression", "gzip") \
    .parquet("s3://data/sales_compressed/")

# zstd (Spark 3.0+): Mejor balance
df.write.mode("overwrite") \
    .option("compression", "zstd") \
    .parquet("s3://data/sales_zstd/")

# Benchmarks (1 GB original):
# - CSV gzipped: 200 MB, read: 30s
# - Parquet snappy: 150 MB, read: 5s
# - Parquet gzip: 100 MB, read: 8s
# - Parquet zstd: 110 MB, read: 6s

# ===== PARQUET: Configurar row group size =====
df.write.mode("overwrite") \
    .option("parquet.block.size", "256MB") \  # Row group size
    .option("parquet.page.size", "1MB") \     # Page size
    .parquet("s3://data/sales/")

# Row group más grande = mejor compresión, pero menos paralelismo
```

### 2. Compresión en Databases

```sql
-- ===== REDSHIFT: Encoding automático =====
CREATE TABLE sales (
    order_id VARCHAR(50) ENCODE LZO,        -- LZO para strings
    customer_id VARCHAR(50) ENCODE LZO,
    amount DECIMAL(10,2) ENCODE AZ64,       -- AZ64 para números
    order_date DATE ENCODE AZ64,
    status VARCHAR(20) ENCODE BYTEDICT      -- Dict encoding para enums
);

-- Encoding automático (analiza datos)
CREATE TABLE sales AS
SELECT * FROM staging_sales;

ANALYZE COMPRESSION sales;  -- Recomienda encodings

-- ===== SNOWFLAKE: Compresión automática =====
-- Snowflake comprime automáticamente (no configurable)
-- Compresión ~10:1 típica
-- Gratis (no pagas por compresión CPU)

-- ===== BIGQUERY: Compresión automática =====
-- Columnar storage comprimido
-- No configurable, gratis
```

---

## Profiling y Monitoring

### 1. Query Profiling

```python
# ===== SPARK: Web UI =====
# Spark UI: http://localhost:4040
# Ver:
# - Jobs → Stages → Tasks
# - Storage → Cached DataFrames
# - Executors → Memory/CPU usage
# - SQL → Query plans

# Programáticamente: Capturar query plan
df = spark.read.parquet("s3://data/sales/")
result = df.groupBy("customer_id").sum("amount")

# Logical plan
print(result.explain(mode="simple"))

# Physical plan con estadísticas
print(result.explain(mode="formatted"))

# ===== PROFILING con listener =====
from pyspark import SparkContext

class ProfilingListener:
    def __init__(self, spark_context):
        spark_context._jvm.org.apache.spark.api.python.PythonAccumulatorV2()

    def onJobEnd(self, job_end):
        print(f"Job {job_end.jobId()} took {job_end.time()} ms")

listener = ProfilingListener(spark.sparkContext)
spark.sparkContext.addSparkListener(listener)

# ===== AWS GLUE: Job metrics =====
import boto3

glue = boto3.client('glue')

response = glue.get_job_runs(JobName='my-etl-job')

for run in response['JobRuns']:
    print(f"""
    Run ID: {run['Id']}
    Status: {run['JobRunState']}
    Duration: {run['ExecutionTime']} seconds
    DPU-hours: {run['MaxCapacity'] * (run['ExecutionTime'] / 3600)}
    """)
```

### 2. Monitoring en Production

```python
# ===== CLOUDWATCH METRICS =====
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch')

# Publicar métricas custom
def publish_etl_metrics(job_name: str, duration_sec: int, rows_processed: int):
    cloudwatch.put_metric_data(
        Namespace='DataPipeline',
        MetricData=[
            {
                'MetricName': 'JobDuration',
                'Value': duration_sec,
                'Unit': 'Seconds',
                'Dimensions': [
                    {'Name': 'JobName', 'Value': job_name}
                ]
            },
            {
                'MetricName': 'RowsProcessed',
                'Value': rows_processed,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobName', 'Value': job_name}
                ]
            },
            {
                'MetricName': 'ThroughputRowsPerSec',
                'Value': rows_processed / duration_sec,
                'Unit': 'Count/Second',
                'Dimensions': [
                    {'Name': 'JobName', 'Value': job_name}
                ]
            }
        ]
    )

# Alarmas
cloudwatch.put_metric_alarm(
    AlarmName='ETL-Job-Duration-High',
    ComparisonOperator='GreaterThanThreshold',
    EvaluationPeriods=1,
    MetricName='JobDuration',
    Namespace='DataPipeline',
    Period=300,
    Statistic='Average',
    Threshold=3600.0,  # 1 hora
    ActionsEnabled=True,
    AlarmActions=['arn:aws:sns:us-east-1:123456789:data-alerts'],
    AlarmDescription='ETL job taking too long'
)

# ===== PROMETHEUS + GRAFANA =====
# Exportar métricas de Spark a Prometheus
# spark-defaults.conf:
"""
spark.metrics.conf.*.sink.prometheus.class=org.apache.spark.metrics.sink.PrometheusSink
spark.metrics.conf.*.sink.prometheus.port=9091
"""

# Dashboard en Grafana:
# - Job duration trends
# - Executor memory usage
# - Shuffle read/write
# - Task failure rate
```

---

## 🎯 Preguntas de Entrevista

**P: Un query tarda 10 minutos. ¿Cómo lo optimizarías?**

R:
1. **EXPLAIN ANALYZE**: Ver plan de ejecución
2. **Identificar bottleneck**: Seq scan? Join costoso? Sort?
3. **Indexing**: Crear índices en columnas de WHERE/JOIN
4. **Reescribir query**: Filtrar antes de joins, evitar subqueries correlacionadas
5. **Partitioning**: Si query escanea tabla completa
6. **Materialized view**: Si query es frecuente y costosa

**P: Un Spark job falla con OOM (Out of Memory). ¿Qué haces?**

R:
1. **Aumentar executor memory**: De 4g a 8g o más
2. **Repartition**: Más partitions → menos datos por partition
3. **Evitar collect()**: No traer datos al driver
4. **Broadcast joins**: Para tablas pequeñas (<100MB)
5. **Spill to disk**: Configurar `spark.memory.fraction`
6. **Verificar data skew**: Algunas partitions muy grandes

**P: ¿Cuándo usarías cache() en Spark?**

R:
- DataFrame usado 2+ veces
- Después de operaciones costosas (join, shuffle)
- Solo si tienes memoria suficiente
- Ejemplo: ML training (mismo dataset múltiples iteraciones)

NO usar si:
- DataFrame usado solo una vez
- Dataset muy grande (> 50% de memoria disponible)
- Pipeline lineal sin re-uso

---

**Siguiente:** [09. Data Quality y Governance](09_data_quality_governance.md)
