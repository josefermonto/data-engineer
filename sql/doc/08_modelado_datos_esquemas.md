# 8. Modelado de Datos y Diseño de Esquemas

---

## ¿Qué es un Schema (Esquema)?

Un **schema** define la estructura lógica de una base de datos:
- Tablas y sus columnas
- Tipos de datos
- Relaciones (Primary Keys, Foreign Keys)
- Constraints (restricciones)
- Índices, vistas, procedimientos

**Dos significados de "schema":**

### 1. Schema como Estructura de BD
La definición completa de cómo se organizan los datos.

### 2. Schema como Namespace (en PostgreSQL, SQL Server, Snowflake)
Un contenedor lógico para agrupar objetos.

```sql
-- PostgreSQL/SQL Server
CREATE SCHEMA ventas;
CREATE TABLE ventas.pedidos (...);
CREATE TABLE ventas.clientes (...);

CREATE SCHEMA rrhh;
CREATE TABLE rrhh.empleados (...);
```

---

## Normalización — Organizar Datos Eficientemente

La **normalización** es el proceso de organizar datos para:
- Reducir redundancia (duplicación)
- Evitar anomalías en INSERT/UPDATE/DELETE
- Mantener integridad de datos

### Primera Forma Normal (1NF)

**Regla:** Cada columna debe contener **valores atómicos** (indivisibles), sin listas o arrays.

**❌ Violación de 1NF:**
```
empleados
┌────┬─────────┬──────────────────────┐
│ id │ nombre  │ telefonos            │
├────┼─────────┼──────────────────────┤
│ 1  │ Ana     │ 555-1234, 555-5678   │  ← Lista de teléfonos
└────┴─────────┴──────────────────────┘
```

**✅ Cumple 1NF:**
```
empleados                    telefonos
┌────┬─────────┐            ┌─────────────┬──────────┐
│ id │ nombre  │            │ empleado_id │ telefono │
├────┼─────────┤            ├─────────────┼──────────┤
│ 1  │ Ana     │            │ 1           │ 555-1234 │
└────┴─────────┘            │ 1           │ 555-5678 │
                            └─────────────┴──────────┘
```

### Segunda Forma Normal (2NF)

**Regla:** Cumple 1NF + no hay **dependencias parciales** (aplicable a primary keys compuestas).

Cada columna no-clave debe depender de **toda** la primary key, no solo de parte de ella.

**❌ Violación de 2NF:**
```
pedidos_detalle
┌───────────┬─────────────┬──────────┬───────────────────┐
│ pedido_id │ producto_id │ cantidad │ producto_nombre   │  ← nombre depende solo de producto_id
├───────────┼─────────────┼──────────┼───────────────────┤
│ 1         │ 10          │ 2        │ Laptop            │
│ 1         │ 20          │ 1        │ Mouse             │
└───────────┴─────────────┴──────────┴───────────────────┘
PRIMARY KEY (pedido_id, producto_id)
```

`producto_nombre` depende solo de `producto_id`, no de la primary key completa.

**✅ Cumple 2NF:**
```
pedidos_detalle                  productos
┌───────────┬─────────────┬──────────┐   ┌─────────────┬─────────┐
│ pedido_id │ producto_id │ cantidad │   │ producto_id │ nombre  │
├───────────┼─────────────┼──────────┤   ├─────────────┼─────────┤
│ 1         │ 10          │ 2        │   │ 10          │ Laptop  │
│ 1         │ 20          │ 1        │   │ 20          │ Mouse   │
└───────────┴─────────────┴──────────┘   └─────────────┴─────────┘
```

### Tercera Forma Normal (3NF)

**Regla:** Cumple 2NF + no hay **dependencias transitivas**.

Columnas no-clave no deben depender de otras columnas no-clave.

**❌ Violación de 3NF:**
```
empleados
┌────┬─────────┬────────────────┬──────────────────┐
│ id │ nombre  │ departamento_id│ departamento_nombre│  ← nombre depende de depto_id
├────┼─────────┼────────────────┼──────────────────┤
│ 1  │ Ana     │ 1              │ IT               │
│ 2  │ Carlos  │ 2              │ Ventas           │
└────┴─────────┴────────────────┴──────────────────┘
```

