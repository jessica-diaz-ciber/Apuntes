# ⚙️ Procesos, Red y Servicios en Linux

> [!abstract] Introducción 
> Linux gestiona los recursos del sistema mediante **procesos** (unidades de ejecución), **servicios** (procesos gestionados por systemd) y una pila de red configurable a bajo nivel. Entender cómo se relacionan entre sí y cómo el kernel los gestiona es fundamental tanto para administración como para seguridad.

---
# 1. ⚙️ Procesos

## 1.1. ¿Qué es un proceso?

Un proceso es un programa en ejecución con su propio espacio de memoria, descriptores de archivo y contexto de seguridad. El kernel asigna a cada proceso un **PID (Process ID)** único.

```
  disco: /usr/bin/nginx
         │
         │ execve()
         ▼
  PROCESO (nginx)
  ├── PID:        1234
  ├── PPID:       1        ← proceso padre (systemd)
  ├── UID/GID:    www-data ← contexto de seguridad
  ├── Estado:     S        ← sleeping
  ├── Memoria:    espacio virtual propio
  ├── FDs:        0 (stdin), 1 (stdout), 2 (stderr), 3 (socket)...
  └── Señales:    máscara de señales pendientes/bloqueadas
```

## 1.2. Árbol de procesos

Todo proceso tiene un padre. El proceso raíz es **systemd (PID 1)**, que arranca todos los demás.

```
  systemd (1)
  ├── sshd (892)              ← daemon SSH
  │   └── sshd (1203)        ← sesión de jessica
  │       └── bash (1204)
  │           └── ps (1250)  ← proceso temporal
  ├── nginx (1100)
  │   ├── nginx worker (1101)
  │   └── nginx worker (1102)
  ├── cron (950)
  └── ...
```

> [!tip] Creación de procesos
> Linux crea nuevos procesos con dos syscalls `fork` y `exec`. Cuando se ejecuta un comando en bash, el sistema sigue este proceso:
> 1. bash llama a `fork()` → crea un proceso hijo idéntico a bash
> 2. El hijo llama a `exec("/usr/bin/ls")` → su imagen se reemplaza con ls
> 3. bash (padre) espera con `wait()` a que el hijo termine

---
## 1.3. Estados de un proceso

|Estado|Código|Descripción|
|---|---|---|
|**Running**|`R`|Ejecutándose en CPU o listo para ejecutarse|
|**Sleeping**|`S`|Esperando un evento (interruptible — puede recibir señales)|
|**Sleeping**|`D`|Espera en I/O de disco (ininterrumpible — no puede recibir señales)|
|**Stopped**|`T`|Detenido por señal SIGSTOP o depurador|
|**Zombie**|`Z`|Terminado pero no recogido por el padre (espera `wait()`)|
|**Dead**|`X`|Muerto, pendiente de limpieza por el kernel|
> [!info] Procesos zombie 
> Un proceso zombie no consume CPU ni memoria real, pero sí ocupa una entrada en la tabla de procesos. Si el padre nunca llama a `wait()`, los zombies se acumulan. Al matar al padre, los zombies son adoptados por systemd, que los recoge automáticamente.

---
## 1.4. `/proc` — el filesystem de procesos

`/proc` es un filesystem virtual en RAM que expone el estado interno del kernel y de cada proceso.

```bash
/proc/
├── <PID>/                   ← directorio de cada proceso
│   ├── cmdline              ← comando con el que se lanzó
│   ├── environ              ← variables de entorno (separadas por \0)
│   ├── exe                  ← symlink al ejecutable
│   ├── fd/                  ← descriptores de archivo abiertos
│   ├── maps                 ← regiones de memoria mapeadas
│   ├── mem                  ← memoria del proceso (requiere ptrace)
│   ├── net/                 ← estado de red del proceso
│   ├── status               ← estado: UID, GID, memoria, señales...
│   └── cwd                  ← directorio de trabajo actual (symlink)
├── cpuinfo                  ← información de la CPU
├── meminfo                  ← información de la RAM
├── net/                     ← estado de la red del sistema
│   ├── tcp                  ← conexiones TCP activas
│   ├── udp                  ← conexiones UDP activas
│   └── if_inet6             ← interfaces de red IPv6
├── mounts                   ← filesystems montados
├── version                  ← versión del kernel
└── sys/                     ← parámetros del kernel (sysctl)
```

| Acción                           | Comando                                  |
| -------------------------------- | ---------------------------------------- |
| Comando del proceso 1234         | `cat /proc/1234/cmdline \| tr '\0' ' '`  |
| Archivos abiertos por el proceso | `ls -la /proc/1234/fd/ `                 |
| Variables de entorno del proceso | `cat /proc/1234/environ \| tr '\0' '\n'` |
| Ruta del ejecutable              | `readlink /proc/1234/exe`                |

---
## 1.5. Señales

Las señales son notificaciones asíncronas enviadas a un proceso para indicar un evento.

