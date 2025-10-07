# 📚 Guía Completa de SQL para Data Engineers

Documentación completa en español de SQL, desde fundamentos hasta temas avanzados, enfocada en Data Engineering.

---

## 📖 Índice de Contenidos

### 🎯 Nivel Principiante

#### [1. Fundamentos de SQL](01_fundamentos_sql.md)
- ¿Qué es SQL y su propósito?
- Database Engine vs Client
- SQL vs NoSQL
- Conceptos de bases de datos relacionales (tablas, filas, columnas, schema)
- Sistemas comunes (Snowflake, PostgreSQL, MySQL, SQL Server, Redshift, BigQuery)

#### [2. Operaciones CRUD](02_operaciones_crud.md)
- **SELECT** — Recuperar datos
- **INSERT** — Agregar datos
- **UPDATE** — Modificar datos
- **DELETE** — Eliminar datos
- Diferencias: DROP vs TRUNCATE vs DELETE
- CREATE y ALTER TABLE

#### [3. Consultas Básicas](03_consultas_basicas.md)
- Seleccionar columnas específicas
- Filtrar con WHERE, AND, OR, IN, BETWEEN, LIKE
- Ordenar con ORDER BY
- Limitar resultados con LIMIT / TOP
- Manejo de NULL (IS NULL, COALESCE, NVL)
- Valores únicos con DISTINCT

---

### 🚀 Nivel Intermedio

#### [4. Agregaciones y Funciones de Ventana](04_agregaciones_funciones_ventana.md)
- Funciones de agregación: COUNT, SUM, AVG, MIN, MAX
- GROUP BY y HAVING
- Window Functions:
  - ROW_NUMBER(), RANK(), DENSE_RANK()
  - NTILE()
  - LAG() / LEAD()
  - FIRST_VALUE() / LAST_VALUE()
- Running totals y moving averages

#### [5. JOINs y Relaciones](05_joins_relaciones.md)
- Primary Key y Foreign Key
- INNER JOIN
- LEFT / RIGHT JOIN
- FULL OUTER JOIN
- CROSS JOIN
- Self Joins
- Errores comunes y mejores prácticas

#### [6. Subqueries y CTEs](06_subqueries_ctes.md)
- Subqueries en WHERE, FROM, SELECT
- EXISTS vs IN
- Common Table Expressions (WITH clause)
- Recursive CTEs
- Diferencias entre CTEs, subqueries y temporary tables

#### [7. Operaciones de Conjuntos](07_operaciones_conjuntos.md)
- UNION vs UNION ALL
- INTERSECT
- EXCEPT / MINUS
- Casos de uso prácticos

---

### 💎 Nivel Avanzado

#### [8. Modelado de Datos y Esquemas](08_modelado_datos_esquemas.md)
- ¿Qué es un Schema?
- **Normalización:** 1NF, 2NF, 3NF, BCNF
- **Denormalización:** Cuándo y por qué
- Star Schema y Snowflake Schema
- Fact vs Dimension tables
- Surrogate keys vs Natural keys
- Integridad referencial y constraints
- OLTP vs OLAP

#### [9. Vistas](09_vistas.md)
- ¿Qué es una vista?
- Standard Views vs Materialized Views
- Casos de uso (simplificación, seguridad, abstracción)
- Refresh y mantenimiento
- Vistas actualizables

#### [10. Índices y Optimización](10_indices_optimizacion.md)
- ¿Qué es un índice?
- Tipos: B-Tree, Bitmap, Hash, Clustered, Non-Clustered
- Ventajas y desventajas
- Cuándo crear índices
- Índices compuestos
- Query execution plans (EXPLAIN)
- Técnicas de optimización

#### [11. Particionamiento y Clustering](11_particionamiento_clustering.md)
- ¿Qué es el particionamiento?
- Tipos: Range, List, Hash, Composite
- Partition pruning
- Clustering en Snowflake y BigQuery
- Redshift: Distribution y Sort keys
- Cuándo particionar
- Eliminar particiones antiguas

#### [12. Transacciones y ACID](12_transacciones_acid.md)
- ¿Qué es una transacción?
- Propiedades ACID:
  - Atomicity
  - Consistency
  - Isolation
  - Durability
- BEGIN, COMMIT, ROLLBACK, SAVEPOINT
- Niveles de aislamiento
- Locking y deadlocks
- Mejores prácticas

