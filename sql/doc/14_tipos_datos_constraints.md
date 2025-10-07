# 14. Tipos de Datos y Constraints

---

## Tipos de Datos Comunes

### Numéricos

| Tipo | Descripción | Rango | Ejemplo |
|------|-------------|-------|---------|
| **INT / INTEGER** | Entero | -2,147,483,648 a 2,147,483,647 | `edad INT` |
| **BIGINT** | Entero grande | -9 quintillones a 9 quintillones | `user_id BIGINT` |
| **SMALLINT** | Entero pequeño | -32,768 a 32,767 | `cantidad SMALLINT` |
| **DECIMAL(p,s)** | Número exacto | p=precisión, s=escala | `precio DECIMAL(10,2)` |
| **NUMERIC(p,s)** | Igual que DECIMAL | Mismo | `salario NUMERIC(10,2)` |
| **FLOAT** | Punto flotante | Aproximado | `porcentaje FLOAT` |
| **DOUBLE** | Doble precisión | Aproximado | `coordenada DOUBLE` |

```sql
CREATE TABLE productos (
    id INT,
    nombre VARCHAR(100),
    precio DECIMAL(10, 2),     -- 99999999.99
    descuento FLOAT,           -- 0.15
    stock SMALLINT             -- 0-32767
);
```

### Texto y Cadenas

| Tipo | Descripción | Longitud | Ejemplo |
|------|-------------|----------|---------|
| **CHAR(n)** | Longitud fija | n caracteres | `codigo CHAR(5)` → 'US   ' |
| **VARCHAR(n)** | Longitud variable | Máx n | `nombre VARCHAR(100)` |
| **TEXT** | Texto ilimitado | Sin límite | `descripcion TEXT` |

```sql
CREATE TABLE clientes (
    codigo CHAR(5),              -- Siempre 5 caracteres
    nombre VARCHAR(100),         -- Hasta 100
    notas TEXT                   -- Sin límite
);
```

**CHAR vs VARCHAR:**
- `CHAR`: Rellena con espacios, fijo
- `VARCHAR`: Longitud variable, más eficiente

### Fecha y Hora

| Tipo | Descripción | Formato | Ejemplo |
|------|-------------|---------|---------|
| **DATE** | Solo fecha | YYYY-MM-DD | `'2024-01-15'` |
| **TIME** | Solo hora | HH:MM:SS | `'14:30:00'` |
| **TIMESTAMP** | Fecha + hora | YYYY-MM-DD HH:MM:SS | `'2024-01-15 14:30:00'` |
| **TIMESTAMPTZ** | Timestamp con zona | Con timezone | PostgreSQL |
| **DATETIME** | Fecha + hora | MySQL, SQL Server | `'2024-01-15 14:30:00'` |

```sql
CREATE TABLE eventos (
    id INT,
    fecha DATE,                          -- 2024-01-15
    hora TIME,                           -- 14:30:00
    timestamp TIMESTAMP,                 -- 2024-01-15 14:30:00
    timestamp_tz TIMESTAMPTZ             -- 2024-01-15 14:30:00+00
);
```

### Booleano

```sql
-- PostgreSQL, MySQL 8+
CREATE TABLE empleados (
    id INT,
    activo BOOLEAN                -- TRUE/FALSE
);

-- SQL Server (no tiene BOOLEAN nativo)
CREATE TABLE empleados (
    id INT,
    activo BIT                    -- 0/1
);
```

### JSON

```sql
-- PostgreSQL
CREATE TABLE configuraciones (
    id INT,
    datos JSON,              -- JSON texto
    datos_b JSONB            -- JSON binario (más eficiente)
);

-- MySQL
CREATE TABLE logs (
    id INT,
    datos JSON
);

-- Snowflake
CREATE TABLE eventos (
    id INT,
    datos VARIANT            -- Tipo semi-estructurado
);
```

### Otros Tipos

```sql
-- UUID
CREATE TABLE usuarios (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid()  -- PostgreSQL
);

-- Array (PostgreSQL)
CREATE TABLE productos (
    id INT,
    tags TEXT[]              -- Array de texto
);

-- Enum (PostgreSQL, MySQL)
CREATE TYPE estado AS ENUM ('pendiente', 'aprobado', 'rechazado');
CREATE TABLE pedidos (
    id INT,
    estado estado
);
```

---

## Constraints (Restricciones)

Las **constraints** aseguran integridad de datos.

### PRIMARY KEY

Identifica únicamente cada fila.

```sql
-- Inline
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100)
);

-- Named constraint
CREATE TABLE empleados (
    id INT,
    nombre VARCHAR(100),
    CONSTRAINT pk_empleados PRIMARY KEY (id)
);

-- Primary key compuesta
CREATE TABLE inscripciones (
    estudiante_id INT,
    curso_id INT,
    PRIMARY KEY (estudiante_id, curso_id)
);
```

### FOREIGN KEY

Referencia otra tabla.