|Señal|Número|Descripción|Por defecto|
|---|---|---|---|
|`SIGHUP`|1|Colgar terminal / recargar config|Terminar|
|`SIGINT`|2|Interrupción (Ctrl+C)|Terminar|
|`SIGQUIT`|3|Salir con core dump (Ctrl+\)|Volcar y terminar|
|`SIGKILL`|9|Matar inmediatamente (**no se puede ignorar**)|Terminar (forzado)|
|`SIGSEGV`|11|Fallo de segmentación (acceso inválido a memoria)|Volcar y terminar|
|`SIGTERM`|15|Terminar limpiamente (se puede ignorar/capturar)|Terminar|
|`SIGSTOP`|19|Parar proceso (**no se puede ignorar**)|Parar|
|`SIGCONT`|18|Continuar proceso parado|Continuar|
|`SIGUSR1/2`|10/12|Señales de usuario definidas por la aplicación|Terminar|
Tenemos estos comandos:

| Acción                                              | Comando                               |
| --------------------------------------------------- | ------------------------------------- |
| Terminar limpiamente / Forzar terminación (SIGKILL) | `kill -SIGTERM 1234` / `kill -9 1234` |
| Recargar configuración                              | `kill -SIGHUP 1234`                   |
| SIGTERM a todos los procesos nginx                  | `killall nginx`                       |
| SIGTERM a todos los procesos de jessica             | `pkill -u jessica`                    |
| Enviar señal a un grupo de procesos - antes del PID | `kill -SIGTERM -1234`                 |

---
# 2. 🔧 Servicios — systemd

## 2.1. La pila de red del kernel

```
  Aplicación (nginx, curl...)
       │ syscalls: send(), recv(), connect()...
       ▼
  ┌─────────────────────────────────┐
  │   Sockets BSD (API)             │  ← interfaz para las aplicaciones
  ├─────────────────────────────────┤
  │   TCP             UDP           │  ← capa de transporte
  ├─────────────────────────────────┤
  │   IP (IPv4 / IPv6)              │  ← enrutamiento
  ├─────────────────────────────────┤
  │   Netfilter (iptables/nftables) │  ← filtrado, NAT, mangling
  ├─────────────────────────────────┤
  │   Driver de red (e1000, virtio) │  ← interfaz física
  └─────────────────────────────────┘
       │
  eth0, lo, wlan0...
```

---
## 2.2. ¿Qué es systemd?

**systemd** es el sistema de init y gestor de servicios estándar en las distribuciones Linux modernas. Reemplaza al init clásico de SysV. Es el PID 1 del sistema — el primer proceso que arranca el kernel y del que dependen todos los demás.

```
Kernel arranca
     │
     ▼
systemd (PID 1)
     │ lee /etc/systemd/system/ y /lib/systemd/system/
     ├── Monta filesystems
     ├── Configura la red (NetworkManager / systemd-networkd)
     ├── Inicia servicios (nginx, sshd, cron...)
     └── Activa targets (multi-user.target, graphical.target...)
```

---
## 2.3. Units — los bloques de systemd

Todo en systemd se gestiona mediante **units** (archivos de configuración):

|Tipo de unit|Extensión|Descripción|
|---|---|---|
|**Service**|`.service`|Daemon o proceso gestionado|
|**Timer**|`.timer`|Equivalente a cron, dispara services|
|**Socket**|`.socket`|Activa un service cuando hay conexión al socket|
|**Target**|`.target`|Agrupa units (como runlevels de SysV)|
|**Mount**|`.mount`|Punto de montaje del filesystem|
|**Path**|`.path`|Monitoriza un archivo/directorio y activa un service|
|**Device**|`.device`|Dispositivo de hardware|
|**Slice**|`.slice`|Jerarquía de cgroups para agrupar procesos|

Ubicación de los archivos unit
```bash
/lib/systemd/system/       # units del sistema (paquetes instalados)
/etc/systemd/system/       # units del administrador (sobreescribe a lib/)
~/.config/systemd/user/    # units del usuario (--user)
```

---
## 2.4. Anatomía de un archivo `.service`

Por ejemplo `/etc/systemd/system/mi-app.service`
```bash
[Unit]
Description=Mi aplicación web
Documentation=https://mi-app.com/docs
After=network.target         # arrancar después de la red
Requires=postgresql.service  # depende de postgresql
Wants=redis.service          # preferible que redis esté, pero no obligatorio

[Service]
Type=simple                  # simple / forking / oneshot / notify / dbus
ExecStart=/usr/bin/mi-app --config /etc/mi-app/config.yaml
ExecReload=/bin/kill -HUP $MAINPID   # recarga con SIGHUP
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure           # reiniciar si falla
RestartSec=5                 # esperar 5s antes de reiniciar
User=www-data                # usuario bajo el que corre
Group=www-data
WorkingDirectory=/opt/mi-app

# Variables de entorno
Environment=NODE_ENV=production
EnvironmentFile=/etc/mi-app/env   # o desde archivo

# Límites de recursos
LimitNOFILE=65536            # máximo de descriptores de archivo
MemoryMax=512M               # límite de RAM (cgroup)
CPUQuota=50%                 # máximo 50% de CPU

# Seguridad
NoNewPrivileges=yes          # no puede ganar privilegios
PrivateTmp=yes               # /tmp privado
ReadOnlyPaths=/etc           # /etc de solo lectura
ProtectSystem=strict         # filesystem de solo lectura excepto los marcados

[Install]
WantedBy=multi-user.target   # a qué target pertenece (arranca con el sistema)
```

