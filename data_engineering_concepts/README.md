# 📚 Guía Completa de Conceptos de Data Engineering

Documentación completa en español sobre conceptos fundamentales de Data Engineering, desde arquitectura hasta MLOps, enfocada en preparación para entrevistas técnicas.

---

## 📖 Índice de Contenidos

### 🎯 Fundamentos (Nivel Entry-Mid)

#### ✅ [1. ¿Qué es Data Engineering?](01_que_es_data_engineering.md)
- Definición y propósito
- Componentes de un sistema de datos moderno
- Flujo típico de datos (ejemplo e-commerce)
- Responsabilidades del Data Engineer
- Skills técnicos clave
- Métricas de éxito
- Evolución de la profesión

#### ✅ [2. Roles en Data](02_roles_data_engineer_analyst_scientist.md)
- Data Engineer vs Data Analyst vs Data Scientist
- Comparación detallada por dimensión
- Día a día de cada rol
- Tech stack típico
- Colaboración entre roles (ejemplo: Churn Prediction)
- ¿Qué rol elegir?
- Transiciones comunes entre roles

#### 📋 3. Tipos de Datos (Próximamente)
- Estructurados (SQL, tablas)
- Semiestructurados (JSON, Parquet, Avro)
- No estructurados (logs, imágenes, audio)
- Comparación y casos de uso

#### 📋 4. Batch vs Streaming (Próximamente)
- Procesamiento por lotes (Batch)
- Procesamiento en tiempo real (Streaming)
- Lambda y Kappa Architecture
- Herramientas y casos de uso

#### 📋 5. Arquitecturas de Datos (Próximamente)
- Data Lake vs Data Warehouse vs Data Mart
- Lakehouse Architecture
- Medallion Architecture (Bronze/Silver/Gold)
- Comparación y casos de uso

#### 📋 6. ETL vs ELT (Próximamente)
- Diferencias fundamentales
- Cuándo usar cada uno
- Tendencia moderna: ELT con dbt
- Ejemplos prácticos

#### 📋 7. Bases de Datos y Modelado (Próximamente)
- SQL vs NoSQL
- OLTP vs OLAP
- Modelado dimensional (Star y Snowflake Schema)
- Slowly Changing Dimensions (SCD)
- Particionamiento y sharding

#### 📋 8. Herramientas de Ingesta (Próximamente)
- Fivetran, Airbyte, Stitch
- AWS Glue, Azure Data Factory
- Custom pipelines con Python
- Change Data Capture (CDC)

---

### 🚀 Procesamiento y Orquestación (Nivel Mid-Senior)

#### 📋 9. Procesamiento Distribuido con Spark (Próximamente)
- Apache Spark fundamentals
- Transformations vs Actions
- Partitioning y shuffling
- Optimization techniques
- Spark SQL y DataFrames

#### 📋 10. Streaming y Event Processing (Próximamente)
- Apache Kafka fundamentals
- AWS Kinesis
- Apache Flink
- Event-driven architectures
- Window functions en streams

#### 📋 11. Orquestación de Pipelines (Próximamente)
- Apache Airflow (DAGs, operators)
- Prefect y Dagster
- Retry policies y error handling
- Monitoring y alerting
- Backfill strategies

#### 📋 12. Cloud Data Platforms (Próximamente)
- AWS: S3, Glue, EMR, Redshift
- GCP: BigQuery, Dataflow
- Azure: Synapse, Data Factory
- Snowflake architecture
- Serverless vs Clustered

---

### 💎 Governance, Calidad y Seguridad (Nivel Senior)

#### 📋 13. Data Quality y Governance (Próximamente)
- Dimensiones de calidad de datos
- Great Expectations, dbt tests
- Data Catalog y Lineage
- Data Observability (Monte Carlo, Soda)

#### 📋 14. Seguridad y Cumplimiento (Próximamente)
- Cifrado y RBAC
- Secrets management
- GDPR, HIPAA, SOC2
- Row/Column-Level Security

---

### 🔧 DevOps y MLOps (Nivel Senior+)

