# 11. Particionamiento y Clustering

---

## ¿Qué es el Particionamiento?

**Particionamiento** divide una tabla grande en **piezas más pequeñas** (particiones) basadas en un criterio (fecha, rango, lista, hash).

**Ventajas:**
- ✅ Queries más rápidas (escanea solo particiones relevantes)
- ✅ Mantenimiento más fácil (eliminar particiones antiguas)
- ✅ Mejor paralelización
- ✅ Reduce costos en cloud (menos datos escaneados)

**Concepto clave:** **Partition Pruning** = eliminar particiones irrelevantes del scan.

---

## Tipos de Particionamiento

### 1. Range Partitioning (Por Rango)

Particiona por **rangos de valores** (común para fechas).

```sql
-- PostgreSQL
CREATE TABLE ventas (
    id SERIAL,
    fecha DATE,
    monto DECIMAL
) PARTITION BY RANGE (fecha);

-- Crear particiones
CREATE TABLE ventas_2023 PARTITION OF ventas
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE ventas_2024 PARTITION OF ventas
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**Uso:**
```sql
-- Solo escanea partición ventas_2024
SELECT * FROM ventas
WHERE fecha >= '2024-01-01' AND fecha < '2024-02-01';
```

### 2. List Partitioning (Por Lista)

Particiona por **valores específicos**.

```sql
-- PostgreSQL
CREATE TABLE empleados (
    id SERIAL,
    nombre VARCHAR(100),
    region VARCHAR(20)
) PARTITION BY LIST (region);

CREATE TABLE empleados_norte PARTITION OF empleados
    FOR VALUES IN ('Norte', 'Noroeste');

CREATE TABLE empleados_sur PARTITION OF empleados
    FOR VALUES IN ('Sur', 'Sureste');
```

### 3. Hash Partitioning (Por Hash)

Particiona usando **función hash**. Distribuye datos uniformemente.

```sql
-- PostgreSQL
CREATE TABLE pedidos (
    id SERIAL,
    cliente_id INT,
    fecha DATE
) PARTITION BY HASH (cliente_id);

CREATE TABLE pedidos_p0 PARTITION OF pedidos
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE pedidos_p1 PARTITION OF pedidos
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

-- ... p2, p3
```

**Uso:** Cuando no hay criterio natural de rango/lista, pero quieres distribuir carga.

### 4. Composite Partitioning (Particionamiento Compuesto)

Combina múltiples estrategias.

```sql
-- Primero por rango (fecha), luego por hash (región)
CREATE TABLE ventas (
    id SERIAL,
    fecha DATE,
    region VARCHAR(20),
    monto DECIMAL
) PARTITION BY RANGE (fecha);

CREATE TABLE ventas_2024 PARTITION OF ventas
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01')
    PARTITION BY LIST (region);

-- Sub-particiones
CREATE TABLE ventas_2024_norte PARTITION OF ventas_2024
    FOR VALUES IN ('Norte');
```

---

## Particionamiento en Diferentes Sistemas

### Snowflake - Clustering Keys

Snowflake usa **micro-particiones automáticas** + **clustering keys** para organización.

```sql
-- Crear tabla con clustering key
CREATE TABLE ventas (
    id INT,
    fecha DATE,
    cliente_id INT,
    monto DECIMAL
) CLUSTER BY (fecha);

-- Clustering compuesto
CREATE TABLE ventas
CLUSTER BY (fecha, cliente_id);

-- Agregar clustering a tabla existente
ALTER TABLE ventas CLUSTER BY (fecha);
```

**Cómo funciona:**
- Snowflake automáticamente particiona en micro-particiones (50-500 MB)
- Clustering key organiza datos dentro de micro-particiones
- Auto-mantenimiento en background (puede tener costo)

**Verificar clustering:**
```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('ventas', '(fecha)');
```

### BigQuery - Partitioning & Clustering

```sql
-- Particionamiento por fecha
CREATE TABLE proyecto.dataset.ventas (
    id INT64,
    fecha DATE,
    producto STRING,
    monto FLOAT64
)
PARTITION BY fecha
CLUSTER BY producto;

-- Particionamiento por mes
CREATE TABLE ventas
PARTITION BY DATE_TRUNC(fecha, MONTH)
CLUSTER BY producto, region;

