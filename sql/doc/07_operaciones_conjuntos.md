# 7. Operaciones de Conjuntos (Set Operations)

Las operaciones de conjuntos combinan resultados de **dos o más queries** en un único resultado.

**Reglas importantes:**
- Las queries deben tener el **mismo número de columnas**
- Los **tipos de datos** deben ser compatibles
- Los nombres de columnas se toman de la **primera query**

---

## UNION — Combinar Resultados (Sin Duplicados)

Combina resultados de múltiples queries y **elimina duplicados**.

### Sintaxis

```sql
SELECT columnas FROM tabla1
UNION
SELECT columnas FROM tabla2;
```

### Ejemplos

```sql
-- Todos los nombres de empleados y clientes
SELECT nombre FROM empleados
UNION
SELECT nombre FROM clientes;

-- Con filtros diferentes
SELECT nombre, 'Empleado' AS tipo
FROM empleados
WHERE departamento = 'IT'
UNION
SELECT nombre, 'Cliente' AS tipo
FROM clientes
WHERE ciudad = 'Madrid';
```

**Resultado:**
```
nombre  | tipo
--------|----------
Ana     | Empleado
Carlos  | Empleado
María   | Cliente
Juan    | Cliente
```

**Nota:** UNION ordena y elimina duplicados, lo que puede ser costoso.

---

## UNION ALL — Combinar Resultados (Con Duplicados)

Similar a UNION pero **mantiene duplicados** y es **más rápido**.

### Sintaxis

```sql
SELECT columnas FROM tabla1
UNION ALL
SELECT columnas FROM tabla2;
```

### Ejemplos

```sql
-- Todos los registros, incluyendo duplicados
SELECT nombre FROM empleados
UNION ALL
SELECT nombre FROM clientes;

-- Combinar datos históricos con actuales
SELECT * FROM ventas_2023
UNION ALL
SELECT * FROM ventas_2024;
```

### UNION vs UNION ALL

| UNION | UNION ALL |
|-------|-----------|
| Elimina duplicados | Mantiene duplicados |
| Más lento (ordena y compara) | Más rápido |
| Usa cuando necesitas únicos | Usa cuando quieres todo |

**Ejemplo:**
```sql
-- Si ambas tablas tienen "Ana"
SELECT nombre FROM tabla1  -- Ana
UNION
SELECT nombre FROM tabla2; -- Ana
-- Resultado: Ana (1 vez)

SELECT nombre FROM tabla1  -- Ana
UNION ALL
SELECT nombre FROM tabla2; -- Ana
-- Resultado: Ana (2 veces)
```

---

## INTERSECT — Intersección (Valores Comunes)

Devuelve solo las filas que **existen en ambas queries**.

### Sintaxis

```sql
SELECT columnas FROM tabla1
INTERSECT
SELECT columnas FROM tabla2;
```

### Ejemplos

```sql
-- Personas que son empleados Y clientes
SELECT nombre FROM empleados
INTERSECT
SELECT nombre FROM clientes;

-- Productos vendidos en ambos años
SELECT producto_id FROM ventas_2023
INTERSECT
SELECT producto_id FROM ventas_2024;
```

**Diagrama de Venn:**
```
Empleados     Clientes
   ┌───┐       ┌───┐
   │   │   ∩   │   │  ← INTERSECT (solo la intersección)
   └───┘       └───┘
```

**⚠️ MySQL no soporta INTERSECT**. Alternativa con INNER JOIN:
```sql
-- INTERSECT simulado en MySQL
SELECT DISTINCT e.nombre
FROM empleados e
INNER JOIN clientes c ON e.nombre = c.nombre;

-- O con EXISTS
SELECT nombre FROM empleados
WHERE nombre IN (SELECT nombre FROM clientes);
```

---

## EXCEPT / MINUS — Diferencia (Excluir Valores)

Devuelve filas de la **primera query que NO están en la segunda**.

### Sintaxis

```sql
-- PostgreSQL, SQL Server, BigQuery
SELECT columnas FROM tabla1
EXCEPT
SELECT columnas FROM tabla2;

-- Oracle, Snowflake (también soportan EXCEPT)
SELECT columnas FROM tabla1
MINUS
SELECT columnas FROM tabla2;
```