#### 📋 15. DataOps y CI/CD (Próximamente)
- Git para proyectos de datos
- CI/CD para pipelines (GitHub Actions)
- Environment management (dev/test/prod)
- Testing y data versioning

#### 📋 16. MLOps Basics (Próximamente)
- Rol del Data Engineer en MLOps
- Feature engineering pipelines
- Feature stores (Feast, Tecton)
- Model serving y monitoring

---

### 🎯 Preparación para Entrevistas

#### 📋 17. Preguntas Técnicas Comunes (Próximamente)
- System design de arquitecturas
- Casos de optimización
- Troubleshooting
- Trade-offs técnicos

#### 📋 18. Casos Prácticos y Proyectos (Próximamente)
- Arquitectura completa: Snowflake + dbt + Airflow
- Pipeline de datos en tiempo real
- Migración on-premise → cloud
- Event-driven con Kafka
- Cost optimization

---

## 🎓 Rutas de Estudio Recomendadas

### Ruta Junior → Mid (3-6 meses)
1. Fundamentos (01)
2. SQL avanzado (ver carpeta `sql/`)
3. Python para data (pandas, básicos de Spark)
4. Bases de datos y modelado (02)
5. ETL/ELT (03)
6. Un cloud provider (AWS o GCP)

**Proyecto:** Pipeline ETL completo con Airflow + dbt

### Ruta Mid → Senior (6-12 meses)
1. Dominar Spark y procesamiento distribuido
2. Streaming con Kafka/Kinesis
3. Orquestación avanzada (Airflow)
4. Data quality y testing
5. DevOps para datos (CI/CD)
6. Optimización y cost management

**Proyecto:** Data platform completa con governance

### Ruta Senior → Staff (12+ meses)
1. Arquitecturas a escala (PB de datos)
2. MLOps integration
3. Data mesh y domain-driven design
4. Multi-cloud strategies
5. Liderazgo técnico

**Proyecto:** Diseñar data platform empresarial

---

## 🛠️ Stack Tecnológico Típico

### Must-Know (Obligatorios)
- **SQL** avanzado (PostgreSQL, Snowflake)
- **Python** (pandas, PySpark)
- **Cloud** (AWS/GCP/Azure - al menos uno)
- **Airflow** o Prefect
- **dbt** para transformaciones
- **Git** para versionamiento

### Nice-to-Have (Muy valorados)
- **Spark** (batch y streaming)
- **Kafka** o Kinesis
- **Docker** y Kubernetes básico
- **Terraform** para IaC
- **Great Expectations** o similar
- **Snowflake** específicamente

### Emerging (Tendencias)
- **dbt Cloud**
- **Delta Lake / Iceberg**
- **Databricks**
- **Fivetran / Airbyte**
- **Dagster / Prefect 2.0**
- **DuckDB** para analytics

---

## 📊 Herramientas por Categoría

### Ingesta
- **Batch:** Fivetran, Airbyte, Stitch
- **Streaming:** Kafka, Kinesis, Pub/Sub
- **Custom:** Python scripts, AWS Lambda

### Transformación
- **SQL:** dbt
- **Python:** pandas, Polars
- **Distributed:** Spark, Flink

### Orquestación
- **Open Source:** Airflow, Prefect, Dagster
- **Cloud:** AWS Step Functions, Cloud Composer, Azure Data Factory

### Storage
- **Warehouse:** Snowflake, BigQuery, Redshift
- **Lake:** S3, GCS, ADLS
- **Lakehouse:** Databricks, Delta Lake

### BI & Analytics
- **Self-Service:** Power BI, Tableau, Looker
- **Embedded:** Metabase, Superset
- **Code-First:** Jupyter, Hex, Deepnote

### Monitoring
- **Data Quality:** Great Expectations, Soda, Monte Carlo
- **Infra:** Datadog, CloudWatch, Prometheus
- **Pipeline:** Airflow UI, Prefect Cloud

---

## 💡 Conceptos Clave para Dominar

