# 13. Tablas Temporales y Gestión de Tablas

---

## Tablas Temporales (Temporary Tables)

Tablas que existen **solo durante la sesión** o transacción actual.

### Características

- ✅ Visibles solo para la sesión que las creó
- ✅ Se eliminan automáticamente al cerrar sesión
- ✅ Útiles para cálculos intermedios
- ✅ Pueden tener índices

---

## Crear Tablas Temporales

### PostgreSQL

```sql
-- Tabla temporal (se elimina al final de la sesión)
CREATE TEMP TABLE temp_empleados AS
SELECT * FROM empleados WHERE departamento = 'IT';

-- O con estructura explícita
CREATE TEMPORARY TABLE temp_calculos (
    id SERIAL,
    valor DECIMAL,
    resultado DECIMAL
);

-- Insertar datos
INSERT INTO temp_calculos (valor, resultado)
SELECT salario, salario * 1.10 FROM empleados;

-- Usar
SELECT * FROM temp_calculos;
```

### MySQL

```sql
CREATE TEMPORARY TABLE temp_empleados (
    id INT,
    nombre VARCHAR(100),
    salario DECIMAL
);

-- Desde query
CREATE TEMPORARY TABLE temp_empleados AS
SELECT * FROM empleados WHERE activo = true;
```

### SQL Server

```sql
-- # = tabla temporal local (solo esta sesión)
CREATE TABLE #temp_empleados (
    id INT,
    nombre VARCHAR(100),
    salario DECIMAL
);

-- ## = tabla temporal global (todas las sesiones)
CREATE TABLE ##temp_global (
    id INT,
    dato VARCHAR(100)
);

-- Desde query
SELECT *
INTO #temp_empleados
FROM empleados
WHERE activo = 1;
```

### Snowflake

```sql
-- Tabla temporal (se elimina al final de la sesión)
CREATE TEMPORARY TABLE temp_empleados AS
SELECT * FROM empleados WHERE activo = true;

-- O TRANSIENT (más económica, sin Time Travel completo)
CREATE TRANSIENT TABLE transient_logs (
    id INT,
    mensaje VARCHAR,
    fecha TIMESTAMP
);
```

---

## Temporary vs Transient vs Permanent

### Snowflake

| Tipo | Duración | Time Travel | Fail-safe | Costo |
|------|----------|-------------|-----------|-------|
| **TEMPORARY** | Sesión | 0-1 días | No | Bajo |
| **TRANSIENT** | Permanente | 0-1 días | No | Medio |
| **PERMANENT** | Permanente | 0-90 días | Sí | Alto |

```sql
-- Temporary: solo para sesión actual
CREATE TEMPORARY TABLE temp_data (...);

-- Transient: permanente pero sin fail-safe
CREATE TRANSIENT TABLE staging_data (...);

-- Permanent: con protección completa
CREATE TABLE production_data (...);
```

**Cuándo usar:**
- **TEMPORARY:** Cálculos intermedios en sesión
- **TRANSIENT:** Staging, ETL, datos reproducibles
- **PERMANENT:** Datos críticos de producción

---

## CREATE TABLE AS SELECT (CTAS)

Crea tabla desde resultado de query.

```sql
-- PostgreSQL, MySQL, Snowflake
CREATE TABLE empleados_backup AS
SELECT * FROM empleados;

-- Con filtros
CREATE TABLE empleados_it AS
SELECT id, nombre, salario
FROM empleados
WHERE departamento = 'IT';

-- Con agregaciones
CREATE TABLE resumen_departamentos AS
SELECT
    departamento,
    COUNT(*) as total,
    AVG(salario) as salario_promedio
FROM empleados
GROUP BY departamento;
```

### SQL Server

```sql
-- INTO clause
SELECT *
INTO empleados_backup
FROM empleados;

-- Con filtro
SELECT id, nombre, salario
INTO empleados_it
FROM empleados
WHERE departamento = 'IT';
```

---

## Casos de Uso de Tablas Temporales

### 1. Cálculos Intermedios Complejos

```sql
-- Paso 1: calcular totales por cliente
CREATE TEMP TABLE temp_totales AS
SELECT
    cliente_id,
    SUM(monto) as total_compras,
    COUNT(*) as num_pedidos
FROM pedidos
WHERE fecha >= '2024-01-01'
GROUP BY cliente_id;

-- Paso 2: unir con datos de clientes
CREATE TEMP TABLE temp_resultado AS
SELECT
    c.nombre,
    c.email,
    t.total_compras,
    t.num_pedidos,
    c.fecha_registro
FROM clientes c
JOIN temp_totales t ON c.id = t.cliente_id;

-- Paso 3: análisis final
SELECT *
FROM temp_resultado
WHERE total_compras > 1000
ORDER BY total_compras DESC;
```

