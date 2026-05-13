> [!abstract] Introducción Linux gestiona el acceso al sistema mediante **usuarios** (identidades individuales), **grupos** (colecciones de usuarios con los mismos permisos) y **capacidades** (capabilities). Cada usuario tiene un identificador único llamado **UID** y se autentica principalmente mediante contraseñas almacenadas como hashes en `/etc/shadow`.

# 1. 🏷️ Nombres e identificadores

## 1.1. UID y GID

Linux identifica a cada usuario con un **UID (User ID)** y a cada grupo con un **GID (Group ID)**. Son números enteros que el kernel usa internamente para todo control de acceso. Los nombres de usuario son solo una traducción legible de esos números.

```c
UID  GID  Grupos suplementarios
│    │    │
▼    ▼    ▼
1001:1001:sudo,developers,docker
```

|Rango de UID|Tipo|Descripción|
|---|---|---|
|`0`|**root**|Superusuario, control total del sistema|
|`1–999`|**Sistema**|Cuentas de daemons y servicios (www-data, nobody, syslog...)|
|`1000+`|**Usuarios**|Cuentas creadas manualmente|
|`65534`|**nobody**|Usuario sin privilegios, usado para accesos anónimos (NFS, etc.)|

## 1.2. Nombres de usuario

Al igual que Windows traduce SIDs a nombres, Linux traduce UIDs. Los archivos clave son:

|Archivo|Contenido|
|---|---|
|`/etc/passwd`|Usuarios del sistema: nombre, UID, GID, home, shell|
|`/etc/group`|Grupos del sistema: nombre, GID, miembros|
|`/etc/shadow`|Hashes de contraseñas (solo legible por root)|
|`/etc/gshadow`|Contraseñas de grupos (raramente usado)|
El `/etc/passwd` sigue este formato
```c
usuario:x:UID:GID:GECOS(nombre completo):directorio home:shell de login

jessica:x:1001:1001:Jessica García:/home/jessica:/bin/bash
        └── x = contraseña en /etc/shadow
```

El `/etc/shaow` sigue este otro
```c
usuario:hash:último_cambio:min:max:aviso:inactivo:expiración

jessica:$6$salt$hash...:19800:0:99999:7:::
   │      │               │   │   │   │
   │      │               │   │   │   └── días de aviso antes de expirar
   │      │               │   │   └── días máximos de validez
   │      │               │   └── días mínimos entre cambios
   │      │               └── días desde epoch del último cambio
   │      └── hash de la contraseña ($6$ = SHA-512)
   └── nombre de usuario
```

| Prefijo   | Cifrado                                 |
| --------- | --------------------------------------- |
| `$1$`     | MD5 (obsoleto)                          |
| `$5$`     | SHA-256                                 |
| `$6$`     | SHA-512 (estándar actual)               |
| `$y$`     | yescrypt (moderno, Ubuntu 22.04+)       |
| `*` o `!` | cuenta bloqueada, sin contraseña válida |

---
# 2. 👤 Tipos de usuarios y grupos

## 2.1. Tipos de usuarios

|Usuario|UID|Descripción|
|---|---|---|
|**root**|0|Superusuario. Control total, sin restricciones. Equivale al Administrador de Windows|
|**Usuarios de sistema**|1–999|Cuentas para daemons y servicios. Sin shell interactiva (`/sbin/nologin` o `/bin/false`)|
|**Usuarios normales**|1000+|Cuentas reales de personas. Shell interactiva, directorio home propio|
|**nobody**|65534|Sin privilegios, sin home. Usado por servicios que necesitan un usuario sin permisos|

> [!warning] root vs sudo 
> En Linux, root es la única cuenta con UID 0. El acceso root suele gestionarse via **sudo**: el usuario normal ejecuta comandos específicos con privilegios elevados sin necesitar la contraseña de root directamente.

## 2.2. Tipos de grupos

En Linux los grupos son más simples que en Windows. Cada usuario tiene:

- Un **grupo principal** (definido en `/etc/passwd`, el GID)
- **Grupos suplementarios** (definidos en `/etc/group`)

