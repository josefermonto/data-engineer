# 12. Transacciones y Propiedades ACID

---

## ¿Qué es una Transacción?

Una **transacción** es un conjunto de operaciones SQL que se ejecutan como **una sola unidad** lógica de trabajo.

**Principio:** **Todo o nada** (all or nothing)

### Ejemplo

```sql
-- Transferencia bancaria
BEGIN;

UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;  -- Retirar de cuenta A
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2;  -- Depositar en cuenta B

COMMIT;  -- Si todo OK
-- O ROLLBACK si hay error
```

Sin transacciones, si falla la segunda operación, ¡el dinero desaparece!

---

## Propiedades ACID

ACID garantiza la **confiabilidad** de las transacciones.

### A - Atomicity (Atomicidad)

**"Todo o nada"**: La transacción se completa totalmente o no se completa en absoluto.

```sql
BEGIN;
INSERT INTO pedidos (cliente_id, total) VALUES (1, 100);
INSERT INTO pedidos_detalle (pedido_id, producto_id) VALUES (1, 10);
COMMIT;  -- Ambas se completan

-- Si la segunda falla:
BEGIN;
INSERT INTO pedidos (cliente_id, total) VALUES (1, 100);
INSERT INTO pedidos_detalle (pedido_id, producto_id) VALUES (1, 10);  -- ERROR
ROLLBACK;  -- Se deshace TODO, incluyendo la primera
```

### C - Consistency (Consistencia)

La transacción lleva la base de datos de un **estado válido a otro estado válido**.

- Respeta todas las reglas: constraints, triggers, cascadas
- No viola PRIMARY KEY, FOREIGN KEY, CHECK, etc.

```sql
-- Consistencia: la suma total se mantiene
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;  -- Suma antes: 1000
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2;  -- Suma después: 1000
COMMIT;  -- Estado consistente

-- Violación de consistencia (se previene):
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;
-- Sin la segunda actualización
ROLLBACK;  -- Evita estado inconsistente
```

### I - Isolation (Aislamiento)

Las transacciones concurrentes **no se afectan entre sí**.

Una transacción no ve cambios **no confirmados** de otras transacciones.

```sql
-- Transacción 1                    Transacción 2
BEGIN;
SELECT saldo FROM cuentas           BEGIN;
WHERE id = 1;  -- 1000
                                    UPDATE cuentas SET saldo = 500
                                    WHERE id = 1;
SELECT saldo FROM cuentas           -- (no commit todavía)
WHERE id = 1;  -- Sigue siendo 1000
                                    COMMIT;
SELECT saldo FROM cuentas
WHERE id = 1;  -- Ahora 500
COMMIT;
```

### D - Durability (Durabilidad)

Una vez hecho **COMMIT**, los cambios son **permanentes**, incluso si hay fallo del sistema.

```sql
BEGIN;
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 1;
COMMIT;  -- Confirmado

-- Aunque el servidor se apague aquí, el cambio persiste
```

Los datos se escriben al **transaction log** antes de COMMIT.

---

## Comandos de Transacción

### BEGIN / START TRANSACTION

Inicia una transacción.

```sql
BEGIN;
-- O
START TRANSACTION;
-- O
BEGIN TRANSACTION;  -- SQL Server
```

### COMMIT

Confirma (guarda) los cambios permanentemente.

```sql
BEGIN;
UPDATE empleados SET salario = salario * 1.10 WHERE departamento = 'IT';
COMMIT;  -- Cambios confirmados
```

### ROLLBACK

Deshace los cambios y cancela la transacción.

```sql
BEGIN;
DELETE FROM empleados WHERE departamento = 'IT';
-- ¡Ups! Error
ROLLBACK;  -- Deshace el DELETE
```

### SAVEPOINT (Punto de Guardado)

Crea un punto intermedio en la transacción para hacer ROLLBACK parcial.

```sql
BEGIN;

UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;
SAVEPOINT sp1;

UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2;
SAVEPOINT sp2;

UPDATE cuentas SET saldo = saldo + 50 WHERE id = 3;
-- Ups, error en la última

ROLLBACK TO SAVEPOINT sp2;  -- Deshace solo la última
-- Las primeras dos actualizaciones siguen activas

COMMIT;  -- Confirma las primeras dos
```

