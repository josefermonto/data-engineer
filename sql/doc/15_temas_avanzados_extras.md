# 15-18. Temas Avanzados, Ecosistema, Entrevistas y Mejores Prácticas

Este documento consolida los temas restantes del plan de estudio SQL.

---

## 15. Temas Avanzados de SQL

### Stored Procedures (Procedimientos Almacenados)

Código SQL reutilizable almacenado en la base de datos.

```sql
-- PostgreSQL
CREATE OR REPLACE PROCEDURE actualizar_salarios(
    p_departamento VARCHAR,
    p_porcentaje DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE empleados
    SET salario = salario * (1 + p_porcentaje / 100)
    WHERE departamento = p_departamento;

    COMMIT;
END;
$$;

-- Ejecutar
CALL actualizar_salarios('IT', 10);
```

### Functions (Funciones)

Retornan un valor.

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION calcular_bonus(p_salario DECIMAL)
RETURNS DECIMAL
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN p_salario * 0.10;
END;
$$;

-- Usar
SELECT nombre, salario, calcular_bonus(salario) as bonus
FROM empleados;
```

### Triggers (Disparadores)

Se ejecutan automáticamente en eventos (INSERT, UPDATE, DELETE).

```sql
-- PostgreSQL: Auditoría automática
CREATE OR REPLACE FUNCTION audit_empleados()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO empleados_audit (
        empleado_id,
        operacion,
        fecha,
        usuario
    ) VALUES (
        NEW.id,
        TG_OP,
        NOW(),
        CURRENT_USER
    );
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_empleados_audit
AFTER INSERT OR UPDATE OR DELETE ON empleados
FOR EACH ROW
EXECUTE FUNCTION audit_empleados();
```

### Pivoting y Unpivoting

Rotar datos de filas a columnas y viceversa.

```sql
-- Pivot (filas → columnas)
SELECT *
FROM (
    SELECT departamento, mes, ventas
    FROM ventas_mensuales
) src
PIVOT (
    SUM(ventas)
    FOR mes IN ('Enero', 'Febrero', 'Marzo')
) piv;

-- Resultado:
-- departamento | Enero | Febrero | Marzo
-- IT           | 1000  | 1200    | 1100
```

```sql
-- UNPIVOT (columnas → filas)
SELECT departamento, mes, ventas
FROM ventas_pivot
UNPIVOT (
    ventas FOR mes IN (enero, febrero, marzo)
) unpiv;
```

### Dynamic SQL

SQL generado dinámicamente en tiempo de ejecución.

```sql
-- PostgreSQL
DO $$
DECLARE
    tabla VARCHAR := 'empleados';
    query TEXT;
BEGIN
    query := 'SELECT COUNT(*) FROM ' || tabla;
    EXECUTE query;
END $$;

-- SQL Server
DECLARE @sql NVARCHAR(MAX);
SET @sql = 'SELECT * FROM ' + @tabla;
EXEC sp_executesql @sql;
```

### Error Handling

```sql
-- PostgreSQL
DO $$
BEGIN
    UPDATE empleados SET salario = -1000 WHERE id = 1;
EXCEPTION
    WHEN check_violation THEN
        RAISE NOTICE 'Error: Salario no puede ser negativo';
    WHEN OTHERS THEN
        RAISE NOTICE 'Error desconocido: %', SQLERRM;
END $$;

-- SQL Server
BEGIN TRY
    UPDATE empleados SET salario = -1000 WHERE id = 1;
END TRY
BEGIN CATCH
    PRINT 'Error: ' + ERROR_MESSAGE();
END CATCH;
```

---

## 16. SQL en el Ecosistema de Data Engineering

### SQL en ETL/ELT

#### dbt (Data Build Tool)

```sql
-- models/staging/stg_empleados.sql
{{ config(materialized='view') }}

SELECT
    id,
    UPPER(nombre) as nombre,
    email,
    departamento_id,
    salario
FROM {{ source('raw', 'empleados') }}
WHERE activo = TRUE
```

```sql
-- models/marts/fct_ventas.sql
{{ config(materialized='table') }}