| Grupo            | Descripción                                                 |
| ---------------- | ----------------------------------------------------------- |
| **root** (GID 0) | El grupo del superusuario                                   |
| **sudo / wheel** | Miembros pueden ejecutar comandos como root con `sudo`      |
| **shadow**       | Permite leer `/etc/shadow` — muy sensible                   |
| **disk**         | ⚠️ Acceso directo a dispositivos de disco                   |
| **docker**       | ⚠️ Permite usar Docker — equivale a root en la práctica     |
| **adm**          | Puede leer logs del sistema en `/var/log/`                  |
| **lxd**          | ⚠️ Permite usar LXD/LXC — vector de escalada de privilegios |
| **www-data**     | Usuario y grupo del servidor web (Apache, Nginx)            |
| **nogroup**      | Grupo por defecto para procesos sin grupo asignado          |

---
# 3. 🔑 Permisos y capacidades

## 3.1. Permisos Unix (DAC)

Linux usa un modelo de control de acceso discrecional (**DAC**) basado en tres entidades y tres tipos de permiso:

```c
-rwxr-xr--  1  jessica  developers  4096  may 12  archivo.sh
│└──┘└──┘└──┘
│  │   │   └── otros (others)
│  │   └────── grupo propietario
│  └────────── propietario (user)
└───────────── tipo: - archivo / d directorio / l symlink
```

| Permiso | Permiso numérico | Archivo             | Directorio                     |
| ------- | ---------------- | ------------------- | ------------------------------ |
| `r`     | +4               | Leer contenido      | Listar contenido (`ls`)        |
| `w`     | +2               | Modificar contenido | Crear/eliminar archivos dentro |
| `x`     | +1               | Ejecutar            | Entrar al directorio (`cd`)    |

Los permisos se pueden asignar de modo letra para cada uno separado (Ej. `chmod u+x`, el propietario puede ejecutarlo) o por número (Ej. `chmod 777`, todo el mundo tiene permiso de todo)

Luego tenemos los bits especiales

| Bit                                | Sobre ejecutable                                         | Sobre directorio                                           |
| ---------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------- |
| **SUID** (`s` en user, `4xxx`)     | Se ejecuta con el UID del propietario, no del invocador. | Sin efecto especial                                        |
| **SGID** (`s` en group, `2xxx`)    | Se ejecuta con el GID del propietario                    | Archivos nuevos heredan el grupo del directorio            |
| **Sticky** (`t` en others, `1xxx`) | Sin efecto especial                                      | Solo el propietario puede borrar sus archivos (ej: `/tmp`) |
>  ⚠️ Si un binario tiene SUID y pertenece a root, cualquier usuario puede ejecutarlo **con privilegios de root**.  


## 3.2. ACLs (Access Control Lists)

Las ACLs extienden el modelo Unix clásico permitiendo permisos por usuario concreto sobre un archivo, sin cambiar el propietario ni el grupo.

| Acción                                   | Comando                                                         |
| ---------------------------------------- | --------------------------------------------------------------- |
| Ver ACLs de un archivo                   | `getfacl archivo.txt`                                           |
| Dar permisos a un usuario/grupo concreto | `setfacl -m u:jessica:r archivo.txt` (grupo `g:developers:rw `) |
| Eliminar ACL de un usuario               | `setfacl -x u:jessica archivo.txt`                              |
| Eliminar todas las ACLs                  | `setfacl -b archivo.txt`                                        |

## 3.3. Capabilities — el equivalente a privilegios Windows

Las capabilities dividen los privilegios de root en unidades más pequeñas, permitiendo dar a un proceso una capacidad específica sin darle UID 0.

|Capability|Permite|Riesgo|
|---|---|---|
|`cap_setuid`|Cambiar el UID del proceso a cualquier valor, incluyendo 0|🔴 Escalada directa a root|
|`cap_sys_admin`|Casi equivalente a root: montar filesystems, namespaces, ptrace...|🔴 Muy alto|
|`cap_sys_ptrace`|Depurar y acceder a la memoria de cualquier proceso|🔴 Alto|
|`cap_dac_read_search`|Leer cualquier archivo ignorando permisos|🔴 Alto|
|`cap_net_raw`|Crear sockets raw (sniffing, spoofing)|🟡 Medio|
|`cap_net_bind_service`|Bind a puertos < 1024 sin ser root|🟢 Bajo (legítimo)|
|`cap_sys_module`|Cargar módulos del kernel → rootkit|🔴 Crítico|
|`cap_chown`|Cambiar el propietario de cualquier archivo|🔴 Alto|
Por tanto tenemos estos comandos para gestionarlas:

