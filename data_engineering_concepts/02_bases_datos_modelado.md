# 2. Bases de Datos y Modelado

---

## SQL vs NoSQL

### Bases de Datos SQL (Relacionales)

**Características:**
- Estructura basada en **tablas** con filas y columnas
- **Schema rígido** definido de antemano
- Soporta **relaciones** entre tablas (Foreign Keys)
- **ACID** garantizado (Atomicity, Consistency, Isolation, Durability)
- Lenguaje estándar: SQL

**Ejemplos:**
- **PostgreSQL** - Open source, extensible, feature-rich
- **MySQL** - Popular, rápido para lectura
- **SQL Server** - Microsoft, integración enterprise
- **Snowflake** - Cloud data warehouse moderno

**Casos de uso:**
- Sistemas transaccionales (OLTP)
- Data warehouses (OLAP)
- Aplicaciones que requieren consistencia fuerte
- Reportes y BI

### Bases de Datos NoSQL (No Relacionales)

**Tipos:**

#### 1. Document Stores
```json
// MongoDB
{
  "_id": "user123",
  "nombre": "Ana García",
  "edad": 28,
  "direcciones": [
    {"tipo": "casa", "ciudad": "Madrid"},
    {"tipo": "oficina", "ciudad": "Barcelona"}
  ]
}
```
**Ejemplos:** MongoDB, CouchDB

#### 2. Key-Value Stores
```
user:123 → {"nombre": "Ana", "email": "ana@example.com"}
session:abc → {"user_id": 123, "expires": "2024-01-15"}
```
**Ejemplos:** Redis, DynamoDB, Memcached

#### 3. Column-Family Stores
```
Row Key: user123
  Column Family: personal
    nombre: "Ana"
    edad: 28
  Column Family: contacto
    email: "ana@example.com"
```
**Ejemplos:** Cassandra, HBase

#### 4. Graph Databases
```
(Ana)-[:AMIGO_DE]->(Carlos)
(Ana)-[:TRABAJA_EN]->(Empresa X)
```
**Ejemplos:** Neo4j, Amazon Neptune

### Comparación SQL vs NoSQL

| Aspecto | SQL | NoSQL |
|---------|-----|-------|
| **Schema** | Fijo (schema-on-write) | Flexible (schema-on-read) |
| **Escalabilidad** | Vertical (más CPU/RAM) | Horizontal (más servidores) |
| **Transacciones** | ACID completo | Eventual consistency |
| **Relaciones** | Joins nativos | Denormalizado |
| **Consultas** | SQL estándar | APIs específicas |
| **Consistencia** | Fuerte | Eventual (generalmente) |
| **Casos de uso** | Transacciones, reportes | Big data, tiempo real |

---

## OLTP vs OLAP

### OLTP (Online Transaction Processing)

**Propósito:** Manejar **transacciones diarias** del negocio.

**Características:**
- Muchas **escrituras** pequeñas (INSERT, UPDATE, DELETE)
- **Baja latencia** (milisegundos)
- **Normalizado** (3NF para evitar redundancia)
- Queries simples
- Alta concurrencia

**Ejemplos:**
- Sistema de ventas (crear pedido)
- Banca (transferencias)
- Registro de usuarios
- E-commerce

**Bases de datos:**
- PostgreSQL, MySQL, SQL Server (OLTP mode)
- Oracle Database

**Ejemplo de query:**
```sql
-- Crear pedido
INSERT INTO orders (customer_id, product_id, quantity, price)
VALUES (123, 456, 2, 29.99);

-- Actualizar inventario
UPDATE inventory SET stock = stock - 2 WHERE product_id = 456;
```

### OLAP (Online Analytical Processing)

**Propósito:** **Análisis y reportes** sobre datos históricos.

**Características:**
- Muchas **lecturas** complejas (SELECT)
- **Alta latencia** tolerada (segundos/minutos)
- **Denormalizado** (Star/Snowflake schema)
- Queries complejas con agregaciones
- Menos usuarios concurrentes

**Ejemplos:**
- Dashboards ejecutivos
- Reportes de ventas mensuales
- Análisis de tendencias
- Data mining