---

## Niveles de Aislamiento (Isolation Levels)

Controlan el **grado de aislamiento** entre transacciones concurrentes.

### Trade-off: Consistencia vs Performance

Más aislamiento = Más consistencia, pero menos concurrencia

| Nivel | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| **Read Uncommitted** | ✓ Sí | ✓ Sí | ✓ Sí |
| **Read Committed** | ✗ No | ✓ Sí | ✓ Sí |
| **Repeatable Read** | ✗ No | ✗ No | ✓ Sí |
| **Serializable** | ✗ No | ✗ No | ✗ No |

### 1. Read Uncommitted (Menos Estricto)

Lee datos **no confirmados** de otras transacciones.

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
SELECT * FROM cuentas;  -- Puede ver cambios no confirmados
COMMIT;
```

**Problema - Dirty Read:**
- T1 actualiza pero no hace COMMIT
- T2 lee el dato no confirmado
- T1 hace ROLLBACK
- T2 leyó dato que "nunca existió"

### 2. Read Committed (Default en mayoría)

Lee solo datos **confirmados**.

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- Default en PostgreSQL, SQL Server
```

**Problema - Non-Repeatable Read:**
- T1 lee un registro: saldo = 1000
- T2 actualiza y hace COMMIT: saldo = 500
- T1 lee el mismo registro: saldo = 500 (¡cambió!)

### 3. Repeatable Read

Una vez leído un registro, **no cambiará** durante la transacción (se bloquea).

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- Default en MySQL
```

**Problema - Phantom Read:**
- T1 lee registros con WHERE: 10 filas
- T2 inserta nueva fila que cumple WHERE y hace COMMIT
- T1 ejecuta misma query: 11 filas (¡apareció nueva!)

### 4. Serializable (Más Estricto)

Transacciones se ejecutan **como si fueran en serie** (una tras otra).

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Previene todos los problemas**, pero:
- ❌ Menor concurrencia
- ❌ Mayor posibilidad de deadlocks
- ❌ Peor performance

---

## Locking y Control de Concurrencia

### Tipos de Locks

#### Shared Lock (S) - Lectura
- Permite a otras transacciones **leer** pero no **escribir**
- Múltiples shared locks pueden existir simultáneamente

#### Exclusive Lock (X) - Escritura
- No permite a otras transacciones **leer ni escribir**
- Solo una transacción puede tener exclusive lock

```sql
-- Explicit locking en PostgreSQL
BEGIN;
SELECT * FROM cuentas WHERE id = 1 FOR UPDATE;  -- Exclusive lock
-- Otros no pueden modificar esta fila hasta COMMIT
COMMIT;

-- Shared lock
SELECT * FROM cuentas WHERE id = 1 FOR SHARE;  -- Otros pueden leer
```

### Deadlocks (Interbloqueos)

Ocurre cuando dos transacciones se **bloquean mutuamente**.

```sql
-- Transacción 1                      Transacción 2
BEGIN;                                BEGIN;
UPDATE cuentas SET saldo = 1000       UPDATE cuentas SET saldo = 2000
WHERE id = 1;  -- Lock en fila 1      WHERE id = 2;  -- Lock en fila 2

UPDATE cuentas SET saldo = 1500       UPDATE cuentas SET saldo = 2500
WHERE id = 2;  -- Espera lock...      WHERE id = 1;  -- Espera lock...

-- ¡DEADLOCK! Ambas esperan a la otra
```

**Solución:** DB detecta y aborta una transacción.

**Prevención:**
- Acceder recursos en el mismo orden
- Mantener transacciones cortas
- Usar índices (reduce tiempo de lock)

---

## Transacciones en Diferentes Sistemas

### PostgreSQL

```sql
BEGIN;
-- operaciones
COMMIT;  -- o ROLLBACK

-- Nivel de aislamiento
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### MySQL

```sql
START TRANSACTION;
-- operaciones
COMMIT;  -- o ROLLBACK

-- Nivel de aislamiento
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### SQL Server

```sql
BEGIN TRANSACTION;
-- operaciones
COMMIT TRANSACTION;  -- o ROLLBACK TRANSACTION