| Tipo      | Descripción                                                   |
| --------- | ------------------------------------------------------------- |
| `simple`  | El proceso de `ExecStart` es el proceso principal. Default    |
| `forking` | El proceso hace fork y el padre termina (daemons clásicos)    |
| `oneshot` | Ejecuta y termina. Útil para scripts de configuración         |
| `notify`  | El daemon avisa a systemd cuando está listo (via `sd_notify`) |
| `dbus`    | Listo cuando adquiere un nombre en el bus D-Bus               |
| `idle`    | Como `simple` pero espera a que no haya otros jobs pendientes |

---
## 2.5. Logs — journalctl

systemd centraliza todos los logs en el **journal** (binario, en `/var/log/journal/`).

| Acción                                            | Comando                                              |
| ------------------------------------------------- | ---------------------------------------------------- |
| Ver todos los logs                                | `journalctl`                                         |
| Logs de un servicio en tiempo real                | `journalctl -u nginx -f`                             |
| Logs de arranque actual / anterior tras un crash  | `journalctl -b` / `journalctl -b -1`                 |
| Solo errores                                      | `journalctl -p err`                                  |
| Filtrar por PID / nombre de proceso               | `journalctl _PID=1234` / `_COMM=nginx`               |
| Eliminar logs de más de 30 días / reducir a 50 mb | `journalctl --vacuum-time=30d` / `--vacuum-size=50M` |

---
## 2.6. Timers — el cron de systemd

Los timers de systemd sustituyen a cron con más control y logging integrado.

```bash
# /etc/systemd/system/backup.timer
[Unit]
Description=Backup diario a las 2:00

[Timer]
OnCalendar=*-*-* 02:00:00      # todos los días a las 2:00
Persistent=true                # ejecutar si se perdió la ejecución anterior
Unit=backup.service            # service que dispara

[Install]
WantedBy=timers.target
```

Sintaxis de `OnCalendar`

| cada hora            | una vez al día (00:00) | lunes 00:00         | días laborables a las 9        | cada 15 minutos     |
| -------------------- | ---------------------- | ------------------- | ------------------------------ | ------------------- |
| `OnCalendar=hourly ` | `OnCalendar=daily`     | `OnCalendar=weekly` | `OnCalendar=Mon..Fri 09:00:00` | `OnCalendar=*:0/15` |

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Script de backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=backup
```

| Acción                                 | Comando                               |
| -------------------------------------- | ------------------------------------- |
| Activar timer                          | `systemctl enable --now backup.timer` |
| Ver todos los timers y cuándo disparan | `systemctl list-timers`               |
| Ver logs de las ejecuciones            | `journalctl -u backup.service`        |

---
## 2.7. cron — el gestor de tareas clásico

Aunque systemd timers son la alternativa moderna, cron sigue siendo muy usado.

| Acción                                      | Comando                                |
| ------------------------------------------- | -------------------------------------- |
| Editar el crontab del usuario actual        | `crontab -e`                           |
| Ver el crontab del usuario actual           | `crontab -l`                           |
| Crontab de root                             | `sudo crontab -e`                      |
| Archivos de cron del sistema / aplicaciones | `cat /etc/crontab` / `ls /etc/cron.d/` |

```bash
# m  h  dom  mon  dow  comando
# │  │   │    │    │
# │  │   │    │    └── día de la semana (0-7, 0 y 7 = domingo)
# │  │   │    └─────── mes (1-12)
# │  │   └──────────── día del mes (1-31)
# │  └──────────────── hora (0-23)
# └─────────────────── minuto (0-59)

*/5  *  *  *  *  /usr/local/bin/check.sh        # cada 5 minutos
0    2  *  *  *  /usr/local/bin/backup.sh        # cada día a las 2:00
0    9  *  *  1-5 /usr/local/bin/informe.sh      # lunes a viernes a las 9:00
@reboot            /usr/local/bin/arranque.sh    # al iniciar el sistema
@hourly            /usr/local/bin/limpieza.sh    # cada hora
```

---

## 🔗 Ver también

- [[Linux; gestión de identidades]]
- [[Linux; escalada de privilegios]]
- [[Suricata]]
- [[Docker]]
- [[Comandos Linux]]