WITH ventas_limpias AS (
    SELECT * FROM {{ ref('stg_ventas') }}
),
clientes AS (
    SELECT * FROM {{ ref('stg_clientes') }}
)
SELECT
    v.*,
    c.nombre as cliente_nombre
FROM ventas_limpias v
LEFT JOIN clientes c ON v.cliente_id = c.id
```

#### Airflow

```python
from airflow.providers.postgres.operators.postgres import PostgresOperator

sql_task = PostgresOperator(
    task_id='cargar_ventas',
    postgres_conn_id='warehouse',
    sql="""
        INSERT INTO ventas_agregadas
        SELECT
            DATE(fecha) as dia,
            SUM(monto) as total
        FROM ventas
        WHERE fecha = '{{ ds }}'
        GROUP BY dia;
    """
)
```

### SQL en Data Warehouses

#### Snowflake

```sql
-- Crear warehouse (recursos de cómputo)
CREATE WAREHOUSE etl_warehouse
WITH WAREHOUSE_SIZE = 'MEDIUM'
AUTO_SUSPEND = 300
AUTO_RESUME = TRUE;

-- Usar warehouse
USE WAREHOUSE etl_warehouse;

-- Cargar datos desde S3
COPY INTO ventas
FROM 's3://mi-bucket/datos/'
CREDENTIALS = (AWS_KEY_ID='...' AWS_SECRET_KEY='...')
FILE_FORMAT = (TYPE = CSV FIELD_DELIMITER = ',');

-- Time Travel
SELECT * FROM empleados
AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);

-- Zero-copy clone
CREATE TABLE empleados_dev CLONE empleados;
```

#### BigQuery

```sql
-- Cargar desde GCS
LOAD DATA INTO proyecto.dataset.ventas
FROM FILES (
    format = 'CSV',
    uris = ['gs://mi-bucket/ventas/*.csv']
);

-- Particionamiento y clustering
CREATE TABLE ventas
PARTITION BY DATE(fecha)
CLUSTER BY producto_id, region
AS SELECT * FROM ventas_temp;

-- Optimización de costos
SELECT
    COUNT(*) as total,
    SUM(monto) as ventas_totales
FROM ventas
WHERE DATE(fecha) = '2024-01-15'  -- Partition pruning
  AND producto_id = 123;          -- Clustering
```

#### Redshift

```sql
-- Crear tabla con distribution y sort keys
CREATE TABLE ventas (
    id INT,
    fecha DATE,
    cliente_id INT,
    monto DECIMAL
)
DISTKEY(cliente_id)
SORTKEY(fecha);

-- Cargar desde S3
COPY ventas
FROM 's3://mi-bucket/ventas/'
IAM_ROLE 'arn:aws:iam::...'
CSV
GZIP;

-- Vacuum para reorganizar
VACUUM ventas;
ANALYZE ventas;
```

### Query Cost Optimization

```sql
-- BigQuery: Ver costo estimado
-- UI muestra bytes que se procesarán

-- Snowflake: Ver costo de query
SELECT *
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_ID = LAST_QUERY_ID();

-- Optimizaciones comunes:
-- 1. Filtrar por partition key
WHERE DATE(fecha) = '2024-01-15'  -- ✓

-- 2. Evitar SELECT *
SELECT id, nombre, monto  -- ✓ Solo columnas necesarias

-- 3. Usar clustering
WHERE producto_id = 123  -- ✓ Si tabla está clustered por producto_id

-- 4. Limitar datos
LIMIT 1000  -- Para pruebas
```

### SQL para Data Quality

```sql
-- Tests de calidad con dbt
-- tests/schema.yml
version: 2
models:
  - name: fct_ventas
    columns:
      - name: id
        tests:
          - unique
          - not_null
      - name: monto
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
      - name: cliente_id
        tests:
          - relationships:
              to: ref('dim_clientes')
              field: id
```

```sql
-- Queries de validación manuales
-- Duplicados
SELECT id, COUNT(*) as duplicados
FROM empleados
GROUP BY id
HAVING COUNT(*) > 1;

