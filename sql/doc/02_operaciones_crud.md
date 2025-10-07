# 2. Operaciones CRUD en SQL

**CRUD** = **C**reate, **R**ead, **U**pdate, **D**elete

Estas son las cuatro operaciones básicas para gestionar datos en cualquier base de datos.

---

## SELECT — Recuperar Datos (Read)

Consulta datos de una o más tablas.

### Sintaxis básica:
```sql
SELECT columna1, columna2, ...
FROM tabla
WHERE condición;
```

### Ejemplos:
```sql
-- Seleccionar todas las columnas
SELECT * FROM empleados;

-- Seleccionar columnas específicas
SELECT nombre, salario FROM empleados;

-- Con filtro
SELECT nombre, salario
FROM empleados
WHERE salario > 50000;
```

---

## INSERT — Agregar Nuevos Datos (Create)

Inserta nuevos registros en una tabla.

### Sintaxis:
```sql
-- Insertar especificando columnas
INSERT INTO tabla (columna1, columna2, columna3)
VALUES (valor1, valor2, valor3);

-- Insertar múltiples filas
INSERT INTO tabla (columna1, columna2)
VALUES
    (valor1, valor2),
    (valor3, valor4),
    (valor5, valor6);
```

### Ejemplos:
```sql
-- Insertar un empleado
INSERT INTO empleados (id, nombre, salario, departamento)
VALUES (1, 'Ana García', 55000, 'IT');

-- Insertar múltiples empleados
INSERT INTO empleados (id, nombre, salario, departamento)
VALUES
    (2, 'Carlos Ruiz', 60000, 'Ventas'),
    (3, 'María López', 58000, 'Marketing');

-- Insertar desde otra tabla (INSERT SELECT)
INSERT INTO empleados_historico
SELECT * FROM empleados WHERE fecha_salida IS NOT NULL;
```

---

## UPDATE — Modificar Datos Existentes

Actualiza valores en registros existentes.

### Sintaxis:
```sql
UPDATE tabla
SET columna1 = valor1, columna2 = valor2
WHERE condición;
```

### Ejemplos:
```sql
-- Actualizar un registro específico
UPDATE empleados
SET salario = 65000
WHERE id = 2;

-- Actualizar múltiples columnas
UPDATE empleados
SET salario = salario * 1.10,
    departamento = 'IT'
WHERE id = 3;

-- Actualizar múltiples registros
UPDATE empleados
SET salario = salario * 1.05
WHERE departamento = 'Ventas';
```

**⚠️ IMPORTANTE:** Siempre usa `WHERE` en UPDATE. Sin WHERE, se actualizarán TODAS las filas.

```sql
-- ¡PELIGRO! Esto actualiza TODOS los empleados
UPDATE empleados SET salario = 50000;
```

---

## DELETE — Eliminar Datos

Elimina registros de una tabla.

### Sintaxis:
```sql
DELETE FROM tabla
WHERE condición;
```

### Ejemplos:
```sql
-- Eliminar un registro específico
DELETE FROM empleados WHERE id = 5;

-- Eliminar múltiples registros
DELETE FROM empleados WHERE departamento = 'Marketing';

-- Eliminar con subconsulta
DELETE FROM empleados
WHERE id IN (SELECT empleado_id FROM inactivos);
```

**⚠️ IMPORTANTE:** Siempre usa `WHERE` en DELETE. Sin WHERE, se eliminarán TODAS las filas.

```sql
-- ¡PELIGRO! Esto elimina TODOS los empleados
DELETE FROM empleados;
```

---

## DROP vs TRUNCATE vs DELETE — Diferencias Clave

| Comando | Qué hace | Velocidad | Rollback | Estructura |
|---------|----------|-----------|----------|------------|
| **DELETE** | Elimina filas (con o sin WHERE) | Lento (log cada fila) | Sí, reversible | Mantiene tabla |
| **TRUNCATE** | Elimina TODAS las filas | Rápido (no log individual) | No (en la mayoría) | Mantiene tabla |
| **DROP** | Elimina tabla completa | Muy rápido | No | Elimina tabla |

### DELETE
```sql
-- Elimina filas específicas o todas
DELETE FROM empleados WHERE departamento = 'IT';
DELETE FROM empleados;  -- Elimina todas las filas

-- Características:
-- ✓ Puedes usar WHERE
-- ✓ Dispara triggers
-- ✓ Puede hacer ROLLBACK
-- ✗ Más lento en tablas grandes
```

