
> **MySQL** es el sistema de gestión de bases de datos relacionales (RDBMS) de código abierto más extendido del mundo. Almacena datos en tablas con filas y columnas, relacionadas entre sí mediante claves. 


---
# 1. 🗄️ ¿Cómo funciona MySQL?

MySQL funciona según el **principio cliente-servidor** y consta de un servidor MySQL y uno o más clientes. 

- El servidor gestiona el almacenamiento, procesamiento y distribución de los datos, que se organizan en tablas con filas, columnas y diferentes tipos de datos. Las bases de datos pueden exportarse o respaldarse como un archivo `.sql`, por ejemplo `wordpress.sql`.
- Los clientes interactúan con el servidor mediante consultas SQL para **insertar, eliminar, modificar y recuperar datos**. Varios clientes pueden realizar consultas simultáneamente y el acceso puede realizarse desde una red interna o a través de Internet. Un ejemplo habitual es `WordPress`, que almacena en MySQL información como publicaciones, usuarios, contraseñas y configuraciones. En arquitecturas más complejas, las bases de datos también pueden distribuirse entre varios servidores.

Es la base de datos de la mayoría de aplicaciones web (WordPress, Drupal, Magento...) y forma parte del stack LAMP (Linux, Apache, MySQL, PHP). Desde su adquisición por Oracle en 2010, la comunidad mantiene el fork **MariaDB** como alternativa compatible.

> [!CAUTION]
> Cuando se produce un error, la aplicación web puede devolver información sobre este al cliente. Los errores pueden revelar detalles sobre la interacción entre la aplicación y la base de datos y, en determinados casos, ayudar a identificar vulnerabilidades como las `inyecciones SQL`.

Si la consulta se procesa correctamente, la aplicación recibe el resultado y puede utilizarlo para funciones como autenticación, búsquedas o generación de contenido.

> [!NOTE]
> `MariaDB` es un **fork de MySQL** creado a partir de su código fuente original. Surgió tras la adquisición de MySQL AB por Oracle y la salida de parte del equipo original de desarrollo. MariaDB mantiene una alta compatibilidad con MySQL, aunque actualmente ambos proyectos han evolucionado de forma independiente.

----
## 1.1. 🗄️ Flujo
El flujo de trabajo con MySQL es el siguiente: 

1️⃣ **La aplicación cliente se conecta al servidor MySQL**: La aplicación realiza la conexión a través del código backend, normalmente mediante una librería o driver específico de MySQL.

2️⃣ **MySQL autentica y procesa la consulta**: El servidor verifica las credenciales del cliente y, una vez autenticado, recibe la consulta SQL. A continuación, analiza su sintaxis, genera un plan de ejecución y lo optimiza para determinar la forma más eficiente de ejecutarla.

3️⃣ **El Storage Engine ejecuta las operaciones sobre los datos**: El _Storage Engine_ es el componente de MySQL responsable de gestionar cómo se almacenan, organizan y recuperan físicamente los datos. Es el encargado de realizar las operaciones necesarias sobre las estructuras de almacenamiento, índices y archivos de datos. 

4️⃣ **MySQL devuelve el resultado al backend**: Una vez ejecutada la consulta, MySQL devuelve los resultados al backend, que los procesa y finalmente los entrega a la aplicación cliente.


---
## 1.2. 🗄️ Flujo Base de datos → Tabla → Fila → Columna

```
Base de datos: tienda
└── Tabla: clientes
    ├── Columna: id (INT, PRIMARY KEY)
    ├── Columna: nombre (VARCHAR)
    ├── Columna: email (VARCHAR)
    └── Columna: fecha_alta (DATE)

    Filas:
    │ id │ nombre  │ email           │ fecha_alta │
    │ 1  │ Jessica │ j@empresa.com   │ 2024-01-15 │
    │ 2  │ Carlos  │ c@empresa.com   │ 2024-02-20 │
```

--------
#### Relaciones entre tablas
Las tablas pueden compartir datos que estén relacionados. Si se actualiza el dato en una tabla, se actualiza en el resto.

```
  clientes                    pedidos
  ─────────                   ───────
  id (PK) ◀───────────────── cliente_id (FK)
  nombre                     id (PK)
  email                      producto
                             cantidad...
```

Una clave foránea (FK) en `pedidos.cliente_id` apunta a `clientes.id`. 

MySQL puede garantizar la **integridad referencial**: no puedes añadir un pedido con un `cliente_id` que no existe en `clientes`.