#### [13. Tablas Temporales](13_tablas_temporales.md)
- Temporary tables
- TRANSIENT tables (Snowflake)
- CREATE TABLE AS SELECT (CTAS)
- Casos de uso
- Temporary tables vs CTEs
- Zero-copy cloning (Snowflake)
- Time Travel

#### [14. Tipos de Datos y Constraints](14_tipos_datos_constraints.md)
- Tipos numéricos (INT, DECIMAL, FLOAT)
- Tipos de texto (CHAR, VARCHAR, TEXT)
- Fechas y horas (DATE, TIME, TIMESTAMP)
- Booleanos y JSON
- Constraints:
  - PRIMARY KEY
  - FOREIGN KEY
  - UNIQUE
  - NOT NULL
  - CHECK
  - DEFAULT
- Casting y conversión de tipos
- Manejo de zonas horarias

#### [15-18. Temas Avanzados y Ecosistema](15_temas_avanzados_extras.md)
- **Temas Avanzados:**
  - Stored Procedures y Functions
  - Triggers
  - Pivoting y Unpivoting
  - Dynamic SQL
  - Error handling
- **SQL en Data Engineering:**
  - dbt, Airflow, Fivetran
  - Snowflake, BigQuery, Redshift
  - Optimización de costos
  - Data Quality
- **Escenarios de Entrevistas:**
  - Segundo salario más alto
  - Encontrar duplicados
  - Running totals
  - Comparaciones YoY
  - OLTP vs OLAP
  - Optimización de queries
- **Mejores Prácticas:**
  - Código limpio
  - Convenciones de nombres
  - Documentación
  - Testing SQL
  - Control de versiones
  - Anti-patrones

---

## 🎓 Cómo Usar Esta Guía

