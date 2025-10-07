# 3. Consultas Básicas en SQL (Data Retrieval Essentials)

---

## Seleccionar Columnas Específicas

```sql
-- Todas las columnas
SELECT * FROM empleados;

-- Columnas específicas
SELECT nombre, salario FROM empleados;

-- Columnas con alias
SELECT
    nombre AS empleado,
    salario AS sueldo_mensual
FROM empleados;

-- Cálculos en columnas
SELECT
    nombre,
    salario,
    salario * 12 AS salario_anual,
    salario * 0.15 AS impuestos
FROM empleados;
```

---

## Filtrar Datos con WHERE

### Operadores de Comparación

```sql
-- Igualdad
SELECT * FROM empleados WHERE departamento = 'IT';

-- Mayor que
SELECT * FROM empleados WHERE salario > 50000;

-- Mayor o igual
SELECT * FROM empleados WHERE salario >= 50000;

-- Menor que
SELECT * FROM empleados WHERE edad < 30;

-- Diferente
SELECT * FROM empleados WHERE departamento != 'Ventas';
SELECT * FROM empleados WHERE departamento <> 'Ventas';  -- Alternativa
```

### AND, OR, NOT

```sql
-- AND: Ambas condiciones deben cumplirse
SELECT * FROM empleados
WHERE departamento = 'IT' AND salario > 60000;

-- OR: Al menos una condición debe cumplirse
SELECT * FROM empleados
WHERE departamento = 'IT' OR departamento = 'Ventas';

-- NOT: Niega la condición
SELECT * FROM empleados
WHERE NOT departamento = 'Marketing';

-- Combinación con paréntesis (importante para precedencia)
SELECT * FROM empleados
WHERE (departamento = 'IT' OR departamento = 'Ventas')
  AND salario > 55000;
```

### IN — Múltiples Valores Posibles

```sql
-- En lugar de múltiples OR
SELECT * FROM empleados
WHERE departamento IN ('IT', 'Ventas', 'Marketing');

-- Equivalente a:
SELECT * FROM empleados
WHERE departamento = 'IT'
   OR departamento = 'Ventas'
   OR departamento = 'Marketing';

-- NOT IN
SELECT * FROM empleados
WHERE departamento NOT IN ('IT', 'HR');

-- IN con subconsulta
SELECT * FROM empleados
WHERE departamento_id IN (
    SELECT id FROM departamentos WHERE presupuesto > 100000
);
```

### BETWEEN — Rangos de Valores

```sql
-- Salario entre 40,000 y 60,000 (inclusivo)
SELECT * FROM empleados
WHERE salario BETWEEN 40000 AND 60000;

-- Equivalente a:
SELECT * FROM empleados
WHERE salario >= 40000 AND salario <= 60000;

-- Fechas
SELECT * FROM pedidos
WHERE fecha_pedido BETWEEN '2024-01-01' AND '2024-12-31';

-- NOT BETWEEN
SELECT * FROM empleados
WHERE salario NOT BETWEEN 40000 AND 60000;
```

### LIKE — Patrones de Texto

```sql
-- % = cualquier secuencia de caracteres (0 o más)
-- _ = exactamente un carácter

-- Nombres que empiezan con 'A'
SELECT * FROM empleados WHERE nombre LIKE 'A%';

-- Nombres que terminan con 'ez'
SELECT * FROM empleados WHERE nombre LIKE '%ez';

-- Nombres que contienen 'ana'
SELECT * FROM empleados WHERE nombre LIKE '%ana%';

-- Nombres de exactamente 5 caracteres
SELECT * FROM empleados WHERE nombre LIKE '_____';

-- Nombres que empiezan con A y tienen al menos 4 caracteres
SELECT * FROM empleados WHERE nombre LIKE 'A___%';

-- Case-insensitive (depende del DB)
SELECT * FROM empleados WHERE nombre ILIKE 'ana%';  -- PostgreSQL

-- NOT LIKE
SELECT * FROM empleados WHERE email NOT LIKE '%@gmail.com';
```