### 2. Staging para ETL

```sql
-- Cargar datos crudos
CREATE TEMP TABLE staging_ventas (
    fecha VARCHAR,
    producto VARCHAR,
    cantidad VARCHAR,
    monto VARCHAR
);

COPY staging_ventas FROM '/data/ventas.csv' CSV;

-- Limpiar y transformar
INSERT INTO ventas (fecha, producto_id, cantidad, monto)
SELECT
    TO_DATE(fecha, 'YYYY-MM-DD'),
    (SELECT id FROM productos WHERE nombre = producto),
    cantidad::INT,
    monto::DECIMAL
FROM staging_ventas
WHERE fecha IS NOT NULL;
```

### 3. Optimizar Queries Complejas

```sql
-- En vez de query larga con múltiples CTEs
-- Usar tablas temporales para claridad y performance

CREATE TEMP TABLE temp_ventas_mes AS
SELECT DATE_TRUNC('month', fecha) as mes, SUM(monto) as total
FROM ventas
GROUP BY mes;

CREATE TEMP TABLE temp_objetivos AS
SELECT mes, objetivo FROM objetivos_mensuales;

SELECT
    v.mes,
    v.total,
    o.objetivo,
    v.total - o.objetivo as diferencia
FROM temp_ventas_mes v
JOIN temp_objetivos o ON v.mes = o.mes;
```

---

## Eliminar Tablas

### DROP TABLE

```sql
-- Eliminar tabla permanentemente
DROP TABLE empleados_backup;

-- Si existe
DROP TABLE IF EXISTS empleados_backup;

-- Múltiples tablas
DROP TABLE IF EXISTS tabla1, tabla2, tabla3;
```

### TRUNCATE vs DROP vs DELETE

Ya cubierto en [02_operaciones_crud.md](02_operaciones_crud.md), pero resumen:

```sql
-- DELETE: elimina filas, reversible, lento
DELETE FROM empleados;

-- TRUNCATE: elimina todas las filas, rápido, no reversible
TRUNCATE TABLE empleados;

-- DROP: elimina tabla completa (estructura + datos)
DROP TABLE empleados;
```

---

## Tablas Temporales vs CTEs

| Característica | Temp Table | CTE |
|----------------|------------|-----|
| **Duración** | Sesión completa | Solo la query |
| **Reutilizable** | En múltiples queries | Solo en una query |
| **Indexable** | ✓ Sí | ✗ No |
| **Estadísticas** | ✓ Sí (ANALYZE) | ✗ No |
| **Overhead** | Mayor | Menor |

### Cuándo usar cada uno:

**Temp Table:**
```sql
-- Cuando necesitas reutilizar en múltiples queries
CREATE TEMP TABLE datos_procesados AS
SELECT ... FROM ... WHERE ...;

-- Query 1
SELECT * FROM datos_procesados WHERE ...;

-- Query 2
SELECT COUNT(*) FROM datos_procesados;

-- Query 3
SELECT * FROM datos_procesados JOIN otra_tabla ...;
```

**CTE:**
```sql
-- Cuando solo necesitas en una query
WITH datos_procesados AS (
    SELECT ... FROM ... WHERE ...
)
SELECT * FROM datos_procesados WHERE ...;
```

---

## Clonación de Tablas (Snowflake)

### Zero-Copy Clone

Snowflake permite **clonar tablas instantáneamente** sin copiar datos.

```sql
-- Clone completo
CREATE TABLE empleados_dev CLONE empleados;

-- Clone en momento específico (Time Travel)
CREATE TABLE empleados_ayer CLONE empleados
AT(OFFSET => -86400);  -- 24 horas atrás

-- Clone desde timestamp
CREATE TABLE empleados_backup CLONE empleados
AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);

-- Clone de schema completo
CREATE SCHEMA dev_schema CLONE production_schema;
```

**Ventajas:**
- ✅ Instantáneo (no copia datos físicamente)
- ✅ Sin costo de almacenamiento inicial
- ✅ Solo paga por cambios divergentes

---

## Ciclo de Vida de Tablas

### 1. Crear

```sql
CREATE TABLE empleados (...);
```

### 2. Modificar