**Bases de datos:**
- Snowflake, BigQuery, Redshift
- Teradata, Oracle (DW mode)

**Ejemplo de query:**
```sql
-- Análisis de ventas
SELECT
    DATE_TRUNC('month', order_date) AS mes,
    product_category,
    SUM(revenue) AS total_ventas,
    COUNT(DISTINCT customer_id) AS clientes_unicos
FROM fact_sales
WHERE order_date >= '2023-01-01'
GROUP BY mes, product_category
ORDER BY total_ventas DESC;
```

### Comparación OLTP vs OLAP

| Aspecto | OLTP | OLAP |
|---------|------|------|
| **Propósito** | Transacciones operacionales | Análisis y reportes |
| **Usuarios** | Miles (clientes, empleados) | Decenas (analysts, managers) |
| **Operaciones** | INSERT, UPDATE, DELETE | SELECT (queries complejas) |
| **Volumen por query** | Pocas filas | Millones de filas |
| **Normalización** | Normalizado (3NF) | Denormalizado (Star) |
| **Latencia** | Milisegundos | Segundos/minutos |
| **Histórico** | Datos actuales | Histórico (años) |
| **Índices** | En claves primarias | Muchos índices |

---

## Modelado de Datos

### Modelos Entidad-Relación (ER)

**Para OLTP:** Diagrama que muestra entidades y sus relaciones.

```
┌───────────┐         ┌───────────┐
│  CLIENTE  │         │  PEDIDO   │
├───────────┤         ├───────────┤
│ id (PK)   │1      n │ id (PK)   │
│ nombre    │◄────────│ cliente_id│
│ email     │         │ fecha     │
└───────────┘         │ total     │
                      └───────────┘
                           │1
                           │
                           │n
                      ┌───────────┐
                      │ DETALLE   │
                      ├───────────┤
                      │ id (PK)   │
                      │ pedido_id │
                      │ producto  │
                      │ cantidad  │
                      └───────────┘
```

### Modelos Dimensionales (Star Schema)

**Para OLAP:** Optimizado para análisis.

#### Star Schema (Esquema Estrella)

```
        ┌─────────────┐
        │ Dim Tiempo  │
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───┴────┐ ┌──┴──────┐ ┌─┴────────┐
│Dim     │ │  Fact   │ │Dim       │
│Producto│ │  Ventas │ │Cliente   │
└────────┘ └─────────┘ └──────────┘
               │
        ┌──────┴──────┐
        │             │
    ┌───┴────┐   ┌────┴─────┐
    │Dim     │   │Dim       │
    │Tienda  │   │Promoción │
    └────────┘   └──────────┘
```

**Ejemplo en SQL:**

```sql
-- Tabla Fact (métricas)
CREATE TABLE fact_ventas (
    venta_id INT PRIMARY KEY,
    fecha_id INT,              -- FK a dim_tiempo
    producto_id INT,           -- FK a dim_producto
    cliente_id INT,            -- FK a dim_cliente
    tienda_id INT,             -- FK a dim_tienda
    cantidad INT,              -- Métrica
    monto DECIMAL,             -- Métrica
    descuento DECIMAL,         -- Métrica
    ganancia DECIMAL           -- Métrica
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
    es_fin_semana BOOLEAN,
    es_feriado BOOLEAN
);

-- Dimension: Producto
CREATE TABLE dim_producto (
    producto_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    categoria VARCHAR(50),
    subcategoria VARCHAR(50),
    marca VARCHAR(50),
    precio_lista DECIMAL
);

-- Dimension: Cliente
CREATE TABLE dim_cliente (
    cliente_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    segmento VARCHAR(20),      -- Premium, Regular, Basic
    ciudad VARCHAR(50),
    pais VARCHAR(50),
    fecha_registro DATE
);
```