-- Particionamiento por columna timestamp
CREATE TABLE eventos
PARTITION BY DATE(_PARTITIONTIME)
CLUSTER BY usuario_id;
```

**Límites BigQuery:**
- Hasta 4,000 particiones por tabla
- Hasta 4 columnas de clustering

### Amazon Redshift - Distribution & Sort Keys

```sql
-- Distribution key (cómo se distribuye entre nodos)
CREATE TABLE ventas (
    id INT,
    fecha DATE,
    cliente_id INT,
    monto DECIMAL
)
DISTKEY(cliente_id)     -- Distribuir por cliente_id
SORTKEY(fecha);         -- Ordenar por fecha

-- Compound sort key (múltiples columnas)
SORTKEY(fecha, cliente_id);

-- Interleaved sort key (para queries con diferentes filtros)
INTERLEAVED SORTKEY(fecha, cliente_id, region);
```

---

## Partition Pruning — Reducción de Costos

**Partition Pruning** = Base de datos **ignora particiones** que no cumplen filtro WHERE.

### Ejemplo en BigQuery

```sql
-- Tabla particionada por fecha
CREATE TABLE ventas
PARTITION BY DATE(fecha);

-- ✅ BUENO: Escanea solo 1 día
SELECT * FROM ventas
WHERE DATE(fecha) = '2024-01-15';
-- Costo: ~1 día de datos

-- ❌ MALO: Escanea TODAS las particiones
SELECT * FROM ventas
WHERE EXTRACT(YEAR FROM fecha) = 2024;
-- Costo: TODO el año (función rompe partition pruning)

-- ✅ BUENO: Rango específico
SELECT * FROM ventas
WHERE fecha BETWEEN '2024-01-01' AND '2024-01-31';
-- Costo: ~1 mes de datos
```

### Ejemplo en Snowflake

```sql
-- Tabla clustered por fecha
CREATE TABLE ventas CLUSTER BY (fecha);

-- ✅ Partition pruning efectivo
SELECT * FROM ventas
WHERE fecha = '2024-01-15';

-- Ver particiones escaneadas
SELECT *
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_ID = LAST_QUERY_ID();
```

---

## Cuándo Particionar

### ✅ SÍ particionar cuando:
1. **Tablas muy grandes** (> 100 GB, > 100M filas)
2. **Queries filtran por columna específica** (fecha, región)
3. **Mantenimiento por período** (eliminar datos antiguos)
4. **Reducir costos** en cloud (menos datos escaneados)
5. **Paralelizar** queries grandes

### ❌ NO particionar cuando:
1. **Tablas pequeñas** (< 10 GB)
2. **No hay patrón claro** de acceso
3. **Queries escanean toda la tabla** siempre
4. **Overhead > beneficio**

---

## Estrategias de Particionamiento Común

### 1. Por Fecha (Más Común)

```sql
-- BigQuery
CREATE TABLE logs
PARTITION BY DATE(timestamp);

-- PostgreSQL
CREATE TABLE logs (...) PARTITION BY RANGE (timestamp);

-- Snowflake
CREATE TABLE logs (...) CLUSTER BY (DATE(timestamp));
```

**Casos de uso:**
- Logs, eventos, transacciones
- Eliminar datos antiguos fácilmente
- Queries por período de tiempo

### 2. Por Región/Ubicación

```sql
CREATE TABLE ventas
PARTITION BY LIST (region);
```

**Casos de uso:**
- Datos geográficos
- Multi-tenant (un tenant por partición)

### 3. Por ID (Hash)

```sql
CREATE TABLE usuarios
PARTITION BY HASH (user_id);
```

**Casos de uso:**
- Distribución uniforme
- Escalabilidad horizontal

---

## Clustering en Cloud Data Warehouses

### Snowflake Clustering

```sql
-- Crear tabla con clustering
CREATE TABLE ventas (
    fecha DATE,
    producto_id INT,
    monto DECIMAL
) CLUSTER BY (fecha, producto_id);

-- Ver información de clustering
SELECT SYSTEM$CLUSTERING_INFORMATION('ventas');

