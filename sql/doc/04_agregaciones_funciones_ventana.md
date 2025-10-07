# 4. Agregaciones y Funciones de Ventana (Window Functions)

---

## Funciones de Agregación

Las funciones de agregación operan sobre **múltiples filas** y devuelven **un único valor**.

### COUNT — Contar Filas

```sql
-- Contar todas las filas
SELECT COUNT(*) FROM empleados;

-- Contar valores no-NULL
SELECT COUNT(email) FROM empleados;

-- Contar valores únicos
SELECT COUNT(DISTINCT departamento) FROM empleados;

-- Con filtro
SELECT COUNT(*) FROM empleados WHERE salario > 50000;
```

### SUM — Suma Total

```sql
-- Suma de todos los salarios
SELECT SUM(salario) FROM empleados;

-- Suma por departamento
SELECT
    departamento,
    SUM(salario) AS total_nomina
FROM empleados
GROUP BY departamento;

-- SUM ignora NULL
SELECT SUM(bonus) FROM empleados;  -- NULL no se suma
```

### AVG — Promedio

```sql
-- Salario promedio
SELECT AVG(salario) FROM empleados;

-- Promedio por departamento
SELECT
    departamento,
    AVG(salario) AS salario_promedio
FROM empleados
GROUP BY departamento;

-- AVG ignora NULL
SELECT AVG(bonus) FROM empleados;
```

### MIN y MAX — Mínimo y Máximo

```sql
-- Salario más bajo y más alto
SELECT
    MIN(salario) AS salario_minimo,
    MAX(salario) AS salario_maximo
FROM empleados;

-- Fecha más antigua y más reciente
SELECT
    MIN(fecha_ingreso) AS primer_ingreso,
    MAX(fecha_ingreso) AS ultimo_ingreso
FROM empleados;
```

### Combinando Agregaciones

```sql
SELECT
    departamento,
    COUNT(*) AS total_empleados,
    SUM(salario) AS nomina_total,
    AVG(salario) AS salario_promedio,
    MIN(salario) AS salario_minimo,
    MAX(salario) AS salario_maximo
FROM empleados
GROUP BY departamento;
```

---

## GROUP BY — Agrupar Datos

Agrupa filas que tienen los mismos valores en columnas especificadas.

### Sintaxis Básica

```sql
SELECT
    columna_agrupacion,
    funcion_agregacion(columna)
FROM tabla
GROUP BY columna_agrupacion;
```

### Ejemplos

```sql
-- Contar empleados por departamento
SELECT
    departamento,
    COUNT(*) AS total
FROM empleados
GROUP BY departamento;

-- Múltiples columnas de agrupación
SELECT
    departamento,
    ciudad,
    COUNT(*) AS total,
    AVG(salario) AS salario_promedio
FROM empleados
GROUP BY departamento, ciudad;

-- Con ORDER BY
SELECT
    departamento,
    COUNT(*) AS total
FROM empleados
GROUP BY departamento
ORDER BY total DESC;
```

### Regla Importante

**Toda columna en SELECT que NO esté en una función de agregación DEBE estar en GROUP BY**

```sql
-- ✓ CORRECTO
SELECT departamento, COUNT(*)
FROM empleados
GROUP BY departamento;

-- ✗ INCORRECTO (nombre no está en GROUP BY)
SELECT departamento, nombre, COUNT(*)
FROM empleados
GROUP BY departamento;

-- ✓ CORRECTO (agregando nombre a GROUP BY)
SELECT departamento, nombre, COUNT(*)
FROM empleados
GROUP BY departamento, nombre;
```

---

## HAVING — Filtrar Grupos

`WHERE` filtra filas **antes** de agrupar.
`HAVING` filtra grupos **después** de agrupar.

```sql
-- Departamentos con más de 5 empleados
SELECT
    departamento,
    COUNT(*) AS total
FROM empleados
GROUP BY departamento
HAVING COUNT(*) > 5;

-- Departamentos con salario promedio > 60000
SELECT
    departamento,
    AVG(salario) AS salario_promedio
FROM empleados
GROUP BY departamento
HAVING AVG(salario) > 60000;

-- Combinando WHERE y HAVING
SELECT
    departamento,
    COUNT(*) AS total,
    AVG(salario) AS salario_promedio
FROM empleados
WHERE fecha_ingreso >= '2020-01-01'  -- Filtra filas primero
GROUP BY departamento
HAVING COUNT(*) >= 3;                -- Filtra grupos después
```

### WHERE vs HAVING

| WHERE | HAVING |
|-------|--------|
| Filtra **filas** | Filtra **grupos** |
| Antes de GROUP BY | Después de GROUP BY |
| No puede usar funciones de agregación | Puede usar funciones de agregación |
| `WHERE salario > 50000` | `HAVING AVG(salario) > 50000` |