### TRUNCATE
```sql
-- Elimina TODAS las filas rápidamente
TRUNCATE TABLE empleados;

-- Características:
-- ✗ NO puedes usar WHERE
-- ✗ NO dispara triggers (en la mayoría de DBs)
-- ✗ NO puedes hacer ROLLBACK (en la mayoría)
-- ✓ Mucho más rápido
-- ✓ Resetea auto-increment/sequences
-- ✓ Mantiene la estructura de la tabla
```

### DROP
```sql
-- Elimina la tabla completamente
DROP TABLE empleados;

-- Elimina si existe (evita errores)
DROP TABLE IF EXISTS empleados;

-- Características:
-- ✗ Elimina estructura Y datos
-- ✗ NO reversible
-- ✓ Libera espacio inmediatamente
```

### ¿Cuándo usar cada uno?

| Escenario | Usa |
|-----------|-----|
| Eliminar algunas filas | `DELETE` con WHERE |
| Vaciar tabla para recargar datos | `TRUNCATE` |
| Eliminar tabla permanentemente | `DROP` |
| Necesitas rollback | `DELETE` |
| Tabla gigante y quieres vaciarla rápido | `TRUNCATE` |

---

## CREATE TABLE — Crear Tablas

Define una nueva tabla con su estructura.

### Sintaxis:
```sql
CREATE TABLE nombre_tabla (
    columna1 tipo_dato [constraints],
    columna2 tipo_dato [constraints],
    ...
);
```

### Ejemplos:
```sql
-- Tabla básica
CREATE TABLE empleados (
    id INT,
    nombre VARCHAR(100),
    salario DECIMAL(10, 2),
    fecha_ingreso DATE
);

-- Con constraints
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    salario DECIMAL(10, 2) CHECK (salario > 0),
    departamento_id INT,
    fecha_ingreso DATE DEFAULT CURRENT_DATE,
    FOREIGN KEY (departamento_id) REFERENCES departamentos(id)
);

-- Crear tabla desde otra (CTAS - Create Table As Select)
CREATE TABLE empleados_backup AS
SELECT * FROM empleados;
```

---

## ALTER TABLE — Modificar Estructura de Tablas

Modifica una tabla existente.

### Agregar columna:
```sql
ALTER TABLE empleados
ADD COLUMN telefono VARCHAR(20);
```

### Eliminar columna:
```sql
ALTER TABLE empleados
DROP COLUMN telefono;
```

### Modificar columna:
```sql
-- Cambiar tipo de dato
ALTER TABLE empleados
ALTER COLUMN salario TYPE NUMERIC(12, 2);

-- Renombrar columna
ALTER TABLE empleados
RENAME COLUMN salario TO sueldo;
```

### Agregar/eliminar constraints:
```sql
-- Agregar primary key
ALTER TABLE empleados
ADD PRIMARY KEY (id);

-- Agregar foreign key
ALTER TABLE empleados
ADD CONSTRAINT fk_departamento
FOREIGN KEY (departamento_id) REFERENCES departamentos(id);

-- Eliminar constraint
ALTER TABLE empleados
DROP CONSTRAINT fk_departamento;
```

### Renombrar tabla:
```sql
ALTER TABLE empleados
RENAME TO trabajadores;
```

---

## Resumen de Comandos CRUD

| Operación | SQL Command | Descripción |
|-----------|-------------|-------------|
| **Create** | INSERT | Agregar nuevos registros |
| **Read** | SELECT | Consultar/leer datos |
| **Update** | UPDATE | Modificar registros existentes |
| **Delete** | DELETE | Eliminar registros |

## Comandos DDL (Data Definition Language)

| Comando | Propósito |
|---------|-----------|
| CREATE TABLE | Crear nueva tabla |
| ALTER TABLE | Modificar tabla existente |
| DROP TABLE | Eliminar tabla |
| TRUNCATE TABLE | Vaciar tabla rápidamente |

---

## Buenas Prácticas

1. **Siempre usa WHERE en UPDATE/DELETE** (a menos que realmente quieras afectar todas las filas)
2. **Prueba con SELECT primero:** Antes de UPDATE/DELETE, ejecuta un SELECT con el mismo WHERE para verificar qué filas se afectarán
3. **Usa transacciones para operaciones críticas:**
   ```sql
   BEGIN;
   DELETE FROM empleados WHERE departamento = 'Ventas';
   -- Verificar resultado
   ROLLBACK;  -- o COMMIT si está correcto
   ```
4. **Backups antes de cambios masivos:** Especialmente con ALTER TABLE o TRUNCATE
5. **Constraints para integridad:** Usa PRIMARY KEY, FOREIGN KEY, NOT NULL para proteger tus datos
