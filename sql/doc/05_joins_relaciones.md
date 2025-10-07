# 5. JOINs y Relaciones entre Tablas

---

## Conceptos: Primary Key y Foreign Key

### Primary Key (Clave Primaria)
Identifica **únicamente** cada fila en una tabla.

**Características:**
- Valor único para cada fila
- No puede ser NULL
- Una tabla solo puede tener UNA primary key (pero puede ser compuesta)

```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,           -- Primary key simple
    nombre VARCHAR(100),
    email VARCHAR(100)
);

-- Primary key compuesta
CREATE TABLE inscripciones (
    estudiante_id INT,
    curso_id INT,
    fecha_inscripcion DATE,
    PRIMARY KEY (estudiante_id, curso_id)  -- Combinación única
);
```

### Foreign Key (Clave Foránea)
Referencia la primary key de otra tabla, estableciendo una **relación** entre tablas.

**Características:**
- Asegura integridad referencial
- Puede ser NULL (a menos que se defina NOT NULL)
- Puede tener duplicados

```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100),
    departamento_id INT,
    FOREIGN KEY (departamento_id) REFERENCES departamentos(id)
);

CREATE TABLE departamentos (
    id INT PRIMARY KEY,
    nombre VARCHAR(50)
);
```

**Relación visual:**
```
departamentos                 empleados
┌────┬──────────┐            ┌────┬────────┬────────────────┐
│ id │ nombre   │            │ id │ nombre │ departamento_id│
├────┼──────────┤            ├────┼────────┼────────────────┤
│ 1  │ IT       │◄───────────│ 1  │ Ana    │ 1              │
│ 2  │ Ventas   │            │ 2  │ Carlos │ 2              │
│ 3  │ Marketing│◄───────────│ 3  │ María  │ 1              │
└────┴──────────┘            └────┴────────┴────────────────┘
```

---

## INNER JOIN — Intersección

Devuelve **solo** las filas que tienen coincidencias en **ambas** tablas.

### Sintaxis

```sql
SELECT columnas
FROM tabla1
INNER JOIN tabla2 ON tabla1.columna = tabla2.columna;
```

### Ejemplos

```sql
-- Empleados con su departamento
SELECT
    e.nombre AS empleado,
    d.nombre AS departamento
FROM empleados e
INNER JOIN departamentos d ON e.departamento_id = d.id;

-- Múltiples columnas en JOIN
SELECT
    e.nombre,
    d.nombre AS departamento,
    p.titulo AS proyecto
FROM empleados e
INNER JOIN departamentos d ON e.departamento_id = d.id
INNER JOIN proyectos p ON e.id = p.empleado_id;
```

**Resultado visual:**
```
Empleados:              Departamentos:          Resultado INNER JOIN:
id | depto_id           id | nombre             empleado | departamento
1  | 1                  1  | IT                 Ana      | IT
2  | 2                  2  | Ventas             Carlos   | Ventas
3  | 1                  3  | Marketing          María    | IT
4  | NULL                                       (Juan NO aparece - NULL)
```

**Juan no aparece porque `departamento_id` es NULL (no hay coincidencia).**

---

## LEFT JOIN (LEFT OUTER JOIN) — Todos de la Izquierda

Devuelve **todas** las filas de la tabla izquierda, con las coincidencias de la derecha (NULL si no hay coincidencia).

### Sintaxis

```sql
SELECT columnas
FROM tabla1
LEFT JOIN tabla2 ON tabla1.columna = tabla2.columna;
```

### Ejemplos

```sql
-- Todos los empleados, con o sin departamento
SELECT
    e.nombre AS empleado,
    d.nombre AS departamento
FROM empleados e
LEFT JOIN departamentos d ON e.departamento_id = d.id;
```

**Resultado:**
```
empleado | departamento
---------|-------------
Ana      | IT
Carlos   | Ventas
María    | IT
Juan     | NULL         ← Aparece aunque no tenga departamento
```

**Casos de uso:**
- Encontrar empleados sin departamento asignado
- Clientes sin pedidos
- Productos sin ventas

```sql
-- Empleados sin departamento
SELECT
    e.nombre
FROM empleados e
LEFT JOIN departamentos d ON e.departamento_id = d.id
WHERE d.id IS NULL;
```

---

## RIGHT JOIN (RIGHT OUTER JOIN) — Todos de la Derecha

Devuelve **todas** las filas de la tabla derecha, con las coincidencias de la izquierda (NULL si no hay coincidencia).

### Sintaxis

```sql
SELECT columnas
FROM tabla1
RIGHT JOIN tabla2 ON tabla1.columna = tabla2.columna;
```

### Ejemplo

```sql
-- Todos los departamentos, tengan o no empleados
SELECT
    e.nombre AS empleado,
    d.nombre AS departamento
FROM empleados e
RIGHT JOIN departamentos d ON e.departamento_id = d.id;
```