`departamento_nombre` depende de `departamento_id` (no de `id`).

**✅ Cumple 3NF:**
```
empleados                        departamentos
┌────┬─────────┬────────────────┐   ┌────┬─────────┐
│ id │ nombre  │ departamento_id│   │ id │ nombre  │
├────┼─────────┼────────────────┤   ├────┼─────────┤
│ 1  │ Ana     │ 1              │   │ 1  │ IT      │
│ 2  │ Carlos  │ 2              │   │ 2  │ Ventas  │
└────┴─────────┴────────────────┘   └────┴─────────┘
```

### Forma Normal de Boyce-Codd (BCNF)

**Regla:** Versión más estricta de 3NF.

Para toda dependencia funcional X → Y, X debe ser una superkey (candidate key).

**Generalmente, 3NF es suficiente** para la mayoría de aplicaciones. BCNF se usa en casos especiales.

---

## Denormalización — Cuándo y Por Qué

**Desnormalizar** significa **agregar redundancia** intencionalmente para mejorar rendimiento de lectura.

### ¿Cuándo desnormalizar?

1. **Queries de lectura muy frecuentes** con múltiples JOINs
2. **Data Warehouses** (OLAP) donde lectura > escritura
3. **Reportes/Analytics** que necesitan respuesta rápida
4. **Datos históricos** que no cambian

### Ejemplo de Denormalización

**Normalizado (3NF):**
```sql
-- Necesita 3 JOINs para reporte de ventas
SELECT
    v.id,
    c.nombre AS cliente,
    p.nombre AS producto,
    d.nombre AS departamento
FROM ventas v
JOIN clientes c ON v.cliente_id = c.id
JOIN productos p ON v.producto_id = p.id
JOIN departamentos d ON p.departamento_id = d.id;
```

**Desnormalizado:**
```sql
-- Tabla de ventas con información redundante
CREATE TABLE ventas_denorm (
    id INT,
    cliente_id INT,
    cliente_nombre VARCHAR(100),     -- ← Redundante
    producto_id INT,
    producto_nombre VARCHAR(100),    -- ← Redundante
    departamento_nombre VARCHAR(50), -- ← Redundante
    monto DECIMAL
);

-- Query simple y rápida
SELECT * FROM ventas_denorm;
```

**Trade-offs:**
- ✅ Queries más rápidas (sin JOINs)
- ✅ Menos carga en el servidor
- ❌ Más espacio de almacenamiento
- ❌ Complejidad en mantenimiento (actualizar múltiples lugares)
- ❌ Riesgo de inconsistencias

---

## Star Schema — Esquema en Estrella (Data Warehouse)

**Diseño común en Data Warehousing** con:
- 1 tabla **Fact** (hechos/métricas) en el centro
- Múltiples tablas **Dimension** alrededor

### Estructura

```
        Dimension           Dimension
         (Tiempo)          (Producto)
            │                  │
            │                  │
            └────► Fact ◄──────┘
                  (Ventas)
                     ▲
                     │
                Dimension
               (Cliente)
```

### Ejemplo

```sql
-- Tabla Fact: contiene métricas y foreign keys
CREATE TABLE fact_ventas (
    venta_id INT PRIMARY KEY,
    fecha_id INT,              -- FK a dim_tiempo
    producto_id INT,           -- FK a dim_producto
    cliente_id INT,            -- FK a dim_cliente
    tienda_id INT,             -- FK a dim_tienda
    cantidad INT,              -- Métrica
    monto DECIMAL,             -- Métrica
    descuento DECIMAL,         -- Métrica
    FOREIGN KEY (fecha_id) REFERENCES dim_tiempo(fecha_id),
    FOREIGN KEY (producto_id) REFERENCES dim_producto(producto_id),
    FOREIGN KEY (cliente_id) REFERENCES dim_cliente(cliente_id),
    FOREIGN KEY (tienda_id) REFERENCES dim_tienda(tienda_id)
);

-- Dimension: Tiempo
CREATE TABLE dim_tiempo (
    fecha_id INT PRIMARY KEY,
    fecha DATE,
    dia INT,
    mes INT,
    trimestre INT,
    año INT,
    dia_semana VARCHAR(10),
    es_fin_semana BOOLEAN
);

-- Dimension: Producto
CREATE TABLE dim_producto (
    producto_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    categoria VARCHAR(50),
    marca VARCHAR(50),
    precio_lista DECIMAL
);

-- Dimension: Cliente
CREATE TABLE dim_cliente (
    cliente_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    ciudad VARCHAR(50),
    pais VARCHAR(50),
    segmento VARCHAR(20)
);
```