### Ejemplos

```sql
-- Empleados que NO son clientes
SELECT nombre FROM empleados
EXCEPT
SELECT nombre FROM clientes;

-- Productos del catálogo que nunca se vendieron
SELECT producto_id FROM productos
EXCEPT
SELECT DISTINCT producto_id FROM ventas;

-- Clientes del año pasado que no compraron este año
SELECT cliente_id FROM ventas_2023
EXCEPT
SELECT cliente_id FROM ventas_2024;
```

**Diagrama de Venn:**
```
Empleados     Clientes
   ┌───┐       ┌───┐
   │ A │       │ B │  ← EXCEPT (solo A, excluyendo intersección)
   └───┘       └───┘
```

**⚠️ MySQL no soporta EXCEPT**. Alternativa con LEFT JOIN:
```sql
-- EXCEPT simulado en MySQL
SELECT e.nombre
FROM empleados e
LEFT JOIN clientes c ON e.nombre = c.nombre
WHERE c.nombre IS NULL;

-- O con NOT IN
SELECT nombre FROM empleados
WHERE nombre NOT IN (SELECT nombre FROM clientes WHERE nombre IS NOT NULL);

-- O con NOT EXISTS (mejor performance)
SELECT nombre FROM empleados e
WHERE NOT EXISTS (
    SELECT 1 FROM clientes c WHERE c.nombre = e.nombre
);
```

---

## Combinar Múltiples Operaciones

```sql
-- Combinando UNION y EXCEPT
(SELECT nombre FROM empleados_it
 UNION
 SELECT nombre FROM empleados_ventas)
EXCEPT
SELECT nombre FROM empleados_inactivos;

-- Con paréntesis para controlar precedencia
SELECT nombre FROM tabla1
UNION
(SELECT nombre FROM tabla2
 INTERSECT
 SELECT nombre FROM tabla3);
```

---

## Ordenar Resultados de Set Operations

`ORDER BY` debe ir **al final** de todas las operaciones.

```sql
SELECT nombre, salario FROM empleados
UNION ALL
SELECT nombre, salario FROM contractors
ORDER BY salario DESC;  -- Ordena el resultado combinado

-- ✗ INCORRECTO (ORDER BY en medio)
SELECT nombre FROM empleados ORDER BY nombre
UNION
SELECT nombre FROM clientes;

-- ✓ CORRECTO
SELECT nombre FROM empleados
UNION
SELECT nombre FROM clientes
ORDER BY nombre;
```

---

## Usar Alias en Set Operations

```sql
-- Los nombres de columnas vienen de la primera query
SELECT nombre AS empleado, departamento
FROM empleados
UNION
SELECT nombre, empresa  -- 'empresa' se mostrará como 'departamento'
FROM contractors;

-- Mejor: asegurar mismos alias
SELECT nombre AS persona, departamento AS organizacion
FROM empleados
UNION
SELECT nombre AS persona, empresa AS organizacion
FROM contractors;
```

---

## Casos de Uso Prácticos

### 1. Consolidar Datos de Múltiples Tablas

```sql
-- Reporte consolidado de todas las transacciones
SELECT
    fecha,
    'Venta' AS tipo,
    monto,
    cliente_id
FROM ventas
UNION ALL
SELECT
    fecha,
    'Devolución' AS tipo,
    -monto,
    cliente_id
FROM devoluciones
ORDER BY fecha DESC;
```

### 2. Encontrar Registros Huérfanos

```sql
-- Pedidos sin cliente válido
SELECT pedido_id FROM pedidos
EXCEPT
SELECT p.pedido_id
FROM pedidos p
JOIN clientes c ON p.cliente_id = c.id;
```

### 3. Análisis de Cambios (Nuevos, Eliminados, Comunes)

```sql
-- Clientes nuevos este mes
SELECT cliente_id FROM clientes_marzo
EXCEPT
SELECT cliente_id FROM clientes_febrero;

-- Clientes que se dieron de baja
SELECT cliente_id FROM clientes_febrero
EXCEPT
SELECT cliente_id FROM clientes_marzo;

-- Clientes que permanecen
SELECT cliente_id FROM clientes_febrero
INTERSECT
SELECT cliente_id FROM clientes_marzo;
```

