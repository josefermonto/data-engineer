# 9. Vistas (Views)

---

## ¿Qué es una Vista?

Una **vista** es una **consulta guardada** que se comporta como una tabla virtual.
- No almacena datos físicamente (solo la definición de la query)
- Se calcula en tiempo real cuando se consulta
- Simplifica queries complejas
- Proporciona capa de abstracción y seguridad

---

## Tipos de Vistas

### 1. Standard View (Vista Lógica)

Vista tradicional que **no almacena datos**.

#### Crear Vista

```sql
CREATE VIEW vista_empleados_activos AS
SELECT
    id,
    nombre,
    departamento,
    salario
FROM empleados
WHERE activo = true;
```

#### Usar Vista

```sql
-- Se usa como si fuera una tabla
SELECT * FROM vista_empleados_activos;

SELECT nombre, salario
FROM vista_empleados_activos
WHERE departamento = 'IT';
```

#### Actualizar Vista

```sql
CREATE OR REPLACE VIEW vista_empleados_activos AS
SELECT
    id,
    nombre,
    departamento,
    salario,
    email  -- Nueva columna agregada
FROM empleados
WHERE activo = true;
```

#### Eliminar Vista

```sql
DROP VIEW vista_empleados_activos;

-- Si existe
DROP VIEW IF EXISTS vista_empleados_activos;
```

### 2. Materialized View (Vista Materializada)

Vista que **almacena físicamente los datos** en disco.

#### Crear Vista Materializada

```sql
-- PostgreSQL, Snowflake, Redshift
CREATE MATERIALIZED VIEW mv_ventas_mensuales AS
SELECT
    DATE_TRUNC('month', fecha) AS mes,
    SUM(monto) AS total_ventas,
    COUNT(*) AS num_transacciones,
    AVG(monto) AS ticket_promedio
FROM ventas
GROUP BY DATE_TRUNC('month', fecha);
```

#### Refrescar Vista Materializada

```sql
-- PostgreSQL
REFRESH MATERIALIZED VIEW mv_ventas_mensuales;

-- Snowflake (automático según configuración)
-- Las MVs en Snowflake se actualizan automáticamente en background

-- BigQuery (actualización automática o manual)
-- BigQuery refresca MVs automáticamente, pero puedes forzar:
-- No hay comando manual en BigQuery, es automático
```

#### Eliminar Vista Materializada

```sql
DROP MATERIALIZED VIEW mv_ventas_mensuales;
```

---

## Standard View vs Materialized View

| Característica | Standard View | Materialized View |
|----------------|---------------|-------------------|
| **Almacenamiento** | No (solo query) | Sí (datos físicos) |
| **Performance** | Depende de query base | Rápido (pre-calculado) |
| **Datos** | Siempre actuales | Pueden estar desactualizados |
| **Espacio** | Mínimo | Requiere espacio |
| **Refresh** | Automático | Manual o programado |
| **Índices** | No | Sí (indexable) |
| **Uso** | Simplicidad, seguridad | Performance, agregaciones |

### Cuándo Usar Cada Una

**Standard View:**
- Queries simples
- Datos deben estar siempre actualizados
- No hay problema de performance
- Simplificar acceso/seguridad

**Materialized View:**
- Queries complejas con agregaciones
- Datos no necesitan estar al segundo
- Performance crítica
- Reportes frecuentes sobre datos grandes

---

## Casos de Uso de Vistas

### 1. Simplificación de Queries Complejas

```sql
-- En vez de escribir esto cada vez:
SELECT
    e.nombre AS empleado,
    d.nombre AS departamento,
    c.nombre AS ciudad,
    e.salario,
    e.fecha_ingreso
FROM empleados e
JOIN departamentos d ON e.departamento_id = d.id
JOIN ciudades c ON e.ciudad_id = c.id
WHERE e.activo = true;

-- Crear vista:
CREATE VIEW v_empleados_completo AS
SELECT
    e.id,
    e.nombre AS empleado,
    d.nombre AS departamento,
    c.nombre AS ciudad,
    e.salario,
    e.fecha_ingreso
FROM empleados e
JOIN departamentos d ON e.departamento_id = d.id
JOIN ciudades c ON e.ciudad_id = c.id
WHERE e.activo = true;

-- Usar vista simple:
SELECT * FROM v_empleados_completo WHERE departamento = 'IT';
```