**Características:**
- ✅ Queries simples (pocos JOINs)
- ✅ Fácil de entender
- ✅ Optimizado para BI tools
- ❌ Redundancia en dimensiones (desnormalizado)

---

## Snowflake Schema — Esquema Copo de Nieve

Similar a Star Schema pero con **dimensiones normalizadas** (divididas en sub-tablas).

### Ejemplo

```sql
-- Fact (igual que Star Schema)
CREATE TABLE fact_ventas (
    venta_id INT PRIMARY KEY,
    producto_id INT,
    -- ...
);

-- Dimension Producto (normalizada)
CREATE TABLE dim_producto (
    producto_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    categoria_id INT,  -- FK a dim_categoria
    marca_id INT       -- FK a dim_marca
);

CREATE TABLE dim_categoria (
    categoria_id INT PRIMARY KEY,
    nombre VARCHAR(50),
    departamento_id INT  -- FK a dim_departamento
);

CREATE TABLE dim_marca (
    marca_id INT PRIMARY KEY,
    nombre VARCHAR(50),
    pais_origen VARCHAR(50)
);
```

**Star vs Snowflake:**

| Star Schema | Snowflake Schema |
|-------------|------------------|
| Dimensiones desnormalizadas | Dimensiones normalizadas |
| Menos tablas | Más tablas |
| Más espacio | Menos espacio |
| Queries más simples | Queries más complejas (más JOINs) |
| Mejor para BI/Analytics | Mejor para integridad de datos |

---

## Fact Tables vs Dimension Tables

### Fact Table (Tabla de Hechos)
- Contiene **métricas/medidas** (ventas, cantidad, monto)
- Contiene **Foreign Keys** a dimensiones
- Generalmente **muchas filas** (millones)
- Se actualiza **frecuentemente** (transacciones)
- Datos **cuantitativos** (números)

```sql
CREATE TABLE fact_ventas (
    venta_id INT PRIMARY KEY,
    -- Foreign keys (dimensiones)
    fecha_id INT,
    producto_id INT,
    cliente_id INT,
    -- Métricas
    cantidad INT,
    monto_total DECIMAL,
    costo DECIMAL,
    ganancia DECIMAL
);
```

### Dimension Table (Tabla de Dimensiones)
- Contiene **atributos descriptivos** (nombre, categoría, ubicación)
- Tiene **Primary Key**
- Generalmente **pocas filas** (cientos/miles)
- Se actualiza **raramente** (maestros)
- Datos **cualitativos** (texto, fechas)

```sql
CREATE TABLE dim_producto (
    producto_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    descripcion TEXT,
    categoria VARCHAR(50),
    marca VARCHAR(50),
    color VARCHAR(20)
);
```

---

## Surrogate Keys vs Natural Keys

### Natural Key (Clave Natural)
Identificador que tiene **significado de negocio**.

```sql
CREATE TABLE paises (
    codigo_iso CHAR(2) PRIMARY KEY,  -- 'US', 'ES', 'MX'
    nombre VARCHAR(100)
);

CREATE TABLE empleados (
    email VARCHAR(100) PRIMARY KEY,  -- Natural key
    nombre VARCHAR(100)
);
```

**Ventajas:** Significativo para usuarios
**Desventajas:** Puede cambiar, puede ser largo, puede ser compuesto

### Surrogate Key (Clave Sustituta)
Identificador **artificial sin significado de negocio**.