| Acción                                                | Comando                               |
| ----------------------------------------------------- | ------------------------------------- |
| Ver capabilities de un binario                        | `getcap /usr/bin/python3`             |
| Ver todos los binarios con capabilities en el sistema | `getcap -r / 2>/dev/null`             |
| Asignar capability a un binario                       | `setcap cap_net_raw+ep /usr/bin/ping` |
> [!danger] Capabilities en binarios interpretados 
> Si un intérprete como Python o Ruby tiene `cap_setuid`, cualquier script que ejecute puede cambiar su UID a 0 y obtener una shell root. Es una de las técnicas de escalada más silenciosas.
> Ej python, con cap_setuid → `python3 -c "import os; os.setuid(0); os.system('/bin/bash')"`

## 3.4. Atributos extendidos

El sistema de ficheros de Linux permite además activar ciertos atributos especiales. Estos se listan con `lsattr <archivo>`

Para cambiar estos permisos especiales se utiliza `sudo chattr +i archivo.txt` (para hacerlo inmutable). 

**Atributos comunes:**
- `i` (Inmutable): El archivo no puede ser modificado, eliminado, renombrado ni enlazado. Ni siquiera por root.
- `a` (Append Only): El archivo solo puede abrirse en modo escritura para añadir información.

---
# 4. 🔐 Autenticación y autorización en Linux

> [!abstract] No son lo mismo
> Linux separa **autenticación** (¿quién eres?) de **autorización** (¿qué puedes hacer?). La autenticación se gestiona mediante **PAM (Pluggable Authentication Modules)**, y la autorización mediante permisos Unix, ACLs y capabilities.

## 4.1. Cómo funciona el login — nivel de proceso

```
Usuario introduce contraseña
        ▼
   getty / sshd / display manager
        │ llama a PAM
        ▼
   PAM (Pluggable Authentication Modules)
        │ consulta módulos configurados en /etc/pam.d/
        ├── pam_unix.so    → verifica hash en /etc/shadow
        ├── pam_ldap.so    → verifica contra LDAP (si está configurado)
        └── pam_google_authenticator.so → 2FA (si está configurado)
        │
        │ autenticación OK
        ▼
   Crea sesión (fork del proceso de login)
        ▼
   Asigna UID, GID y grupos suplementarios al proceso
        ▼
   Ejecuta el shell del usuario (/bin/bash)
        ▼
   Cada proceso hijo hereda el contexto de seguridad
```

## 4.2. PAM — Pluggable Authentication Modules

PAM es la capa de abstracción de autenticación de Linux. Permite cambiar el método de autenticación sin modificar las aplicaciones.
En el directorio `/etc/pam.d/` tenemos una config de pam por cada servicio, por ejemplo `login` es la config para el login en consola. 

Cada línea en un archivo PAM tiene esta estructura:

```c
tipo    control    módulo    argumentos

auth    required   pam_unix.so   nullok
  │         │          │
  │         │          └── módulo que hace la verificación
  │         └── comportamiento si falla: required/sufficient/optional
  └── tipo de comprobación: auth/account/password/session
```

|Tipo|Función|
|---|---|
|`auth`|Verifica la identidad (contraseña, token, etc.)|
|`account`|Comprueba si la cuenta puede iniciar sesión (expirada, bloqueada...)|
|`password`|Gestiona el cambio de contraseña|
|`session`|Configura el entorno de la sesión (montar home, límites, logs...)|

## 4.3. Delegación de privilegios: sudoers

`sudo` permite a usuarios ejecutar comandos como otro usuario (normalmente root) sin conocer su contraseña. La configuración está en `/etc/sudoers` (editar siempre con `visudo`).

```bash
# Formato de /etc/sudoers:
usuario  host=(usuario_destino)  comando

# Ejemplos:
jessica  ALL=(ALL:ALL) ALL           # jessica puede hacer todo como root
carlos   ALL=(ALL) /usr/bin/apt      # carlos solo puede ejecutar apt como root
%sudo    ALL=(ALL:ALL) ALL           # todos los del grupo sudo pueden hacer todo
www-data ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx   # sin contraseña
```