-- NULLs en columnas críticas
SELECT COUNT(*) as registros_invalidos
FROM ventas
WHERE cliente_id IS NULL
   OR monto IS NULL
   OR fecha IS NULL;

-- Valores fuera de rango
SELECT COUNT(*) as valores_invalidos
FROM empleados
WHERE salario < 0 OR edad < 18 OR edad > 100;
```

---

## 17. Escenarios Comunes de Entrevistas

### 1. Segundo Salario Más Alto

```sql
-- Opción 1: Con OFFSET
SELECT DISTINCT salario
FROM empleados
ORDER BY salario DESC
LIMIT 1 OFFSET 1;

-- Opción 2: Con subquery
SELECT MAX(salario)
FROM empleados
WHERE salario < (SELECT MAX(salario) FROM empleados);

-- Opción 3: Con window function
SELECT DISTINCT salario
FROM (
    SELECT salario, DENSE_RANK() OVER (ORDER BY salario DESC) as rank
    FROM empleados
) ranked
WHERE rank = 2;
```

### 2. Encontrar Duplicados

```sql
-- Encontrar duplicados
SELECT email, COUNT(*) as total
FROM empleados
GROUP BY email
HAVING COUNT(*) > 1;

-- Con todos los detalles
SELECT e.*
FROM empleados e
JOIN (
    SELECT email
    FROM empleados
    GROUP BY email
    HAVING COUNT(*) > 1
) dup ON e.email = dup.email
ORDER BY e.email;
```

### 3. Running Total (Suma Acumulativa)

```sql
SELECT
    fecha,
    monto,
    SUM(monto) OVER (ORDER BY fecha) as total_acumulado
FROM ventas
ORDER BY fecha;
```

### 4. Comparación Mes a Mes / Año a Año

```sql
-- Mes actual vs mes anterior
SELECT
    DATE_TRUNC('month', fecha) as mes,
    SUM(monto) as ventas_mes,
    LAG(SUM(monto)) OVER (ORDER BY DATE_TRUNC('month', fecha)) as ventas_mes_anterior,
    SUM(monto) - LAG(SUM(monto)) OVER (ORDER BY DATE_TRUNC('month', fecha)) as diferencia
FROM ventas
GROUP BY DATE_TRUNC('month', fecha);

-- Año sobre año (YoY)
SELECT
    EXTRACT(YEAR FROM fecha) as año,
    EXTRACT(MONTH FROM fecha) as mes,
    SUM(monto) as ventas,
    LAG(SUM(monto), 12) OVER (ORDER BY EXTRACT(YEAR FROM fecha), EXTRACT(MONTH FROM fecha)) as ventas_año_anterior
FROM ventas
GROUP BY año, mes;
```

### 5. Top N por Categoría

```sql
-- Top 3 productos más vendidos por categoría
WITH ranking AS (
    SELECT
        categoria,
        producto,
        ventas_totales,
        DENSE_RANK() OVER (PARTITION BY categoria ORDER BY ventas_totales DESC) as rank
    FROM (
        SELECT
            p.categoria,
            p.nombre as producto,
            SUM(v.monto) as ventas_totales
        FROM ventas v
        JOIN productos p ON v.producto_id = p.id
        GROUP BY p.categoria, p.nombre
    ) agregado
)
SELECT categoria, producto, ventas_totales
FROM ranking
WHERE rank <= 3;
```

### 6. Transformaciones para BI

```sql
-- Crear tabla fact para BI
CREATE TABLE fct_ventas AS
WITH ventas_enriquecidas AS (
    SELECT
        v.id as venta_id,
        v.fecha,
        EXTRACT(YEAR FROM v.fecha) as año,
        EXTRACT(QUARTER FROM v.fecha) as trimestre,
        EXTRACT(MONTH FROM v.fecha) as mes,
        EXTRACT(DOW FROM v.fecha) as dia_semana,
        v.cliente_id,
        c.segmento as cliente_segmento,
        c.region as cliente_region,
        v.producto_id,
        p.categoria as producto_categoria,
        v.cantidad,
        v.monto,
        v.monto / NULLIF(v.cantidad, 0) as precio_unitario
    FROM ventas v
    LEFT JOIN clientes c ON v.cliente_id = c.id
    LEFT JOIN productos p ON v.producto_id = p.id
)
SELECT * FROM ventas_enriquecidas;
```

### 7. OLTP vs OLAP

**OLTP (Online Transaction Processing):**
- Operaciones transaccionales (CRUD)
- Muchas escrituras pequeñas
- Normalizado (3NF)
- Baja latencia
- Bases de datos: PostgreSQL, MySQL, SQL Server

**OLAP (Online Analytical Processing):**
- Queries analíticas complejas
- Muchas lecturas grandes
- Desnormalizado (Star/Snowflake schema)
- Throughput alto
- Data warehouses: Snowflake, BigQuery, Redshift

### 8. Optimización de Query Lenta

**Pasos:**
1. **EXPLAIN** para ver execution plan
2. **Identificar bottlenecks** (full table scan, missing index)
3. **Agregar índices** en columnas de WHERE/JOIN
4. **Reescribir query** (CTEs, eliminar subqueries innecesarias)
5. **Particionar** tabla si es muy grande
6. **Actualizar estadísticas** (ANALYZE)

```sql
-- Query lenta
SELECT e.nombre, d.nombre, COUNT(p.id)
FROM empleados e
JOIN departamentos d ON e.departamento_id = d.id
LEFT JOIN proyectos p ON e.id = p.empleado_id
WHERE e.activo = true
GROUP BY e.nombre, d.nombre;