### 4. Combinar Particiones de Datos

```sql
-- Combinar datos particionados por fecha
SELECT * FROM ventas_q1_2024
UNION ALL
SELECT * FROM ventas_q2_2024
UNION ALL
SELECT * FROM ventas_q3_2024
UNION ALL
SELECT * FROM ventas_q4_2024;
```

---

## Compatibilidad de Tipos de Datos

```sql
-- ✓ CORRECTO: tipos compatibles
SELECT id, nombre FROM empleados    -- INT, VARCHAR
UNION
SELECT id, nombre FROM clientes;    -- INT, VARCHAR

-- ✓ CORRECTO: conversión implícita
SELECT id FROM empleados            -- INT
UNION
SELECT id::TEXT FROM clientes;      -- TEXT (convertido de INT)

-- ✗ ERROR: tipos incompatibles
SELECT id, nombre FROM empleados    -- INT, VARCHAR
UNION
SELECT nombre, id FROM clientes;    -- VARCHAR, INT (orden incorrecto)

-- ✗ ERROR: diferente número de columnas
SELECT id, nombre FROM empleados    -- 2 columnas
UNION
SELECT id FROM clientes;            -- 1 columna
```

---

## Performance Tips

### 1. UNION ALL es más rápido que UNION
```sql
-- Si sabes que no hay duplicados, usa UNION ALL
SELECT * FROM ventas_online
UNION ALL  -- Más rápido
SELECT * FROM ventas_tienda;
```

### 2. Filtrar antes de combinar
```sql
-- ✓ MEJOR: filtra primero
SELECT nombre FROM empleados WHERE activo = true
UNION
SELECT nombre FROM contractors WHERE activo = true;

-- ✗ PEOR: combina primero
SELECT nombre FROM (
    SELECT nombre, activo FROM empleados
    UNION
    SELECT nombre, activo FROM contractors
) AS todos
WHERE activo = true;
```

### 3. Indexar columnas usadas en INTERSECT/EXCEPT
```sql
-- Crear índices para mejorar performance
CREATE INDEX idx_empleados_nombre ON empleados(nombre);
CREATE INDEX idx_clientes_nombre ON clientes(nombre);

SELECT nombre FROM empleados
INTERSECT
SELECT nombre FROM clientes;
```

---

## Resumen de Operaciones de Conjuntos

| Operación | Resultado | Duplicados | Soportado en MySQL |
|-----------|-----------|------------|-------------------|
| **UNION** | A ∪ B (combinados, únicos) | Elimina | ✓ Sí |
| **UNION ALL** | A + B (todos) | Mantiene | ✓ Sí |
| **INTERSECT** | A ∩ B (comunes) | Elimina | ✗ No |
| **EXCEPT/MINUS** | A - B (solo en A) | Elimina | ✗ No |

## Alternativas en MySQL

| Operación SQL | Alternativa MySQL |
|---------------|-------------------|
| `INTERSECT` | `INNER JOIN` o `IN` |
| `EXCEPT` | `LEFT JOIN ... WHERE NULL` o `NOT IN` o `NOT EXISTS` |

---

## Diagrama Visual Completo

```
Conjunto A: Empleados (Ana, Carlos, María, Juan)
Conjunto B: Clientes   (María, Juan, Pedro, Laura)

UNION (únicos):
Ana, Carlos, María, Juan, Pedro, Laura

UNION ALL (todos):
Ana, Carlos, María, Juan, María, Juan, Pedro, Laura

INTERSECT (comunes):
María, Juan

EXCEPT (solo en A):
Ana, Carlos
```

---

## Mejores Prácticas

1. **Usa UNION ALL** si no necesitas eliminar duplicados (más rápido)
2. **Asegura mismo orden** de columnas en todas las queries
3. **Usa alias descriptivos** para claridad
4. **Filtra antes** de combinar para mejor performance
5. **ORDER BY al final** de todas las operaciones
6. **Verifica compatibilidad** de tipos de datos
7. **En MySQL**, usa JOINs/EXISTS para simular INTERSECT/EXCEPT