**Query de ejemplo:**
```sql
SELECT
    t.año,
    t.mes,
    p.categoria,
    c.segmento,
    SUM(v.monto) AS ventas_totales,
    SUM(v.cantidad) AS unidades_vendidas,
    COUNT(DISTINCT v.cliente_id) AS clientes_unicos
FROM fact_ventas v
JOIN dim_tiempo t ON v.fecha_id = t.fecha_id
JOIN dim_producto p ON v.producto_id = p.producto_id
JOIN dim_cliente c ON v.cliente_id = c.cliente_id
WHERE t.año = 2024
GROUP BY t.año, t.mes, p.categoria, c.segmento;
```

#### Snowflake Schema (Esquema Copo de Nieve)

Star schema con **dimensiones normalizadas**.

```
                ┌──────────┐
                │   Fact   │
                │  Ventas  │
                └────┬─────┘
                     │
            ┌────────┴────────┐
            │                 │
       ┌────┴─────┐      ┌────┴─────┐
       │   Dim    │      │   Dim    │
       │ Producto │      │  Tiempo  │
       └────┬─────┘      └──────────┘
            │
    ┌───────┴────────┐
    │                │
┌───┴────┐      ┌────┴────┐
│  Dim   │      │   Dim   │
│Categoría│     │  Marca  │
└────────┘      └─────────┘
```

**Comparación:**

| Aspecto | Star | Snowflake |
|---------|------|-----------|
| **Dimensiones** | Denormalizadas | Normalizadas |
| **Joins** | Menos (más rápido) | Más (más lento) |
| **Espacio** | Más (redundancia) | Menos |
| **Complejidad** | Simple | Complejo |
| **Uso** | Preferido para BI | Menos común |

---

## Slowly Changing Dimensions (SCD)

Manejo de **cambios en dimensiones** a lo largo del tiempo.

### Type 1: Sobrescribir

**No mantiene historial**, simplemente actualiza.

```sql
-- Cliente cambia de dirección
UPDATE dim_cliente
SET ciudad = 'Barcelona'
WHERE cliente_id = 123;
```

**Pros:** Simple
**Contras:** Pierdes historial

### Type 2: Agregar Nueva Fila

**Mantiene historial completo** con múltiples versiones.

```sql
CREATE TABLE dim_cliente (
    cliente_sk INT PRIMARY KEY,    -- Surrogate key (único)
    cliente_id INT,                 -- Business key (puede repetirse)
    nombre VARCHAR(100),
    ciudad VARCHAR(50),
    fecha_inicio DATE,
    fecha_fin DATE,
    es_actual BOOLEAN
);

-- Cliente cambia de ciudad
-- 1. Cerrar registro anterior
UPDATE dim_cliente
SET fecha_fin = CURRENT_DATE,
    es_actual = FALSE
WHERE cliente_id = 123 AND es_actual = TRUE;

-- 2. Insertar nuevo registro
INSERT INTO dim_cliente
VALUES (456, 123, 'Ana García', 'Barcelona',
        CURRENT_DATE, '9999-12-31', TRUE);
```

**Resultado:**
```
cliente_sk | cliente_id | ciudad    | fecha_inicio | fecha_fin  | es_actual
-----------|------------|-----------|--------------|------------|----------
123        | 123        | Madrid    | 2020-01-01   | 2024-01-15 | FALSE
456        | 123        | Barcelona | 2024-01-15   | 9999-12-31 | TRUE
```

**Pros:** Historial completo
**Contras:** Más complejo, más espacio

### Type 3: Agregar Columna

**Mantiene valor anterior** en columna separada.

```sql
CREATE TABLE dim_cliente (
    cliente_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    ciudad_actual VARCHAR(50),
    ciudad_anterior VARCHAR(50),
    fecha_cambio DATE
);

-- Cliente cambia de ciudad
UPDATE dim_cliente
SET ciudad_anterior = ciudad_actual,
    ciudad_actual = 'Barcelona',
    fecha_cambio = CURRENT_DATE
WHERE cliente_id = 123;
```

**Pros:** Simple, mantiene valor anterior
**Contras:** Solo guarda último cambio

---

## Particionamiento y Sharding

### Particionamiento

Dividir tabla grande en **piezas más pequeñas** (particiones).

**Tipos:**