-- Optimizaciones:
-- 1. Índice en empleados.activo
CREATE INDEX idx_empleados_activo ON empleados(activo);

-- 2. Índice en FK
CREATE INDEX idx_empleados_dept ON empleados(departamento_id);
CREATE INDEX idx_proyectos_emp ON proyectos(empleado_id);

-- 3. Filtrar antes de JOIN
WITH empleados_activos AS (
    SELECT id, nombre, departamento_id
    FROM empleados
    WHERE activo = true
)
SELECT ea.nombre, d.nombre, COUNT(p.id)
FROM empleados_activos ea
JOIN departamentos d ON ea.departamento_id = d.id
LEFT JOIN proyectos p ON ea.id = p.empleado_id
GROUP BY ea.nombre, d.nombre;
```

---

## 18. Mejores Prácticas y Tips del Mundo Real

### Código SQL Limpio

```sql
-- ❌ Malo
select * from empleados e join departamentos d on e.departamento_id=d.id where e.activo=1 and d.nombre='IT';

-- ✓ Bueno
SELECT
    e.id,
    e.nombre,
    e.email,
    d.nombre AS departamento
FROM empleados e
INNER JOIN departamentos d
    ON e.departamento_id = d.id
WHERE e.activo = TRUE
  AND d.nombre = 'IT'
ORDER BY e.nombre;
```

### Convenciones de Nombres

```sql
-- Tablas: plural, snake_case
CREATE TABLE empleados (...);
CREATE TABLE pedidos_detalle (...);

-- Columnas: singular, snake_case
id, nombre, email, fecha_creacion

-- Índices: idx_tabla_columna
CREATE INDEX idx_empleados_email ON empleados(email);

-- Foreign keys: fk_tabla_referencia
CONSTRAINT fk_empleados_departamento

-- Views: v_ o vista_
CREATE VIEW v_empleados_activos AS ...

-- Materialized views: mv_
CREATE MATERIALIZED VIEW mv_ventas_mensuales AS ...
```

### Comentarios y Documentación

```sql
-- Comentar tablas
COMMENT ON TABLE empleados IS 'Tabla maestra de empleados activos e inactivos';

-- Comentar columnas
COMMENT ON COLUMN empleados.fecha_ingreso IS 'Fecha del primer día de trabajo';

-- Comentarios inline
SELECT
    e.nombre,
    e.salario * 12 AS salario_anual,  -- Salario anual bruto
    -- Calcular bonus (10% para IT, 5% para otros)
    CASE
        WHEN d.nombre = 'IT' THEN e.salario * 0.10
        ELSE e.salario * 0.05
    END AS bonus_estimado