```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    departamento_id INT,
    FOREIGN KEY (departamento_id) REFERENCES departamentos(id)
);

-- Con acciones
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    departamento_id INT,
    CONSTRAINT fk_empleados_dept
        FOREIGN KEY (departamento_id)
        REFERENCES departamentos(id)
        ON DELETE CASCADE        -- Elimina empleado si se elimina depto
        ON UPDATE CASCADE        -- Actualiza id si cambia
);
```

**Opciones ON DELETE/UPDATE:**
- `CASCADE`: Propaga cambio
- `SET NULL`: Establece NULL
- `SET DEFAULT`: Establece valor default
- `RESTRICT`: Previene si hay referencias
- `NO ACTION`: Similar a RESTRICT

### UNIQUE

Asegura valores únicos.

```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE,           -- No duplicados
    ssn CHAR(11) UNIQUE
);

-- UNIQUE compuesto
CREATE TABLE productos (
    id INT PRIMARY KEY,
    nombre VARCHAR(100),
    categoria VARCHAR(50),
    UNIQUE (nombre, categoria)           -- Combinación única
);
```

### NOT NULL

La columna debe tener un valor.

```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,        -- Obligatorio
    email VARCHAR(100),                  -- Opcional (puede ser NULL)
    fecha_ingreso DATE NOT NULL
);
```

### CHECK

Valida condiciones personalizadas.

```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    edad INT CHECK (edad >= 18 AND edad <= 65),
    salario DECIMAL CHECK (salario > 0),
    email VARCHAR(100) CHECK (email LIKE '%@%')
);

-- Named constraint
CREATE TABLE productos (
    id INT PRIMARY KEY,
    precio DECIMAL,
    descuento DECIMAL,
    CONSTRAINT chk_precio CHECK (precio > 0),
    CONSTRAINT chk_descuento CHECK (descuento >= 0 AND descuento <= 1)
);
```

### DEFAULT

Valor por defecto.

```sql
CREATE TABLE empleados (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    activo BOOLEAN DEFAULT TRUE,
    fecha_ingreso DATE DEFAULT CURRENT_DATE,
    intentos INT DEFAULT 0,
    pais VARCHAR(50) DEFAULT 'México'
);

-- Insertar usando defaults
INSERT INTO empleados (nombre) VALUES ('Ana');
-- activo = TRUE, fecha_ingreso = hoy, intentos = 0, pais = 'México'
```

---

## Casting y Conversión de Tipos

### CAST

```sql
-- Standard SQL
SELECT CAST('123' AS INT);                    -- '123' → 123
SELECT CAST(123.45 AS INT);                   -- 123.45 → 123
SELECT CAST('2024-01-15' AS DATE);            -- '2024-01-15' → DATE

-- PostgreSQL shorthand
SELECT '123'::INT;
SELECT 123.45::VARCHAR;
SELECT '2024-01-15'::DATE;
```

### CONVERT (SQL Server, MySQL)

```sql
-- SQL Server
SELECT CONVERT(INT, '123');
SELECT CONVERT(VARCHAR, 123);
SELECT CONVERT(DATE, '2024-01-15');

-- MySQL
SELECT CONVERT('123', SIGNED);                -- String a INT
SELECT CONVERT('2024-01-15', DATE);
```

### Conversiones Comunes

```sql
-- Texto a número
SELECT CAST('123' AS INT);
SELECT '123'::INT;                            -- PostgreSQL

-- Número a texto
SELECT CAST(123 AS VARCHAR);
SELECT 123::VARCHAR;                          -- PostgreSQL

-- Texto a fecha
SELECT CAST('2024-01-15' AS DATE);
SELECT TO_DATE('2024-01-15', 'YYYY-MM-DD');   -- PostgreSQL, Snowflake

-- Fecha a texto
SELECT CAST(CURRENT_DATE AS VARCHAR);
SELECT TO_CHAR(CURRENT_DATE, 'YYYY-MM-DD');   -- PostgreSQL

-- Decimal a int (trunca)
SELECT CAST(123.99 AS INT);                   -- 123
SELECT ROUND(123.99)::INT;                    -- 124
```

---

## Manejo de Fechas y Zonas Horarias

### Funciones de Fecha

```sql
-- Fecha/hora actual
SELECT CURRENT_DATE;                          -- 2024-01-15
SELECT CURRENT_TIME;                          -- 14:30:00
SELECT CURRENT_TIMESTAMP;                     -- 2024-01-15 14:30:00
SELECT NOW();                                 -- PostgreSQL, MySQL

-- Extraer partes
SELECT EXTRACT(YEAR FROM fecha);
SELECT EXTRACT(MONTH FROM fecha);
SELECT EXTRACT(DAY FROM fecha);

-- PostgreSQL
SELECT DATE_PART('year', fecha);
SELECT DATE_TRUNC('month', timestamp);        -- Truncar a inicio de mes

-- SQL Server
SELECT YEAR(fecha);
SELECT MONTH(fecha);
SELECT DAY(fecha);
SELECT DATEPART(QUARTER, fecha);
```

### Aritmética de Fechas

