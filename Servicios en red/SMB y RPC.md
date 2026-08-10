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