---
## 1.3. 🔧 Storage Engines

El storage engine es el componente que decide **cómo se almacenan y recuperan los datos físicamente en disco**. MySQL permite elegir el engine por tabla.

Un engine por tanto puede tener o no estas características
- **Ser transaccional**: agrupa operaciones para que sean todo-o-nada siguiendo el prinicpio ACID (transacciones Atómicas, Consistentes, Aisladas y Duraderas)
- **Tener journaling**: antre crasheos guarda un log dónde almacena las operaciones y así evita perder datos
- **Foreign Keys:** Soportar referencias y relaciones entre tablas
- **Concurrencia**: bloquear solo las filas modificadas durante la escritura y no bloquear escrituras cuando se hagan lecturas (MVCC)

Por tanto, tenemos estos distintos motores:

| Engine        | Uso típico                                                                                                                                                                                                                                                   |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **InoDB**     | Cuenta con todas las características de concurrencia, transacciones o foreign keys. Siempre recomendable                                                                                                                                                     |
| **MyISAM**    | Es otro engine, pero no es transaccional ni soporta claves foraneas, por tanto puede dejar los datos en estado inconsistente tras un crash. Solo recomendable para tablas de solo lectura o logs históricos donde el rendimiento de lectura sea prioritario. |
| **MEMORY**    | Tablas en RAM; muy rápidas pero se pierden al reiniciar. Para cachés temporales                                                                                                                                                                              |
| **ARCHIVE**   | Tablas de solo escritura/lectura comprimidas. Para logs históricos                                                                                                                                                                                           |
| **CSV**       | Almacena datos como archivos CSV en disco. Para exportación                                                                                                                                                                                                  |
| **BLACKHOLE** | Acepta escrituras pero no guarda nada. Para replicación                                                                                                                                                                                                      |

---

## 1.4. 📊 Tipos de datos

#### Atributos especiales
- **PRIMARY KEY**: Identifica de forma única cada fila. No puede ser NULL ni repetirse
- **NOT NULL**: No puede ser nulo
- **FOREIGN KEY** : Referencia a la PRIMARY KEY de otra tabla — crea la relación entre tablas
- **UNIQUE** : Los valores deben ser únicos en la columna, pero puede ser NULL
- **INDEX** : Estructura auxiliar para acelerar las búsquedas (no garantiza unicidad)

-------
#### Numéricos

| Tipo              | Bytes    | Rango (sin signo)   | Uso típico                                                  |
| ----------------- | -------- | ------------------- | ----------------------------------------------------------- |
| `TINYINT`         | 1        | 0 → 255             | Flags booleanos, estados simples                            |
| `SMALLINT`        | 2        | 0 → 65.535          | Contadores pequeños                                         |
| `MEDIUMINT`       | 3        | 0 → 16.777.215      | —                                                           |
| `INT` / `INTEGER` | 4        | 0 → 4.294.967.295   | IDs, contadores generales                                   |
| `BIGINT`          | 8        | 0 → 18,4 × 10¹⁸     | IDs de alto volumen, timestamps Unix                        |
| `DECIMAL(p,s)`    | Variable | Exacto              | Dinero, precios (evita errores de float)                    |
| `FLOAT`           | 4        | Aprox. ±3,4 × 10³⁸  | Valores científicos donde la precisión exacta no es crítica |
| `DOUBLE`          | 8        | Aprox. ±1,8 × 10³⁰⁸ | Mayor precisión que FLOAT                                   |

> ⚠️ Nunca usar `FLOAT`/`DOUBLE` para dinero Los números en coma flotante tienen errores de redondeo (`0.1 + 0.2 ≠ 0.3` en binario). Para valores monetarios usar siempre `DECIMAL(10,2)`.

----
#### Texto

|Tipo|Tamaño máximo|Descripción|
|---|---|---|
|`CHAR(n)`|255 chars|Longitud fija. Rellena con espacios. Más rápido para longitudes constantes (ej: códigos)|
|`VARCHAR(n)`|65.535 bytes|Longitud variable. Solo usa el espacio necesario. El más común|
|`TEXT`|65.535 bytes|Texto largo. No se puede indexar completo|
|`MEDIUMTEXT`|16 MB|Para textos muy largos|
|`LONGTEXT`|4 GB|Para documentos enteros|
|`ENUM('a','b')`|—|Solo acepta uno de los valores definidos. Eficiente en espacio|
|`SET('a','b')`|—|Puede contener uno o varios valores del conjunto|