```sql
-- PostgreSQL
SELECT CURRENT_DATE + INTERVAL '7 days';
SELECT CURRENT_DATE - INTERVAL '1 month';
SELECT fecha + INTERVAL '2 hours';

-- MySQL
SELECT DATE_ADD(CURRENT_DATE, INTERVAL 7 DAY);
SELECT DATE_SUB(CURRENT_DATE, INTERVAL 1 MONTH);

-- SQL Server
SELECT DATEADD(day, 7, CURRENT_DATE);
SELECT DATEADD(month, -1, CURRENT_DATE);

-- Snowflake
SELECT DATEADD(day, 7, CURRENT_DATE);
SELECT DATEDIFF(day, fecha1, fecha2);
```

### Zonas Horarias

```sql
-- PostgreSQL
CREATE TABLE eventos (
    id INT,
    timestamp_utc TIMESTAMPTZ,                -- Con timezone
    timestamp_local TIMESTAMP                 -- Sin timezone
);

-- Convertir zonas
SELECT timestamp_utc AT TIME ZONE 'America/Mexico_City';
SELECT timestamp_utc AT TIME ZONE 'UTC';

-- Snowflake
SELECT CONVERT_TIMEZONE('UTC', 'America/New_York', timestamp_col);
```

---

## Mejores Prácticas

### Tipos de Datos

1. ✅ **Usa tipo apropiado** para cada dato
   - Fechas → DATE/TIMESTAMP (no VARCHAR)
   - Dinero → DECIMAL (no FLOAT)
   - Booleanos → BOOLEAN/BIT

2. ✅ **DECIMAL para dinero** (no FLOAT - evita errores de redondeo)
   ```sql
   precio DECIMAL(10, 2)  -- NO FLOAT
   ```

3. ✅ **VARCHAR en vez de CHAR** (a menos que longitud fija)

4. ✅ **Tamaño apropiado** (no `VARCHAR(1000)` si solo usas 50)

5. ✅ **TIMESTAMP con timezone** para eventos globales

### Constraints

1. ✅ **Siempre PRIMARY KEY**
   ```sql
   id SERIAL PRIMARY KEY
   ```

2. ✅ **FOREIGN KEY para relaciones**
   ```sql
   FOREIGN KEY (departamento_id) REFERENCES departamentos(id)
   ```

3. ✅ **NOT NULL en columnas críticas**
   ```sql
   nombre VARCHAR(100) NOT NULL
   ```

4. ✅ **UNIQUE para campos únicos** (email, username)

5. ✅ **CHECK para validaciones** de negocio
   ```sql
   CHECK (edad >= 18)
   CHECK (salario > 0)
   ```

6. ✅ **DEFAULT para valores comunes**
   ```sql
   activo BOOLEAN DEFAULT TRUE
   fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   ```

7. ✅ **Nombra constraints** para debugging
   ```sql
   CONSTRAINT chk_edad CHECK (edad >= 18)
   ```

---

## Ejemplo Completo

```sql
CREATE TABLE empleados (
    -- Primary key con auto-increment
    id SERIAL PRIMARY KEY,

    -- Campos obligatorios
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,

    -- Foreign key con cascade
    departamento_id INT NOT NULL,
    manager_id INT,

    -- Tipos apropiados
    salario DECIMAL(10, 2) NOT NULL CHECK (salario > 0),
    fecha_nacimiento DATE CHECK (fecha_nacimiento < CURRENT_DATE),
    edad INT CHECK (edad >= 18 AND edad <= 65),

    -- Defaults
    activo BOOLEAN DEFAULT TRUE,
    fecha_ingreso DATE DEFAULT CURRENT_DATE,
    intentos_login INT DEFAULT 0,

    -- JSON para datos flexibles
    configuraciones JSONB,

    -- Timestamps de auditoría
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints nombradas
    CONSTRAINT fk_departamento
        FOREIGN KEY (departamento_id)
        REFERENCES departamentos(id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,

    CONSTRAINT fk_manager
        FOREIGN KEY (manager_id)
        REFERENCES empleados(id)
        ON DELETE SET NULL,

    CONSTRAINT chk_email
        CHECK (email LIKE '%@%'),

    CONSTRAINT chk_edad_fecha
        CHECK (edad = EXTRACT(YEAR FROM AGE(fecha_nacimiento)))
);
```

---

## Resumen de Constraints

| Constraint | Propósito | Ejemplo |
|------------|-----------|---------|
| **PRIMARY KEY** | Identificador único | `id INT PRIMARY KEY` |
| **FOREIGN KEY** | Relación entre tablas | `FOREIGN KEY (dept_id) REFERENCES...` |
| **UNIQUE** | Valores únicos | `email VARCHAR UNIQUE` |
| **NOT NULL** | Valor obligatorio | `nombre VARCHAR NOT NULL` |
| **CHECK** | Validación personalizada | `CHECK (edad >= 18)` |
| **DEFAULT** | Valor por defecto | `activo BOOLEAN DEFAULT TRUE` |
