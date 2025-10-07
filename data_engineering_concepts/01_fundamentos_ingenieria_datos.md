# 1. Fundamentos de la Ingeniería de Datos

---

## ¿Qué es la Ingeniería de Datos?

La **ingeniería de datos** es la práctica de diseñar, construir y mantener sistemas que recopilan, almacenan, procesan y entregan datos de manera confiable, escalable y eficiente.

### Propósito Clave

- Hacer que los datos estén **disponibles, confiables y accesibles** para análisis, ML y toma de decisiones
- Construir **pipelines automatizados** que muevan y transformen datos
- Garantizar **calidad, seguridad y governance** de los datos
- Optimizar **costos y performance** de infraestructura de datos

---

## Rol del Data Engineer vs Data Analyst vs Data Scientist

| Aspecto | Data Engineer | Data Analyst | Data Scientist |
|---------|---------------|--------------|----------------|
| **Enfoque** | Infraestructura y pipelines | Insights de negocio | Modelos predictivos |
| **Principales tareas** | ETL/ELT, arquitectura, optimización | Reportes, dashboards, SQL | ML, estadística, experimentación |
| **Herramientas** | Airflow, Spark, dbt, Kafka | SQL, Excel, Power BI, Tableau | Python, R, scikit-learn, TensorFlow |
| **Output típico** | Pipelines, data warehouses | Dashboards, reportes | Modelos ML, predicciones |
| **Skills técnicos** | SQL, Python, cloud, orquestación | SQL, BI tools, estadística básica | Python/R, ML, matemáticas |
| **Upstream/Downstream** | Upstream (fuentes → warehouse) | Downstream (warehouse → insights) | Downstream (datos → modelos) |

---

## Componentes de un Sistema de Datos Moderno

```
┌─────────────┐
│   Sources   │  APIs, DBs, Files, Logs, Streaming
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Ingestion  │  Fivetran, Airbyte, Kafka, Kinesis
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Storage   │  S3, Data Lake, Data Warehouse
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Processing  │  Spark, dbt, Glue, Dataflow
└──────┬──────┘
       │
       ▼
┌─────────────┐
│Orchestration│  Airflow, Prefect, Dagster
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Consumption │  BI Tools, ML Models, APIs
└─────────────┘
```

---

## Tipos de Datos

### 1. Datos Estructurados
Datos organizados en **esquemas rígidos** (tablas con filas y columnas).

**Ejemplos:**
- Bases de datos relacionales (PostgreSQL, MySQL)
- Archivos CSV con columnas definidas
- Tablas en data warehouses

**Características:**
- Schema definido
- Fácil de consultar con SQL
- Integridad referencial

### 2. Datos Semiestructurados
Datos con **estructura flexible**.

**Formatos:**
- **JSON:** Objetos con pares clave-valor
- **Parquet:** Formato columnar para analytics
- **Avro:** Formato de fila con schema, ideal para streaming

**Ejemplos:**
```json
{
    "usuario_id": 123,
    "nombre": "Ana García",
    "compras": [
        {"producto": "Laptop", "precio": 1200},
        {"producto": "Mouse", "precio": 25}
    ]
}
```

### 3. Datos No Estructurados
Sin estructura predefinida.

**Ejemplos:**
- Logs de aplicaciones
- Imágenes, audio, video
- Documentos PDF, Word
- Emails

---

## Batch vs Streaming Processing

### Batch Processing (Por Lotes)
Procesa **grandes volúmenes** en **intervalos definidos**.

**Características:**
- Alta latencia (minutos/horas)
- Alto throughput
- Datos históricos completos

**Herramientas:**
- Apache Spark
- AWS Glue
- dbt

**Casos de uso:**
- Reportes diarios
- Agregaciones históricas
- ETL nocturno

### Streaming Processing (Tiempo Real)
Procesa datos **continuamente** a medida que llegan.

**Características:**
- Baja latencia (segundos/milisegundos)
- Datos en movimiento
- Mayor complejidad

**Herramientas:**
- Apache Kafka
- AWS Kinesis
- Apache Flink
- Spark Streaming

**Casos de uso:**
- Detección de fraude en tiempo real
- Dashboards live
- Alertas automáticas
- Recomendaciones en tiempo real

---

## Arquitecturas de Datos

### Data Warehouse (Almacén de Datos)
Sistema centralizado para **analytics y BI**, con datos **estructurados y procesados**.

**Ejemplos:**
- Snowflake
- Amazon Redshift
- Google BigQuery
- Azure Synapse

**Características:**
- Schema-on-write
- Datos limpios y transformados
- Optimizado para lectura
- Arquitectura columnar

### Data Lake (Lago de Datos)
Almacenamiento **flexible** para datos **raw** en cualquier formato.

**Ejemplos:**
- Amazon S3
- Azure Data Lake Storage
- Google Cloud Storage

**Características:**
- Schema-on-read
- Almacena datos raw
- Cualquier tipo de dato
- Bajo costo

### Data Mart
Subconjunto de Data Warehouse enfocado en **área de negocio específica**.

**Ejemplos:**
- Data Mart de Ventas
- Data Mart de Marketing

### Lakehouse Architecture
**Híbrido** entre Data Lake y Data Warehouse.

**Tecnologías:**
- Databricks Delta Lake
- Apache Iceberg
- Apache Hudi

**Ventajas:**
- Almacenamiento barato + performance de warehouse
- ACID transactions en data lake
- Un solo sistema para BI y ML

---

## ETL vs ELT

### ETL (Extract, Transform, Load)
**Transforma antes de cargar**.

```
Source → Extract → Transform → Load → Warehouse
```

**Ventajas:**
- Datos llegan limpios
- Menor carga en warehouse

**Desventajas:**
- Servidor intermedio necesario
- Más lento
- Menos flexible

### ELT (Extract, Load, Transform)
**Carga primero, transforma después**.

```
Source → Extract → Load → Transform (in Warehouse)
```

**Ventajas:**
- Más rápido (paralelización)
- Raw data disponible
- Más flexible
- Aprovecha poder del warehouse

**Desventajas:**
- Requiere warehouse potente
- Raw data ocupa espacio

**Tendencia actual:** ELT con dbt es el estándar moderno

---

## Resumen

✅ Data Engineering construye infraestructura de datos confiable
✅ Data Engineer ≠ Data Analyst ≠ Data Scientist (roles diferentes)
✅ Sistema moderno: Sources → Ingestion → Storage → Processing → Consumption
✅ Datos: Estructurados (SQL), Semiestructurados (JSON/Parquet), No estructurados (logs)
✅ Batch (horario/diario) vs Streaming (tiempo real)
✅ Data Lake (raw) vs Data Warehouse (procesado) vs Lakehouse (híbrido)
✅ ETL (transforma antes) vs ELT (transforma después, tendencia moderna)

**Siguiente:** [02. Bases de Datos y Modelado](02_bases_datos_modelado.md)
