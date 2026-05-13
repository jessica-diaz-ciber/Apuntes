# Índice

[[Docker#1. 🐳 Docker]]
- [[Docker#1.1. Funcionamiento interno de Docker]]
- [[Docker#1.2. Capas de Docker]]
- [[Docker#1.3. Gestión de imágenes]]
- [[Docker#1.4. Creación y gestión un contenedor]]
- [[Docker#1.5. Los DockerFiles]]
- [[Docker#1.6. Archivo DockerIgnore]]
- [[Docker#1.7. Multi-stage builds]]
- [[Docker#1.8. Análisis de un contenedor]]
- [[Docker#1.9. Persistencia Docker]]
- [[Docker#1.10. Redes Docker]]

[[Docker#2. Seguridad Docker]]
- [[Docker#2.1. Contenedor privilegiado y capabilities]]
- [[Docker#2.2.Exposición del socket de Docker]]
- [[Docker#2.3. Vulnerabilidades del runtime — escapes reales]]
- [[Docker#2.4. Imágenes con vulnerabilidades o secretos]]
- [[Docker#2.5. 🛡️ Resumen de hardening]]

[[Docker#3. Docker compose]]
- [[Docker#3.1. Docker compose]]
- [[Docker#3.2. Ejemplos de Docker compose]]
- [[Docker#3.3. Comandos de docker compose]]

# 1. 🐳 Docker

> [!abstract] Introducción
> Docker es una tecnología de virtualización a nivel de aplicación que permite empaquetar una aplicación en un contenedor: un entorno aislado que incluye todo lo necesario para ejecutarla correctamente (código, configuración, dependencias y versiones). Al contenerizar una aplicación, esta puede desplegarse en cualquier sistema compatible sin conflictos con el entorno del host.

A diferencia de una máquina virtual, que virtualiza hardware y ejecuta su propio sistema operativo con su kernel, **un contenedor comparte el kernel de Linux del host.** Solo incorpora las librerías y recursos estrictamente necesarios para la aplicación. Esto lo hace mucho más ligero, rápido y eficiente, tanto en consumo de espacio como de rendimiento, al evitar duplicar kernels y capas innecesarias.

## 1.1. Funcionamiento interno de Docker

Los contenedores son creados y gestionados por el Docker Engine, que interactúa directamente con el kernel de Linux y se apoya en sus mecanismos nativos de aislamiento y control de recursos: los cgroups y los namespaces. 
- **cgroups** limitan y asignan recursos para los procesos
- **namespaces** aíslan el contenedor del sistema host (procesos, red, sistema de archivos, etc.)

```
[dockerd]        ->  [containerd]              ->  [runc]                     -> [kernel] 
Daemon de docker     gestiona el ciclo vital       crea el contenedor usando     namespaces + cgroups + OverlayFS
(API)                de los contenedores           syscalls del kernel             
```

> [!example] Namespaces — qué puede ver
> Los namespaces hacen que un proceso tenga su **propia vista** de los recursos del sistema. El contenedor cree que es el único proceso en un sistema independiente, pero en realidad es un proceso más en el host.
> 
**Cada namespace se encarga de una función**: por ejemplo el namespace `PID` hace que el contenedor no pueda ver los procesos del host y se vea su propio proceso como el primero del que deriva el resto. O por ejemplo `MNT` ve su propio sistema de archivos sin ver los montajes del host

> [!example] cgroups — qué puede usar
> Mientras los namespaces controlan lo que el proceso **ve**, los cgroups controlan lo que **consume**. Son la "policía de recursos" del kernel. Si un contenedor supera el límite de memoria, el OOM killer del kernel mata el proceso y el contenedor termina con exit code **137** (SIGKILL).

**Los contenedores, se crean en base a imágenes**. Una imagen en Docker es una plantilla inmutable que describe qué va a tener un contenedor y cómo debe arrancar, el molde a partir del que crear contenedores.

---
## 1.2. Capas de Docker

Una imagen Docker está compuesta por capas, y cada una cumple una función concreta. Imaginemos una aplicación en Python que imprime “Hola mundo”, empaquetada en una imagen basada en Arch Linux.

| Capa                                                 | Descripción                                                                                                                                                                                                                                                       |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Capa base**                                        | Contiene la estructura básica del sistema de archivos (/bin, /usr, /etc, …), la mayoría inicialmente vacíos, junto con las librerías fundamentales del sistema (como glibc) y las herramientas base de la distro, como su gestor de paquetes (Ej, pacman en Arch) |
| **Capa de dependencias del sistema**                 | En esta capa se instalan los paquetes necesarios para ejecutar la aplicación. Ej. Python en una versión concreta, pip y las librerías del sistema requeridas por Python                                                                                           |
| **Capa de dependencias de la aplicación**            | Incluye las librerías específicas de la aplicación, fijadas en las versiones exactas con las que la aplicación funciona correctamente                                                                                                                             |
| **Capa del código y configuración de la aplicación** | Contiene el código fuente de la aplicación y sus archivos de configuración propios                                                                                                                                                                                |
| **Capa de configuración y ejecución**                | Define cómo se inicia el contenedor: el comando de arranque y las variables de entorno necesarias, Ej `CMD ["python", "/app/app.py"]`                                                                                                                             |

> [!tip]+ Como se crean las capas
> Cuando se construye una imagen Docker, esta se crea por capas y cada instrucción del Dockerfile genera una capa nueva
> Al crear nuevas versiones de una capa, docker cachea las capas que no se han cambiado, por tanto en la build solo se generan las capas modificadas
> Por ello es util:
> - **Excluir los archivos necesarios con** `.dockerignore`: No sirve eliminarlos despues, si ya están en una capa previa, siguen ocupando espacio (en la caché).
> - **Ordenar el Dockerfile, poniendo primero las instrucciones que cambian poco**, como la instalación de dependencias. Dejamos para el final las que cambian con frecuencia como el código fuente
> - **Agrupar comandos relacionados en una misma instrucción**, para reducir al máximo el número de capas generadas, eliminando las innecesarias y produciendo imágenes más eficientes y mantenibles. 
> - **Usar multi-stage builds si es posible**

Docker utiliza un driver de almacenamiento llamado **overlay2**, que se encarga de gestionar el sistema de archivos de las imágenes y contenedores. Este driver almacena los datos divididos en capas, cada una con un propósito específico, pero el contenedor las percibe como un único sistema de archivos unificado. 
> Esto permite que varios contenedores compartan la misma imagen, ya que reutilizan las capas de solo lectura comunes, reduciendo el uso de espacio en disco. 

Todas estas capas se almacenan en la ruta: `/var/lib/docker/overlay2/` y son:

| Tipo          | Descripción                                                                                                                                                                                                                                 |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ** LowerDir** | Agrupa las capas de la imagen y es **de solo lectura**. Contiene las capas que forman la imagen base, generadas a partir del Dockerfile, y almacena el sistema de archivos original, que no se modifica durante la ejecución del contenedor |
| ** UpperDir** | Es la **capa de escritura del contenedor** y guarda todos los cambios respecto al estado inicial. Cuando se modifica un archivo, Docker aplica copy-on-write, copiándolo a esta capa y manteniendo intactas las LowerDir                    |
| **MergedDir** | Existe únicamente mientras el contenedor está activo y ofrece una vista unificada de LowerDir y UpperDir, de modo que el contenedor percibe un único sistema de archivos                                                                    |
| **WorkDir**   | Es un directorio interno necesario para el funcionamiento de overlayfs, utilizado por Docker para operaciones técnicas, pero no visible ni accesible desde el contenedor                                                                    |

---
## 1.3. Gestión de imágenes

Primero, descargamos una imagen de docker como alpine, o mysql, la que necesitemos. En los [repositorios de docker](https://hub.docker.com/search?type=image), existen imágenes de todo tipo de aplicaciones y sistemas, lo que nos ahorra tener que instalarlos y configurarlos manualmente.

| Acción                                                              | Comando                                        |
| ------------------------------------------------------------------- | ---------------------------------------------- |
| Instalar imágenes (última versión por defecto si no se indica)      | `docker pull <imagen>:<version>`               |
| Listar imagenes descargadas                                         | `docker images` (alias de `docker image list`) |
| Listar solo el ID de las imagenes descargadas (útil para scripting) | `docker images -q`                             |
| Eliminar una imagen                                                 | `docker rmi <imagen>:<version>`                |
| Borrar todas las imágenes                                           | `docker rmi -f $(docker images -q)`            |
> [!warning]+ Imágenes dangling
> Las imágenes dangling son imágenes sin etiqueta (tag), que suelen generarse al sobrescribir una imagen existente durante un build o cuando el proceso falla, y pueden eliminarse con `docker image prune`. 
> > Luego con `docker system prune` eliminamos imagenes dangling, contenedores parados, redes sin usar y caché, recuperando bastante espacio

---
## 1.4. Creación y gestión un contenedor

Ahora, ya teniendo la imagen descargada, podemos hacer un `docker run` para crear y ejecutar comandos en el contenedor. 

| Acción                                                                                        | Comando                             |
| --------------------------------------------------------------------------------------------- | ----------------------------------- |
| Ejecutar un contenedor: en segundo plano (`-d` ) y en modo interactivo (`-i`)  con tty (`-t`) | `docker run -dit <imagen> sh`       |
| Mapear una carpeta del host en una del contenedor                                             | `docker run (...) -v $(pwd):/datos` |
| Mapear un puerto local al del contenedor                                                      | `docker run (...) -p 80:80`         |
| Configurar una variable de entorno                                                            | `docker run (...) -e 'PASS=pass123` |
| Crear un contenedor con nombre, en lugar que se le ponga uno aleatorio                        | `docker run (...) --name=<nombre>`  |
| Crear contenedores de pruebas que se eliminen al salir de ellos o pararlos                    | `docker run (...) --rm`             |
> El contenedor en segundo plano permite que nos podamos conectar a él y salir cuando queramos

Una vez que tenemos creado un contenedor:

| Acción                                                   | Comando                                 |
| -------------------------------------------------------- | --------------------------------------- |
| Ver los contenedores                                     | `docker ps`                             |
| Ver solo el id de los contenedores (útil para scripting) | `docker ps -q`                          |
| Mostrar los contenedores parados (combinable con `-q`)   | `docker ps -a`                          |
| Ejecutar comandos en el contenedor                       | `docker exec it <contenedor> <comando>` |
| Parar un contenedor                                      | `docker stop <contenedor>`              |
| Eliminar un contenedor parado                            | `docker rm <contenedor>`                |
Podemos copiar archivos del contenedor al host y viceversa con `docker cp`. Aun así lo más eficiente es crear la carpeta compartida con `docker run -v`.

| Acción                            | Comando                                                          |
| --------------------------------- | ---------------------------------------------------------------- |
| Del contenedor al host            | `docker cp <path_del_host> <contenedor>:<ruta en el contenedor>` |
| Del contenedor al host (al revés) | `docker cp <contenedor>:<ruta en el contenedor> <path_del_host>` |

---
## 1.5. Los DockerFiles

Dockerizar una aplicación consiste en **empaquetar todo lo necesario para que funcione** (dependencias, librerías, configuración y código) dentro de una **imagen**, a partir de la cual se crean **contenedores reproducibles** que pueden ejecutarse en cualquier sistema con Docker, sin instalaciones adicionales.

La dockerización se compone de:

- **Dockerfile**: archivo de texto que define cómo se construye la imagen.
- **.dockerignore**: lista de archivos y directorios que **no deben copiarse** al contenedor.

---
Un `Dockerfile` define paso a paso la construcción de una imagen mediante instrucciones:

```bash
FROM <base_image>          # Definen la imagen base de la que se construye, el entorno inicial
ENV <variable>:<valor>     # Define variables de entorno usadas por la aplicación o el sistema
RUN <command>              # Ejecuta comandos durante la construcción de la imagen
WORKDIR /app               # Establece el directorio de trabajo del contenedor
COPY <origen> <destino>    # Copia archivos desde el host al contenedor. 
# COPY . . copia todo el directorio actual en la workdir
USER 1001                  # Usuario que ejecutará el contenedor y sus comandos (UID)
CMD                        # Comando por defecto al iniciar el contenedor (puede sobrescribirse)  
ENTRYPOINT                 # Comando principal que siempre se ejecuta
EXPOSE <puerto>            # Documenta qué puerto usa la aplicación y que se debe exponer con "docker run -p"
```

Luego creamos la imagen a partir del Dockerfile con `docker build -t <nombre> .`

> [!tip] Versionado
> Cada modificación del Dockerfile da lugar a una nueva imagen que deriva de la anterior, por lo que es importante gestionar correctamente el versionado de las imágenes para evitar la acumulación de imágenes dangling, por ejemplo etiquetándolas como `prueba:1`, `prueba:2`, etc.

En cuando a las base images que elegir, tenemos que tener en cuenta el amaño, seguridad, tiempos de build y operación.

|Tipo|Ejemplos|Ventajas|Desventajas|
|---|---|---|---|
|**Sistemas operativos completos**|ubuntu, debian, centos|Ofrecen máxima compatibilidad con librerías, shell y utilidades, lo que facilita el aprendizaje, las pruebas y el debugging|Son imágenes grandes que aumentan el tiempo de build, el consumo de espacio y la superficie de ataque|
|**Entornos de lenguaje**|node, python, golang|Incluyen el runtime y librerías estándar del lenguaje, reduciendo errores de instalación y configuración|Siguen siendo relativamente pesadas e incluyen herramientas innecesarias para solo ejecutar la aplicación|
|**Alpine Linux**|alpine|Proporciona una base muy ligera y rápida, ideal para microservicios con dependencias bien controladas|Puede causar incompatibilidades debido al uso de musl en lugar de glibc y dificulta el debugging|
|**Distroless**|distroless/nodejs, distroless/java|Incluye únicamente lo necesario para ejecutar la aplicación, ofreciendo una superficie de ataque mínima en producción|No dispone de shell ni gestor de paquetes, lo que impide su uso para desarrollo o depuración interactiva|
|**Scratch**|scratch|Permite crear contenedores extremadamente pequeños y seguros para ejecutar binarios estáticos|No incluye ningún componente del sistema, lo que limita su uso a casos muy específicos y sin posibilidad de depuración|
|**Aplicaciones**|Imágenes oficiales de apps|Proporcionan una imagen lista para ejecutar una aplicación concreta sin configuración adicional|Son poco flexibles y dependen completamente de las decisiones del mantenedor de la imagen|

Hay que tener en cuenta estos parámetros:

> [!warning] Exec form vs Shell form
> El comando CMD puede declararse en shell form o exec form, lo que afecta a la ejecución del proceso.
> - **Shell form**: Se ejecuta una shell  (proceso padre) que a su vez ejecuta el comando hijo (proceso hijo). Ej `CMD node index.js` → `/bin/sh -c "node index.js"`
> - **Exec form**: Se ejecuta directamente el comando, creando un solo proceso pero sin interpretar variables de entorno `CMD ["node", "index.js"]`

> [!warning] CMD vs Entrypoint
> Aunque ambos definen el comando que se ejecuta al iniciar el contenedor: 
> - **CMD**: Establece un comando por defecto que puede sobrescribirse. Ej: *Si ponemos `CMD ["node"]` y en ese contenedor ponemos `docker run imagen bash` ejecutará una bash en lugar de node* 
> - **ENTRYPOINT**: Define un comando fijo al que se le pasan los argumentos indicados en `docker run`. *Si ponemos `ENTRYPOINT ["node"]` y hacemos `docker run imagen bash` se ejecutará "`node bash` dando un error*
> 
> > Al combinar ambos, se puede definir un ejecutable obligatorio con parámetros por defecto, logrando mayor flexibilidad en la ejecución del contenedor con docker exec.

> [!example] Redes
> Docker crea por defecto una red privada tipo `bridge` asociada a la interfaz `docker0`, normalmente en el rango `172.17.0.0/16`, accesible solo desde el host. Por tanto:
> - El contenedor y el host se comunican por esa interfaz
> - El contenedor puede acceder a otras máquinas de la red por medio de la ip del host (traducción NAT)
> - Para que otras máquinas accedan a los servicios del contenedor, tiene que haber port forwarding. _Por ejemplo si se ejecuta con `docker run -p 80:80`, cuando otras máquinas consulten al puerto 80 del host, en realidad acceden al servicio del contenedor_

> [!warning] Usuarios y permisos
> **Los usuarios del contenedor son independientes de los del host, pero comparten los mismos UID**, lo que puede causar problemas de permisos. 
> *Ej, el usuario root del contenedor comparte UID con root del host, y el primer usuario suele tener el UID 1000, igual que el primer usuario del sistema anfitrión*. 
> 
> Ademas siempre se crea el contenedor como root, así que un atacante al comprometer al contenedor puede afectar al host. Para ello, tras instalar las dependencias, hay que migrar a otro usuario para crear archivos y ejecutar comandos con la línea `USER 1000`. Esto  fuerza la ejecución como un UID no privilegiado, ahorrándonos tener que crear usuarios.

> [!warning] Problemas con el prompt
> Un problema habitual surge cuando al crear la imagen, algunas herramientas piden un input del usuario, como por ejemplo elegir la zona horaria en contenedores debian (los contenedores no heredan la hora del host), esto rompe la build y genera un error. 
> Para evitarlo, se desactiva la interactividad y se predefinen las configuraciones mediante variables de entorno, por ejemplo la zona horaria con:
> ```
> ENV DEBIAN_FRONTEND=noninteractive
> ENV TZ=Europe/Madrid
> ```

---
## 1.6. Archivo DockerIgnore

Normalmente el Dockerfile se ubica en el mismo directorio que los archivos que se copiarán a la imagen. Así que todo el código fuente o binarios estáticos se deben colocar en ese directorio y copiar al contenedor con la instrucción `COPY . . `

El archivo `.dockerignore` indica los archivos que no se deben copiar, cada uno en una línea. Por motivos de seguridad y reducción del tamaño de la imagen, debemos excluir todo lo que no sea estrictamente necesario para que funcione la aplicación:
- **Control de versiones**: .git* 
- **Archivos de editores** como: .vscode
- **Logs**: que contienen credenciales o información interna: *.log, logs/
- **Dependencias locales:** estas ya no se necesitan al compilar: node_modules/, .venv/, build/, target/, dist/
- **Caches**: .npm, .yarn, __pycache__/...
- **Archivos sensibles** que puedan contener credenciales como variables de entorno (.env), llaves y certificados (.pem, .key y .cert)
- **El propio Dockerfile** ¿para que lo necesitamos en el contenedor?

---
## 1.7. Multi-stage builds

Es recomendable utilizar los **multi-stage builds**. Esta técnica permite crear una imagen en varias etapas, usando una imagen base distinta para cada una. En la imagen final, solo se tendrá en cuenta la última etapa, reduciendo drásticamente el tamaño de la imagen.

1. **Construcción o compilación:** se usa una imagen más pesada, que incluya todo lo necesario. Podemos utilizar una imagen propia de ese lenguaje, por ejemplo: gcc para binarios en c, golang para los de go o node para los javascript.
|
2. **Ejecución:** para esta fase utilizamos una imagen minimalista, la cual no contenga ni una shell ni apenas librerías reduciendo al máximo el tamaño y la superficie de ataque.   
    - Podemos usar una imagen `scratch` para binarios estáticos, esta imagen está prácticamente vacía
    - Para los programas interpretados, podemos usar una versión distroless.Ej, gcr.io/distroless/nodejs para node.

Por ejemplo creamos esta simple aplicación GO
```go
packege main
import "fmt"
func main() { fmt.Println("hello world") }
```

Y luego el Dockerfile
```bash
FROM golang:1.24 AS base
WORKDIR /src
COPY ./main.go . 
RUN go build -o /bin/hello ./main.go # Compilar el binario

FROM scratch                           # Usamos la imagen scratch
COPY --from=base /bin/hello /bin/hello # Copiar el binario de la imagen base
USER 1000                              # Ejecución no privilegiada
CMD ["/bin/hello"]                     # En scratch no hay una shell, por tanto tiene que ser exec form si o si
```

---
## 1.8. Análisis de un contenedor

Con `docker top` podemos inspeccionar los procesos activos en el contenedor.

Y con `docker inspect <ID>`, podemos inspeccionar información de una imagen o un contenedor. Podemos inspeccionarlo con:
- `-f {{campos}}`: es la manera nativa, más ineficiente. Ej. `docker inspect <ID> -f "{{.NetworkSettings.IPAddress}}"`
- Conbinándolo con herramientas como `jq` o `grep` : Ej. `docker inspect <ID> |jq ".[] | .NetworkSettings.IPAddress, .Name"`

Los campos que nos interesan son:
- **.GraphDriver.Data** : las rutas de UpperDir, LowerDir... etc
- **.NetworkSettings**: datos de redes
- **.Mount**s: carpetas compartidas (monturas)
- **.Config**: configuraciones (usuariom, variables de entorno, comandos)

> [!example] Analizar las capas
> Para analizar las capas podemos usar la herramienta `dive` la cual se instala desde [su repositorio](https://github.com/wagoodman/dive#installation), copiando los comandos que nos indican. 
Se maneja desde atajos del teclado:
> - **Tab**: moverse entre las columnas de información de imágenes y el árbol de directorios. Vemos en verde los archivos nuevos, en rojo los borrados
> - **Espacio**: en el árbol de directorios, podemos expandir o colapsar una carpeta
> - **Flechas arriba/abajo**: con las flechas nos movemos entre directorios y versiones
>
Para inspeccionar los archivos, se buscan en la ruta de **overlay2**: `cat $(find /var/lib/docker/overlay2 -name hola.txt 2>/dev/null)`

---
## 1.9. Persistencia Docker

Los contenedores Docker son efímeros: cuando se elimina un contenedor, se pierden todos sus archivos. 

Para evitar la pérdida de datos (por ejemplo bases de datos), Docker proporciona los volúmenes, que son unidades de almacenamiento persistente independientes del contenedor y que están en la ruta `/var/lib/docker/volumes/db_data` del host.

| Acción                        | Comando                         |
| ----------------------------- | ------------------------------- |
| Crear un volumen              | `docker volume create db_data`  |
| Listar los volúmenes          | `docker volume ls`              |
| Ver información de un volumen | `docker volume inspect db_data` |
| Eliminar un volumen           | `docker volume rm db_data`      |
Por tanto podemos crear un contenedor con ese volumen. Por ejemplo usaremos un volumen que se llame "**db_data**" y que hayamos creado previamente:
```bash
docker run -dit -e MYSQL_ROOT_PASSWORD=asdfg -p 3306:3306 -v db_data:/var/lib/mysql mysql:8.0 
```

Por tanto **si el contenedor se elimina y se crea otro usando el mismo volumen, los datos se conservan**.

> [!example] Los Commits
> Docker permite crear una imagen a partir del estado actual de un contenedor mediante
> 1. `docker commit <contenedor> <imagen>` : permite crear una imagen a partir del estado actual del contenedor
> 2. `docker save <imagen> -o <archivo>.tar` : guarda la imagen en un archivo .tar que enviarle a otra persona
> 3. `docker load -i <archivo>.tar` : permite cargar la imagen a partir de un archivo tar que hayamos descargado
>
⚠️**Esto tiene varias limitaciones**: no permite conocer cómo se ha creado el contenedor ni reproducir el proceso, y además no incluye los datos almacenados en volúmenes o monturas. 
> >  Por este motivo, su uso se limita a casos muy concretos, en los que no es necesario conocer el procedimiento de creación, como ejercicios de ciberseguridad o laboratorios prácticos, y siempre que el contenedor no haya sido ejecutado utilizando la opción `-v` de `docker run`.

> [!example] COPY
> La forma correcta y recomendada de compartir una aplicación Docker con otra persona es construir la imagen de forma reproducible, no capturarla. Esto se hace entregando un proyecto que incluya:
> - 📁 Los archivos del código y binarios
> - 📄 El dockerfile con la instrucción `COPY . .`
> - 📄 El dockerignore para excluir archivos innecesarios (como el propio Dockerfile o la carpeta .git, etc.)
> 
> Luego, la otra persona puede construir la imagen con: `docker build -t mi_imagen .`

---
## 1.10. Redes Docker

Primero, los comandos son:

| Acción                                                                               | Comando                         |
| ------------------------------------------------------------------------------------ | ------------------------------- |
| Listar redes. Encontraremos siempre la bridge, la host y la none más las que creemos | `docker network ls`             |
| Eliminar una red                                                                     | `docker network rm <id/nombre>` |
Existen 7 tipos de redes docker:

> [!example] Default bridge
> Es la que se asigna por defecto a los contenedores. Es ideal para desarrollo local y aplicaciones sencillas. 
La red es la `172.17.0.0/16` por la interfaz `docker0`. El host (gateway para NAT) es la primera dirección `172.17.0.1` y a partir de ahí se asignan en orden direcciones para los contenedores que se vayan creando.
> - Los contenedores de la red pueden comunicarse entre sí y con el host usando sus nombres (DNS interno) o IPs
> - Los contenedores pueden acceder a internet y a la red local por medio de traducción NAT
> - Para exponer servicios a la red local hay que hacer port forwarding (`docker run -p`)
> 
> Ej: `docker run --name contenedor -dit alpine`

> [!example] User defined bridge
> Es una red bridge como la anterior, solo que la crea el usuario y por tanto tiene la opción de configurarla manualmente (asignar rangos de red, IPS...etc). Por tanto en un sistema se pueden crear varias redes bridge aisladas entre sí para crear distintos entornos. 
> ```bash
> docker network create mi_red
> docker run --name contenedor2 --network mi_red -dit alpine
> ```

> [!example] Host
> Esta red elimina el aislamiento de red, y no hay que crearla, ya que existe por defecto. En esta red el contenedor comparte la IP y puertos del host. Esto hace que no haya NAT ni mapeo de puertos, lo que ofrece máximo rendimiento, pero reduce la seguridad y puede generar conflictos de puertos. Se usa cuando el rendimiento o el acceso directo a la red es crítico.
> Ej: `docker run --name contenedor3 --network host -it python:3.12 python -m http.server 80`. Bloquea el puerto 80 del host

> [!example] None
> Esta red es muy sencilla de entender, ya que deshabilita completamente la conectividad. El contenedor no tiene interfaz de red (salvo la localhost) y queda totalmente aislado. Es útil para tareas offline o escenarios donde se busca la máxima seguridad.
> Ej: `docker run --name contenedor4 --network none -it alpine`

> [!example] MavLAN
> En esta red, cada contenedor obtiene una IP propia dentro de la red física, apareciendo como un dispositivo independiente en la LAN. Esto permite que los servicios externos puedan acceder directamente al contenedor sin port forwarding.
> 
Para crearla, lo primero que debemos hacer en nuestra máquina es identificar la red física a la que estamos conectados y obtener tanto la interfaz como la gateway. Para ello utilizamos el comando `ip route`. Es especialmente importante definir el rango de redes para que Docker asigne las IPs de forma controlada y evitar conflictos con direcciones que ya estén siendo utilizadas por otros dispositivos de la red. Para ello asignaremos direcciones altas, por ejemplo a partir de la 200.
> ```bash
/usr/sbin/ip route # default via 192.168.159.2 dev eth0 proto dhcp src 192.168.159.139 metric 100
docker network create --driver macvlan --subnet=192.168.159.0/24 --gateway=192.168.159.2 \
 --ip-range=192.168.159.200/29 -o parent=eth0  red_macvlan
docker --name contenedor5 --network red_macvlan -it tomcat:9-jdk17 
docker inspect contenedor5 | grep IPAddress # 192.168.159.200
curl http://192.168.159.200:8080 # El tomcat
> ```

Existen otros 2 tipos más complejos
- **IPvlan:** En esta red, los contenedores comparten solo la MAC del host, no la IP. Esto es útil para redes grandes y entornos de producción
- **Overlay**:  Permite que contenedores ubicados en distintos hosts se comuniquen como si estuvieran en la misma red. Funciona sobre Docker Swarm y es clave en sistemas de microservicios.

---
# 2. Seguridad Docker

> [!danger] Kernel compartido
> El aislamiento de Docker no es absoluto Docker aísla procesos, no el kernel. Todas las vulnerabilidades a continuación explotan que el contenedor y el host **comparten el mismo kernel** o que hay configuraciones que rompen el aislamiento deliberadamente.

> [!danger] Conectividad con el host
> Por defecto, todos los contenedores en la misma red `bridge` se pueden comunicar entre sí sin restricciones. Si un contenedor es comprometido, el atacante puede escanear y atacar al resto: 
> ```bash
> for i in $(seq 1 254); do ping -c1 172.17.0.$i 2>/dev/null && echo "172.17.0.$i alive"; done
> ```

---
## 2.1. Contenedor privilegiado y capabilities

`--privileged` es la opción más peligrosa de Docker. Desactiva **todos** los mecanismos de seguridad: el contenedor hereda todas las capabilities del host, puede montar filesystems, cargar módulos del kernel y acceder a dispositivos. 

Las capabilities que hereda son todas estas, pero con que tenga alguna de estas ya es peligroso. Se puede comprobar con `capsh --print`

| Capability            | Riesgo                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------ |
| `CAP_SYS_ADMIN`       | Permite montar filesystems, modificar namespaces, usar `ptrace`...                               |
| `CAP_NET_ADMIN`       | Modificar interfaces de red, iptables, sniffing de tráfico del host                              |
| `CAP_SYS_PTRACE`      | Acceder a la memoria de cualquier proceso. Base de muchos escapes                                |
| `CAP_DAC_READ_SEARCH` | Leer cualquier archivo del host ignorando permisos                                               |
| `CAP_SYS_MODULE`      | Cargar módulos del kernel → [explotación](https://blog.nody.cc/posts/container-breakouts-part2/) |
| `CAP_NET_RAW`         | Crear sockets raw → sniffing, spoofing                                                           |

> Un contenedor privilegiado tiene efectivamente el mismo nivel de acceso que el proceso raíz del host. Si un atacante compromete la aplicación dentro, tiene el host.

```bash
docker run --privileged ubuntu bash
# Dentro del contenedor:
fdisk -l                          # ve todos los discos del host
mount /dev/sda1 /mnt              # monta el disco del host
chroot /mnt                       # sale al sistema de archivos real del host → escape completo al host
```

> [!error] Directorios críticos del host
> Montar directorios críticos del host da acceso directo a su contenido desde el contenedor. Por ejemplo la raíz `/`, pero tambien
> - El `etc` para modificar passwd, sudoers. Por ejemplo `echo 'wtf::0:0::/root:/bin/bash' >> /etc/passwd`

> [!warning]+ UID 0
Por defecto, los procesos dentro del contenedor corren como **root (UID 0)**. Si hay un escape, el atacante llega al host como root.
Con la opción `--user`, se puede especificar un usuario y grupo no privilegiado para el primer proceso del contenedor, limitando el impacto potencial de vulnerabilidades de seguridad. `docker run --user 1000:1000 ubuntu whoami` o `USER nonroot` en el Dockerfile

---
## 2.2.Exposición del socket de Docker

El socket de Docker es la API del daemon. Se encuentra en `/var/run/docker.sock`.
Si se monta dentro de un contenedor, ese contenedor puede controlar Docker creando por ejemplo contenedores privilegiados
```bash
docker_api(){ curl -s -X POST --unix-socket /var/run/docker.sock "http://localhost$1"; }
docker_api /images/json  # Listar imágenes. Ej test:1
docker_api /images/json /containers/create -H 'Content-Type: application/json' \
-d '{"Image":"test:1","HostConfig":{"Binds":["/:/wtf"]},"Cmd":["/bin/sh","-c","chmod u+s /wtf/bin/bash"], "Tty":true}' 
docker_api /containers/<id>/start # Arrancarlo
```

> [!warning] Casos legítimos y su riesgo 
> Herramientas como Portainer, Watchtower o algunos CI/CD montan el socket por diseño. En esos casos, el contenedor debe ser tratado con el mismo nivel de confianza que el propio host.

---
## 2.3. Vulnerabilidades del runtime — escapes reales

Vulnerabilidades como CVE-2024-21626 (Leaky Vessels) demostraron el impacto generalizado en entornos cloud, mostrando cómo un solo fallo del kernel puede afectar a toda la infraestructura en contenedores.

|CVE|Componente|Descripción|
|---|---|---|
|**CVE-2019-5736**|runc|Permitía sobrescribir el binario `runc` del host durante la ejecución del contenedor. Afectó a Docker, Kubernetes, containerd|
|**CVE-2022-0492**|kernel (cgroups)|Race condition en cgroups v1 permitía escape de contenedor sin privilegios especiales|
|**CVE-2024-21626**|runc (Leaky Vessels)|File descriptor leak permitía al contenedor acceder al filesystem del host durante el arranque|
|**CVE-2024-1086**|kernel (netfilter)|Use-after-free en netfilter explotado activamente por ransomware para escalada de privilegios|
|**CVE-2025-31133**|runc|Permite reemplazar `/dev/null` con un symlink a `/proc/sys/kernel/core_pattern` → escritura arbitraria en el host|
> [!tip] La conclusión de los escapes de runtime La mayor parte de escapes de contenedor históricos han sido bugs en `runc` o en el kernel. Mantener el host y el runtime actualizados es la mitigación más efectiva.

---
## 2.4. Imágenes con vulnerabilidades o secretos

La imagen es la superficie de ataque más olvidada. Una imagen con paquetes desactualizados o secretos embebidos es vulnerable aunque el runtime sea perfecto.

```bash
ENV AWS_SECRET_KEY=PASS123 # MAL — secreto embebido en la imagen (queda en las capas)
FROM ubuntu:18.04          # MAL — Imagen base seactualizada, llena de CVEs sin parchear
RUN apt install -y python3 # MAL — instalar paquetes sin fijar versiones (no reproducible)
```

Podemos buscar CVEs en una imagen con `docker scout cves <imagen>:<version>`

Tambien podemos buscar secretos en capas de la imagen `trufflehog docker --image ubuntu:22.04`

---
## 2.5. 🛡️ Resumen de hardening

| Práctica                                                                         | Problema que mitiga                       |
| -------------------------------------------------------------------------------- | ----------------------------------------- |
| Nunca `--privileged` salvo necesidad absoluta                                    | Acceso total al host                      |
| No montar `/var/run/docker.sock` en contenedores                                 | Control del daemon desde el contenedor    |
| Ejecutar como usuario no root (`--user` o `USER` en Dockerfile)                  | Reduce impacto de escape                  |
| `--cap-drop ALL --cap-add <solo lo necesario>`                                   | Minimiza superficie de capabilities       |
| Limitar recursos (`--memory`, `--cpus`)                                          | Protege contra DoS por contenedor ruidoso |
| Usar `--read-only` cuando el contenedor no necesita escribir                     | Reduce superficie de escritura            |
| No montar `/`, `/proc`, `/sys`, `/dev`                                           | Escapes directos al host                  |
| Escanear imágenes con Trivy/Scout antes de producción                            | CVEs en dependencias                      |
| No embeber secretos en imágenes — usar secrets o variables de entorno en runtime | Exposición de credenciales                |
| Mantener kernel, Docker y runc actualizados                                      | Vulnerabilidades de runtime               |
| Usar redes bridge separadas por servicio                                         | Movimiento lateral entre contenedores     |
| Activar seccomp y AppArmor (activos por defecto en Docker)                       | Filtrado de syscalls peligrosas           |

---
# 3. Docker compose

> [!abstract] Introducción
> Docker Compose es una herramienta que facilita la definición, gestión y ejecución de aplicaciones basadas en contenedores Docker, especialmente cuando están formadas por varios servicios (por ejemplo, un contenedor para la aplicación web y otro para la base de datos). Permite centralizar toda la configuración del entorno en un único archivo, simplificando su administración.

Gracias a Docker Compose, todo el ecosistema de la aplicación puede iniciarse y detenerse de forma conjunta, lo que ahorra tiempo, reduce errores manuales y mejora la consistencia entre entornos. En la práctica, Docker Compose permite evitar la proliferación de comandos docker run largos y complejos, llenos de variables de entorno, volúmenes y puertos expuestos, sustituyéndolos por una configuración clara y reutilizable. 

> ⚠️Para que un despliegue con Docker Compose funcione correctamente, los contenedores deben ser capaces de ejecutarse de forma individual, ya que Compose se limita a coordinarlos y gestionarlos como un conjunto.

## 3.1. Docker compose

El funcionamiento de compose se basa en un archivo denominado `docker-compose.yml`, en el que se describe la estructura de la aplicación de forma legible y declarativa. 

> [!abstract] Sintaxis YAML
> 1. La clave y el valor se separan mediante dos puntos y un espacio, con el formato `clave: valor`
> 2. La jerarquía y los distintos niveles se definen por la indentación, que suele ser de 2 espacios y sin tabulaciones. Al igual que pasa con Python, un error de indentación puede provocar un fallo
> 3. Las comillas no son obligatorias en la mayoría de los casos y solo se utilizan cuando el valor contiene caracteres especiales o rutas, por ejemplo: `ruta: '/home/kali'`
> 4. Las listas se representan mediante un guion delante de cada elemento, permitiendo definir conjuntos de valores u objetos, como en el caso de `- aplicación: web`

En este caso, la estructura raíz de nuestro compose.yaml va a ser:

```yaml
version:       # Version de docker compose (opcional)
services:      # Los distintos contenedores que formarán la aplicación
  <contenedor>:
    <configuraciones>  
    
networks:           # La red interna a la que se conectarán los contenedores
  <red>:
    driver: bridge  # tipo de driver, Ej user_defined_bridge 
    
volumes:            # Los volúmenes para conservar datos persistentes
  <volúmen>:        # Cada volumen
```

Dentro de services, tenemos estas opciones:

> [!example] Image y build
> Primero tenemos que definir la imagen base. Tenmos dos opciones
> - **imagen predefinida**: Es lo más sencillo y eficiente, ya que hay imágenes para todo tipo de usos. Ej. `image: mysql:8.0`
> - **imagen personalizada:** Para casos muy específicos, utilizamos `build` y en cuyo caso, `image` equivale al nombre que le pondremos al construir la imagen. `dockerfile` indica el dockerfile a utilizar
> ```
> services:   
  >   db:   
  >     image: mi_base_de_datos:1  
  >     build:  
   >      dockerfile: ./dockerfile_db
> ```

> [!example] Redes
> Podemos mapear puertos (equivalente al `docker run -p`) 
> ```yaml
>   ports:                      # puertos expuestos (-p 3306:3306)
>     - "3306:3306"
> ```
> Y tambien pedir al contenedor que se conecte a una red interna que hemos definido con `networks`
> ```yaml
>   networks:                      # la red dónde se conecta
 >      - wordpress-network
> ```

> [!example] Variables de entorno
> El parámetro `environment` permite definir las variables de entorno de un servicio, estas sirven para definir usuarios, contraseñas y demás opciones de cada servicio. Es el equivalente a la opción `-e` en el Dockerfile tradicional. Según el servicio que se ejecute, existirán unas variables de entorno predefinidas que podremos usar. *Ej. `MYSQL_PASSWORD` para las contraseñas en mysql.*
> 
> Pues bien, es un error poner en el docker-compose los valores directamente (Ej `MYSQL_PASSWORD: asdfg`), lo ideal en todo caso es crear en el mismo directorio un archivo llamado `.env` que contenga todas variables de este modo:
> ```yaml
> # En docker-compose.yml
>  env_file: .env
 >  enviroment: 
  >   MYSQL_ROOT_PASSWORD: ${DB_PASS}
  > 
> # En .env
> DB_PASS=password
> ```

> [!example] Restart y depends_on
> `restart` define qué hace Docker si un contenedor se para (por error, por crasheo, o porque el daemon de Docker reinició). Puede ser
> 
> | valor          | uso                                                                                   |
> | -------------- | ------------------------------------------------------------------------------------- |
> | no             | Si se para, no lo levanta otra vez                                                    |
> | always         | Lo vuelve a levantar siempre si se para o al reiniciar Docker                         |
> | unless-stopped | Como always, pero si se paró manualmente, no se vuelve a levantar al reiniciar Docker |
> `depends_on` controla el orden de arranque/parada entre servicios. Por tanto si queremos que el servicio (por ejemplo web) no se levante hasta que no lo haga la base de datos (y así evitar fallos), utilizamos:
> ```yaml
> depends_on:
>   - db
> ```

> [!example] Volúmenes
> Un volumen es una forma de persistir datos fuera del contenedor. Estas pueden ser:
> - **Bind mount**: montar una carpeta del sistema (equivalente a `docker run -v`). 
> ```bash
> volumes:
>   - ./src:/app  # ./src -> /app en el contenedor
>                 # Podemos añadir :ro para que sea de solo lectura ./src:/app:ro
> ```
> 
> - **Named volumes:** Son volúmenes persistentes de Docker, bajo el directorio `/var/lib/docker/volumes/`. Se definen en `volumes`, fuera del campo `services`
> ```bash
>   volumes:
>     - db_data:/var/lib/mysql  # En el contenedor
> 
> volumes:       # Fuera, en volumes
>   db_data:
> ```

## 3.2. Ejemplos de Docker compose

Ejemplo:
```yaml
services:
  mysql:
    image: mysql:8.0            # imagen base 
    container_name: mysql-db    # nombre del contenedor (--name mysql-db)
    ports:                      # puertos expuestos (-p 3306:3306)
      - "3306:3306"
    enviroment:                     # variables de entorno (-e)
      MYSQL_ROOT_PASSWORD: asdfgh   
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: pass123
    volumes:                       # volúmenes (-v mysql_data:/var/lib/mysql)
      - mysql_data:/var/lib/mysql
```

Construido a partir de la estructura de este Dockerfile
```bash
docker run -dit --name mysql-db -p 3306:3306 -v mysql_data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=asdfgh -e MYSQL_DATABASE=appdb -e MYSQL_USER=appuser -e MYSQL_PASSWORD=pass123 \
mysql:8.0
```

Ejemplo de mysql 
```yaml
version: '3.8'
services:
  db:
    image: mysql:8.0
    container_name: wordpress_db
    restart: always
    env_file: .env
    environment:
      MYSQL_DATABASE: ${MYSQL_DB}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASS}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASS}
    volumes:
      - ./db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    container_name: wordpress_app
    restart: always
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: ${WP_DB_HOST}
      WORDPRESS_DB_USER: ${WP_DB_USER}
      WORDPRESS_DB_PASSWORD: ${WP_DB_PASS}
      WORDPRESS_DB_NAME: ${WP_DB_NAME}
    depends_on:
      - db
    volumes:
      - ./wp_data:/var/www/html
```

---
## 3.3. Comandos de docker compose

| Descripción                                   | Comando                                      |
| --------------------------------------------- | -------------------------------------------- |
| Desplegar los contenedores en segundo plano   | `docker compose up -d`                       |
| Ver los estados y nombres de los contenedores | `docker compose ps`                          |
| Ver el estado de un contenedor en concreto    | `docker compose ps <contenedor>`             |
| Ver los logs en tiempo real de un contenedor  | `docker compose log <contenedor>`            |
| Ejecutar un comando en un contenedor          | `docker compose exec <contenedor> <comando>` |
| Ver en yaml la configuración del contenedor   | `docker compose version`                     |
| Parar los contenedores                        | `docker compose stop`                        |
| Eliminar los contenedores                     | `docker compose down`                        |
| Eliminar contenedores y volúmenes             | `docker compose down -v`                     |




## 🔗 Ver también

- [[Suricata]]
