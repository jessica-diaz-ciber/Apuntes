# 1. 💻 Básico

## 1.1. Redirecciones y operadores

| Comando              | Descripción                                                                                                                                                                              |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cmd1 \| cmd2`       | Pipe: pasa la salida de`cmd1`como entrada de`cmd2`                                                                                                                                       |
| `cmd > archivo.txt`  | Redirige la salida estándar a un archivo. Pero lo sobrescribe si existe                                                                                                                  |
| `cmd >> archivo.txt` | Añade el contenido de la salida al final del archivo sin sobrescribir                                                                                                                    |
| `cmd 2> errores.txt` | Redirige la salida de **error**(stderr) a un archivo                                                                                                                                     |
| `cmd 2>&1`           | Redirige stderr al mismo destino que stdout (fusiona salidas)                                                                                                                            |
| `cmd1 && cmd2`       | Ejecuta `cmd2` solo si `cmd1` tuvo éxito (exit code 0)                                                                                                                                   |
| `cmd1 \|\| cmd2`     | Ejecuta `cmd2` solo si `cmd1` falló (exit code distinto a 0)                                                                                                                             |
| `cmd1 $(cmd2)`       | Permite ejecutar un comando y usar su resultado como texto dentro de otro comando (ejecuta y remplaza por el resultado). <br>Para un `echo` solo se interpreta entre comillas dobles "". |
| `cmd1; cmd2`         | Ejecuta ambos comandos **siempre**, en secuencia                                                                                                                                         |
| `cmd1 \| xargs cmd2` | Aplica el segundo comando a todos los resultados                                                                                                                                         |
| `sponge archivo.txt` | Aplica la salida al mismo archivo                                                                                                                                                        |
**Referencia a directorios**
- `.` : Directorio actual
- `..` : Directorio padre

Luego tenemos las expresiones regulares

| Regex | Significado                                  | Ejemplo                                                                                                         |
| ----- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `*`   | Cualquier cadena de caracteres               | `cat *.txt` lee todos los archivos que acaben en “txt”                                                          |
| `?`   | Cero o una repetición del carácter anterior  | `grep "colou?r"` Filtra “color” y “colour”                                                                      |
| `+`   | Una o más repeticiones del carácter anterior | `grep "go+gle"` Filtra “gogle”, “google”, “gooogle”                                                             |
| `.`   | Un carácter cualquiera                       | `grep "b.t"` Filtra “bat”, “bit”, “but”…                                                                        |
| `^`   | Todo lo que empiece por                      | `grep "^root"` Filtra líneas que empiezan con “root”                                                            |
| `$`   | Todo lo que acabe por                        | `grep "sh$"` Filtra líneas que terminan en “sh” (ej: “bash”, “zsh”)                                             |
| `[]`  | Cualquier carácter dentro de los corchetes   | `grep "[bz]sh"` Filtra “bash” o “zsh”;  <br>`grep "[A-Z]"` Filtra mayúsculas                                    |
| `{}`  | Numero de repeticiones                       | `grep "[0-9]\{3\}"` Filtra números con exactamente 3 dígitos  <br>`grep "a\{2,4\}"` Filtra “aa”, “aaa” o “aaaa” |

Y por último las funciones, que nos ahorran tener que repetir muchos comandos:
```bash
curl_api(){ curl -s -H "Authorization: Bearer ey(...)" http://api$1"; }
curl_api /v1/users
```


## 1.2. Variables de entorno

Tanto en windows como en  Linux existen las variables de entorno. Estas son pares `clave=valor` que se almacenan en la consola, y que pueden ser leídas por cualquier programa, sirven para almacenar configuraciones o datos que sean útiles.

Estas se pueden consultar con el comando `env` para verlas todas o `echo` para ver una en concreto, siempre tienen un "$" delante. Por ejemplo la variable $PATH es muy importante, aunque existen otras como:

| Variable | Uso                                                                                                                                                                                               |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| HOME     | Directorio personal del usuario (/home/usuario)                                                                                                                                                   |
| USER     | Usuario actual (lo mismo que al hacer whoami)                                                                                                                                                     |
| PATH     | Serie de rutas separadas por ":" en las que el sistema busca los programas al invocarlos                                                                                                          |
| SHELL    | Shell actual                                                                                                                                                                                      |
| TERM     | El tipo de terminal, de ello depende: el número de colores, como interpretar teclas especiales, como mover el cursor, como hacer scroll y como limpiar la pantalla. Por defecto es xterm-256color |
| PWD      | Directorio actual, lo mismo que hacer "pwd"                                                                                                                                                       |
| LANG     | Idioma del sistema y conjunto de caracteres, por ejemplo para español es "LANG=es_ES.UTF-8"                                                                                                       |
| EDITOR   | Editor por defecto, puede ser nano, vim, nvim, emacs...etc                                                                                                                                        |
Se asignan con `export clave=valor; echo $clave # valor`. Admiten la sintaxis `$()`.


---
# 2. 📁 Gestión de ficheros

## 2.1. Listado

| Acción                          | CMD             |
| ------------------------------- | --------------- |
| Cambiar de directorio           | `cd <ruta>`     |
| Listado                         | `ls <ruta>`     |
| Listado detallado               | `ls -l <ruta>`  |
| Listar archivos ocultos         | `dir -a <ruta>` |
| Leer el contenido de un archivo | `cat <archivo>` |

> [!tip] Batcat
> Es una versión mejorada de cat que nos da resaltado de sintaxis

> [!tip] Lsd / exa
> Son versiones mejoradas de ls con iconos y colores

## 2.2. Crear y borrar archivos y carpetas

| Acción                             | CMD                  |
| ---------------------------------- | -------------------- |
| Crear una carpeta                  | `mkdir <carpeta>`    |
| Eliminar un archivo                | `rm <archivo>`       |
| Eliminar una carpeta vacía         | `rmdir <carpeta>`    |
| Eliminar una carpeta con contenido | `rm -rf <carpeta>`   |
| Vaciar una carpeta                 | `rm -rf <carpeta>/*` |

## 2.3. Copiar y mover ficheros

| Acción                         | CMD                          |
| ------------------------------ | ---------------------------- |
| Copiar un archivo              | `cp <origen> <destino>`      |
| Mover/renombrar un archivo     | `mv <origen> <destino>`      |
| Copiar/Mover todo el contenido | `cp/mv <origen>\* <destino>` |

## 2.4. Permisos

| Acción                                  | CMD                                                                             |
| --------------------------------------- | ------------------------------------------------------------------------------- |
| Cambiar un permiso                      | `chmod <permiso> <archivo>`. Ej `chmod +x ./programa` o `chmod 777 archivo.txt` |
| Cambiar propietario                     | `chown <usuario>:<grupo> <archivo>`                                             |
| Cambiar un permiso de manera recursiva  | `chmod <permiso> -R <carpeta>`                                                  |
| Cambiar propietario de manera recursiva | `chown <usuario>:<grupo> -R <archivo>`                                          |

---
# 3. Filtrado, propiedades y búsquedas

## 3.1. Búsquedas

🔍 `grep`: buscar patrones de texto

| Acción                                                  | CMD                                |
| ------------------------------------------------------- | ---------------------------------- |
| Buscar patrón en archivo                                | `grep "patrón" archivo`            |
| Ignorar mayúsculas/minúsculas                           | `grep -i "patrón" archivo`         |
| Mostrar líneas que NO coinciden                         | `grep -v "patrón" archivo`         |
| Buscar de forma recursiva en directorio                 | `grep -r "patrón" /ruta/`          |
| Mostrar número de línea del resultado                   | `grep -n "patrón" archivo`         |
| Mostrar solo el texto que coincide (no la línea entera) | `grep -o "patrón" archivo`         |
| Contar líneas que coinciden                             | `grep -c "patrón" archivo`         |
| Buscar palabra completa (no subcadenas)                 | `grep -w "root" /etc/passwd`       |
| Buscar mas de un patrón                                 | `grep -E "error\|warning" archivo` |
| Mostrar N líneas de contexto antes y después            | `grep -A 2 -B 2 "patrón" archivo`  |
| Buscar en múltiples archivos                            | `grep "patrón" *.log`              |
| Mostrar solo el nombre del archivo con coincidencias    | `grep -l "patrón" *.log`           |
| Combinar con pipe                                       | `cat archivo \| grep "patrón"`     |
> Podemos quitar las lineas comentadas de un archivo con `cat archivo.txt | grep -v "^#" | grep -v "^$"`

> [!tip] Ripgrep
> Ripgrep (`rg`) es una versión mejorada de grep para busquedas recursivas. Se instala con `sudo apt install ripgrep`
> Ponemos `rg -i <patrón>` y encuentra los archivos en los que está casi instantaneamente

> [!tip] FZF
> Este comando nos permite buscar tanto en nuestro historial (`Ctrol+R`) como en los archivos del directorio actual `Ctrol+T`.
> - Se activa poniendo en la .zshrc `source <(fzf --zsh)` o en la .bashrc `eval "$(fzf --bash)"`

🔍 `find`: búsquedas

| Acción                                            | CMD                                                                              |
| ------------------------------------------------- | -------------------------------------------------------------------------------- |
| Buscar por nombre exacto                          | `find /ruta -name "archivo.txt"` (`iname` case insensitive), (`"*.sh"` patrones) |
| Buscar solo archivos / directorios                | `find /ruta -type f/d`                                                           |
| Buscar archivos modificados en los últimos N días | `find /ruta -mtime -7`                                                           |
| Buscar archivos más grandes que N                 | `find /ruta -size +100M`                                                         |
| Buscar archivos con permisos exactos              | `find /ruta -perm 777`. Ej `find / -perm -4000 2>/dev/null` para los SUIDS       |
| Buscar archivos de un usuario/grupo concreto      | `find /ruta -user/-group usuario/grupo`                                          |
| Buscar y ejecutar comando (más eficiente)         | `find /ruta -name "*.log" -exec ls -lh {} +`                                     |
| Buscar y mostrar con detalle                      | `find /ruta -name "*.conf" -exec ls -lh {} \;`                                   |
| Limitar profundidad de búsqueda                   | `find /ruta -maxdepth 2 -name "*.txt"`                                           |

> [!tip] Fd-find
> Fdfind (`fd`) es una versión mejorada de find . Se instala con `apt install fd-find`. Su sintaxis es `fd <ruta> <archivo>`
> - Nos permite buscar por tipos de fichero `-e jpg`, solo ficheros `-f`, solo directorios `-d`

## 3.2. Transformaciones de texto

✂️ `cut`: extraer columnas o campos. Por defecto utiliza el tabulador como delimitador, pero se puede cambiar.

| Acción                                      | CMD                              |
| ------------------------------------------- | -------------------------------- |
| Extraer caracteres por posición             | `cut -c1-5 archivo`              |
| Extraer múltiples campos con delimitador    | `cut -d: -f1,3 /etc/passwd`      |
| Extraer desde el campo N hasta el final     | `cut -d: -f3- /etc/passwd`       |
| Extraer columnas de la salida de un comando | `cat /etc/passwd \| cut -d: -f1` |

✂️ `tr`: transformar o eliminar caracteres

| Acción                                                            | CMD                                     |
| ----------------------------------------------------------------- | --------------------------------------- |
| Convertir minúsculas a mayúsculas (y al reves cambiando el orden) | `echo "hola" \| tr 'a-z' 'A-Z'`         |
| Eliminar caracteres concretos                                     | `echo "h-o-l-a" \| tr -d '-'`           |
| Reemplazar carácter por otro                                      | `echo "a:b:c" \| tr ':' '/'`            |
| Comprimir caracteres repetidos en uno                             | `echo "hola mundo" \| tr -s ' '`        |
🔁 `sed`: editar flujos de texto

| Acción                                          | CMD                                                                    |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| Sustituir primera ocurrencia por línea          | `sed 's/viejo/nuevo/' archivo`                                         |
| Sustituir todas las ocurrencias                 | `sed 's/viejo/nuevo/g' archivo`                                        |
| Sustituir ignorando mayúsculas                  | `sed 's/viejo/nuevo/gi' archivo`                                       |
| Sustituir y guardar en el archivo (in-place)    | `sed -i 's/viejo/nuevo/g' archivo`                                     |
| Eliminar líneas que coincidan con patrón        | `sed '/patrón/d' archivo`. Ej `sed '/^$/d' archivo` para líneas vacias |
| Eliminar línea N                                | `sed '3d' archivo`                                                     |
| Mostrar línea / rango de líneas                 | `sed -n '5,10p' archivo`                                               |
| Insertar línea antes/despues de la que coincide | `sed '/patrón/i nueva línea' archivo` (Despues es `/patrón/a`)         |
| Usar múltiples expresiones                      | `sed -e 's/a/A/g' -e 's/b/B/g' archivo`                                |

🧮 `awk`: procesado de columnas y lógica

> `$0` = línea entera · `$1`, `$2`... = campos · `NR` = nº de línea · `NF` = nº de campos · `FS` = separador de entrada · `OFS` = separador de salida

| Acción                                   | CMD                                |
| ---------------------------------------- | ---------------------------------- |
| Imprimir un campo concreto/varios campos | `awk '{print $1, $3}' archivo`     |
| Cambiar el separador de campo            | `awk -F: '{print $1}' /etc/passwd` |
| Imprimir número de línea y contenido     | `awk '{print NR": "$0}' archivo`   |
| Imprimir solo la última columna          | `awk '{print $NF}' archivo`        |
| Imprimir líneas entre dos patrones       | `awk '/inicio/,/fin/' archivo`     |

---
# 4. 👤 Gestión de usuarios y grupos

## 4.1. Consultar información

| Acción                                         | CMD                                  |
| ---------------------------------------------- | ------------------------------------ |
| Ver el usuario actual                          | `whoami`                             |
| Ver nuestro usuario, grupo y uid/gid numéricos | `id`                                 |
| Ver a qué grupos pertenecemos                  | `groups`                             |
| Ver quién está conectado ahora mismo           | `who`                                |
| Listar todos los usuarios / grupos del sistema | `cat /etc/passwd` / `cat /etc/group` |
| Ver información de un usuario                  | `id <usuario>`                       |

## 4.2. Crear y eliminar usuarios

| Acción                                        | CMD                                                      |
| --------------------------------------------- | -------------------------------------------------------- |
| Crear un usuario (con home y shell)           | `useradd -m -s /bin/bash <usuario>`                      |
| Crear un usuario sin home ni shell (servicio) | `useradd -M -s /sbin/nologin <usuario>`                  |
| Asignar o cambiar contraseña                  | `passwd <usuario>`                                       |
| Eliminar un usuario                           | `userdel <usuario>` (`-r` para eliminar su home tambien) |
| Modificar el shell de un usuario              | `usermod -s /bin/bash <usuario>`                         |
| Bloquear/Desbloquear una cuenta               | `usermod -L/-U <usuario>`                                |
| Cambiar el home de un usuario                 | `usermod -d /nuevo/home -m <usuario>`                    |

## 4.3. Gestionar grupos

|Acción|CMD|
|---|---|
|Crear un grupo|`groupadd <grupo>`|
|Eliminar un grupo|`groupdel <grupo>`|
|Añadir un usuario a un grupo|`usermod -aG <grupo> <usuario>`|
|Añadir un usuario a varios grupos|`usermod -aG grupo1,grupo2 <usuario>`|
|Ver los miembros de un grupo|`getent group <grupo>`|
|Cambiar el grupo principal de un usuario|`usermod -g <grupo> <usuario>`|
|Cambiar temporalmente de grupo en la sesión|`newgrp <grupo>`|
> `usermod -G grupo usuario` **sobreescribe** todos los grupos secundarios del usuario. Usar siempre `-aG` (append) para añadir sin quitar los grupos existentes.

---
# 5. ⚙️ Gestión de procesos y red

## 5.1. Gestionar procesos

| Acción                                   | CMD                                 |
| ---------------------------------------- | ----------------------------------- |
| Listar procesos del usuario actual       | `ps`                                |
| Listar todos los procesos con detalle    | `ps aux`                            |
| Listar procesos en árbol (padre-hijo)    | `ps auxf` / `pstree`                |
| Filtrar procesos por nombre              | `ps aux \| grep nginx`              |
| Ver el PID de un proceso por nombre      | `pidof nginx` / `pgrep nginx`       |
| Monitor interactivo de procesos          | `top` / `htop`                      |
| Ver procesos ordenados por CPU/RAM       | `top` → tecla `P` (CPU) / `M` (RAM) |
| Ver información de un proceso por PID    | `cat /proc/<pid>/status`            |
| Ver la ruta del ejecutable de un proceso | `ls -l /proc/<pid>/exe`             |
| Ver los archivos abiertos por un proceso | `lsof -p <pid>`                     |
| Enviar señal a un proceso (terminar)     | `kill <pid>` /(`-9` = SIGKILL)      |
| Terminar todos los procesos por nombre   | `killall nginx`                     |
| Terminar proceso y sus hijos por nombre  | `pkill -TERM -P <pid>`              |
| Ejecutar proceso en background           | `<comando> &`                       |
| Ver procesos en background               | `jobs`                              |
| Traer proceso de background a foreground | `fg %1`                             |

## 5.2. Conexiones y puertos

|Acción|CMD|
|---|---|
|Ver puertos en escucha con PID|`ss -tlnp`|
|Ver todas las conexiones activas con PID|`ss -tunap`|
|Equivalente legacy de `ss`|`netstat -tunap`|
|Filtrar por puerto concreto|`ss -tunap \| grep :443`|
|Ver qué proceso usa un puerto|`lsof -i :80`|
|Ver todas las conexiones de un proceso|`lsof -i -p <pid>`|
|Comprobar si un puerto remoto responde|`nc -zv <ip> 443`|
|Realizar un ping|`ping <ip>`|
|Trazar la ruta a un host|`traceroute <ip>` / `mtr <ip>`|
|Ver la tabla de rutas|`ip route` / `route -n`|
|Ver interfaces de red y sus IPs|`ip a` / `ifconfig`|
|Ver la tabla ARP|`arp -a` / `ip neigh`|
|Resolver DNS de un dominio|`nslookup <dominio>` / `dig <dominio>`|

> [!tip] `ss` vs `netstat`
>  `ss` es el sustituto moderno de `netstat`, más rápido y con más información. Flags más usados: `t` TCP · `u` UDP · `l` solo escuchando · `n` sin resolver nombres · `a` todas · `p` con proceso.

## 5.3. Servicios (systemd)

| Acción                                      | CMD                                                   |
| ------------------------------------------- | ----------------------------------------------------- |
| Listar todos los servicios                  | `systemctl list-units --type=service`                 |
| Listar solo los servicios activos           | `systemctl list-units --type=service --state=running` |
| Ver el estado de un servicio                | `systemctl status <servicio>`                         |
| Iniciar/Detener/reiniciar un servicio       | `systemctl start/stop/restart <servicio>`             |
| Recargar config sin reiniciar               | `systemctl reload <servicio>`                         |
| Habilitar/Deshabilitar servicio al arranque | `systemctl enable/disable <servicio>`                 |
| Habilitar y arrancar en un solo comando     | `systemctl enable --now <servicio>`                   |
| Ver si un servicio está habilitado          | `systemctl is-enabled <servicio>`                     |
| Ver los logs de un servicio en tiempo real  | `journalctl -u <servicio> -f`                         |
| Ver los logs de los últimos N minutos       | `journalctl -u <servicio> --since "5 min ago"`        |
| Recargar los daemons tras editar una unit   | `systemctl daemon-reload`                             |