### 2. Seguridad y Control de Acceso

```sql
-- Ocultar columnas sensibles
CREATE VIEW v_empleados_publico AS
SELECT
    id,
    nombre,
    departamento,
    ciudad
    -- SIN: salario, SSN, dirección
FROM empleados;

-- Otorgar acceso solo a la vista
GRANT SELECT ON v_empleados_publico TO usuario_lectura;
REVOKE SELECT ON empleados FROM usuario_lectura;
```

### 3. Abstracción de Estructura de Datos

```sql
-- Mantener compatibilidad cuando cambia estructura
CREATE VIEW clientes AS
SELECT
    customer_id AS id,              -- Renombrar columnas
    customer_name AS nombre,
    email_address AS email
FROM nuevo_sistema_clientes;

-- Aplicaciones antiguas siguen funcionando con "clientes"
```

### 4. Agregaciones Pre-calculadas (con Materialized Views)

```sql
CREATE MATERIALIZED VIEW mv_dashboard_ventas AS
SELECT
    DATE_TRUNC('day', fecha) AS dia,
    producto_id,
    SUM(cantidad) AS total_unidades,
    SUM(monto) AS total_ventas,
    AVG(monto) AS ticket_promedio,
    COUNT(DISTINCT cliente_id) AS clientes_unicos
FROM ventas
GROUP BY DATE_TRUNC('day', fecha), producto_id;

-- Dashboard usa la vista materializada (rápido)
SELECT * FROM mv_dashboard_ventas
WHERE dia >= CURRENT_DATE - 30;
```

---

## Vistas Actualizables (Updatable Views)

Algunas vistas permiten INSERT/UPDATE/DELETE.

### Requisitos para Vista Actualizable:
1. SELECT de **una sola tabla**
2. Sin DISTINCT, GROUP BY, HAVING, UNION
3. Sin funciones de agregación
4. Todas las columnas NOT NULL deben estar en la vista

```sql
-- Vista actualizable
CREATE VIEW v_empleados_it AS
SELECT id, nombre, email, salario
FROM empleados
WHERE departamento_id = 1;

-- Se puede actualizar
UPDATE v_empleados_it
SET salario = salario * 1.10
WHERE id = 5;

-- Se puede insertar
INSERT INTO v_empleados_it (id, nombre, email, salario)
VALUES (100, 'Pedro', 'pedro@empresa.com', 55000);
-- Nota: departamento_id se inserta como NULL o default
```

### WITH CHECK OPTION

Evita que INSERT/UPDATE violen la condición WHERE de la vista.

```sql
CREATE VIEW v_empleados_it AS
SELECT id, nombre, email, salario, departamento_id
FROM empleados
WHERE departamento_id = 1
WITH CHECK OPTION;

-- ✓ PERMITIDO
UPDATE v_empleados_it SET salario = 60000 WHERE id = 5;

-- ✗ ERROR: violaría WHERE departamento_id = 1
UPDATE v_empleados_it SET departamento_id = 2 WHERE id = 5;
```

---

## Refresh y Mantenimiento de Materialized Views

### PostgreSQL

```sql
-- Refresh completo (bloquea la vista)
REFRESH MATERIALIZED VIEW mv_ventas_mensuales;

-- Refresh sin bloquear (requiere UNIQUE index)
CREATE UNIQUE INDEX idx_mv_ventas ON mv_ventas_mensuales(mes);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_ventas_mensuales;
```

### Snowflake

```sql
-- Snowflake maneja MVs automáticamente
-- Se actualizan en background cuando cambian datos base

-- Crear con clustering (opcional)
CREATE MATERIALIZED VIEW mv_ventas_mensuales
CLUSTER BY (mes)
AS
SELECT
    DATE_TRUNC('month', fecha) AS mes,
    SUM(monto) AS total_ventas
FROM ventas
GROUP BY mes;

-- Ver estado de la MV
SHOW MATERIALIZED VIEWS LIKE 'mv_ventas_mensuales';
```

### BigQuery

