# 10. Índices y Optimización de Performance

---

## ¿Qué es un Índice?

Un **índice** es una estructura de datos que mejora la velocidad de las operaciones de consulta en una tabla, similar a un índice en un libro.

**Analogía:** En vez de leer todo el libro para encontrar "SQL", miras el índice al final que dice "SQL - página 42".

### Cómo Funciona

**Sin índice:**
```
Buscar WHERE nombre = 'Ana' en 1,000,000 filas
→ Escanea TODAS las filas (Full Table Scan)
→ Tiempo: Alto
```

**Con índice en 'nombre':**
```
Buscar WHERE nombre = 'Ana' en índice
→ Búsqueda directa (como árbol binario)
→ Tiempo: Bajo
```

---

## Tipos de Índices

### 1. B-Tree Index (Más Común)

Estructura de árbol balanceado. **Default en la mayoría de DBs**.

**Mejor para:**
- Búsquedas exactas: `WHERE id = 5`
- Rangos: `WHERE salario BETWEEN 40000 AND 60000`
- Ordenamiento: `ORDER BY fecha`
- Comparaciones: `WHERE fecha > '2024-01-01'`

```sql
-- Crear índice B-tree
CREATE INDEX idx_empleados_nombre ON empleados(nombre);

-- Índice compuesto (múltiples columnas)
CREATE INDEX idx_empleados_dept_salario ON empleados(departamento_id, salario);
```

### 2. Bitmap Index

Usa mapas de bits para representar valores. **Eficiente para columnas con baja cardinalidad** (pocos valores únicos).

**Mejor para:**
- Columnas con pocos valores distintos: `genero`, `activo`, `estado`
- Data warehouses (Oracle, Snowflake, Redshift)
- Queries con múltiples condiciones AND/OR

```sql
-- Oracle, Snowflake
CREATE BITMAP INDEX idx_empleados_genero ON empleados(genero);
```

**Ejemplo:** Columna `genero` con solo 'M', 'F', 'Otro':
```
Registro | M | F | Otro
1        | 1 | 0 | 0
2        | 0 | 1 | 0
3        | 1 | 0 | 0
```

### 3. Hash Index

Usa función hash para mapear valores a ubicaciones. **Solo para igualdad exacta**.

**Mejor para:**
- Búsquedas exactas: `WHERE id = 123`
- NO para rangos o ORDER BY

```sql
-- PostgreSQL (raro en la práctica)
CREATE INDEX idx_empleados_email USING HASH ON empleados(email);
```

❌ No soporta: `WHERE id > 100`, `ORDER BY id`

### 4. Clustered Index (Índice Agrupado)

**Reorganiza físicamente** los datos de la tabla según el índice.
- **Solo puede haber UNO** por tabla
- En SQL Server, Primary Key es clustered por defecto

```sql
-- SQL Server
CREATE CLUSTERED INDEX idx_empleados_id ON empleados(id);

-- PostgreSQL (CLUSTER reorganiza, no mantiene orden)
CLUSTER empleados USING idx_empleados_id;
```

### 5. Non-Clustered Index (Índice No Agrupado)

Estructura separada que apunta a los datos.
- **Pueden ser múltiples** por tabla
- Todos los índices no-PK en SQL Server son non-clustered

```sql
-- SQL Server
CREATE NONCLUSTERED INDEX idx_empleados_nombre ON empleados(nombre);
```

---

## Ventajas y Desventajas de Índices

### Ventajas
✅ **Queries más rápidas** (SELECT con WHERE, JOIN, ORDER BY)
✅ **Mejora performance** de búsquedas
✅ **Acelera JOINs**
✅ **Mejora ORDER BY** y GROUP BY

### Desventajas
❌ **Espacio adicional** en disco
❌ **INSERT/UPDATE/DELETE más lentos** (índice debe actualizarse)
❌ **Mantenimiento** (fragmentación)
❌ **Demasiados índices** pueden ser contraproducentes

---

## Cuándo Crear Índices

### ✅ SÍ crear índice en:

1. **Primary Keys** (automático)
2. **Foreign Keys** (para JOINs)
3. **Columnas en WHERE frecuente**
   ```sql
   -- Si haces frecuentemente:
   SELECT * FROM empleados WHERE departamento_id = 5;
   -- Crear:
   CREATE INDEX idx_empleados_dept ON empleados(departamento_id);
   ```