```sql
-- Agregar columna
ALTER TABLE empleados ADD COLUMN telefono VARCHAR(20);

-- Eliminar columna
ALTER TABLE empleados DROP COLUMN telefono;

-- Renombrar
ALTER TABLE empleados RENAME TO trabajadores;
```

### 3. Respaldar

```sql
-- Backup
CREATE TABLE empleados_backup AS
SELECT * FROM empleados;

-- O export
COPY empleados TO '/backup/empleados.csv' CSV HEADER;
```

### 4. Restaurar

```sql
-- Desde backup
INSERT INTO empleados
SELECT * FROM empleados_backup;

-- O import
COPY empleados FROM '/backup/empleados.csv' CSV HEADER;
```

### 5. Eliminar

```sql
DROP TABLE empleados;
```

---

## Data Retention (Snowflake)

### Time Travel

Acceder a datos **históricos**.

```sql
-- Ver datos de hace 1 hora
SELECT * FROM empleados
AT(OFFSET => -3600);

-- Restaurar tabla eliminada (dentro de retention period)
UNDROP TABLE empleados;

-- Restaurar a estado anterior
CREATE TABLE empleados_restaurada CLONE empleados
AT(TIMESTAMP => '2024-01-15 09:00:00'::TIMESTAMP);
```

### Configurar Retention

```sql
-- Tabla con 7 días de Time Travel
CREATE TABLE empleados (...)
DATA_RETENTION_TIME_IN_DAYS = 7;

-- Cambiar retention
ALTER TABLE empleados
SET DATA_RETENTION_TIME_IN_DAYS = 30;  -- Máximo 90 en Enterprise
```

---

## Mejores Prácticas

### Tablas Temporales

1. ✅ **Nombra claramente:** `temp_`, `staging_`, `tmp_`
2. ✅ **Limpia explícitamente** si es crítico (aunque auto-eliminan)
   ```sql
   DROP TABLE IF EXISTS temp_calculos;
   ```
3. ✅ **Usa para staging ETL**
4. ✅ **Indexa si necesario** para performance
5. ❌ **No uses para datos críticos** (se pierden al cerrar sesión)

### Gestión de Tablas

1. ✅ **Backups regulares** de tablas importantes
2. ✅ **Documenta estructura** con comments
   ```sql
   COMMENT ON TABLE empleados IS 'Tabla maestra de empleados activos';
   ```
3. ✅ **Monitorea tamaño** de tablas
4. ✅ **Archiva datos antiguos** (particiones antiguas)
5. ✅ **Usa TRANSIENT** en Snowflake para staging (ahorra costos)

### Snowflake Específico

1. ✅ **TRANSIENT para staging/temp**
2. ✅ **CLONE para dev/test** (zero-copy)
3. ✅ **Time Travel** para recuperación rápida
4. ✅ **Retention apropiado** (balance costo vs necesidad)

---

## Ejemplo Práctico: Pipeline ETL

```sql
-- 1. Crear staging temporal
CREATE TEMP TABLE staging_ventas (
    fecha VARCHAR,
    producto VARCHAR,
    cantidad VARCHAR,
    monto VARCHAR,
    region VARCHAR
);

-- 2. Cargar datos crudos
COPY staging_ventas
FROM 's3://bucket/ventas.csv'
FILE_FORMAT = (TYPE = CSV);

-- 3. Validar y limpiar
CREATE TEMP TABLE staging_limpio AS
SELECT
    TO_DATE(fecha, 'YYYY-MM-DD') as fecha,
    TRIM(producto) as producto,
    cantidad::INT as cantidad,
    monto::DECIMAL(10,2) as monto,
    UPPER(region) as region
FROM staging_ventas
WHERE fecha IS NOT NULL
  AND cantidad ~ '^[0-9]+$'
  AND monto ~ '^[0-9.]+$';

-- 4. Insertar en tabla final
INSERT INTO ventas (fecha, producto_id, cantidad, monto, region_id)
SELECT
    s.fecha,
    p.id,
    s.cantidad,
    s.monto,
    r.id
FROM staging_limpio s
JOIN productos p ON s.producto = p.nombre
JOIN regiones r ON s.region = r.codigo;

-- 5. Temp tables se eliminan automáticamente al finalizar sesión
```

---

## Resumen

| Tipo de Tabla | Duración | Uso Principal |
|---------------|----------|---------------|
| **TEMPORARY** | Sesión | Cálculos intermedios |
| **TRANSIENT** | Permanente (Snowflake) | Staging, datos reproducibles |
| **PERMANENT** | Permanente | Datos de producción |
| **CTE** | Solo la query | Simplificación de queries |