```sql
CREATE TABLE empleados (
    id SERIAL PRIMARY KEY,           -- Surrogate key
    email VARCHAR(100) UNIQUE,       -- Natural key como UNIQUE
    nombre VARCHAR(100)
);
```

**Ventajas:**
- Nunca cambia
- Siempre simple (INT)
- Mejor performance en JOINs
- Evita problemas si natural key cambia

**Desventajas:**
- No tiene significado
- Requiere índice adicional en natural key si se busca por él

### ¿Cuál usar?

| Usa Natural Key cuando | Usa Surrogate Key cuando |
|------------------------|--------------------------|
| Valor estable (códigos ISO) | Valores pueden cambiar |
| Corto y simple | Natural key es compuesto o largo |
| No cambiará nunca | Mayor flexibilidad |
| Es un estándar | Performance crítica |

**Mejor práctica:** Surrogate key como PK + natural key con constraint UNIQUE.

```sql
CREATE TABLE empleados (
    id SERIAL PRIMARY KEY,              -- Surrogate
    email VARCHAR(100) UNIQUE NOT NULL, -- Natural con constraint
    nombre VARCHAR(100)
);
```

---

## Integridad Referencial y Constraints

### Primary Key
```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    -- ...
);

-- O
CREATE TABLE empleados (
    id INT,
    -- ...
    PRIMARY KEY (id)
);

-- Primary key compuesta
CREATE TABLE inscripciones (
    estudiante_id INT,
    curso_id INT,
    PRIMARY KEY (estudiante_id, curso_id)
);
```

### Foreign Key
```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    departamento_id INT,
    FOREIGN KEY (departamento_id) REFERENCES departamentos(id)
);

-- Con acciones en cascada
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    departamento_id INT,
    FOREIGN KEY (departamento_id) REFERENCES departamentos(id)
        ON DELETE CASCADE        -- Elimina empleados si se elimina depto
        ON UPDATE CASCADE        -- Actualiza id si cambia en depto
);

-- Opciones: CASCADE, SET NULL, SET DEFAULT, RESTRICT, NO ACTION
```

### UNIQUE
```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE  -- No se permiten duplicados
);
```

### NOT NULL
```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL  -- Obligatorio
);
```

### CHECK
```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    salario DECIMAL CHECK (salario > 0),
    edad INT CHECK (edad >= 18 AND edad <= 65),
    email VARCHAR(100) CHECK (email LIKE '%@%')
);
```

### DEFAULT
```sql
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    fecha_ingreso DATE DEFAULT CURRENT_DATE,
    activo BOOLEAN DEFAULT true,
    intentos INT DEFAULT 0
);
```

---

## Resumen de Formas Normales

| Forma Normal | Requisito |
|--------------|-----------|
| **1NF** | Valores atómicos, sin listas |
| **2NF** | 1NF + no dependencias parciales |
| **3NF** | 2NF + no dependencias transitivas |
| **BCNF** | 3NF + toda dependencia tiene superkey a la izquierda |

**En la práctica:** Normaliza hasta **3NF** para OLTP, desnormaliza para **OLAP/Data Warehouses**.

---

## OLTP vs OLAP

| OLTP (Transaccional) | OLAP (Analítico) |
|---------------------|------------------|
| Muchas escrituras | Muchas lecturas |
| Normalizado (3NF) | Desnormalizado (Star/Snowflake) |
| Datos actuales | Datos históricos |
| Queries simples | Queries complejas |
| MySQL, PostgreSQL, SQL Server | Snowflake, BigQuery, Redshift |
| ERP, CRM, apps web | BI, Reports, Analytics |

---

## Mejores Prácticas

1. **OLTP:** Normaliza hasta 3NF
2. **OLAP:** Usa Star/Snowflake Schema
3. **Usa Surrogate Keys** para Primary Keys en la mayoría de casos
4. **Foreign Keys** para integridad referencial
5. **Constraints** (NOT NULL, CHECK) para validación
6. **Índices** en Foreign Keys y columnas de búsqueda frecuente
7. **Documenta** el schema (ERD - Entity Relationship Diagram)
8. **Versionamiento** del schema (migrations, Flyway, Liquibase)