----
#### Fecha y hora

|Tipo|Rango|Descripción|
|---|---|---|
|`DATE`|1000-01-01 → 9999-12-31|Solo fecha: `2024-01-15`|
|`TIME`|-838:59:59 → 838:59:59|Solo hora: `14:30:00`|
|`DATETIME`|1000-01-01 → 9999-12-31|Fecha y hora: `2024-01-15 14:30:00`|
|`TIMESTAMP`|1970-01-01 → 2038-01-19|Como DATETIME pero se guarda en UTC y se convierte a la zona horaria local. Se actualiza automáticamente|
|`YEAR`|1901 → 2155|Solo el año|

> [!NOTE]
>  **DATETIME vs TIMESTAMP**:  `TIMESTAMP` se almacena en UTC y cambia según la zona horaria de la sesión. `DATETIME` guarda el valor tal cual, sin conversión. Para registros de auditoría donde la consistencia entre zonas horarias importa, usar `TIMESTAMP`. Para fechas de negocio (cumpleaños, fechas de contrato) donde no debe cambiar con la zona horaria, usar `DATETIME`.

----
#### Binarios y JSON

|Tipo|Descripción|
|---|---|
|`BLOB`, `MEDIUMBLOB`, `LONGBLOB`|Datos binarios: imágenes, archivos (aunque generalmente es mejor guardarlos en disco y solo la ruta en la BD)|
|`JSON`|Desde MySQL 5.7. Almacena JSON validado con acceso por ruta (`$.campo`)|

----
## 1.5.🔐 Instalación, Usuarios y permisos

Mysql se instala con
```
sudo apt install mariadb-client mariadb-server
sudo service mysql start
```

MySQL tiene su propio sistema de usuarios y permisos, independiente del SO. Se identifican por este formato:  `nombre@host`

`'jessica'@'localhost'` (solo para el sistema local) es distinto de `'jessica'@'%'` (para todos los sistemas)

Los privilegios son estos:

| Privilegio                     | Alcance               | Descripción                                                     |
| ------------------------------ | --------------------- | --------------------------------------------------------------- |
| `ALL PRIVILEGES`               | Global                | Todos los privilegios (equivale a root)                         |
| `SELECT`                       | Tabla                 | Leer datos                                                      |
| `INSERT` / `UPDATE` / `DELETE` | Tabla                 | Insertar / modificar / eliminar filas                           |
| `CREATE` / `DROP`              | Base de datos / Tabla | Crear / eliminar bases de datos o tablas                        |
| `ALTER` / `INDEX`              | Tabla                 | Modificar la estructura de una tabla / Crear y eliminar índices |
| `GRANT OPTION`                 | —                     | Puede otorgar sus propios privilegios a otros usuarios          |
| `SUPER`                        | Global                | Operaciones administrativas avanzadas                           |

> [!CAUTION]
> - **Usuario root sin contraseña:** En instalaciones por defecto o sin ejecutar `mysql_secure_installation`, el usuario root puede no tener contraseña. Siempre ejecutar `mysql_secure_installation` tras instalar.
> - **Usuario con `'%'` como host y privilegios amplios**: Un usuario definido como `'jessica'@'%'` puede conectarse desde cualquier IP del mundo. Limitar siempre el host al mínimo necesario.

---------
#### Gestión:
Por tanto, tenemos estos comandos de gestión:

| Acción                                    | Comando                                                     |
| ----------------------------------------- | ----------------------------------------------------------- |
| Ver usuarios existentes                   | `SELECT user, host FROM mysql.user;`                        |
| Crear usuario para todos los hosts        | `CREATE USER 'jessica'@'%' IDENTIFIED BY 'pass'; `          |
| Darle todos los privilegios para el admin | `GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost'; `      |
| Darle un privilegio sobre una tabla       | `GRANT SELECT, INSERT ON tienda.clientes TO 'jessica'@'%';` |
| Ver permisos de un usuario                | `SHOW GRANTS FOR 'jessica'@'%';`                            |
| Revocar privilegios                       | `REVOKE ALL PRIVILEGES ON tienda.* FROM 'jessica'@'%';`     |
| Aplicar cambios                           | `FLUSH PRIVILEGES;`                                         |
| Cambiar contraseña                        | `ALTER USER 'jessica'@'%' IDENTIFIED BY 'pass123';`         |
| Eliminar usuario                          | `DROP USER 'jessica'@'%';`                                  |

