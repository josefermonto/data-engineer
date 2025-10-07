# 6. Subqueries y CTEs (Common Table Expressions)

---

## Subqueries (Subconsultas)

Una subquery es una consulta **dentro de otra consulta**. Se puede usar en:
- Cláusula **WHERE**
- Cláusula **FROM**
- Cláusula **SELECT**
- Cláusula **HAVING**

---

## Subqueries en WHERE

### Comparación con Valor Único

```sql
-- Empleados con salario mayor al promedio
SELECT nombre, salario
FROM empleados
WHERE salario > (
    SELECT AVG(salario) FROM empleados
);

-- Empleado con el salario más alto
SELECT nombre, salario
FROM empleados
WHERE salario = (
    SELECT MAX(salario) FROM empleados
);
```

### IN — Múltiples Valores

```sql
-- Empleados en departamentos con presupuesto > 100k
SELECT nombre, departamento_id
FROM empleados
WHERE departamento_id IN (
    SELECT id
    FROM departamentos
    WHERE presupuesto > 100000
);

-- NOT IN
SELECT nombre
FROM empleados
WHERE departamento_id NOT IN (
    SELECT id FROM departamentos WHERE nombre = 'Marketing'
);
```

**⚠️ Cuidado con NULL en NOT IN:**
```sql
-- Si la subquery devuelve NULL, NOT IN puede no funcionar como esperas
SELECT nombre
FROM empleados
WHERE departamento_id NOT IN (1, 2, NULL);  -- No devuelve filas

-- Mejor: excluir NULLs en subquery
SELECT nombre
FROM empleados
WHERE departamento_id NOT IN (
    SELECT id FROM departamentos WHERE id IS NOT NULL
);
```

---

## EXISTS vs IN

### EXISTS — Verifica Existencia

```sql
-- Empleados que tienen al menos un proyecto
SELECT e.nombre
FROM empleados e
WHERE EXISTS (
    SELECT 1
    FROM proyectos p
    WHERE p.empleado_id = e.id
);

-- NOT EXISTS
SELECT e.nombre
FROM empleados e
WHERE NOT EXISTS (
    SELECT 1
    FROM proyectos p
    WHERE p.empleado_id = e.id
);
```

### EXISTS vs IN: ¿Cuál usar?

| EXISTS | IN |
|--------|-----|
| Mejor para grandes datasets | Mejor para listas pequeñas |
| Para cuando hay correlated subquery | Para lista estática o subquery simple |
| Corta ejecución al encontrar match | Evalúa toda la subquery |
| Maneja NULL mejor | Problemas con NULL en NOT IN |

**Rendimiento:**
```sql
-- EXISTS (mejor performance generalmente)
WHERE EXISTS (SELECT 1 FROM tabla2 WHERE tabla1.id = tabla2.id)

-- IN (más simple pero puede ser más lento)
WHERE id IN (SELECT id FROM tabla2)
```

**Correlated Subquery** (subconsulta correlacionada):
```sql
-- EXISTS es correlated (referencia tabla externa)
SELECT e.nombre
FROM empleados e
WHERE EXISTS (
    SELECT 1
    FROM departamentos d
    WHERE d.id = e.departamento_id  -- ← Referencia a 'e'
    AND d.presupuesto > 100000
);
```

---

## Subqueries en FROM (Derived Tables)

```sql
-- Salario promedio por departamento, luego filtrar
SELECT *
FROM (
    SELECT
        departamento,
        AVG(salario) AS salario_promedio
    FROM empleados
    GROUP BY departamento
) AS promedios
WHERE salario_promedio > 60000;
```

**⚠️ Importante:** La subquery en FROM **debe tener un alias**.

```sql
-- ✗ INCORRECTO (falta alias)
SELECT * FROM (SELECT * FROM empleados);

-- ✓ CORRECTO
SELECT * FROM (SELECT * FROM empleados) AS emp;
```

---

## Subqueries en SELECT (Scalar Subquery)

Devuelve un **único valor** por cada fila.

```sql
-- Mostrar salario y promedio general
SELECT
    nombre,
    salario,
    (SELECT AVG(salario) FROM empleados) AS salario_promedio_empresa,
    salario - (SELECT AVG(salario) FROM empleados) AS diferencia_promedio
FROM empleados;

-- Correlated: salario vs promedio del departamento
SELECT
    nombre,
    departamento_id,
    salario,
    (
        SELECT AVG(salario)
        FROM empleados e2
        WHERE e2.departamento_id = e1.departamento_id
    ) AS salario_promedio_dept
FROM empleados e1;
```

