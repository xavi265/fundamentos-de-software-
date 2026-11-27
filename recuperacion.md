**Materia:** administracion de base de datos 
**Alumno:** Acevedo Yañez Christian Xavier
**Fecha:** 26 novienmbre de 2026 
**Profesor/a:** diego alberto bollas paredes


---
# Práctica de Respaldo y Recuperación  - MariaDB

## 1. Creación de la base de datos `tienda` y carga de datos iniciales
Se creó la base de datos `tienda`, la tabla `clientes` y se insertaron 5 registros iniciales.

```sql
CREATE DATABASE tienda;
USE tienda;
CREATE TABLE clientes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO clientes (nombre, email) VALUES
('Ana Torres', 'ana@correo.com'),
('Luis Perez', 'luis@correo.com'),
('Maria Gomez', 'maria@correo.com'),
('Carlos Ruiz', 'carlos@correo.com'),
('Sofia Diaz', 'sofia@correo.com');

SELECT COUNT(*) FROM clientes;
```
<sr=img"C:\Users\xavi\Pictures\Screenshots\Screenshot175722.png">

---
## 2. Respaldo inicial, borrado de prueba y restauración básica
Se generó el respaldo inicial `tienda_backup.sql`, se eliminó la base de datos `tienda` para probar el respaldo y se restauró.

```bash
mariadb-dump -u root -p --databases tienda > tienda_backup.sql
ls -lh tienda_backup.sql
```

```sql
DROP DATABASE tienda;
SHOW DATABASES;
```

```bash
mariadb -u root -p < tienda_backup.sql
mariadb -u root -p -e "SELECT * FROM tienda.clientes;"
```

## 3. Verificación del binary log y respaldo con `--flush-logs` y `--master-data=2`
Se reinició el servicio de MariaDB, se verificó que `log_bin` estuviera activo (`ON`), se generó `tienda_pitr.sql` y se consultó la referencia del archivo y posición del binary log (`mariadb-bin.000025`, posición `389`).

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
sudo systemctl restart mariadb
mariadb -u root -p -e "SHOW VARIABLES LIKE 'log_bin';"
mariadb-dump -u root -p --databases tienda --flush-logs --master-data=2 > tienda_pitr.sql
grep "CHANGE MASTER" tienda_pitr.sql
```

## 4. Nuevas inserciones, borrado de registros y restauración
Se insertaron 2 clientes adicionales (totalizando 7 registros), se ejecutó `DELETE FROM clientes;` y se restauró con `tienda_pitr.sql`, además de los comandos ejecutados con `mariadb-binlog` y la verificación final.

```sql
USE tienda;
INSERT INTO clientes (nombre, email) VALUES
('Pedro Nunez', 'pedro@correo.com'),
('Laura Vega', 'laura@correo.com');

SELECT COUNT(*) FROM clientes;
SELECT NOW();
DELETE FROM clientes;
SELECT COUNT(*) FROM clientes;
```

```bash
mariadb -u root -p < tienda_pitr.sql
sudo mariadb-binlog --start-position=330 --stop-datetime="2026-11-24 14:31:20" \
    /var/lib/mysql/mariadb-bin.000002 | mariadb -u root -p
sudo mariadb-binlog --start-position=330 --stop-datetime="2026-11-26 18:02:54"     /var/lib/mysql/mariadb-bin.000002 | mariadb -u root -p
mariadb -u root -p -e "SELECT * FROM tienda.clientes;"
```

## 5. Preguntas y respuestas

### 1. ¿Qué datos se habrían perdido si solo hubieras usado el respaldo de la Parte A?
Si solo restaurabas `tienda_backup.sql`, se habrían perdido los 2 registros insertados después del respaldo inicial (**Pedro Nunez** y **Laura Vega**), conservando únicamente los 5 clientes originales.

### 2. ¿Para qué sirve la opción `--master-data=2`?
Sirve para incluir en el dump (como comentario `-- CHANGE MASTER TO ...`) el nombre del archivo de binary log y la posición exacta al momento del respaldo, lo que indica desde qué punto continuar en una recuperación punto en el tiempo.

### 3. ¿Por qué es necesario activar el binlog antes de que ocurra el error?
Porque el binary log registra los cambios realizados en la base de datos (`INSERT`, `UPDATE`, `DELETE`, etc.). Si `log_bin` no está activado antes del incidente, no se guardan los movimientos posteriores al respaldo para recuperarlos después.

### 4. ¿Por qué se esperaron 5 segundos antes y después de anotar la hora?
Para tener un margen de tolerancia temporal al delimitar las marcas de tiempo (`--stop-datetime`), evitando cortar justo en el límite de un segundo o incluir por error sentencias ejecutadas inmediatamente después.

[def]: <C:/Users/xavi/Pictures/Screenshots/Screenshot 2026-11-26 175722.png>