--------
# 2. 💻 Cheatsheet de comandos

## 2.1. Conexión y acceso

| Acción                                            | Comando                                              |
| ------------------------------------------------- | ---------------------------------------------------- |
| Conectar al servidor local                        | `mysql -u root -p`                                   |
| Conectar a servidor remoto                        | `mysql -u jessica -p -h 192.168.1.50 -P 3306`        |
| Conectar y seleccionar base de datos directamente | `mysql -u jessica -p tienda`                         |
| Ejecutar un comando aislado                       | `mysql -u jessica -p -e "SHOW DATABASES;"`           |

## 2.2. Creación y listado

| Acción                                       | Comando                                                   |
| -------------------------------------------- | --------------------------------------------------------- |
| Listar bases de datos                        | `SHOW DATABASES;`                                         |
| Ver base de datos actual                     | `SELECT DATABASE();`                                      |
| Eliminar base de datos                       | `DROP DATABASE tienda;`                                   |
| Listar tablas de una base de datos           | `SHOW TABLES FROM tienda;`                                |
| Ver estructura de una tabla                  | `DESCRIBE clientes;` / `SHOW COLUMNS FROM clientes;`      |
| Ver la sentencia CREATE de una base de datos | `SHOW CREATE DATABASE tienda;`                            |
| Ver la sentencia CREATE de una tabla         | `SHOW CREATE TABLE clientes;`                             |
| Renombrar tabla                              | `RENAME TABLE clientes TO customers;`                     |
| Eliminar tabla                               | `DROP TABLE pedidos;`                                     |
| Vaciar tabla (mantiene la estructura)        | `TRUNCATE TABLE pedidos;`                                 |
| Copiar estructura de una tabla               | `CREATE TABLE clientes_backup LIKE clientes;`             |
| Copiar estructura y datos                    | `CREATE TABLE clientes_backup AS SELECT * FROM clientes;` |

```sql
-- Crear base de datos y entrar en ella
CREATE DATABASE tienda;
USE tienda;
 
-- CREATE TABLE nombre (columna1 tipo(bytes), columna2 tipo(bytes), columnaN tipo(bytes)); 
-- Crear tabla
CREATE TABLE clientes (
    id         INT          NOT NULL AUTO_INCREMENT,
    nombre     VARCHAR(100) NOT NULL,
    email      VARCHAR(150) NOT NULL UNIQUE,
    edad       TINYINT      UNSIGNED,
    activo     TINYINT(1)   DEFAULT 1,
    fecha_alta DATE         DEFAULT (CURRENT_DATE),
    PRIMARY KEY (id)
);

-- Crear tabla con foreign key
CREATE TABLE pedidos (
    id          INT  NOT NULL AUTO_INCREMENT,
    cliente_id  INT  NOT NULL,
    total       DECIMAL(10,2) NOT NULL,
    fecha       DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    FOREIGN KEY (cliente_id) REFERENCES clientes(id) ON DELETE CASCADE
);
```


Podemos crear además un archivo `.sql` con las instrucciones para crear la tabla
```SQL
sql> create database star_wars; use star_wars;
sql> source /media/shared/sw_characters.sql
sql> CREATE USER 'Paco'@'localhost' IDENTIFIED BY 'pass123';
sql> grant select on star_wars.* to 'Paco'@'localhost';
```

Tambien tenemos el comando **mysqldump**:
```bash
sudo mysqldump prueba > ~/Documentos/ejercicios.sql # exportar
sudo mysqldump prueba < ~/Documentos/ejercicios.sql # importar
```


---
## 2.3. Consultar datos

#### SELECT — consultar datos