```sql
-- BigQuery actualiza MVs automáticamente
CREATE MATERIALIZED VIEW proyecto.dataset.mv_ventas_mensuales AS
SELECT
    DATE_TRUNC(fecha, MONTH) AS mes,
    SUM(monto) AS total_ventas
FROM proyecto.dataset.ventas
GROUP BY mes;

-- Ver última actualización
SELECT * FROM proyecto.dataset.INFORMATION_SCHEMA.MATERIALIZED_VIEWS
WHERE table_name = 'mv_ventas_mensuales';
```

---

## Diferencias entre Vistas y Tablas

| Característica | Vista | Tabla |
|----------------|-------|-------|
| **Datos físicos** | No | Sí |
| **Espacio en disco** | Mínimo | Sí |
| **Performance** | Depende de query | Rápido |
| **Actualización** | Automática | Manual (INSERT/UPDATE) |
| **Índices** | No (excepto MV) | Sí |
| **Constraints** | No | Sí (PK, FK, etc.) |

---

## Ventajas y Desventajas

### Ventajas de Vistas
✅ Simplifica queries complejas
✅ Reutilización de lógica
✅ Seguridad (ocultar columnas/filas)
✅ Abstracción (independencia de estructura física)
✅ Mantenimiento más fácil (cambio en un lugar)

### Desventajas de Vistas
❌ Performance puede ser peor (especialmente con vistas anidadas)
❌ No se pueden indexar (excepto Materialized Views)
❌ Dificultad para debugging (query oculta)
❌ Limitaciones en INSERT/UPDATE/DELETE

### Ventajas de Materialized Views
✅ Performance excelente
✅ Indexables
✅ Ideal para agregaciones complejas

### Desventajas de Materialized Views
❌ Datos pueden estar desactualizados
❌ Requiere espacio en disco
❌ Costo de refresh
❌ Complejidad de mantenimiento

---

## Mejores Prácticas

1. **Nombra claramente:** Prefijo `v_` para vistas, `mv_` para materialized views
2. **Documenta** el propósito de cada vista
3. **Evita vistas sobre vistas** (impacto en performance)
4. **Usa Materialized Views** para reportes pesados
5. **Programa refresh** de MVs en horarios de baja carga
6. **Monitorea performance** de vistas complejas
7. **WITH CHECK OPTION** para vistas actualizables
8. **Índices en MVs** para mejorar queries sobre ellas

---

## Ejemplos Prácticos

### Vista de Seguridad

```sql
-- Solo mostrar empleados del departamento del usuario
CREATE VIEW v_mi_departamento AS
SELECT id, nombre, email, salario
FROM empleados
WHERE departamento_id = (
    SELECT departamento_id FROM empleados WHERE email = CURRENT_USER
);
```

### Vista de Agregación

```sql
CREATE VIEW v_resumen_departamentos AS
SELECT
    d.nombre AS departamento,
    COUNT(e.id) AS total_empleados,
    AVG(e.salario) AS salario_promedio,
    SUM(e.salario) AS nomina_total,
    MIN(e.fecha_ingreso) AS empleado_mas_antiguo
FROM departamentos d
LEFT JOIN empleados e ON d.id = e.departamento_id
GROUP BY d.id, d.nombre;
```

### Materialized View para Dashboard

```sql
CREATE MATERIALIZED VIEW mv_kpis_ventas AS
SELECT
    DATE(fecha) AS dia,
    COUNT(*) AS transacciones,
    SUM(monto) AS ventas_totales,
    AVG(monto) AS ticket_promedio,
    COUNT(DISTINCT cliente_id) AS clientes_unicos,
    SUM(CASE WHEN monto > 100 THEN 1 ELSE 0 END) AS ventas_premium
FROM ventas
WHERE fecha >= CURRENT_DATE - 365
GROUP BY DATE(fecha);

-- Refresh diario programado
-- (configurar en cron/Airflow/scheduler)
```

---

## Consultar Metadatos de Vistas

```sql
-- PostgreSQL
SELECT * FROM information_schema.views
WHERE table_schema = 'public';

-- Ver definición
SELECT pg_get_viewdef('v_empleados_activos', true);

-- SQL Server
SELECT * FROM INFORMATION_SCHEMA.VIEWS;

-- MySQL
SHOW FULL TABLES WHERE table_type = 'VIEW';

-- Snowflake
SHOW VIEWS IN SCHEMA nombre_schema;
```
