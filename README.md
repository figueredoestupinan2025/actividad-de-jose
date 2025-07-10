# Base de Datos Pizza - Eventos MySQL

## 📋 Descripción General

Este documento contiene la implementación de eventos MySQL para la base de datos de pizzería, utilizando tanto `ON COMPLETION PRESERVE` como `ON COMPLETION NOT PRESERVE` para diferentes casos de uso.

## 🗂️ Estructura de Tablas

### Tabla `resumen_ventas`
```sql
CREATE TABLE IF NOT EXISTS resumen_ventas (
    fecha           DATE         PRIMARY KEY,
    total_pedidos   INT,
    total_ingresos  DECIMAL(12,2),
    creado_en       DATETIME     DEFAULT CURRENT_TIMESTAMP
);
```

### Tabla `alerta_stock`
```sql
CREATE TABLE IF NOT EXISTS alerta_stock (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    ingrediente_id  INT UNSIGNED NOT NULL,
    stock_actual    INT NOT NULL,
    fecha_alerta    DATETIME NOT NULL,
    creado_en       DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (ingrediente_id) REFERENCES ingrediente(id)
);
```

## 🎯 Eventos Implementados

### 1. Resumen Diario Único
**Propósito**: Generar un resumen de ventas del día anterior una sola vez y luego autodestruirse.

```sql
-- Verificar si el scheduler está habilitado
SET GLOBAL event_scheduler = ON;

-- Crear evento que se ejecuta una sola vez y luego se elimina
CREATE EVENT ev_resumen_diario_unico
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 MINUTE
ON COMPLETION NOT PRESERVE
DO
BEGIN
    -- Insertar resumen del día anterior
    INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
    SELECT 
        DATE_SUB(CURDATE(), INTERVAL 1 DAY) as fecha,
        COUNT(*) as total_pedidos,
        COALESCE(SUM(total), 0) as total_ingresos
    FROM pedido 
    WHERE DATE(fecha_pedido) = DATE_SUB(CURDATE(), INTERVAL 1 DAY)
    ON DUPLICATE KEY UPDATE
        total_pedidos = VALUES(total_pedidos),
        total_ingresos = VALUES(total_ingresos);
END;
```

### 2. Resumen Semanal Recurrente
**Propósito**: Generar resumen semanal cada lunes a las 01:00 AM, manteniendo el evento activo.

```sql
-- Crear evento recurrente que se mantiene activo
CREATE EVENT ev_resumen_semanal
ON SCHEDULE EVERY 1 WEEK
STARTS CONCAT(DATE_ADD(CURDATE(), INTERVAL (7 - WEEKDAY(CURDATE())) DAY), ' 01:00:00')
ON COMPLETION PRESERVE
DO
BEGIN
    DECLARE fecha_inicio DATE;
    DECLARE fecha_fin DATE;
    
    -- Calcular fechas de la semana pasada (lunes a domingo)
    SET fecha_fin = DATE_SUB(CURDATE(), INTERVAL WEEKDAY(CURDATE()) + 1 DAY);
    SET fecha_inicio = DATE_SUB(fecha_fin, INTERVAL 6 DAY);
    
    -- Insertar resumen semanal
    INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
    SELECT 
        fecha_fin as fecha,
        COUNT(*) as total_pedidos,
        COALESCE(SUM(total), 0) as total_ingresos
    FROM pedido 
    WHERE DATE(fecha_pedido) BETWEEN fecha_inicio AND fecha_fin
    ON DUPLICATE KEY UPDATE
        total_pedidos = VALUES(total_pedidos),
        total_ingresos = VALUES(total_ingresos);
END;
```

### 3. Alerta de Stock Bajo Única
**Propósito**: Generar alertas de stock bajo una sola vez en el futuro y luego autodestruirse.

```sql
-- Crear evento único para alertas de stock bajo
CREATE EVENT ev_alerta_stock_unico
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 5 MINUTE
ON COMPLETION NOT PRESERVE
DO
BEGIN
    -- Insertar alertas para ingredientes con stock < 5
    INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
    SELECT 
        id as ingrediente_id,
        stock as stock_actual,
        NOW() as fecha_alerta
    FROM ingrediente 
    WHERE stock < 5;
    
    -- Log opcional para debugging
    INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
    VALUES (CURDATE(), -1, -1.00); -- Valor especial para indicar ejecución de alerta
END;
```

### 4. Monitoreo Continuo de Stock
**Propósito**: Revisar stock cada 30 minutos y mantener el evento activo permanentemente.