```sql
-- Básico
SELECT * FROM clientes;
SELECT nombre, email FROM clientes;

-- Con condición
SELECT * FROM clientes WHERE activo = 1;
SELECT * FROM clientes WHERE edad > 25 AND activo = 1; -- dos valores
SELECT * FROM clientes WHERE nombre LIKE 'J%';       -- empieza por J
SELECT * FROM clientes WHERE email LIKE '%@gmail%';  -- contiene @gmail
SELECT * FROM clientes WHERE edad BETWEEN 20 AND 30; -- edad entre 20 y 30 años
SELECT * FROM clientes WHERE edad IN (20, 25, 30); -- edad que sean 20. 25 o 30 años
SELECT * FROM clientes WHERE telefono IS NOT NULL;
SELECT concat(nombre,0x3a,email) FROM clientes; -- formatear: nombre:email

-- Ordenar
SELECT * FROM clientes ORDER BY nombre ASC; -- El nombre por orden alfabético
SELECT * FROM clientes ORDER BY fecha_alta DESC; -- La fecha de alta desde las mas reciente
SELECT * FROM clientes ORDER BY nombre ASC, edad DESC; -- Juan 23, Juan 19, Lucas 30

-- Limitar resultados
SELECT * FROM clientes LIMIT 10;
SELECT * FROM clientes LIMIT 20, 10;  -- por cual empieza (offset), cuantos resultados

-- Eliminar duplicados
SELECT DISTINCT ciudad FROM clientes; -- Madrid, Barcelona y Cadiz

-- Alias de columna y tabla (renombrado temporal de tablas y columnas)
SELECT nombre AS cliente, email AS correo FROM clientes AS c;
```

Funciones:

| Acción                          | Comando                                                              |
| ------------------------------- | -------------------------------------------------------------------- |
| Promedio `AVG`                  | `SELECT AVG(sueldo) FROM plantilla;` 🡆 800 €                        |
| Número de resultados `COUNT`    | `SELECT count(ciudad) FROM clientes;` 🡆 3 ciudades diferentes       |
| Resultados distintos `DISTINCT` | `SELECT DISTINCT ciudad FROM clientes;` 🡆 Madrid, Cadiz y Barcelona |
| Máximo `MAX()`, mínimo `MIN()`  | `SELECT MAX(edad) FROM plantilla;`🡆 56 años                         |
| Suma `SUM`                      | `SELECT SUM(ganancias) FROM pedidos;` 🡆 21.000 €                    |

----
#### Subconsultas
Permite hacer una primera consulta para obtener un dato que usar como filtro para una segunda consulta (que puede ser en la misma tabla o en otra). Es como un filtro inteligente.

```sql
-- Ej 1. Muestra el nombre de los que tengan edad por encima del promedio
-- Edad promedio = 35, Query 🡆 SELECT nombre FROM clientes WHERE edad > 35;
SELECT nombre FROM clientes WHERE edad > (SELECT AVG(edad) FROM clientes);

-- Ej 2. Muestra los datos de los clientes que tengan mas de 100 pedidos en la tabla pedidos
SELECT * FROM clientes WHERE id IN (SELECT cliente_id FROM pedidos WHERE total > 100);

-- Ej 3: Queremos hacer la media de la suma de ganacias de todas las empresas por sector
-- 1. Creamos una tabla virtual con la suma de ganancias ordenadas por sector (2 columnas)
-- 2. Hacemos una media sobre esa columna de esa tabla virtual
SELECT AVG(ganancias)
    FROM (SELECT sector, SUM(revenue) AS ganancias FROM fortune GROUP BY sector) 
AS tabla1 GROUP BY sector;
```

-------
#### Case When
Permite crear una nueva columna basandonos en características

```sql
-- De la tabla star_wars_characters 
-- Seleccionar nombre, planeta natal, color de ojos y altura
-- Nueva columna “tamaño” 🡆 “grande” o “pequeño” si supera o no los 2 metros
-- Solo mostrar los que sean de los mundos Chandrila, Stewjon o Tatooine
-- Por último ordenar por nombre en orden alfabético

SELECT name, homeworld, eye_color, height, CASE
  WHEN height>=200 then 'Grande' ELSE 'Pequeño' END as tamaño
  WHEN species='Gungan' OR eye_color='yellow' THEN 'Lord_Sith' ELSE 'Normal' END AS rol
FROM star_wars_characters_2 
   WHERE homeworld IN ('Chandrila','Stewjon','Tatooine')
ORDER BY name;
```


