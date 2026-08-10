> **Microsoft SQL Server** es el RDBMS de Microsoft. A diferencia de MySQL, está pensado para entornos enterprise Windows con integración nativa en Active Directory, y usa **T-SQL (Transact-SQL)**, una extensión propietaria de SQL con funciones y variables propias. **Es el componente de base de datos más frecuente en entornos corporativos Windows y un objetivo habitual en pentesting de AD.**

Utiliza el protocolo 1433/TCP y permite múltiples instancias por servidor. Como storage engine utiliza el propio motor de SQL server

---
# 1. 🗄️ ¿Cómo funciona MSSQL?

## 1.1. 🏛️ Instancias
Una de las diferencias más importantes con MySQL: un servidor Windows puede alojar **múltiples instancias** de SQL Server, cada una con su propio nombre, puerto y configuración independiente.

El servicio **SQL Server Browser** (UDP 1434) resuelve el nombre de instancia al puerto real (igual que hace rpcbind/portmapper para RPC)
```
  SERVIDOR (Windows)
  ├── Instancia por defecto (servidor\MSSQLSERVER)  → puerto 1433 
  ├── Instancia nombrada (servidor\DESARROLLO)      → puerto dinámico (SQL Browser)
  └── Instancia nombrada (servidor\PRODUCCION)      → puerto fijo configurado manualmente
```

---
## 1.1. 🔐 Autenticación — dos modos
MSSQL tiene dos modos de autenticación que pueden coexistir:

- **Windows Authentication (recomendado):** Usa las credenciales de Active Directory o del sistema local. No requiere contraseña separada para SQL Server. El token de Kerberos o NTLM del usuario de Windows se usa directamente (SSO)
- **Login y contraseña propios de SQL Server**, independientes de AD. Necesario para el acceso desde fuera del dominio o desde Linux aparte de la cuenta `sa` (System Administrator, es el equivalente a root de MySQL)

Para configurar el modo desde `SSMS`
```
Clic derecho en el servidor 🡆 Properties 🡆 Security 🡆 Windows Authentication mode 🡆 SQL Server and Windows Authentication mode (modo mixto)
```

Además, tenemos estos dos niveles: por un lado está el login, que sirve de manera global, y por otro lado los usuarios, que se crean a nivel de tabla por cada login.

Desde linea de comandos

| Acción                             | Comando                                              |
| ---------------------------------- | ---------------------------------------------------- |
| Crear login de servidor SQL        | `CREATE LOGIN jessica WITH PASSWORD = 'pass123';`    |
| Crear login de Windows             | `CREATE LOGIN [EMPRESA\jessica] FROM WINDOWS;`       |
| Crear usuario a partir de un login | `USE tienda; CREATE USER jessica FOR LOGIN jessica;` |

Además, podemos impersonar a un usuario si tenemos suficientes permisos
```sql
EXECUTE AS LOGIN = 'sa'; EXEC xp_cmdshell 'whoami'; REVERT; 
```

---
## 1.2. 🏗️ Estructura interna

#### Bases de datos del sistema
MSSQL incluye cuatro bases de datos de sistema que no deben modificarse:

| Base de datos | Descripción                                                                                                                                   |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `master`      | Configuración global del servidor, logins, bases de datos, configuración de instancia. La más crítica, si se corrompe, el servidor no arranca |
| `model`       | Plantilla para nuevas bases de datos. Cualquier objeto creado aquí aparecerá en las nuevas BDs                                                |
| `msdb`        | Datos del Agente SQL (jobs, alertas, historial de backups)                                                                                    |
| `tempdb`      | Base de datos temporal que se recrea en cada reinicio. Tablas temporales, operaciones de ordenación, row versioning                           |

----
#### Esquemas (schemas)
MSSQL introduce el concepto de **schema** como capa adicional de organización dentro de una base de datos:

```
  Base de datos: tienda
  ├── Schema: dbo (default)
  │   ├── dbo.clientes
  │   └── dbo.pedidos
  ├── Schema: ventas
  │   ├── ventas.facturas
  │   └── ventas.descuentos
  └── Schema: rrhh
      └── rrhh.empleados
```

El schema por defecto es `dbo` (database owner). Los objetos se referencian como `schema.tabla` o `basededatos.schema.tabla`.

---
#### Linked Servers
MSSQL permite conectar instancias entre sí o con otras fuentes de datos (Oracle, MySQL, Excel...) mediante **Linked Servers**. Las queries pueden cruzar servidores:

```sql
-- Query a un servidor enlazado
SELECT * FROM SERVIDOR_REMOTO.tienda.dbo.clientes;
```