-- Clustering ratio: 0-100 (más alto = mejor)
-- Si < 50, considerar re-clustering manual
ALTER TABLE ventas RECLUSTER;
```

**Costos:**
- Auto-clustering consume créditos
- Considera suspender auto-clustering si tabla no cambia mucho

### BigQuery Clustering

```sql
CREATE TABLE ventas (
    fecha DATE,
    producto_id INT64,
    region STRING,
    monto FLOAT64
)
PARTITION BY fecha
CLUSTER BY producto_id, region;
```

**Beneficios:**
- Hasta 4 columnas de clustering
- Mejora queries con filtros en columnas clustered
- Reduce bytes escaneados (menor costo)

**Orden de clustering:** Más selectiva primero.

---

## Eliminar Particiones Antiguas

### PostgreSQL

```sql
-- Eliminar partición completa (rápido)
DROP TABLE ventas_2020;

-- Desconectar sin eliminar
ALTER TABLE ventas DETACH PARTITION ventas_2020;
```

### BigQuery

```sql
-- Eliminar particiones antiguas (> 90 días)
DELETE FROM ventas
WHERE fecha < CURRENT_DATE() - 90;

-- O configurar expiración automática
ALTER TABLE ventas
SET OPTIONS (
    partition_expiration_days = 90
);
```

### Snowflake

```sql
-- Eliminar datos antiguos
DELETE FROM ventas
WHERE fecha < DATEADD(day, -365, CURRENT_DATE());

-- Opción: usar Time Travel para recuperar si necesario
```

---

## Redshift: Distribution Styles

### EVEN (Default)

```sql
CREATE TABLE productos (...)
DISTSTYLE EVEN;
-- Distribuye filas uniformemente (round-robin)
```

### KEY

```sql
CREATE TABLE ventas (...)
DISTKEY(cliente_id);
-- Distribuye por hash de columna (co-location para JOINs)
```

### ALL

```sql
CREATE TABLE dim_producto (...)
DISTSTYLE ALL;
-- Replica tabla en TODOS los nodos (para tablas pequeñas de dimensión)
```

### AUTO

```sql
CREATE TABLE ventas (...)
DISTSTYLE AUTO;
-- Redshift decide automáticamente
```

---

## Mejores Prácticas

### Particionamiento
1. ✅ **Particiona por fecha** en tablas de eventos/logs
2. ✅ **Usa partition pruning** (filtra por columna de partición)
3. ✅ **Evita funciones** en columna de partición en WHERE
4. ✅ **Elimina particiones antiguas** para mantener performance
5. ✅ **Documenta estrategia** de particionamiento

### Clustering (Snowflake/BigQuery)
1. ✅ **Clustering en columnas de filtro frecuente**
2. ✅ **Orden: alta cardinalidad → baja cardinalidad**
3. ✅ **Máximo 4 columnas** (BigQuery) o 3-4 (Snowflake)
4. ✅ **Monitorea clustering ratio** (Snowflake)
5. ✅ **No sobre-clusterizar** (costo vs beneficio)

### Redshift
1. ✅ **DISTKEY en columna de JOIN** principal
2. ✅ **SORTKEY en columna de filtro/ORDER BY**
3. ✅ **DISTSTYLE ALL** para tablas pequeñas de dimensión
4. ✅ **Vacuum y Analyze** regularmente

---

## Resumen: Estrategias por Sistema

| Sistema | Estrategia | Sintaxis Principal |
|---------|------------|-------------------|
| **PostgreSQL** | Particionamiento manual | `PARTITION BY RANGE/LIST/HASH` |
| **Snowflake** | Micro-particiones auto + clustering | `CLUSTER BY (col)` |
| **BigQuery** | Particionamiento + clustering | `PARTITION BY col CLUSTER BY col` |
| **Redshift** | Distribution + sort keys | `DISTKEY/SORTKEY` |

---

## Ejemplo Completo: Tabla de Logs

```sql
-- BigQuery
CREATE TABLE proyecto.dataset.logs (
    timestamp TIMESTAMP,
    usuario_id INT64,
    evento STRING,
    datos JSON
)
PARTITION BY DATE(timestamp)
CLUSTER BY usuario_id, evento
OPTIONS (
    partition_expiration_days = 365,
    description = "Logs de eventos con retención de 1 año"
);

-- Query optimizada (usa partition pruning y clustering)
SELECT evento, COUNT(*) as total
FROM logs
WHERE DATE(timestamp) = '2024-01-15'  -- Partition pruning
  AND usuario_id = 12345              -- Clustering
GROUP BY evento;
```