4. **Columnas en JOIN**
   ```sql
   -- Para JOIN rápido
   CREATE INDEX idx_pedidos_cliente ON pedidos(cliente_id);
   ```
5. **Columnas en ORDER BY**
6. **Columnas con alta cardinalidad** (muchos valores únicos)

### ❌ NO crear índice en:

1. **Tablas pequeñas** (< 1000 filas)
2. **Columnas raramente usadas** en WHERE/JOIN
3. **Columnas con baja cardinalidad** (ej: booleanos) - excepto bitmap
4. **Columnas que cambian mucho** (overhead en UPDATE)
5. **Cuando hay más escrituras** que lecturas

---

## Índices Compuestos (Multi-columna)

Índice sobre **múltiples columnas**. Orden importa.

```sql
CREATE INDEX idx_empleados_dept_salario
ON empleados(departamento_id, salario);
```

### Regla del Prefijo Izquierdo

El índice se usa si la query filtra por:
- ✅ Primera columna sola: `WHERE departamento_id = 1`
- ✅ Ambas columnas: `WHERE departamento_id = 1 AND salario > 50000`
- ❌ Solo segunda columna: `WHERE salario > 50000` (NO usa el índice)

**Orden óptimo:** Columna más selectiva primero (o la más usada en WHERE).

```sql
-- Si haces frecuentemente:
SELECT * FROM empleados
WHERE departamento_id = 1 AND salario > 50000;

-- Índice óptimo:
CREATE INDEX idx_dept_salario ON empleados(departamento_id, salario);
```

---

## Query Execution Plans (Planes de Ejecución)

Muestra **cómo** la base de datos ejecuta una query.

### PostgreSQL

```sql
EXPLAIN SELECT * FROM empleados WHERE salario > 50000;

-- Con costos detallados
EXPLAIN ANALYZE SELECT * FROM empleados WHERE salario > 50000;
```

**Resultado:**
```
Seq Scan on empleados  (cost=0.00..15.00 rows=100 width=200)
  Filter: (salario > 50000)

-- Seq Scan = Full Table Scan (malo para tablas grandes)
```

**Con índice:**
```
Index Scan using idx_salario on empleados  (cost=0.29..8.45 rows=100 width=200)
  Index Cond: (salario > 50000)

-- Index Scan = usa índice (bueno)
```

### MySQL

```sql
EXPLAIN SELECT * FROM empleados WHERE salario > 50000;
```

### SQL Server

```sql
SET SHOWPLAN_TEXT ON;
GO
SELECT * FROM empleados WHERE salario > 50000;
GO
SET SHOWPLAN_TEXT OFF;
```

### Snowflake

```sql
-- Ver query profile en UI
SELECT * FROM empleados WHERE salario > 50000;

-- O usar EXPLAIN
EXPLAIN USING TEXT SELECT * FROM empleados WHERE salario > 50000;
```

---

## Optimización de Queries

### 1. Evitar SELECT *

```sql
-- ❌ Malo: trae columnas innecesarias
SELECT * FROM empleados WHERE id = 5;

-- ✅ Bueno: solo las columnas necesarias
SELECT id, nombre, salario FROM empleados WHERE id = 5;
```

### 2. Usar Índices en Columnas de Filtro/JOIN

```sql
-- Asegurar índices en:
CREATE INDEX idx_pedidos_cliente ON pedidos(cliente_id);
CREATE INDEX idx_pedidos_fecha ON pedidos(fecha);

-- Para queries como:
SELECT * FROM pedidos
WHERE cliente_id = 123
  AND fecha > '2024-01-01';
```

### 3. Filtrar Temprano (WHERE antes de JOIN cuando sea posible)

```sql
-- ❌ Malo: JOIN primero, filtra después
SELECT e.nombre
FROM empleados e
JOIN departamentos d ON e.departamento_id = d.id
WHERE d.nombre = 'IT';

-- ✅ Mejor: filtra departamento primero
SELECT e.nombre
FROM empleados e
JOIN (SELECT id FROM departamentos WHERE nombre = 'IT') d
  ON e.departamento_id = d.id;
```

### 4. Evitar Funciones en WHERE (rompe índices)

```sql
-- ❌ Malo: función en columna indexada
SELECT * FROM empleados
WHERE UPPER(nombre) = 'ANA';  -- No usa índice en 'nombre'

-- ✅ Bueno: comparación directa
SELECT * FROM empleados
WHERE nombre = 'Ana';  -- Usa índice

-- ✅ Alternativa: índice funcional
CREATE INDEX idx_empleados_nombre_upper ON empleados(UPPER(nombre));
```

