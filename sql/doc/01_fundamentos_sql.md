# 1. Fundamentos de SQL

## ¿Qué es SQL y cuál es su propósito?

**SQL** (Structured Query Language) es un lenguaje de programación estándar diseñado para gestionar y manipular datos en sistemas de bases de datos relacionales.

**Propósito principal:**
- Consultar datos (recuperar información)
- Insertar, actualizar y eliminar datos
- Crear y modificar estructuras de bases de datos (tablas, índices, vistas)
- Controlar el acceso a los datos (permisos y seguridad)
- Gestionar transacciones y garantizar la integridad de los datos

## Database Engine vs Client

### Database Engine (Motor de Base de Datos)
Es el software principal que:
- Almacena los datos físicamente en disco
- Procesa y ejecuta las consultas SQL
- Gestiona transacciones, seguridad y concurrencia
- Optimiza el rendimiento de las consultas

**Ejemplos:** PostgreSQL Server, MySQL Server, Snowflake Engine, SQL Server Engine

### Client (Cliente)
Es la herramienta o aplicación que:
- Permite al usuario conectarse al motor de base de datos
- Envía consultas SQL al motor
- Recibe y muestra los resultados
- Proporciona interfaz para interactuar con la base de datos

**Ejemplos:** DBeaver, pgAdmin, DataGrip, SQL Server Management Studio (SSMS), SnowSQL, psql

**Analogía:** El motor es como un servidor web (procesa peticiones), el cliente es como un navegador (envía peticiones y muestra resultados).

## SQL vs NoSQL

| Característica | SQL (Relacional) | NoSQL (No Relacional) |
|---------------|------------------|----------------------|
| **Estructura** | Tablas con esquema fijo | Documentos, key-value, grafos, columnas |
| **Esquema** | Definido y rígido (schema-on-write) | Flexible o sin esquema (schema-on-read) |
| **Escalabilidad** | Vertical (más recursos al servidor) | Horizontal (más servidores) |
| **Transacciones** | ACID garantizado | Eventual consistency (generalmente) |
| **Relaciones** | Joins nativos, claves foráneas | Denormalizado, duplicación de datos |
| **Casos de uso** | Transacciones, reportes, BI | Big data, tiempo real, datos no estructurados |

**Ejemplos SQL:** PostgreSQL, MySQL, SQL Server, Snowflake, BigQuery, Redshift

**Ejemplos NoSQL:** MongoDB (documentos), Redis (key-value), Neo4j (grafos), Cassandra (columnas)

## Conceptos de Bases de Datos Relacionales

### Tabla (Table)
Estructura que organiza datos en filas y columnas. Representa una entidad (clientes, productos, pedidos).

```
Tabla: empleados
┌────┬──────────┬─────────┬────────┐
│ id │ nombre   │ salario │ depto  │
├────┼──────────┼─────────┼────────┤
│ 1  │ Ana      │ 50000   │ IT     │
│ 2  │ Carlos   │ 60000   │ Ventas │
└────┴──────────┴─────────┴────────┘
```

### Fila (Row/Record/Tuple)
Representa un registro individual en una tabla. Cada fila contiene datos sobre una instancia específica de la entidad.

### Columna (Column/Field/Attribute)
Define un atributo de la entidad. Cada columna tiene:
- **Nombre:** identificador único en la tabla
- **Tipo de dato:** INT, VARCHAR, DATE, etc.
- **Restricciones:** NOT NULL, UNIQUE, etc.

### Schema (Esquema)
Define la estructura lógica de la base de datos:
- Qué tablas existen
- Qué columnas tiene cada tabla
- Tipos de datos de cada columna
- Relaciones entre tablas (claves primarias/foráneas)
- Índices, vistas, constraints

**Ejemplo visual:**
```
Schema: empresa
│
├── Tabla: empleados
│   ├── id (INT, PRIMARY KEY)
│   ├── nombre (VARCHAR(100))
│   ├── salario (DECIMAL)
│   └── departamento_id (INT, FOREIGN KEY)
│
└── Tabla: departamentos
    ├── id (INT, PRIMARY KEY)
    └── nombre (VARCHAR(50))
```

## Sistemas de Bases de Datos Comunes

### Snowflake
- **Tipo:** Cloud Data Warehouse
- **Arquitectura:** Almacenamiento y cómputo separados
- **Ventajas:** Auto-escalado, zero-copy cloning, time travel
- **Uso:** Data warehousing, analytics a gran escala

### PostgreSQL
- **Tipo:** RDBMS open source
- **Ventajas:** Extensible, conforme a estándares, soporta JSON
- **Uso:** Aplicaciones transaccionales (OLTP), analytics ligero

### MySQL
- **Tipo:** RDBMS open source
- **Ventajas:** Rápido para lectura, ampliamente usado
- **Uso:** Aplicaciones web, OLTP

### SQL Server
- **Tipo:** RDBMS de Microsoft
- **Ventajas:** Integración con ecosistema Microsoft, herramientas robustas
- **Uso:** Enterprise applications, BI

### Amazon Redshift
- **Tipo:** Cloud Data Warehouse (basado en PostgreSQL)
- **Arquitectura:** Columnar, MPP (Massively Parallel Processing)
- **Uso:** Analytics, BI, grandes volúmenes de datos

### Google BigQuery
- **Tipo:** Serverless Data Warehouse
- **Arquitectura:** Almacenamiento columnar, procesamiento distribuido
- **Ventajas:** Sin gestión de infraestructura, SQL estándar, escalado automático
- **Uso:** Analytics masivo, ML integrado

---

## Conceptos Clave para Recordar

1. **SQL es declarativo:** Describes QUÉ datos quieres, no CÓMO obtenerlos
2. **Las tablas son la unidad básica:** Todo gira alrededor de tablas relacionadas
3. **El esquema define la estructura:** Es el "contrato" de cómo se organizan los datos
4. **SQL vs NoSQL:** Elige según tus necesidades (estructura vs flexibilidad)
5. **Motor vs Cliente:** El motor procesa, el cliente conecta