FROM empleados e
JOIN departamentos d ON e.departamento_id = d.id;
```

### Diseño Modular con CTEs

```sql
-- ✓ Modular y claro
WITH
-- Paso 1: Filtrar empleados activos
empleados_activos AS (
    SELECT * FROM empleados WHERE activo = TRUE
),
-- Paso 2: Calcular totales por departamento
totales_dept AS (
    SELECT
        departamento_id,
        COUNT(*) as total_empleados,
        AVG(salario) as salario_promedio
    FROM empleados_activos
    GROUP BY departamento_id
),
-- Paso 3: Enriquecer con nombres
resultado_final AS (
    SELECT
        d.nombre as departamento,
        t.total_empleados,
        t.salario_promedio
    FROM totales_dept t
    JOIN departamentos d ON t.departamento_id = d.id
)
SELECT * FROM resultado_final
ORDER BY total_empleados DESC;
```

### Control de Versiones (Git)

```bash
# Estructura de proyecto
├── migrations/
│   ├── 001_crear_tablas_base.sql
│   ├── 002_agregar_indices.sql
│   └── 003_agregar_columna_email.sql
├── models/
│   ├── staging/
│   │   └── stg_empleados.sql
│   └── marts/
│       └── fct_ventas.sql
└── tests/
    └── test_data_quality.sql
```

### Testing SQL

```sql
-- Test: No duplicados en primary key
SELECT
    id,
    COUNT(*) as duplicados
FROM empleados
GROUP BY id
HAVING COUNT(*) > 1;
-- Esperado: 0 filas

-- Test: Foreign keys válidos
SELECT e.*
FROM empleados e
LEFT JOIN departamentos d ON e.departamento_id = d.id
WHERE e.departamento_id IS NOT NULL
  AND d.id IS NULL;
-- Esperado: 0 filas

-- Test usando EXCEPT para comparar datasets
SELECT * FROM tabla_esperada
EXCEPT
SELECT * FROM tabla_actual;
-- Esperado: 0 filas (ambas iguales)
```

### Anti-Patrones a Evitar

```sql
-- ❌ SELECT * en producción
SELECT * FROM empleados;

-- ✓ Especificar columnas
SELECT id, nombre, email FROM empleados;

-- ❌ Subqueries en SELECT para muchas filas
SELECT
    e.nombre,
    (SELECT COUNT(*) FROM proyectos WHERE empleado_id = e.id) as total_proyectos
FROM empleados e;  -- N+1 problem

-- ✓ JOIN o window function
SELECT
    e.nombre,
    COUNT(p.id) as total_proyectos
FROM empleados e
LEFT JOIN proyectos p ON e.id = p.empleado_id
GROUP BY e.nombre;

-- ❌ Funciones en WHERE (rompe índices)
WHERE UPPER(nombre) = 'ANA'

-- ✓ Comparación directa o índice funcional
WHERE nombre = 'Ana'
-- O
CREATE INDEX idx_nombre_upper ON empleados(UPPER(nombre));
```

### Performance Checklist

- [ ] Índices en Primary Keys y Foreign Keys
- [ ] Índices en columnas de WHERE frecuente
- [ ] Solo columnas necesarias en SELECT
- [ ] EXPLAIN para verificar query plan
- [ ] Evitar funciones en WHERE
- [ ] Filtrar antes de JOIN
- [ ] Particionar tablas grandes
- [ ] ANALYZE/estadísticas actualizadas
- [ ] Limitar resultados en desarrollo (LIMIT)
- [ ] Monitorear queries lentas

---

## Recursos y Aprendizaje Continuo

- **Práctica:** LeetCode, HackerRank, DataLemur (SQL problems)
- **Documentación:** PostgreSQL docs, MySQL docs, Snowflake docs
- **Herramientas:** dbt, Airflow, Fivetran, Stitch
- **Comunidades:** Stack Overflow, Reddit r/SQL, r/dataengineering
- **Certificaciones:** PostgreSQL, Snowflake SnowPro, Google BigQuery

---

¡SQL es fundamental para Data Engineering! La práctica constante y trabajar con datasets reales es la mejor forma de dominar estos conceptos.