-- Nivel de aislamiento
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### Snowflake

```sql
BEGIN;
-- operaciones
COMMIT;  -- o ROLLBACK

-- Multi-statement transactions
BEGIN TRANSACTION;
INSERT INTO tabla1 VALUES (...);
UPDATE tabla2 SET ...;
COMMIT;
```

**Nota:** Snowflake usa **MVCC** (Multi-Version Concurrency Control) - alta concurrencia sin locks.

### BigQuery

**BigQuery NO soporta transacciones tradicionales** (OLAP optimizado).

Alternativas:
- Queries atómicas individuales
- Scripts multi-statement (pero sin rollback parcial)
- Usar tablas temporales

---

## Transacciones Implícitas vs Explícitas

### Implícita (Autocommit)

Cada statement es una transacción automática.

```sql
-- Autocommit ON (default en muchos DBs)
UPDATE empleados SET salario = 50000 WHERE id = 1;  -- Auto-COMMIT

INSERT INTO empleados (nombre) VALUES ('Ana');  -- Auto-COMMIT
```

### Explícita

Tú controlas BEGIN y COMMIT.

```sql
BEGIN;
UPDATE empleados SET salario = 50000 WHERE id = 1;
INSERT INTO logs (mensaje) VALUES ('Salario actualizado');
COMMIT;  -- Ambas se confirman juntas
```

---

## Mejores Prácticas

1. ✅ **Usa transacciones para operaciones relacionadas**
   ```sql
   BEGIN;
   INSERT INTO pedidos (...);
   INSERT INTO pedidos_detalle (...);
   UPDATE inventario SET stock = stock - 1;
   COMMIT;
   ```

2. ✅ **Mantén transacciones CORTAS** (evita locks prolongados)

3. ✅ **Maneja errores con ROLLBACK**
   ```sql
   BEGIN;
   -- operaciones
   IF error THEN
       ROLLBACK;
   ELSE
       COMMIT;
   END IF;
   ```

4. ✅ **Usa nivel de aislamiento apropiado**
   - OLTP: Read Committed o Repeatable Read
   - Reportes: Read Uncommitted (si tolerancia a dirty reads)

5. ✅ **Evita deadlocks: acceso en orden consistente**

6. ❌ **NO hagas transacciones largas** (bloquea recursos)

7. ❌ **NO uses transacciones para queries de solo lectura** (a menos que necesites consistencia snapshot)

---

## Ejemplo Práctico: E-commerce Order

```sql
BEGIN;

-- 1. Crear pedido
INSERT INTO pedidos (cliente_id, fecha, total)
VALUES (123, CURRENT_DATE, 150.00)
RETURNING id INTO @pedido_id;

-- 2. Agregar items
INSERT INTO pedidos_detalle (pedido_id, producto_id, cantidad, precio)
VALUES
    (@pedido_id, 10, 2, 50.00),
    (@pedido_id, 20, 1, 50.00);

-- 3. Reducir inventario
UPDATE productos SET stock = stock - 2 WHERE id = 10;
UPDATE productos SET stock = stock - 1 WHERE id = 20;

-- 4. Verificar stock suficiente
IF EXISTS (SELECT 1 FROM productos WHERE id IN (10, 20) AND stock < 0) THEN
    ROLLBACK;  -- No hay suficiente stock
ELSE
    COMMIT;    -- Todo OK
END IF;
```

---

## Resumen ACID

| Propiedad | Garantiza | Ejemplo |
|-----------|-----------|---------|
| **Atomicity** | Todo o nada | Transferencia completa o ninguna |
| **Consistency** | Estado válido | No viola constraints |
| **Isolation** | Sin interferencia | No ve cambios no confirmados |
| **Durability** | Cambios permanentes | Sobrevive a fallos del sistema |

---

## Cuando NO Usar Transacciones

- ❌ Queries de solo lectura sin necesidad de snapshot consistente
- ❌ Bulk loads (usa COPY/BULK INSERT sin transacción)
- ❌ Data Warehouses (OLAP) - generalmente no necesitan
- ❌ Operaciones muy largas (considera partirlas)