> [!danger] sudo misconfiguration — escalada habitual 
> Si un usuario puede ejecutar un intérprete, editor o comando que permita ejecución de código con `sudo`, tiene acceso root. Ejemplos clásicos en [GTFOBins](https://gtfobins.github.io/):
> 
> ```bash
> sudo vim     → :!/bin/bash    → shell root
> sudo python3 → os.system()    → shell root
> sudo find    → -exec /bin/bash \; → shell root
> sudo less    → !/bin/bash     → shell root
> ```

## 4.4. Almacenamiento de contraseñas — `/etc/shadow`

Linux no almacena contraseñas en texto plano. Las almacena como hashes en `/etc/shadow`, accesible solo por root.

```bash
jessica:$6$rounds=5000$salt$hash_sha512...:19800:0:99999:7:::
```

El hash incluye el **algoritmo**, la **sal (salt)** y el **hash resultante**, todo en el mismo campo separado por `$`.

> [!danger] Volcado y cracking de `/etc/shadow` 
> Un atacante con acceso a `/etc/shadow` puede intentar crackear los hashes offline:
> 
> ```bash
> unshadow /etc/passwd /etc/shadow > hashes.txt # Combinar passwd y shadow para John/Hashcat
> john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt # Crackear con John the Ripper
> hashcat -m 1800 hashes.txt /usr/share/wordlists/rockyou.txt # Crackear con Hashcat (modo SHA-512 crypt = 1800)
> ```

## 4.5. Tipos de autenticación remota

|Mecanismo|Descripción|
|---|---|
|**Contraseña + PAM**|Método básico. Contraseña verificada contra `/etc/shadow`|
|**SSH con clave pública**|El servidor verifica que el cliente tiene la clave privada correspondiente a la pública registrada en `~/.ssh/authorized_keys`. La contraseña no viaja por la red|
|**Kerberos (GSSAPI)**|Autenticación SSO mediante tickets. Habitual en entornos con Active Directory o FreeIPA|
|**LDAP / SSSD**|Autenticación centralizada contra un servidor LDAP (AD, OpenLDAP). SSSD es el daemon que lo gestiona en Linux|
|**Certificate-based**|Autenticación con certificados X.509, usada en entornos con CA corporativa|

## 4.6. Tipos de sesión y qué dejan en memoria

|Tipo de sesión|Cómo|Deja credenciales en memoria|
|---|---|---|
|**Login interactivo** (consola/TTY)|Contraseña directa|✅ Sí — en la sesión PAM|
|**SSH con contraseña**|Contraseña por red (cifrada)|✅ Sí — en sshd y sesión|
|**SSH con clave pública**|Clave privada del cliente|⚠️ Solo si se usa `ssh-agent`|
|**sudo**|Contraseña del usuario|✅ Cacheada durante `timestamp_timeout` (5 min por defecto)|
|**su**|Contraseña del usuario destino|✅ Sí — mientras dure la sesión|
|**Cron / servicio**|Según la cuenta del servicio|❌ No interactivo, no hay caché|

> [!info] `ssh-agent` y el robo de claves
>  `ssh-agent` mantiene las claves privadas descifradas en memoria para no pedir passphrase en cada conexión. Un atacante con acceso al socket del agente (`$SSH_AUTH_SOCK`) puede usarlo para autenticarse en nombre del usuario sin ver la clave privada directamente. Esto es el equivalente Linux al Pass-the-Ticket de Kerberos.

## 4.7. Proceso de autorización — cómo decide Linux si permite una acción

Cuando un proceso intenta acceder a un recurso, el kernel sigue este orden:

```
Proceso intenta abrir /etc/shadow
        ▼
1. ¿El proceso tiene UID 0 (root)?
   └── Sí → acceso concedido (root ignora DAC)
   └── No → ¿El proceso tiene una capability relevante? (cap_dac_read_search)
           └── Sí → acceso concedido
           └── No → Comprobar permisos Unix (DAC):
                     ├── ¿UID del proceso == UID del archivo? → aplica permisos de propietario
                     ├── ¿GID del proceso está en los grupos del archivo? → aplica permisos de grupo
                     └── En caso contrario → aplica permisos de "otros"
                           └── ¿Hay ACL extendida en el archivo? → verificar ACL
                                  └── ¿SELinux / AppArmor activo? → Verificar política MAC
                                              ▼
                               ✅ Acceso permitido / ❌ Acceso denegado (EACCES / EPERM)
```

> [!info] MAC — SELinux y AppArmor 
> Sobre el modelo DAC, Linux puede añadir una capa de **Mandatory Access Control**:
> - **SELinux** (Red Hat, CentOS, Fedora): modelo basado en etiquetas. Cada proceso y archivo tiene un contexto de seguridad. Muy granular y complejo.
> - **AppArmor** (Ubuntu, Debian): modelo basado en perfiles por aplicación. Más sencillo de configurar que SELinux.
> 
> Ambos pueden denegar acciones que los permisos Unix normales permitirían, añadiendo una capa extra de defensa.

---

## 🔗 Ver también

- [[Escalada de Privilegios Linux]]
- [[Active Directory; Kerberos]]