```sql
-- Crear evento recurrente para monitoreo continuo
CREATE EVENT ev_monitor_stock_bajo
ON SCHEDULE EVERY 30 MINUTE
STARTS CURRENT_TIMESTAMP
ON COMPLETION PRESERVE
DO
BEGIN
    -- Insertar alertas para ingredientes con stock < 10
    INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
    SELECT 
        id as ingrediente_id,
        stock as stock_actual,
        NOW() as fecha_alerta
    FROM ingrediente 
    WHERE stock < 10
    AND NOT EXISTS (
        SELECT 1 FROM alerta_stock a 
        WHERE a.ingrediente_id = ingrediente.id 
        AND DATE(a.fecha_alerta) = CURDATE()
    );
END;
```

### 5. Limpieza de Resúmenes Antiguos
**Propósito**: Eliminar registros antiguos (>365 días) una sola vez y luego autodestruirse.

```sql
-- Crear evento único para limpieza de datos antiguos
CREATE EVENT ev_purgar_resumen_antiguo
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 2 MINUTE
ON COMPLETION NOT PRESERVE
DO
BEGIN
    -- Eliminar registros más antiguos de 365 días
    DELETE FROM resumen_ventas 
    WHERE fecha < DATE_SUB(CURDATE(), INTERVAL 365 DAY);
    
    -- Eliminar alertas de stock más antiguas de 30 días
    DELETE FROM alerta_stock 
    WHERE DATE(fecha_alerta) < DATE_SUB(CURDATE(), INTERVAL 30 DAY);
END;
```

## 📊 Comandos de Administración

### Verificar Estado del Scheduler
```sql
-- Verificar si el scheduler está habilitado
SHOW VARIABLES LIKE 'event_scheduler';

-- Habilitar el scheduler si está deshabilitado
SET GLOBAL event_scheduler = ON;
```

### Consultar Eventos Activos
```sql
-- Ver todos los eventos de la base de datos
SELECT 
    EVENT_NAME,
    EVENT_TYPE,
    EXECUTE_AT,
    INTERVAL_VALUE,
    INTERVAL_FIELD,
    STATUS,
    ON_COMPLETION,
    CREATED,
    LAST_EXECUTED
FROM information_schema.EVENTS 
WHERE EVENT_SCHEMA = DATABASE();
```

### Gestión de Eventos
```sql
-- Deshabilitar un evento
ALTER EVENT ev_monitor_stock_bajo DISABLE;

-- Habilitar un evento
ALTER EVENT ev_monitor_stock_bajo ENABLE;

-- Eliminar un evento
DROP EVENT IF EXISTS ev_resumen_diario_unico;
```

## 🔍 Verificación de Resultados

### Consultar Resúmenes de Ventas
```sql
SELECT * FROM resumen_ventas ORDER BY fecha DESC LIMIT 10;
```

### Consultar Alertas de Stock
```sql
SELECT 
    a.id,
    i.nombre as ingrediente,
    a.stock_actual,
    a.fecha_alerta,
    a.creado_en
FROM alerta_stock a
JOIN ingrediente i ON a.ingrediente_id = i.id
ORDER BY a.fecha_alerta DESC
LIMIT 10;
```

## 📝 Conceptos Clave

### `ON COMPLETION PRESERVE`
- **Uso**: Para eventos que deben **mantenerse activos** después de su ejecución
- **Casos**: Tareas recurrentes, monitoreo continuo
- **Ejemplos**: `ev_resumen_semanal`, `ev_monitor_stock_bajo`

### `ON COMPLETION NOT PRESERVE`
- **Uso**: Para eventos que deben **eliminarse automáticamente** después de su ejecución
- **Casos**: Tareas únicas, limpieza de datos, inicializaciones
- **Ejemplos**: `ev_resumen_diario_unico`, `ev_alerta_stock_unico`, `ev_purgar_resumen_antiguo`

## ⚠️ Consideraciones Importantes

1. **Scheduler**: El `event_scheduler` debe estar habilitado en MySQL
2. **Permisos**: El usuario debe tener privilegios para crear eventos
3. **Horarios**: Los eventos utilizan la zona horaria del servidor MySQL
4. **Rendimiento**: Eventos frecuentes pueden impactar el rendimiento
5. **Logs**: MySQL registra la ejecución de eventos en sus logs

## 🚀 Ejecución del Script

Para implementar todos los eventos:

```sql
-- 1. Habilitar el scheduler
SET GLOBAL event_scheduler = ON;

-- 2. Ejecutar cada CREATE EVENT en orden
-- 3. Verificar con la consulta de eventos activos
-- 4. Monitorear las tablas para verificar funcionamiento
```

---

*Base de datos de pizzería - Sistema de eventos automatizados*