**Ejemplos de patrones:**
- `'A%'` → Ana, Antonio, Alberto
- `'%ez'` → López, Martínez, Pérez
- `'%ana%'` → Ana, Mariana, Susana
- `'M_r_a'` → María, Marta
- `'___'` → Cualquier texto de exactamente 3 caracteres

---

## Ordenar Resultados con ORDER BY

```sql
-- Orden ascendente (por defecto)
SELECT * FROM empleados
ORDER BY salario;

SELECT * FROM empleados
ORDER BY salario ASC;  -- Explícito

-- Orden descendente
SELECT * FROM empleados
ORDER BY salario DESC;

-- Ordenar por múltiples columnas
SELECT * FROM empleados
ORDER BY departamento ASC, salario DESC;
-- Primero por departamento (A-Z), luego por salario (mayor a menor)

-- Ordenar por alias
SELECT
    nombre,
    salario * 12 AS salario_anual
FROM empleados
ORDER BY salario_anual DESC;

-- Ordenar por posición de columna (no recomendado, pero válido)
SELECT nombre, salario FROM empleados
ORDER BY 2 DESC;  -- Ordena por la segunda columna (salario)

-- NULL al final o al principio
SELECT * FROM empleados
ORDER BY fecha_salida NULLS LAST;   -- PostgreSQL

SELECT * FROM empleados
ORDER BY fecha_salida NULLS FIRST;  -- PostgreSQL
```

---

## Limitar Resultados con LIMIT / TOP

### PostgreSQL, MySQL, Snowflake, BigQuery:
```sql
-- Primeros 10 registros
SELECT * FROM empleados
LIMIT 10;

-- Top 5 salarios más altos
SELECT nombre, salario FROM empleados
ORDER BY salario DESC
LIMIT 5;

-- OFFSET para paginación
SELECT * FROM empleados
LIMIT 10 OFFSET 20;  -- Registros 21-30

-- Paginación (página 3, 10 registros por página)
SELECT * FROM empleados
LIMIT 10 OFFSET 20;
```

### SQL Server:
```sql
-- Top 10
SELECT TOP 10 * FROM empleados;

-- Top 10 con porcentaje
SELECT TOP 10 PERCENT * FROM empleados;

-- Con ORDER BY
SELECT TOP 5 nombre, salario
FROM empleados
ORDER BY salario DESC;

-- OFFSET FETCH (SQL Server 2012+)
SELECT * FROM empleados
ORDER BY id
OFFSET 20 ROWS
FETCH NEXT 10 ROWS ONLY;
```

---

## Manejo de Datos Faltantes (NULL)

### IS NULL / IS NOT NULL

```sql
-- Empleados sin email
SELECT * FROM empleados
WHERE email IS NULL;

-- Empleados con email
SELECT * FROM empleados
WHERE email IS NOT NULL;

-- ⚠️ INCORRECTO (NULL no se compara con =)
SELECT * FROM empleados WHERE email = NULL;  -- ¡NO FUNCIONA!
```

### COALESCE — Reemplazar NULL

```sql
-- Devuelve el primer valor no-NULL
SELECT
    nombre,
    COALESCE(telefono, 'Sin teléfono') AS telefono
FROM empleados;

-- Múltiples opciones
SELECT
    nombre,
    COALESCE(telefono_movil, telefono_fijo, telefono_oficina, 'Sin contacto') AS telefono
FROM empleados;

-- Con cálculos
SELECT
    nombre,
    salario + COALESCE(bonus, 0) AS salario_total
FROM empleados;
```

### NVL — Reemplazar NULL (Oracle, Snowflake)

```sql
-- Similar a COALESCE pero con 2 argumentos
SELECT
    nombre,
    NVL(telefono, 'Sin teléfono') AS telefono
FROM empleados;

-- NVL2: tres argumentos (si no es null, si es null)
SELECT
    nombre,
    NVL2(email, 'Tiene email', 'Sin email') AS estado_email
FROM empleados;
```