### Principios Fundamentales
1. **Idempotencia** - Pipelines que pueden ejecutarse múltiples veces sin efectos secundarios
2. **Incrementalidad** - Procesar solo datos nuevos/modificados
3. **Data Lineage** - Rastrear origen y transformaciones
4. **Schema Evolution** - Manejar cambios en estructura de datos
5. **Backpressure** - Manejar cuando downstream no puede procesar rápido
6. **Exactly-Once Semantics** - Garantías en processing distribuido

### Patrones de Diseño
1. **Medallion Architecture** (Bronze → Silver → Gold)
2. **Lambda Architecture** (Batch + Streaming)
3. **Kappa Architecture** (Solo streaming)
4. **Data Mesh** (Domain-driven data)
5. **Slowly Changing Dimensions** (SCD Types)
6. **Event Sourcing** (Log inmutable de eventos)

---

## 🎯 Preparación para Entrevistas

### Tipos de Preguntas

#### 1. System Design (40% de entrevistas senior)
- "Diseña un pipeline para procesar 1TB de logs diarios"
- "Arquitectura para dashboard en tiempo real"
- "Migrar data warehouse on-premise a cloud"

#### 2. Coding/SQL (30%)
- SQL comple jo (window functions, CTEs, optimización)
- Python para ETL
- Spark transformations

#### 3. Behavioral (20%)
- Conflictos en equipo
- Proyectos difíciles
- Decisiones técnicas

#### 4. Troubleshooting (10%)
- "Pipeline está lento, ¿cómo debuggeas?"
- "Datos inconsistentes entre sources"
- "Costos de cloud muy altos"

### Estructura de Respuesta (STAR Method)
- **S**ituation: Contexto
- **T**ask: Qué había que hacer
- **A**ction: Qué hiciste
- **R**esult: Resultado y aprendizajes

---

## 📚 Recursos Complementarios

### Cursos Recomendados
- **DataCamp:** Data Engineering track
- **Udacity:** Data Engineering Nanodegree
- **Coursera:** Google Cloud Data Engineering
- **LinkedIn Learning:** Apache Airflow, dbt

### Libros
- "Designing Data-Intensive Applications" - Martin Kleppmann
- "The Data Warehouse Toolkit" - Ralph Kimball
- "Fundamentals of Data Engineering" - Joe Reis & Matt Housley

### Blogs y Newsletters
- [Data Engineering Weekly](https://www.dataengineeringweekly.com/)
- [Seattle Data Guy](https://www.theseattledataguy.com/)
- [dbt Blog](https://www.getdbt.com/blog/)
- [Locally Optimistic](https://locallyoptimistic.com/)

### Práctica
- [DataLemur](https://datalemur.com/) - SQL para entrevistas
- [StrataScratch](https://www.stratascratch.com/)
- [Kaggle](https://www.kaggle.com/) - Datasets
- Construir proyectos personales en GitHub

---

## 🚀 Próximos Pasos

1. **Completa fundamentos** (archivo 01)
2. **Domina SQL** (ver carpeta `sql/`)
3. **Aprende un cloud** (AWS recomendado)
4. **Construye un proyecto** end-to-end
5. **Practica entrevistas** con LeetCode/DataLemur
6. **Contribuye a open source** (Airflow, dbt, etc.)

---

## 📝 Cómo Usar Esta Guía

### Para Principiantes
- Lee 01-04 en orden
- Practica con datasets pequeños
- Construye un pipeline simple (CSV → PostgreSQL → Dashboard)

### Para Intermedios
- Profundiza en 05-08
- Implementa Airflow + dbt
- Aprende Spark y Kafka

### Para Avanzados
- Enfócate en 09-14
- Diseña arquitecturas completas
- Prepárate para preguntas de system design

---

**Estado del Repositorio:**
- ✅ SQL: Completo (15 archivos detallados)
- 🚧 Data Engineering Concepts: **2/18 archivos** completados
  - ✅ 01. ¿Qué es Data Engineering?
  - ✅ 02. Roles en Data
  - 📋 03-18. Por crear
- 📋 Python: Planeado
- 📋 AWS: Planeado

---

*Última actualización: Octubre 2025*

Para contribuciones o sugerencias, abre un issue en el repositorio.