> [!CAUTION]
> Si se compromete una instancia con un Linked Server configurado hacia otro servidor, se puede ejecutar código remoto en el servidor enlazado. Es un vector habitual de movimiento lateral en entornos SQL Server.

---
## 1.3. Roles y permisos

```sql
-- CONCEDER permisos: leer, insertar y modificar en el esquema dbo
GRANT SELECT, INSERT, UPDATE ON SCHEMA::dbo TO jessica;

-- ASIGNAR un rol de base de datos
ALTER ROLE db_datareader ADD MEMBER jessica;

-- COMPROBAR si es sysadmin: da 0, osea, no tiene ese rol
SELECT IS_SRVROLEMEMBER('sysadmin');

-- QUITAR un permiso concreto: ❌ FALLA porque llegando desde SCHEMA::dbo.
REVOKE UPDATE ON dbo.clientes FROM jessica;

-- PROHIBIR explícitamente un permiso ✅ Si bloquea el permiso
DENY SELECT ON dbo.clientes TO jessica;
```

Tenemos estos roles tanto para el servidor como para las bases de datos (`db_`)

| Rol                | Significado                                                                    |
| ------------------ | ------------------------------------------------------------------------------ |
| `sysadmin`         | **Control total** del servidor con permisos de administrador                   |
| `securityadmin`    | Puede modificar seguridad / permisos                                           |
| `dbcreator`        | Puede **crear, modificar, eliminar y restaurar bases de datos**.               |
| `bulkadmin`        | Puede ejecutar operaciones `BULK INSERT`.                                      |
| `db_owner`         | **Control total sobre una base de datos**.                                     |
| `db_securityadmin` | Puede modificar la pertenencia a roles y permisos de la base de datos.         |
| `db_ddladmin`      | Puede ejecutar comandos DDL (`CREATE`, `ALTER`, `DROP`, etc.).                 |
| `db_datareader`    | Puede hacer `SELECT` sobre **todas las tablas y vistas**.                      |
| `db_datawriter`    | Puede hacer `INSERT`, `UPDATE` y `DELETE` sobre **todas las tablas y vistas**. |

## 1.4. 📁 Archivos importantes

```
C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\
        DATA\
            master.mdf          ← base de datos master
            master.ldf          ← log de transacciones de master
            model.mdf
            msdb.mdf
            tempdb.mdf
        Log\
            ERRORLOG            ← log de errores del servidor
            ERRORLOG.1, .2...   ← logs rotados
        Backup\                 ← backups automáticos

C:\Windows\System32\sqlcmd.exe                  ← cliente CLI

%ProgramFiles%\Microsoft SQL Server Management Studio XX\  ← SSMS
```

---
# 2. 💻 Cheatsheet de comandos MSSQL

## 2.1. Conexión y acceso

```bash
# CLI desde Linux (sqlcmd de mssql-tools)
sqlcmd -S 192.168.1.50 -U jessica -P contraseña
sqlcmd -S 192.168.1.50\INSTANCIA -U jessica -P contraseña

# Con autenticación Windows (desde Windows)
sqlcmd -S servidor -E

# Ejecutar query directa sin entrar al cliente interactivo
sqlcmd -S servidor -U sa -P pass -Q "SELECT name FROM sys.databases"

# Desde Linux con Impacket (pentesting)
Impacket-mssqlclient jessica:contraseña@192.168.1.50 -windows-auth  # auth Windows
```

> En `sqlcmd`, los comandos terminan con `GO` en lugar de `;`. `GO` no es T-SQL sino una directiva del cliente que separa lotes de instrucciones.

---
## 2.2. Gestión básica

#### Bases de datos y esquemas

| Acción                          | Comando                                                  |
| ------------------------------- | -------------------------------------------------------- |
| Listar bases de datos           | `SELECT name FROM sys.databases;` / `EXEC sp_databases;` |
| Seleccionar base de datos       | `USE tienda;`                                            |
| Base de datos actual            | `SELECT DB_NAME();`                                      |
| Eliminar base de datos          | `DROP DATABASE tienda;`                                  |
| Listar esquemas                 | `SELECT name FROM sys.schemas;`                          |
| Crear esquema                   | `CREATE SCHEMA ventas;`                                  |
| Cambiar el esquema de una tabla | `ALTER SCHEMA ventas TRANSFER dbo.facturas;`             |

Crear base de datos
```sql
CREATE DATABASE tienda
    ON PRIMARY (NAME = 'tienda', FILENAME = 'C:\Data\tienda.mdf', SIZE = 100MB)
    LOG ON (NAME = 'tienda_log', FILENAME = 'C:\Data\tienda.ldf', SIZE = 20MB);
```