### Para Principiantes
1. Empieza con **01-03** (fundamentos y consultas básicas)
2. Practica cada concepto con ejemplos propios
3. Usa herramientas como [DB Fiddle](https://www.db-fiddle.com/) o [SQL Fiddle](http://sqlfiddle.com/) para experimentar

### Para Intermedios
1. Domina **04-07** (agregaciones, JOINs, subqueries)
2. Practica con datasets reales
3. Resuelve problemas en [LeetCode SQL](https://leetcode.com/problemset/database/) o [HackerRank](https://www.hackerrank.com/domains/sql)

### Para Avanzados
1. Estudia **08-14** (modelado, optimización, transacciones)
2. Implementa en proyectos reales
3. Profundiza en tu data warehouse específico (Snowflake, BigQuery, etc.)

### Para Data Engineers
1. Enfócate en **11** (particionamiento) y **15-18** (ecosistema)
2. Aprende dbt para transformaciones
3. Practica con pipelines ETL/ELT reales

---

## 📊 Sistemas Cubiertos

| Sistema | Tipo | Uso Principal |
|---------|------|---------------|
| **PostgreSQL** | RDBMS | OLTP, apps web |
| **MySQL** | RDBMS | OLTP, apps web |
| **SQL Server** | RDBMS | Enterprise, Windows |
| **Snowflake** | Cloud DW | OLAP, analytics |
| **BigQuery** | Cloud DW | OLAP, analytics |
| **Amazon Redshift** | Cloud DW | OLAP, analytics |

---

## 🛠️ Herramientas Recomendadas

### Clientes SQL
- **DBeaver** (multi-DB, free)
- **DataGrip** (JetBrains, paid)
- **pgAdmin** (PostgreSQL)
- **MySQL Workbench** (MySQL)
- **Azure Data Studio** (SQL Server)
- **SnowSQL** (Snowflake CLI)

### Data Engineering
- **dbt** - Transformaciones SQL
- **Apache Airflow** - Orquestación
- **Fivetran / Stitch** - Ingesta de datos
- **Great Expectations** - Data quality

### Práctica Online
- [LeetCode SQL Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL](https://www.hackerrank.com/domains/sql)
- [DataLemur](https://datalemur.com/)
- [Mode Analytics SQL Tutorial](https://mode.com/sql-tutorial/)
- [DB Fiddle](https://www.db-fiddle.com/)

---

## 📝 Orden de Estudio Recomendado

### Ruta Rápida (Fundamentos)
1. [Fundamentos SQL](01_fundamentos_sql.md) → 2 horas
2. [Operaciones CRUD](02_operaciones_crud.md) → 2 horas
3. [Consultas Básicas](03_consultas_basicas.md) → 3 horas
4. [Agregaciones](04_agregaciones_funciones_ventana.md) → 4 horas
5. [JOINs](05_joins_relaciones.md) → 4 horas

**Total: ~15 horas** para fundamentos sólidos

### Ruta Completa (Data Engineer)
1-5 (arriba) +
6. [Subqueries y CTEs](06_subqueries_ctes.md) → 3 horas
7. [Operaciones de Conjuntos](07_operaciones_conjuntos.md) → 2 horas
8. [Modelado de Datos](08_modelado_datos_esquemas.md) → 5 horas
9. [Vistas](09_vistas.md) → 2 horas
10. [Índices](10_indices_optimizacion.md) → 4 horas
11. [Particionamiento](11_particionamiento_clustering.md) → 4 horas
12. [Transacciones](12_transacciones_acid.md) → 3 horas
13. [Tablas Temporales](13_tablas_temporales.md) → 2 horas
14. [Tipos de Datos](14_tipos_datos_constraints.md) → 3 horas
15. [Temas Avanzados](15_temas_avanzados_extras.md) → 6 horas

**Total: ~50-60 horas** para dominio completo

---

## 🎯 Objetivos de Aprendizaje

Al completar esta guía, serás capaz de:

✅ Escribir queries SQL complejas con múltiples JOINs y subqueries
✅ Diseñar schemas de bases de datos normalizados
✅ Optimizar queries para mejor performance
✅ Implementar pipelines ETL/ELT con SQL
✅ Trabajar con data warehouses modernos (Snowflake, BigQuery, Redshift)
✅ Aplicar mejores prácticas de SQL en producción
✅ Resolver problemas técnicos de entrevistas
✅ Entender trade-offs entre diferentes enfoques

---

## 💡 Tips para el Éxito

1. **Practica constantemente** - SQL se aprende haciendo
2. **Usa datasets reales** - Kaggle, datos públicos
3. **Explica en voz alta** - Si puedes explicarlo, lo entiendes
4. **Compara approaches** - Múltiples formas de resolver un problema
5. **Lee query plans** - Entiende cómo funciona internamente
6. **Participa en comunidades** - Stack Overflow, Reddit
7. **Construye proyectos** - Portfolio de data engineering

---

## 📚 Próximos Pasos

Después de dominar SQL:
- **Python** para scripting y data engineering
- **Apache Spark** para big data
- **Docker & Kubernetes** para deployment
- **CI/CD** para data pipelines
- **Cloud platforms** (AWS, GCP, Azure)
- **Data modeling tools** (dbt, Looker)

---

## 🤝 Contribuciones

Esta es una guía viva. Sugerencias de mejora:
- Abrir un issue con temas que quisieras ver
- Compartir casos de uso interesantes
- Reportar errores o aclaraciones necesarias

---

## 📜 Licencia

Este contenido es para fines educativos. Siéntete libre de usarlo, compartirlo y adaptarlo.

---

## 🌟 Recursos Adicionales

### Documentación Oficial
- [PostgreSQL Docs](https://www.postgresql.org/docs/)
- [MySQL Docs](https://dev.mysql.com/doc/)
- [SQL Server Docs](https://docs.microsoft.com/en-us/sql/)
- [Snowflake Docs](https://docs.snowflake.com/)
- [BigQuery Docs](https://cloud.google.com/bigquery/docs)

### Blogs y Tutoriales
- [Mode Analytics SQL Tutorial](https://mode.com/sql-tutorial/)
- [SQL Bolt](https://sqlbolt.com/)
- [W3Schools SQL](https://www.w3schools.com/sql/)
- [SQL Zoo](https://sqlzoo.net/)

### Certificaciones
- Snowflake SnowPro Core
- Google Professional Data Engineer
- Microsoft Azure Data Engineer Associate
- AWS Certified Database - Specialty

---

## 📞 Contacto

¿Preguntas? ¿Sugerencias?
- Crea un issue en el repositorio
- Comparte tu progreso usando #SQLDataEngineer

---

**¡Buena suerte en tu journey de Data Engineering! 🚀**

*Última actualización: Octubre 2025*