**Resultado:**
```
empleado | departamento
---------|-------------
Ana      | IT
María    | IT
Carlos   | Ventas
NULL     | Marketing    ← Departamento sin empleados
```

**Nota:** `RIGHT JOIN` es menos común. Generalmente se reescribe como `LEFT JOIN` intercambiando las tablas:

```sql
-- Estas son equivalentes:
SELECT ... FROM empleados e RIGHT JOIN departamentos d ...
SELECT ... FROM departamentos d LEFT JOIN empleados e ...
```

---

## FULL OUTER JOIN — Todos de Ambas Tablas

Devuelve **todas** las filas de ambas tablas, con NULL donde no hay coincidencia.

### Sintaxis

```sql
SELECT columnas
FROM tabla1
FULL OUTER JOIN tabla2 ON tabla1.columna = tabla2.columna;
```

### Ejemplo

```sql
SELECT
    e.nombre AS empleado,
    d.nombre AS departamento
FROM empleados e
FULL OUTER JOIN departamentos d ON e.departamento_id = d.id;
```

**Resultado:**
```
empleado | departamento
---------|-------------
Ana      | IT
María    | IT
Carlos   | Ventas
Juan     | NULL         ← Empleado sin departamento
NULL     | Marketing    ← Departamento sin empleados
```

**⚠️ Nota:** MySQL **NO** soporta FULL OUTER JOIN. Puedes simularlo con UNION:

```sql
-- Simulando FULL OUTER JOIN en MySQL
SELECT e.nombre, d.nombre
FROM empleados e
LEFT JOIN departamentos d ON e.departamento_id = d.id
UNION
SELECT e.nombre, d.nombre
FROM empleados e
RIGHT JOIN departamentos d ON e.departamento_id = d.id;
```

---

## CROSS JOIN — Producto Cartesiano

Combina **cada** fila de la primera tabla con **cada** fila de la segunda tabla.

### Sintaxis

```sql
SELECT columnas
FROM tabla1
CROSS JOIN tabla2;

-- También válido (sin ON):
SELECT columnas
FROM tabla1, tabla2;
```

### Ejemplo

```sql
SELECT
    e.nombre AS empleado,
    d.nombre AS departamento
FROM empleados e
CROSS JOIN departamentos d;
```

**Si hay 4 empleados y 3 departamentos = 4 × 3 = 12 filas**

```
empleado | departamento
---------|-------------
Ana      | IT
Ana      | Ventas
Ana      | Marketing
Carlos   | IT
Carlos   | Ventas
Carlos   | Marketing
...
```

**Casos de uso:**
- Generar combinaciones (tamaños × colores de productos)
- Crear datos de prueba
- Análisis de todas las combinaciones posibles

**⚠️ Cuidado:** Puede generar MUCHAS filas (1000 × 1000 = 1,000,000 filas)

---

## Self Join — Join de una Tabla Consigo Misma

Útil para comparar filas dentro de la misma tabla.

### Ejemplo: Estructura Jerárquica (Empleados y Managers)

```sql
-- Tabla empleados con manager_id
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100),
    manager_id INT,
    FOREIGN KEY (manager_id) REFERENCES empleados(id)
);

-- Empleados con su manager
SELECT
    e.nombre AS empleado,
    m.nombre AS manager
FROM empleados e
LEFT JOIN empleados m ON e.manager_id = m.id;
```

**Datos:**
```
id | nombre  | manager_id
---|---------|----------
1  | Ana     | NULL       (CEO)
2  | Carlos  | 1          (reporta a Ana)
3  | María   | 1          (reporta a Ana)
4  | Juan    | 2          (reporta a Carlos)
```

**Resultado:**
```
empleado | manager
---------|--------
Ana      | NULL
Carlos   | Ana
María    | Ana
Juan     | Carlos
```

### Ejemplo: Comparar Salarios

```sql
-- Pares de empleados del mismo departamento
SELECT
    e1.nombre AS empleado1,
    e2.nombre AS empleado2,
    e1.departamento_id
FROM empleados e1
JOIN empleados e2
    ON e1.departamento_id = e2.departamento_id
    AND e1.id < e2.id;  -- Evita duplicados (A-B y B-A)
```

---

## Uso de Alias en JOINs

```sql
-- Sin alias (difícil de leer)
SELECT empleados.nombre, departamentos.nombre
FROM empleados
INNER JOIN departamentos ON empleados.departamento_id = departamentos.id;

-- Con alias (más claro)
SELECT e.nombre, d.nombre
FROM empleados e
INNER JOIN departamentos d ON e.departamento_id = d.id;

-- Alias para columnas con mismo nombre
SELECT
    e.nombre AS empleado,
    d.nombre AS departamento,
    e.id AS empleado_id,
    d.id AS departamento_id
FROM empleados e
JOIN departamentos d ON e.departamento_id = d.id;
```

---

## Errores Comunes con JOINs

### 1. Duplicados por relaciones 1-a-muchos