---
#### Agregaciones
Las funciones de agregación sirven para convertir **muchas filas en un resultado resumido**.
```sql
-- ¿Cuántos clientes tengo por ciudad?
SELECT ciudad, COUNT(*) FROM clientes GROUP BY ciudad; -- 

-- ¿Cuántos clientes tengo por ciudad quitando las ciudades con menos de 5 clientes?
SELECT ciudad, COUNT(*) AS total FROM clientes GROUP BY ciudad HAVING COUNT(*) > 5;

SELECT ciudad, COUNT(*) AS total, AVG(edad) AS media_edad -- columnas: ciudad, total y media_edad
   FROM clientes WHERE activo = 1 -- de la tabla clientes, pero tomando solo los activos
   GROUP BY ciudad HAVING COUNT(*) > 5 -- pero quitando las ciudades con menos de 5 clientes
ORDER BY total DESC;

-- Crea una tabla temporal con el conteo de sesiones por dispositivo
-- Dentro de cada grupo, suma sessions solamente cuando el dispositivo sea X
SELECT 
	channelgrouping as Canal,
    sum(CASE WHEN device = 'mobile' THEN sessions end) AS 'mobile_sesions',
    sum(CASE WHEN device = 'desktop' THEN sessions end) AS 'desktop_sesions',
    sum(CASE WHEN device = 'tablet' THEN sessions end) AS 'tablet_sesions',
    sum(sessions) AS total
FROM google_analytics where strftime('%Y-%m', date)='2019-10'
GROUP BY channelgrouping
```


---
#### JOINs — combinar tablas
Tenemos estas dos tablas
```
Clientes              Pedidos
┌────┬────────┐       ┌────┬────────────┬───────┐
│ id │ nombre │       │ id │ cliente_id │ total │
├────┼────────┤       ├────┼────────────┼───────┤
│ 1  │ Ana    │       │ 10 │ 1          │ 200   │
│ 2  │ Luis   │       │ 11 │ 1          │ 50    │
│ 3  │ Marta  │       │ 12 │ 2          │ 300   │
└────┴────────┘       └────┴────────────┴───────┘
```

Por tanto queremos combinarlas
```sql
-- INNER JOIN — solo filas que tienen coincidencia en ambas tablas
SELECT c.nombre, p.total FROM clientes c
INNER JOIN pedidos p ON c.id = p.cliente_id;
-- Ana     200
-- Ana      50
-- Luis    300
-- Marta no porque no ha pedido nada

-- LEFT JOIN — todas las filas de la izquierda, con NULL donde no hay coincidencia
SELECT c.nombre, p.total FROM clientes c
LEFT JOIN pedidos p ON c.id = p.cliente_id;
-- Ana     200
-- Ana      50
-- Luis    300
-- Marta   NULL

-- Si queremos quitar a Marta
SELECT c.nombre FROM clientes c LEFT JOIN pedidos p ON c.id = p.cliente_id WHERE p.id IS NULL;

-- RIGHT JOIN — todas las filas de la derecha
FROM clientes c RIGHT JOIN pedidos p ON c.id = p.cliente_id;
-- Conservamos todas las filas de pedidos, aunque no exista cliente correspondiente.

SELECT c.nombre, p.total, pr.nombre AS producto FROM clientes c 
  INNER JOIN pedidos p ON c.id = p.cliente_id
  INNER JOIN productos pr ON p.producto_id = pr.id;

-- cliente    total    producto
----------------------------
-- Ana        200      iPhone
-- Ana         50      Cable USB
-- Luis       300      Monitor
```


---
## 2.4. Modificación

#### INSERT — insertar datos

```sql
-- Insertar fila/filas en orden (nombre, email, edad)
INSERT INTO clientes VALUES ('Jessica', 'jessica@empresa.com', 28),

-- Insertar fila/filas en orden explicito
INSERT INTO clientes (nombre, email, edad) VALUES
    ('Jessica', 'jessica@empresa.com', 28),
    ('Carlos',  'carlos@empresa.com',  35),
    ('Ana',     'ana@empresa.com',     22);

-- Insertar desde otra tabla
INSERT INTO clientes_backup SELECT * FROM clientes WHERE activo = 1;
```

---
#### ALTER — Modificar estructura de tablas
```sql
-- Añadir columna / indice / clave foranea
ALTER TABLE clientes ADD COLUMN telefono VARCHAR(20);
ALTER TABLE clientes ADD COLUMN telefono VARCHAR(20) AFTER email;
ALTER TABLE clientes ADD COLUMN telefono VARCHAR(20) FIRST;
ALTER TABLE clientes ADD UNIQUE INDEX idx_email (email);
ALTER TABLE pedidos ADD CONSTRAINT fk_cliente 
            FOREIGN KEY (cliente_id) REFERENCES clientes(id);

-- Eliminar columna / indice / clave foranea
ALTER TABLE clientes DROP COLUMN telefono;
ALTER TABLE clientes DROP INDEX idx_email;
ALTER TABLE pedidos DROP FOREIGN KEY fk_cliente;

-- Modificar tipo o atributos de una columna
ALTER TABLE clientes RENAME COLUMN nombre TO nombre_completo;
ALTER TABLE clientes MODIFY COLUMN nombre_completo VARCHAR(200) NOT NULL;

-- Cambiar el storage engine
ALTER TABLE clientes ENGINE = InnoDB;
```