#### 1. Range Partitioning (por Rango)
```sql
-- PostgreSQL
CREATE TABLE ventas (
    id INT,
    fecha DATE,
    monto DECIMAL
) PARTITION BY RANGE (fecha);

CREATE TABLE ventas_2023
    PARTITION OF ventas
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE ventas_2024
    PARTITION OF ventas
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

#### 2. List Partitioning (por Lista)
```sql
CREATE TABLE empleados (
    id INT,
    nombre VARCHAR(100),
    region VARCHAR(20)
) PARTITION BY LIST (region);

CREATE TABLE empleados_norte
    PARTITION OF empleados
    FOR VALUES IN ('Norte', 'Noroeste');
```

#### 3. Hash Partitioning
```sql
CREATE TABLE pedidos (
    id INT,
    cliente_id INT
) PARTITION BY HASH (cliente_id);

CREATE TABLE pedidos_p0
    PARTITION OF pedidos
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
```

**Beneficios:**
- Queries más rápidas (partition pruning)
- Mantenimiento más fácil (eliminar particiones antiguas)
- Mejor paralelización

### Sharding

Distribuir datos entre **múltiples servidores**.

```
┌─────────────┐
│   Cliente   │
│  (Router)   │
└──────┬──────┘
       │
   ┌───┴────┬─────────┐
   │        │         │
┌──▼──┐  ┌──▼──┐  ┌──▼──┐
│Shard│  │Shard│  │Shard│
│  1  │  │  2  │  │  3  │
└─────┘  └─────┘  └─────┘
```

**Estrategias:**

#### 1. Hash-based Sharding
```python
shard = hash(customer_id) % num_shards
```

#### 2. Range-based Sharding
```
Shard 1: customer_id 1-1000
Shard 2: customer_id 1001-2000
Shard 3: customer_id 2001-3000
```

#### 3. Geographic Sharding
```
Shard US: clientes de USA
Shard EU: clientes de Europa
Shard ASIA: clientes de Asia
```

**Beneficios:**
- Escala horizontal
- Aislamiento de fallas

**Desafíos:**
- Joins entre shards son costosos
- Complejidad en aplicación
- Rebalanceo difícil

---

## Integridad Referencial

Garantizar **consistencia** entre tablas relacionadas.

### Primary Key (Clave Primaria)

```sql
CREATE TABLE clientes (
    id INT PRIMARY KEY,        -- Único, no NULL
    nombre VARCHAR(100) NOT NULL
);
```

### Foreign Key (Clave Foránea)

```sql
CREATE TABLE pedidos (
    id INT PRIMARY KEY,
    cliente_id INT NOT NULL,
    fecha DATE,
    FOREIGN KEY (cliente_id) REFERENCES clientes(id)
        ON DELETE CASCADE      -- Elimina pedidos si se elimina cliente
        ON UPDATE CASCADE      -- Actualiza id si cambia
);
```

**Opciones de acción:**
- **CASCADE**: Propaga cambio
- **SET NULL**: Establece NULL
- **SET DEFAULT**: Usa valor default
- **RESTRICT**: Previene operación
- **NO ACTION**: Similar a RESTRICT

### Unique Constraint

```sql
CREATE TABLE usuarios (
    id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE,    -- No duplicados
    username VARCHAR(50) UNIQUE
);
```

### Check Constraint

```sql
CREATE TABLE productos (
    id INT PRIMARY KEY,
    precio DECIMAL CHECK (precio > 0),
    descuento DECIMAL CHECK (descuento BETWEEN 0 AND 1)
);
```

---

## Resumen

✅ **SQL** (relacional, ACID) vs **NoSQL** (flexible, escalable horizontalmente)
✅ **OLTP** (transaccional, normalizado) vs **OLAP** (analítico, denormalizado)
✅ **Star Schema** (fact + dimensions) para data warehouses
✅ **SCD Type 2** es el más común para mantener historial
✅ **Particionamiento** divide tabla en una BD, **Sharding** divide entre BDs
✅ **Foreign Keys** garantizan integridad referencial

**Siguiente:** [03. Procesos ETL/ELT](03_procesos_etl_elt.md)