---

## Funciones de Ventana (Window Functions)

Las funciones de ventana operan sobre un **conjunto de filas** relacionadas pero **devuelven un valor para cada fila** (no colapsan las filas como GROUP BY).

### Sintaxis General

```sql
funcion() OVER (
    [PARTITION BY columna]
    [ORDER BY columna]
    [ROWS/RANGE especificacion]
)
```

- **PARTITION BY**: Divide datos en grupos (como GROUP BY pero sin colapsar)
- **ORDER BY**: Define el orden dentro de cada partición
- **ROWS/RANGE**: Define el marco de la ventana (opcional)

---

## ROW_NUMBER() — Número de Fila

Asigna un número secuencial único a cada fila.

```sql
-- Numerar todas las filas
SELECT
    nombre,
    salario,
    ROW_NUMBER() OVER (ORDER BY salario DESC) AS ranking
FROM empleados;

-- Numerar por departamento
SELECT
    departamento,
    nombre,
    salario,
    ROW_NUMBER() OVER (PARTITION BY departamento ORDER BY salario DESC) AS ranking_dept
FROM empleados;
```

**Resultado ejemplo:**
```
departamento | nombre  | salario | ranking_dept
-------------|---------|---------|-------------
IT           | Ana     | 70000   | 1
IT           | Carlos  | 65000   | 2
IT           | María   | 60000   | 3
Ventas       | Juan    | 75000   | 1
Ventas       | Pedro   | 68000   | 2
```

---

## RANK() — Ranking con Empates

Asigna ranking, pero deja huecos cuando hay empates.

```sql
SELECT
    nombre,
    salario,
    RANK() OVER (ORDER BY salario DESC) AS ranking
FROM empleados;
```

**Ejemplo con empates:**
```
nombre  | salario | RANK()
--------|---------|-------
Ana     | 70000   | 1
Carlos  | 70000   | 1  ← empate
María   | 65000   | 3  ← salta el 2
Juan    | 60000   | 4
```

---

## DENSE_RANK() — Ranking Sin Huecos

Similar a RANK() pero sin huecos en la numeración.

```sql
SELECT
    nombre,
    salario,
    DENSE_RANK() OVER (ORDER BY salario DESC) AS ranking
FROM empleados;
```

**Ejemplo con empates:**
```
nombre  | salario | DENSE_RANK()
--------|---------|-------------
Ana     | 70000   | 1
Carlos  | 70000   | 1  ← empate
María   | 65000   | 2  ← no salta
Juan    | 60000   | 3
```

### Diferencia: ROW_NUMBER vs RANK vs DENSE_RANK

```sql
SELECT
    nombre,
    salario,
    ROW_NUMBER() OVER (ORDER BY salario DESC) AS row_num,
    RANK() OVER (ORDER BY salario DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salario DESC) AS dense_rank
FROM empleados;
```

**Resultado:**
```
nombre  | salario | row_num | rank | dense_rank
--------|---------|---------|------|------------
Ana     | 70000   | 1       | 1    | 1
Carlos  | 70000   | 2       | 1    | 1
María   | 65000   | 3       | 3    | 2
Juan    | 60000   | 4       | 4    | 3
```

- **ROW_NUMBER**: Siempre único (1, 2, 3, 4...)
- **RANK**: Con huecos en empates (1, 1, 3, 4...)
- **DENSE_RANK**: Sin huecos en empates (1, 1, 2, 3...)

---

## NTILE() — Dividir en Grupos

Divide filas en N grupos (cuartiles, percentiles, etc.)

```sql
-- Dividir empleados en 4 grupos por salario
SELECT
    nombre,
    salario,
    NTILE(4) OVER (ORDER BY salario DESC) AS cuartil
FROM empleados;

-- Top 25%, siguiente 25%, etc.
```

**Casos de uso:**
- `NTILE(4)` → Cuartiles (25%, 50%, 75%, 100%)
- `NTILE(10)` → Deciles (10%, 20%, ...)
- `NTILE(100)` → Percentiles

---

## LAG() y LEAD() — Valores Anterior/Siguiente

### LAG() — Valor de Fila Anterior

```sql
-- Comparar salario con el anterior
SELECT
    nombre,
    salario,
    LAG(salario) OVER (ORDER BY salario) AS salario_anterior,
    salario - LAG(salario) OVER (ORDER BY salario) AS diferencia
FROM empleados;

-- Con valor por defecto si no hay anterior
SELECT
    nombre,
    salario,
    LAG(salario, 1, 0) OVER (ORDER BY fecha_ingreso) AS salario_empleado_anterior
FROM empleados;
```

### LEAD() — Valor de Fila Siguiente

```sql
-- Comparar con el siguiente
SELECT
    nombre,
    salario,
    LEAD(salario) OVER (ORDER BY salario) AS salario_siguiente
FROM empleados;
```