--------
#### UPDATE — Actualizar datos
Siempre incluir `WHERE` en `UPDATE` para que no modifique TODAS las filas

```sql
-- Actualizar columna/columnas
UPDATE clientes SET nombre='Jessica García', email='jgarcia@empresa.com' WHERE id=1;
UPDATE clientes SET edad=edad + 1 WHERE activo=1;
```

#### DELETE — eliminar datos
Siempre incluir `WHERE` en `DELETE` para que NO BORRE TODAS LAS FILAS

```sql
-- Eliminar filas concretas
DELETE FROM clientes WHERE id = 5;
DELETE FROM clientes WHERE activo = 0 AND fecha_alta < '2020-01-01';

-- Eliminar todos los registros (mantiene la tabla)
DELETE FROM clientes;         -- borra fila a fila, lento, no resetea AUTO_INCREMENT
TRUNCATE TABLE clientes;      -- borra el contenido completo, rápido, resetea AUTO_INCREMENT
```

#### Transacciones

```sql
BEGIN; -- Iniciar transacción
INSERT INTO pedidos (cliente_id, total) VALUES (1, 150.00); -- Operar en la transacción
SAVEPOINT sp1; -- Crear un punto de guardado
ROLLBACK TO sp1; -- Deshacer (revierte todo desde el savepoint)
COMMIT; -- Confirmar (hace las operaciones permanentes)
```


---
# 4. Gestión

Podemos ver la información del servidor,

| Acción                              | Comando                                          |
| ----------------------------------- | ------------------------------------------------ |
| Versión del servidor                | `SELECT VERSION();`                              |
| Conexión actual                     | `SELECT USER(), DATABASE(), CONNECTION_ID();`    |
| Motor de almacenamiento por defecto | `SHOW ENGINES;`                                  |
| Estadísticas de una tabla           | `SHOW TABLE STATUS FROM tienda LIKE 'clientes';` |
| Ver el log de errores               | `SHOW VARIABLES LIKE 'log_error';`               |

---
## 4.1. 📁 Archivos y rutas importantes

```
/etc/mysql/mysql.conf.d/mysqld.cnf    ← configuración principal (Debian/Ubuntu)
/etc/my.cnf                           ← configuración principal (CentOS/RHEL)
/etc/mysql/my.cnf                     ← configuración alternativa
/var/lib/mysql/                       ← datos (bases de datos, tablas, logs)
/var/lib/mysql/<base_de_datos>/       ← archivos de cada base de datos
/var/log/mysql/error.log              ← log de errores
/var/log/mysql/mysql.log              ← log general de queries (si está activado)
```

### Configuración básica (`my.cnf` / `mysqld.cnf`)

```bash
[mysqld]
port            = 3306          # Puerto 
bind-address    = 127.0.0.1     # solo acepta conexiones locales (más seguro que 0.0.0.0)

character-set-server  = utf8mb4 # Charset por defecto
collation-server      = utf8mb4_unicode_ci

innodb_buffer_pool_size = 1G # Tamaño del buffer pool de InnoDB (ideal: 50-70% de la RAM)
max_connections = 150        # Máximo de conexiones simultáneas

# Logs
general_log         = 0         # log de todas las queries (caro en producción)
general_log_file    = /var/log/mysql/mysql.log
slow_query_log      = 1         # log de queries lentas
slow_query_log_file = /var/log/mysql/slow.log
long_query_time     = 2         # queries que tardan más de 2 segundos
```


> [!CAUTION] 
> `LOAD DATA INFILE` / `INTO OUTFILE` Permite leer y escribir archivos del sistema desde SQL. Si un atacante tiene acceso a MySQL, puede leer `/etc/passwd` o escribir webshells.

```sql
-- Leer archivo del sistema (si el usuario tiene FILE privilege)
SELECT LOAD_FILE('/etc/passwd');

-- Escribir archivo en disco (webshell)
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';
```

> Deshabilitar con `secure_file_priv = /tmp` en `my.cnf` (restringe a ese directorio) o `secure_file_priv = ""` para deshabilitar completamente.