### 5. LIMIT para Pruebas

```sql
-- Prueba con subset primero
SELECT * FROM ventas
WHERE fecha > '2024-01-01'
LIMIT 100;  -- Verifica query antes de correr en millones
```

### 6. EXISTS vs IN

```sql
-- ✅ Mejor performance (especialmente con grandes datasets)
SELECT * FROM empleados e
WHERE EXISTS (
    SELECT 1 FROM proyectos p WHERE p.empleado_id = e.id
);

-- Puede ser más lento
SELECT * FROM empleados
WHERE id IN (SELECT empleado_id FROM proyectos);
```

### 7. Evitar OR con Diferentes Columnas

```sql
-- ❌ Malo: difícil de optimizar
SELECT * FROM empleados
WHERE nombre = 'Ana' OR salario > 50000;

-- ✅ Mejor: UNION (puede usar índices en cada parte)
SELECT * FROM empleados WHERE nombre = 'Ana'
UNION
SELECT * FROM empleados WHERE salario > 50000;
```

### 8. JOINs en Orden Correcto

```sql
-- Unir tabla pequeña primero
SELECT *
FROM tabla_pequeña tp
JOIN tabla_grande tg ON tp.id = tg.tp_id;
```

---

## Analizar Performance

### PostgreSQL - ANALYZE

```sql
-- Actualizar estadísticas de la tabla
ANALYZE empleados;

-- Ver estadísticas
SELECT * FROM pg_stats WHERE tablename = 'empleados';
```

### MySQL - ANALYZE TABLE

```sql
ANALYZE TABLE empleados;
```

### Snowflake - Query Profile

```sql
-- Ver en UI Web después de ejecutar query
SELECT * FROM empleados WHERE salario > 50000;

-- O historial de queries
SELECT *
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_TEXT LIKE '%empleados%'
ORDER BY START_TIME DESC
LIMIT 10;
```

---

## Mantenimiento de Índices

### Ver Índices Existentes

```sql
-- PostgreSQL
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'empleados';

-- MySQL
SHOW INDEX FROM empleados;

-- SQL Server
EXEC sp_helpindex 'empleados';

-- Snowflake
SHOW INDEXES IN empleados;
```

### Eliminar Índices

```sql
DROP INDEX idx_empleados_nombre;

-- Si existe
DROP INDEX IF EXISTS idx_empleados_nombre;
```

### Reconstruir Índices (desfragmentar)

```sql
-- PostgreSQL
REINDEX TABLE empleados;
REINDEX INDEX idx_empleados_nombre;

-- SQL Server
ALTER INDEX idx_empleados_nombre ON empleados REBUILD;

-- MySQL
ALTER TABLE empleados ENGINE=InnoDB;  -- Reconstruye tabla e índices
```

---

## Resumen de Tipos de Índices

| Tipo | Uso Principal | DB Común |
|------|---------------|----------|
| **B-Tree** | General (rangos, búsquedas) | Todos |
| **Bitmap** | Baja cardinalidad, DW | Oracle, Snowflake, Redshift |
| **Hash** | Igualdad exacta | PostgreSQL (raro) |
| **Clustered** | 1 por tabla, reorganiza datos | SQL Server |
| **Non-Clustered** | Múltiples, estructura separada | SQL Server |

---

## Anti-Patrones Comunes

1. ❌ **Índice en todo:** Demasiados índices ralentizan INSERT/UPDATE
2. ❌ **No analizar query plans:** Optimizar sin evidencia
3. ❌ **SELECT * siempre:** Trae datos innecesarios
4. ❌ **Funciones en WHERE:** Rompe uso de índices
5. ❌ **Índices sin usar:** Ocupan espacio sin beneficio

---

## Mejores Prácticas

1. ✅ **Indexar Primary Keys y Foreign Keys**
2. ✅ **Usar EXPLAIN/ANALYZE** antes de optimizar
3. ✅ **Índices compuestos** para queries comunes
4. ✅ **Monitorear uso** de índices (eliminar no usados)
5. ✅ **Actualizar estadísticas** regularmente (ANALYZE)
6. ✅ **Solo columnas necesarias** en SELECT
7. ✅ **Filtrar antes de JOIN** cuando sea posible
8. ✅ **Testear en producción-like data** (volumen similar)