**⚠️ La subquery debe devolver 1 valor o dará error:**
```sql
-- ✗ ERROR: devuelve múltiples filas
SELECT nombre, (SELECT salario FROM empleados) FROM empleados;

-- ✓ CORRECTO: devuelve 1 valor
SELECT nombre, (SELECT MAX(salario) FROM empleados) FROM empleados;
```

---

## Common Table Expressions (CTEs) — WITH clause

CTEs son **subqueries nombradas** que se definen al inicio de la query. Son más legibles y reutilizables.

### Sintaxis

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT ...
FROM cte_name;
```

### Ejemplos

```sql
-- En vez de subquery en FROM
WITH promedios AS (
    SELECT
        departamento,
        AVG(salario) AS salario_promedio
    FROM empleados
    GROUP BY departamento
)
SELECT *
FROM promedios
WHERE salario_promedio > 60000;
```

### Múltiples CTEs

```sql
WITH
-- Primera CTE
empleados_it AS (
    SELECT * FROM empleados WHERE departamento = 'IT'
),
-- Segunda CTE (puede referenciar la primera)
salarios_it AS (
    SELECT
        AVG(salario) AS promedio,
        MAX(salario) AS maximo
    FROM empleados_it
)
SELECT * FROM salarios_it;
```

### CTEs vs Subqueries

```sql
-- SUBQUERY (difícil de leer si se reutiliza)
SELECT e.nombre, e.salario
FROM empleados e
WHERE e.salario > (SELECT AVG(salario) FROM empleados)
  AND e.departamento_id IN (SELECT id FROM departamentos WHERE presupuesto > 100000);

-- CTE (más claro)
WITH
promedio_empresa AS (
    SELECT AVG(salario) AS promedio FROM empleados
),
departamentos_grandes AS (
    SELECT id FROM departamentos WHERE presupuesto > 100000
)
SELECT e.nombre, e.salario
FROM empleados e, promedio_empresa
WHERE e.salario > promedio_empresa.promedio
  AND e.departamento_id IN (SELECT id FROM departamentos_grandes);
```

---

## Recursive CTEs — Consultas Recursivas

Para datos jerárquicos o grafos (organigramas, categorías, rutas).

### Sintaxis

```sql
WITH RECURSIVE cte_name AS (
    -- Caso base (anchor)
    SELECT ...

    UNION ALL

    -- Caso recursivo
    SELECT ...
    FROM cte_name
    WHERE condición_de_parada
)
SELECT * FROM cte_name;
```

### Ejemplo: Jerarquía de Empleados

```sql
-- Tabla empleados con manager_id
CREATE TABLE empleados (
    id INT,
    nombre VARCHAR(100),
    manager_id INT
);

-- Encontrar todos los subordinados de un manager
WITH RECURSIVE subordinados AS (
    -- Caso base: el manager inicial
    SELECT id, nombre, manager_id, 1 AS nivel
    FROM empleados
    WHERE id = 1  -- CEO

    UNION ALL

    -- Caso recursivo: subordinados de subordinados
    SELECT e.id, e.nombre, e.manager_id, s.nivel + 1
    FROM empleados e
    INNER JOIN subordinados s ON e.manager_id = s.id
)
SELECT * FROM subordinados;
```

**Resultado:**
```
id | nombre  | manager_id | nivel
---|---------|------------|------
1  | Ana     | NULL       | 1
2  | Carlos  | 1          | 2
3  | María   | 1          | 2
4  | Juan    | 2          | 3
```

### Ejemplo: Categorías Anidadas

```sql
WITH RECURSIVE categoria_arbol AS (
    -- Categorías raíz
    SELECT id, nombre, parent_id, 0 AS profundidad
    FROM categorias
    WHERE parent_id IS NULL

    UNION ALL

    -- Subcategorías
    SELECT c.id, c.nombre, c.parent_id, ca.profundidad + 1
    FROM categorias c
    INNER JOIN categoria_arbol ca ON c.parent_id = ca.id
)
SELECT
    REPEAT('  ', profundidad) || nombre AS categoria_indentada