**Caso de uso real: Calcular crecimiento**
```sql
-- Ventas mes a mes
SELECT
    mes,
    ventas,
    LAG(ventas) OVER (ORDER BY mes) AS ventas_mes_anterior,
    ventas - LAG(ventas) OVER (ORDER BY mes) AS crecimiento,
    ROUND(100.0 * (ventas - LAG(ventas) OVER (ORDER BY mes)) / LAG(ventas) OVER (ORDER BY mes), 2) AS porcentaje_crecimiento
FROM ventas_mensuales;
```

---

## FIRST_VALUE() y LAST_VALUE() — Primer/Último Valor

### FIRST_VALUE()

```sql
-- Comparar salario con el más alto del departamento
SELECT
    departamento,
    nombre,
    salario,
    FIRST_VALUE(salario) OVER (
        PARTITION BY departamento
        ORDER BY salario DESC
    ) AS salario_mas_alto_dept,
    salario - FIRST_VALUE(salario) OVER (
        PARTITION BY departamento
        ORDER BY salario DESC
    ) AS diferencia_con_top
FROM empleados;
```

### LAST_VALUE()

```sql
-- Comparar con el más bajo del departamento
SELECT
    departamento,
    nombre,
    salario,
    LAST_VALUE(salario) OVER (
        PARTITION BY departamento
        ORDER BY salario DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS salario_mas_bajo_dept
FROM empleados;
```

**⚠️ Importante con LAST_VALUE**: Por defecto, la ventana es `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, lo que puede no dar el resultado esperado. Usa `UNBOUNDED FOLLOWING` para considerar todas las filas.

---

## Combinando Agregaciones con Window Functions

```sql
-- Salario individual vs promedio del departamento
SELECT
    departamento,
    nombre,
    salario,
    AVG(salario) OVER (PARTITION BY departamento) AS salario_promedio_dept,
    salario - AVG(salario) OVER (PARTITION BY departamento) AS diferencia_promedio,
    COUNT(*) OVER (PARTITION BY departamento) AS total_empleados_dept
FROM empleados;
```

**Resultado ejemplo:**
```
departamento | nombre | salario | salario_promedio_dept | diferencia_promedio | total_empleados_dept
-------------|--------|---------|----------------------|---------------------|---------------------
IT           | Ana    | 70000   | 65000                | 5000                | 3
IT           | Carlos | 65000   | 65000                | 0                   | 3
IT           | María  | 60000   | 65000                | -5000               | 3
```

---

## Running Totals y Moving Averages

### Running Total (Suma Acumulativa)

```sql
SELECT
    fecha,
    ventas,
    SUM(ventas) OVER (ORDER BY fecha ROWS UNBOUNDED PRECEDING) AS ventas_acumuladas
FROM ventas_diarias;
```

### Moving Average (Promedio Móvil)

```sql
-- Promedio de los últimos 7 días
SELECT
    fecha,
    ventas,
    AVG(ventas) OVER (
        ORDER BY fecha
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS promedio_7_dias
FROM ventas_diarias;
```

---

## Resumen: Funciones de Agregación

| Función | Descripción | Ignora NULL |
|---------|-------------|-------------|
| `COUNT(*)` | Cuenta filas | No |
| `COUNT(columna)` | Cuenta valores no-NULL | Sí |
| `SUM(columna)` | Suma valores | Sí |
| `AVG(columna)` | Promedio | Sí |
| `MIN(columna)` | Valor mínimo | Sí |
| `MAX(columna)` | Valor máximo | Sí |

## Resumen: Window Functions

| Función | Propósito | Requiere ORDER BY |
|---------|-----------|-------------------|
| `ROW_NUMBER()` | Número secuencial único | Sí |
| `RANK()` | Ranking con huecos en empates | Sí |
| `DENSE_RANK()` | Ranking sin huecos | Sí |
| `NTILE(n)` | Dividir en n grupos | Sí |
| `LAG()` | Valor de fila anterior | Sí |
| `LEAD()` | Valor de fila siguiente | Sí |
| `FIRST_VALUE()` | Primer valor en ventana | Sí |
| `LAST_VALUE()` | Último valor en ventana | Sí |

---

## Diferencia Clave: GROUP BY vs Window Functions

```sql
-- GROUP BY: colapsa filas
SELECT
    departamento,
    AVG(salario) AS salario_promedio
FROM empleados
GROUP BY departamento;
-- Resultado: 1 fila por departamento

-- Window Function: mantiene todas las filas
SELECT
    departamento,
    nombre,
    salario,
    AVG(salario) OVER (PARTITION BY departamento) AS salario_promedio
FROM empleados;
-- Resultado: 1 fila por empleado, con promedio repetido
```