### NULLIF — Convertir Valores a NULL

```sql
-- Convierte '' a NULL
SELECT
    nombre,
    NULLIF(telefono, '') AS telefono
FROM empleados;

-- Si salario es 0, devuelve NULL
SELECT
    nombre,
    NULLIF(salario, 0) AS salario
FROM empleados;
```

### IFNULL (MySQL) / ISNULL (SQL Server)

```sql
-- MySQL
SELECT nombre, IFNULL(telefono, 'N/A') FROM empleados;

-- SQL Server
SELECT nombre, ISNULL(telefono, 'N/A') FROM empleados;
```

---

## DISTINCT — Valores Únicos

```sql
-- Todos los departamentos únicos
SELECT DISTINCT departamento FROM empleados;

-- Combinaciones únicas de departamento y ciudad
SELECT DISTINCT departamento, ciudad FROM empleados;

-- Contar valores únicos
SELECT COUNT(DISTINCT departamento) AS total_departamentos
FROM empleados;

-- DISTINCT con ORDER BY
SELECT DISTINCT departamento
FROM empleados
ORDER BY departamento;
```

**Diferencia DISTINCT vs GROUP BY:**
```sql
-- DISTINCT (más simple, solo para eliminar duplicados)
SELECT DISTINCT departamento FROM empleados;

-- GROUP BY (más poderoso, permite agregaciones)
SELECT departamento, COUNT(*) AS total
FROM empleados
GROUP BY departamento;
```

---

## Ejemplos Combinados

```sql
-- Consulta completa combinando varios conceptos
SELECT DISTINCT
    departamento,
    COALESCE(ciudad, 'Sin asignar') AS ciudad,
    COUNT(*) AS total_empleados,
    AVG(salario) AS salario_promedio
FROM empleados
WHERE salario IS NOT NULL
  AND departamento IN ('IT', 'Ventas', 'Marketing')
  AND fecha_ingreso BETWEEN '2020-01-01' AND '2024-12-31'
GROUP BY departamento, ciudad
HAVING COUNT(*) >= 5
ORDER BY salario_promedio DESC
LIMIT 10;
```

---

## Resumen de Operadores

| Categoría | Operadores |
|-----------|------------|
| **Comparación** | `=`, `!=`, `<>`, `>`, `<`, `>=`, `<=` |
| **Lógicos** | `AND`, `OR`, `NOT` |
| **Rangos** | `BETWEEN`, `NOT BETWEEN` |
| **Listas** | `IN`, `NOT IN` |
| **Patrones** | `LIKE`, `NOT LIKE`, `ILIKE` |
| **NULL** | `IS NULL`, `IS NOT NULL` |

## Funciones para NULL

| Función | DB | Descripción |
|---------|-----|-------------|
| `COALESCE(val1, val2, ...)` | Todos | Primer valor no-NULL |
| `NVL(val, default)` | Oracle, Snowflake | Reemplaza NULL |
| `IFNULL(val, default)` | MySQL | Reemplaza NULL |
| `ISNULL(val, default)` | SQL Server | Reemplaza NULL |
| `NULLIF(val1, val2)` | Todos | NULL si val1 = val2 |

---

## Orden de Ejecución de una Query

```sql
SELECT DISTINCT columnas          -- 5. Selecciona columnas
FROM tabla                        -- 1. Define tabla
WHERE condiciones                 -- 2. Filtra filas
GROUP BY columnas                 -- 3. Agrupa
HAVING condiciones_grupo          -- 4. Filtra grupos
ORDER BY columnas                 -- 6. Ordena resultados
LIMIT n;                          -- 7. Limita resultados
```

Este orden lógico es importante para entender por qué:
- No puedes usar alias de SELECT en WHERE
- Puedes usar alias de SELECT en ORDER BY
- HAVING filtra después de GROUP BY, WHERE filtra antes