FROM categoria_arbol
ORDER BY profundidad;
```

**⚠️ Cuidado con loops infinitos:**
```sql
-- Agregar límite de profundidad
WITH RECURSIVE subordinados AS (
    SELECT id, nombre, manager_id, 1 AS nivel
    FROM empleados
    WHERE id = 1

    UNION ALL

    SELECT e.id, e.nombre, e.manager_id, s.nivel + 1
    FROM empleados e
    INNER JOIN subordinados s ON e.manager_id = s.id
    WHERE s.nivel < 10  -- ← Límite de profundidad
)
SELECT * FROM subordinados;
```

---

## Diferencias: CTEs, Subqueries, Temporary Tables

| Característica | CTE | Subquery | Temp Table |
|----------------|-----|----------|------------|
| **Sintaxis** | `WITH nombre AS (...)` | `(SELECT ...)` | `CREATE TEMP TABLE` |
| **Reutilizable en query** | ✓ Sí | ✗ No (se repite) | ✓ Sí |
| **Persiste entre queries** | ✗ No | ✗ No | ✓ Sí (en sesión) |
| **Recursión** | ✓ Sí | ✗ No | ✗ No |
| **Legibilidad** | ✓✓✓ Alta | ✓ Media | ✓✓ Alta |
| **Performance** | Similar | Similar | Puede ser mejor |
| **Indexable** | ✗ No | ✗ No | ✓ Sí |

### CTE
```sql
WITH empleados_recientes AS (
    SELECT * FROM empleados WHERE fecha_ingreso > '2023-01-01'
)
SELECT * FROM empleados_recientes WHERE salario > 50000;
```

### Subquery
```sql
SELECT * FROM (
    SELECT * FROM empleados WHERE fecha_ingreso > '2023-01-01'
) AS empleados_recientes
WHERE salario > 50000;
```

### Temporary Table
```sql
CREATE TEMP TABLE empleados_recientes AS
SELECT * FROM empleados WHERE fecha_ingreso > '2023-01-01';

-- Crear índice
CREATE INDEX idx_salario ON empleados_recientes(salario);

-- Reutilizar en múltiples queries
SELECT * FROM empleados_recientes WHERE salario > 50000;
SELECT * FROM empleados_recientes WHERE departamento = 'IT';

-- Se elimina al cerrar sesión (o manualmente)
DROP TABLE empleados_recientes;
```

---

## Cuándo Usar Cada Uno

### Usa **Subquery** cuando:
- Es simple y se usa una sola vez
- En WHERE con IN/EXISTS
- Necesitas un valor escalar

### Usa **CTE** cuando:
- Necesitas reutilizar la query
- Quieres mejorar legibilidad
- Necesitas recursión
- Query compleja con múltiples pasos

### Usa **Temporary Table** cuando:
- Resultado intermedio grande
- Lo usarás en múltiples queries separadas
- Necesitas indexar el resultado
- Quieres optimizar performance con estadísticas

---

## Ejemplos Prácticos Combinados

### Encontrar Top 3 Salarios por Departamento

```sql
-- Con window function y CTE
WITH ranking_salarios AS (
    SELECT
        departamento,
        nombre,
        salario,
        DENSE_RANK() OVER (PARTITION BY departamento ORDER BY salario DESC) AS rank
    FROM empleados
)
SELECT departamento, nombre, salario
FROM ranking_salarios
WHERE rank <= 3;
```

### Comparar con Promedios

```sql
WITH estadisticas AS (
    SELECT
        departamento,
        AVG(salario) AS promedio_dept,
        (SELECT AVG(salario) FROM empleados) AS promedio_empresa
    FROM empleados
    GROUP BY departamento
)
SELECT
    e.nombre,
    e.departamento,
    e.salario,
    s.promedio_dept,
    s.promedio_empresa,
    e.salario - s.promedio_dept AS diff_dept,
    e.salario - s.promedio_empresa AS diff_empresa
FROM empleados e
JOIN estadisticas s ON e.departamento = s.departamento;
```

---

## Resumen

| Concepto | Uso Principal |
|----------|---------------|
| **Subquery en WHERE** | Filtrar con IN, EXISTS, comparaciones |
| **Subquery en FROM** | Crear tabla derivada |
| **Subquery en SELECT** | Calcular valor escalar por fila |
| **CTE (WITH)** | Mejorar legibilidad, reutilizar queries |
| **Recursive CTE** | Jerarquías, grafos, datos recursivos |
| **EXISTS vs IN** | EXISTS mejor para performance, IN más simple |

## Mejores Prácticas

1. **Prefiere CTEs** sobre subqueries anidadas para legibilidad
2. **Usa EXISTS** en vez de IN para grandes datasets
3. **Cuidado con NULL** en NOT IN
4. **Agrega alias siempre** a subqueries en FROM
5. **Usa RECURSIVE CTEs** para jerarquías en vez de loops en código
6. **Considera temporary tables** para resultados grandes reutilizables