```sql
-- Un empleado puede tener múltiples proyectos
SELECT e.nombre, p.titulo
FROM empleados e
JOIN proyectos p ON e.id = p.empleado_id;

-- Si Ana tiene 3 proyectos, aparecerá 3 veces
```

**Solución: usar DISTINCT o agregaciones**
```sql
SELECT DISTINCT e.nombre
FROM empleados e
JOIN proyectos p ON e.id = p.empleado_id;

-- O contar proyectos
SELECT e.nombre, COUNT(p.id) AS total_proyectos
FROM empleados e
LEFT JOIN proyectos p ON e.id = p.empleado_id
GROUP BY e.nombre;
```

### 2. Filtrar después del JOIN vs Filtrar en la condición ON

```sql
-- WHERE filtra después del JOIN (INNER JOIN)
SELECT e.nombre, d.nombre
FROM empleados e
LEFT JOIN departamentos d ON e.departamento_id = d.id
WHERE d.nombre = 'IT';
-- Esto convierte el LEFT JOIN en INNER JOIN efectivamente

-- Filtrar en ON mantiene el LEFT JOIN
SELECT e.nombre, d.nombre
FROM empleados e
LEFT JOIN departamentos d ON e.departamento_id = d.id AND d.nombre = 'IT';
-- Muestra todos los empleados, con 'IT' o NULL
```

### 3. NULLs en JOINs

```sql
-- NULL nunca coincide con NULL en JOINs
SELECT *
FROM tabla1 t1
JOIN tabla2 t2 ON t1.columna = t2.columna;
-- Si columna es NULL en ambas, NO se unen

-- Para incluir NULLs (raro en la práctica):
SELECT *
FROM tabla1 t1
JOIN tabla2 t2 ON (t1.columna = t2.columna OR (t1.columna IS NULL AND t2.columna IS NULL));
```

### 4. CROSS JOIN Accidental

```sql
-- ⚠️ OLVIDAR la condición ON crea un CROSS JOIN
SELECT e.nombre, d.nombre
FROM empleados e, departamentos d;
-- Genera todas las combinaciones

-- ✓ CORRECTO
SELECT e.nombre, d.nombre
FROM empleados e
JOIN departamentos d ON e.departamento_id = d.id;
```

---

## Orden de Filtrado: ON vs WHERE

```sql
-- ON: aplica durante el JOIN
SELECT e.nombre, d.nombre
FROM empleados e
LEFT JOIN departamentos d
    ON e.departamento_id = d.id
    AND d.presupuesto > 100000;  -- Solo une depto con presupuesto > 100k
-- Muestra TODOS los empleados

-- WHERE: aplica después del JOIN
SELECT e.nombre, d.nombre
FROM empleados e
LEFT JOIN departamentos d ON e.departamento_id = d.id
WHERE d.presupuesto > 100000;     -- Filtra resultados después
-- Solo muestra empleados en deptos con presupuesto > 100k
```

---

## Resumen Visual de JOINs

```
Tabla A (empleados)        Tabla B (departamentos)

  ┌─────────┐                ┌─────────┐
  │    A    │                │    B    │
  │  ┌───┐  │                │  ┌───┐  │
  │  │   │  │                │  │   │  │
  └──│───│──┘                └──│───│──┘
     │ ∩ │                      │ ∩ │
     └───┘                      └───┘

INNER JOIN (∩)              LEFT JOIN (A + ∩)
  Solo la intersección        Todo A + matches B

FULL OUTER JOIN             RIGHT JOIN (B + ∩)
  Todo A + Todo B             Todo B + matches A

CROSS JOIN
  A × B (todas combinaciones)
```

---

## Resumen de JOINs

| JOIN Type | Resultado | Cuándo Usar |
|-----------|-----------|-------------|
| **INNER JOIN** | Solo coincidencias | Necesitas datos que existan en ambas tablas |
| **LEFT JOIN** | Todos de izq. + coincidencias | Quieres todos los registros de tabla principal |
| **RIGHT JOIN** | Todos de der. + coincidencias | Menos común, usa LEFT JOIN invertido |
| **FULL OUTER JOIN** | Todos de ambas | Necesitas ver todo, tengan o no coincidencia |
| **CROSS JOIN** | Todas las combinaciones | Producto cartesiano, matrices |
| **SELF JOIN** | Tabla consigo misma | Jerarquías, comparar filas de misma tabla |

---

## Mejores Prácticas

1. **Usa alias siempre** para mayor claridad
2. **Especifica columnas** en vez de `SELECT *` en JOINs
3. **LEFT JOIN es más común** que RIGHT JOIN
4. **Cuidado con duplicados** en relaciones 1-a-muchos
5. **Filtra en WHERE** para INNER JOIN, **en ON** para OUTER JOINs cuando quieras mantener filas sin coincidencia
6. **Usa INNER JOIN explícito** en vez de la sintaxis antigua `FROM t1, t2 WHERE ...`
7. **Indexa las columnas** de JOIN para mejor rendimiento