---
#### Tablas

```sql
-- Listar tablas de la BD actual
SELECT name FROM sys.tables;

-- Crear tabla
CREATE TABLE clientes (
    id       INT          IDENTITY(1,1) NOT NULL,
    nombre   VARCHAR(100) NOT NULL,
    email    VARCHAR(150) NOT NULL,
    fecha_alta DATETIME   DEFAULT GETDATE(),
    CONSTRAINT PK_clientes PRIMARY KEY (id),
    CONSTRAINT UQ_email UNIQUE (email)
);

-- Ver estructura
EXEC sp_help 'clientes';
EXEC sp_columns 'clientes';
```

---
#### Vistas del sistema (equivalentes a INFORMATION_SCHEMA)

| Acción                       | Comando                                                              |
| ---------------------------- | -------------------------------------------------------------------- |
| Listar bases de datos        | `SELECT * FROM sys.databases;`                                       |
| Tablas                       | `SELECT * FROM sys.tables;`                                          |
| Columnas                     | `SELECT * FROM sys.columns WHERE object_id = OBJECT_ID('clientes');` |
| Usuarios de la base de datos | `SELECT name, type_desc FROM sys.database_principals;`               |
| Permisos                     | `SELECT * FROM sys.database_permissions;`                            |


---
## 2.3. ⚡ Ejecución de comandos del SO

`xp_cmdshell` es un procedimiento almacenado que ejecuta comandos del sistema operativo directamente desde SQL Server. Está **deshabilitado por defecto** desde SQL Server 2005.

```sql
-- Habilitar xp_cmdshell (requiere sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

-- Ejecutar comandos del SO
EXEC xp_cmdshell 'whoami';

-- Deshabilitar
EXEC sp_configure 'xp_cmdshell', 0; RECONFIGURE;
```

> [!CAUTION] 
> Si se consigue acceso a SQL Server con privilegios de `sysadmin`, habilitar `xp_cmdshell` permite ejecutar comandos como la cuenta de servicio de SQL Server, que habitualmente tiene privilegios elevados en el sistema. 

## 2.4. 📝 T-SQL — Transact-SQL
MSSQL tiene el lenguaje Transact-SQL, algo diferente al SQL estándar

```sql
-- Declarar y asignar variables
DECLARE @nombre VARCHAR(100); SET @nombre = 'Jessica';
SELECT @nombre = nombre FROM clientes WHERE id = 1;
PRINT @nombre;

-- Cierto control de flujo y procedimientos
IF @edad >= 18
	BEGIN PRINT 'Mayor de edad'; END
ELSE
	BEGIN PRINT 'Menor de edad'; END

-- TOP en lugar de LIMIT
SELECT TOP 10 * FROM clientes;	

-- Creación de tablas temporales
CREATE TABLE #temp_global (id INT, nombre VARCHAR(100));
```

| MySQL                      | MSSQL T-SQL                    | Descripción                 |
| -------------------------- | ------------------------------ | --------------------------- |
| `CONCAT(a, b)`             | `a + b` o `CONCAT(a, b)`       | Concatenar cadenas          |
| `SUBSTRING(str, pos, len)` | `SUBSTRING(str, pos, len)`     | Igual                       |
| `LENGTH(str)`              | `LEN(str)`                     | Longitud de cadena          |
| `NOW()`                    | `GETDATE()`                    | Fecha y hora actual         |
| `CURDATE()`                | `CAST(GETDATE() AS DATE)`      | Fecha actual                |
| `LIMIT n`                  | `TOP n` (al inicio del SELECT) | Limitar filas               |
| `GROUP_CONCAT()`           | `STRING_AGG(col, sep)`         | Concatenar valores de grupo |
| `IFNULL(a, b)`             | `ISNULL(a, b)`                 | Valor alternativo si NULL   |
| `AUTO_INCREMENT`           | `IDENTITY(1,1)`                | Columna autoincremental     |


---
## 2.5. Ejecutar comandos en linked servers

```sql
-- Listar linked servers configurados
SELECT name, data_source FROM sys.servers WHERE is_linked = 1;

-- Ejecutar query en el servidor enlazado
SELECT * FROM SERVIDOR_REMOTO.master.dbo.sysdatabases;

-- Ejecutar xp_cmdshell en el servidor enlazado (si tiene permisos)
EXEC ('xp_cmdshell ''whoami''') AT SERVIDOR_REMOTO;
```


---
