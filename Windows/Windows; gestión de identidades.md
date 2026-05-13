# Índice

[[Windows; gestión de identidades#1. 🏷️ Nombres e identificadores]]
- [[Windows; gestión de identidades#1.1. SID — Security Identifier]]
- [[Windows; gestión de identidades#1.2. Nombres de usuarios]]

[[Windows; gestión de identidades#2. 👤Tipos de usuarios y grupos]]
- [[Windows; gestión de identidades#2.1. Tipos de usuarios]]
- [[Windows; gestión de identidades#2.2. Tipos de usuarios]]

[[Windows; gestión de identidades#3. 🔑 Privilegios]]
- [[Windows; gestión de identidades#3.1. Lista de privilegios]]

[[Windows; gestión de identidades#4. 🔐 Autenticación y autorización en Windows]]
- [[Windows; gestión de identidades#4.1. Modos de autenticación SSPs]]
- [[Windows; gestión de identidades#4.2. Proceso de Autenticación por NTLM]]
- [[Windows; gestión de identidades#4.3. Tipo de logins]]

---
> [!abstract] Introducción
>  Windows gestiona el acceso al sistema mediante **usuarios** (identidades individuales), **grupos** (colecciones de usuarios con los mismos permisos) y **privilegios** (capacidades específicas asignadas a ambos). Cada usuario tiene un identificador único llamado **SID** y se autentica mediante hashes **NTLM**.

## 1. 🏷️ Nombres e identificadores

### 1.1. SID — Security Identifier

Cada usuario tiene un **SID**, su huella digital única dentro del sistema. Windows internamente gestiona los usuarios mediante su SID, pero para hacer más sencilla la administración, este se  traduce a un nombre (samAccountName). 
Tiene tres partes:

```c
S-1-5-21-<subautoridades>-<RID>
│ │ │  │                   │
│ │ │  └── 21 = usuarios   └── Identificador de cuenta
│ │ └── 5 = NT Authority
│ └── 1 = versión
└── S = SID
```

| Parte              | Descripción                                 | Ejemplo                           |
| ------------------ | ------------------------------------------- | --------------------------------- |
| **Prefijo**        | Siempre `S-1-5-21` para usuarios de dominio | `S-1-5-21`                        |
| **Subautoridades** | Serie numérica que identifica el dominio    | `3623811015-3361044348-30300820`  |
| **RID**            | Identifica el rol de la cuenta              | `500` (Admin), `1001+` (usuarios) |

> [!tip] RIDs relevantes
> 
> - `500` → Administrador built-in
> - `501` → Invitado
> - `1000+` → Usuarios creados manualmente

## 1.2. Nombres de usuarios

A parte del SID, existen otros nombres que se usan para identificar a los usuarios y que utilizan las personas:

> [!example]+ Friendly Name
> Es un alias que permite identificar fácilmente a un usuario y que se muestra en la interfaz del usuario y en las ACLs de archivos y carpetas. El sistema internamente traduce estos alias a SIDs.
> Ej  `juan`

> [!example]+ sAMAccountName
> Es el nombre de inicio de sesión "clásico" o de "nivel inferior" (down-level). Tiene un límite de 20 caracteres y tienen dos partes separadas por un backslash "`\`"
>  
| Formato                 | Uso                                       | Ejemplo                   |
| ----------------------- | ----------------------------------------- | ------------------------- |
| `<dominio>\<usuario>`   | Usuarios del sistema o dominio            | `PC_juan\juan` |
| `BUILTIN\<grupo>`       | Grupos predefinidos del sistema           | `BUILTIN\Guests`          |
| `NT AUTHORITY\<cuenta>` | Cuentas del sistema que ejecutan procesos | `NT AUTHORITY\SYSTEM`     |

> [!example]+ User Principal Name (UPN)
> Es el formato de inicio de sesión moderno, similar a un correo electrónico (ej. `usuario@dominio.com`), utilizado principalmente en entornos de Active Directory y cuentas de Microsoft. 

> [!example]+ Display Name
> Es el nombre completo que aparece en la pantalla de bloqueo o en el menú de inicio. No se utiliza para la autenticación técnica interna, sino meramente para la identificación visual del usuario. 
> Ej `"Juan Cuesta"`

> [!tip] ¿Dónde ver usuarios y grupos? `Win + R` → `compmgmt.msc` → **Administración de equipos** → Usuarios y grupos locales

---
# 2. 👤Tipos de usuarios y grupos

## 2.1. Tipos de usuarios

Encontramos estos usuarios:

| Usuario                | Privilegios | Descripción                                                                                                                            |
| ---------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Administrador**      | 🔴 Altos    | Control total del sistema. Creado en la instalación, deshabilitado por defecto para login directo. Se accede a sus privilegios via UAC |
| **Usuario estándar**   | 🟡 Bajos    | Uso normal del sistema. No puede hacer cambios importantes sin aprobación del Admin                                                    |
| **Invitado**           | 🟢 Mínimos  | Acceso provisional. **Desactivado por defecto** en versiones modernas                                                                  |
| **Default User**       | —           | Plantilla oculta que se copia al crear nuevas cuentas                                                                                  |
| **WDAGUtilityAccount** | 🟢 Bajos    | Desde Windows 10. Ejecuta apps en un contenedor aislado para proteger el sistema                                                       |

---
## 2.2. Tipos de grupos

> [!example]+ Grupos de acceso general
| Grupo                   | Descripción                                                                                                           |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **Administrators**      | Pueden realizar tareas administrativas via UAC. El primer usuario creado en la instalación entra aquí automáticamente |
| **Users**               | Usuarios estándar. Pueden usar el sistema y ejecutar apps, sin tocar configuraciones críticas                         |
| **Guests**              | Acceso anónimo y muy limitado. El usuario _Invitado_ pertenece aquí                                                   |
| **Advanced Users**      | Tareas administrativas limitadas (Ej: modificar configuración de red)                                                 |
| **Authenticated Users** | Todos los usuarios que han iniciado sesión con credenciales válidas                                                   |
| **Everyone**            | Todos: autenticados y no autenticados. ⚠️ Asignar permisos aquí afecta a cualquiera                                   |

> [!example]+ Grupos de acceso remoto
> | Grupo                       | Protocolo | Descripción                                                             |
| --------------------------- | --------- | ----------------------------------------------------------------------- |
| **Remote Desktop Users**    | RDP       | Pueden conectarse remotamente con escritorio gráfico                    |
| **Remote Management Users** | WinRM     | Pueden conectarse remotamente por línea de comandos (PowerShell remoto) |

> [!example]+ Grupos especializados
> | Grupo                         | Descripción                                                                      |
| ----------------------------- | -------------------------------------------------------------------------------- |
| **Event Log Readers**         | Pueden leer el registro de eventos. Útil para administradores y equipos SOC      |
| **Operadores** (varios tipos) | Gestión delegada de tareas específicas: servicios, backups, control de acceso... |
| **IIS_IUSRS**                 | Permisos para ejecutar aplicaciones web en servidores IIS                        |

---
# 3. 🔑 Privilegios

## 3.1. Lista de privilegios

Los privilegios son capacidades específicas asignadas a usuarios o grupos, independientemente de los permisos sobre archivos.

| Privilegio                      | Permite                                          | Riesgo   |
| ------------------------------- | ------------------------------------------------ | -------- |
| `SeImpersonatePrivilege`        | Ejecutar procesos suplantando al Admin           | 🔴 Alto  |
| `SeDebugPrivilege`              | Acceder a la memoria de cualquier proceso        | 🔴 Alto  |
| `SeBackupPrivilege`             | Leer archivos ignorando sus ACLs                 | 🔴 Alto  |
| `SeRestorePrivilege`            | Escribir archivos ignorando sus ACLs             | 🔴 Alto  |
| `SeTakeOwnershipPrivilege`      | Apropiarse de cualquier recurso del sistema      | 🔴 Alto  |
| `SeLoadDriverPrivilege`         | Cargar drivers con altos privilegios             | 🔴 Alto  |
| `SeSecurityPrivilege`           | Administrar el registro de auditoría y seguridad | 🔴 Alto  |
| `SeNetworkLogonRight`           | Conectarse al sistema por red (SMB, NetBIOS...)  | 🟡 Medio |
| `SeRemoteInteractiveLogonRight` | Conectarse por RDP                               | 🟡 Medio |
| `SeIncreaseWorkingSetPrivilege` | Aumentar la RAM asignada a procesos              | 🟡 Medio |
| `SeShutdownPrivilege`           | Apagar el sistema localmente                     | 🟢 Bajo  |
| `SeRemoteShutdownPrivilege`     | Apagar el sistema remotamente                    | 🟢 Bajo  |
| `SeTimeZonePrivilege`           | Cambiar la zona horaria                          | 🟢 Bajo  |
> [!warning] Abuso de privilegios
>  Privilegios como `SeImpersonatePrivilege` o `SeDebugPrivilege` son vectores habituales de escalada de privilegios. Los  Potato attacks o herramientas como Mimikatz los explotan directamente.

---
# 4. 🔐 Autenticación y autorización en Windows

> [!abstract] Autenticación y autorización en Windows  
> Windows separa dos conceptos fundamentales: **autenticación** (comprobar que el usuario es quien dice ser) y **autorización** (qué acciones puede realizar sobre cada recurso).

**Windows utiliza autenticación SSO (Single Sign-On)**, en la que el usuario se autentica una vez y accede a múltiples recursos sin volver a introducir credenciales. Esto funciona de la siguiente manera:

1. **LSASS.exe es el proceso que maneja la autenticación**, la cual varía segun el SSP elegido (proveedor de seguridad), como Kerberos o por NTLM.
    
2. **Una vez validadas las credenciales, el sistema crea una sesión de inicio de sesión (logon session)**, que puede ser local o remota según el tipo de acceso.
    
3. **A partir de esa sesión se genera un token de acceso**, que contiene la identidad del usuario y su _security context_ (sus grupos, privilegios y permisos).
    
4. **Cada vez que el usuario quiere realizar una acción**, como abrir un archivo o ejecutar un programa, **el sistema utiliza las credenciales almacenadas en memoria para la autenticación y crea un proceso con una copia asociada de su token**.
    
5. **Autorización →** Cuando el proceso intenta acceder a un recurso, el sistema compara el token con las ACL ("Access Control Lists") del objeto; dentro de ellas, las ACE ("Access Control Entries") determinan si el acceso se permite o se deniega.

> [!warning] El proceso LSASS.exe - Local Security Authority Subsystem Service
> `lsass.exe` es el proceso central de seguridad de Windows. Corre con privilegios de **SYSTEM** y es el responsable de la autenticación, la creación y gestion de sesiones y la emisión de tokens.
> - **Este proceso almacena en memoria las credenciales usadas, por tanto es el objetivo principal de los atacantes**, por tanto está protegido con meidas de seguridad como **PPL** (protege su memoria de ser leida por otros procesos) y **credential guard** (aisla las credenciales en un entorno virtual separado del kernel)

> [!error] SAM Database - Security Account Manager
> La **SAM** es la base de datos local donde Windows almacena los hashes NTLM de los usuarios locales. Se encuentra en la ruta `C:\Windows\System32\config\SAM` . Esta base de datos está bloqueada mientras Windows está en ejecución y cifrada con una clave almacenada en el registro /(`SYSTEM` hive). LSASS la lee al arrancar para poder autenticar usuarios.
> - Un usuario administrador o con `SeBackupPrivilege` puede crear un volcado de la sam con `reg save HKLM\SAM sam.bak` y `reg save HKLM\SYSTEM system.bak`. También es accesible desde un Live CD o si se obtiene el archivo desde una shadow copy.

---
## 4.1. Modos de autenticación: SSPs

Para la autenticación, existen varias soluciones distintas, aunque la mayoría son para entornos empresariales (Active Directory)

| Mecanismo                      | Descripción                                                                                                                                                                                                |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **NTLM**                       | Método que transforma una contraseña en un hash que se envía y valida                                                                                                                                      |
| **Kerberos**                   | Sistema de autenticación basado en tickets. El usuario se autentica una vez y luego utiliza tickets cifrados para acceder a distintos servicios sin reenviar la contraseña.                                |
| **Certificate-based (PKINIT)** | Variante de Kerberos donde la autenticación inicial se realiza con certificados digitales en lugar de contraseña.                                                                                          |
| **Smart Card**                 | Uso de certificados almacenados en una tarjeta física. Funciona como PKINIT, pero el certificado está protegido por hardware.                                                                              |
| **Windows Hello**              | Método de autenticación local mediante biometría o PIN que desbloquea una clave criptográfica almacenada en el dispositivo, sin enviar la contraseña a la red. Esta clave descifra el contenido del disco. |

---
## 4.2. Proceso de Autenticación por NTLM

Windows no almacena contraseñas, sino **hashes NTLM** en una base de datos protegida llamada **SAM**, leída por el proceso `lsass.exe`.

```
  [Cliente]                                                                                    [Servidor]
     │                                                                                             │
     │──── 1. Introduce contraseña → se convierte en hash NTLM ──────────────────────────────────> │
     │                                                                                             │
     │ <── 2. Challenge (cadena aleatoria) ────────────────────────────────────────────────────────│
     │                                                                                             │
     │──── 3. Response: Cifra el challenge con su hash NTLM ─────────────────────────────────────> │
     │                                                                                             │
     │ <── 4. El servidor cifra el mismo challenge con el hash almacenado en la SAM y compara ─────│
     │                                                                                             │
     │ <── 5. Acceso concedido (si coinciden) ─────────────────────────────────────────────────────│
```

Existen dos versiones de NTLM
- **NTLMv1**: El método mas antiguo, considerado obsoleto e inseguro por utilizar un cifrado muy débil. 
- **NTLMv2**: Es el método actual. Es más robusto, pero vulnerable a **Pass-the-Hash** 

> [!error] Cracking offline de hashes NTLM
> **Un atacante puede interceptar el challenge y descifrarlo probando múltiples contraseñas posibles en un ataque por fuerza bruta**. Para cada contraseña candidata, el atacante calcula su hash, lo usa para cifrar el desafío interceptado y compara el resultado con la respuesta real. Si los valores coinciden, el atacante ha encontrado la contraseña en texto plano

> [!error] Pass The Hash
> Como lo que utiliza la autenticación de windows para completar el "desafío-respuesta" de la autenticación es el jasj, el atacante puede usarlo directante para acceder al sistema. Ademas como los hahes no cambian a menos que se cambie de contraseña, un atacante puede reutilizarlo las veces que quiera.

El formato que usan los hashes NTLMv2 es este:
```c
usuario:RID:hash_LM:hash_NT:::

jessica:1001:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
         │    └── siempre igual (LM obsoleto)    └── este es el que se crackea
         └── RID de la cuenta
```

> [!info] El hash LM El hash LM siempre es el mismo (`aad3b435...`) porque está obsoleto y desactivado. El que importa es el **hash NT**.

---
## 4.3. Tipo de logins

Cuando un usuario se autentica, Windows registra el **tipo de logon**. Esto es crítico en seguridad porque determina **qué credenciales se almacenan en memoria**.

| Logon Type            | ID  | Descripción                                                                                        | Qué queda en memoria                                          |
| --------------------- | --- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Interactive**       | 2   | Login físico en el equipo (teclado).                                                               | ✅ Se cachean las credenciales del usuario                     |
| **Network**           | 3   | Se accede por red a un recurso compartido, sin crear una sesión interactiva                        | ❌ Se crea un token temporal, pero no se cachean credenciales  |
| **Batch**             | 4   | Tareas programadas por procesos automáticos                                                        | ⚠️ Solo si la tarea fue configurada con usuario y contraseña. |
| **Service**           | 5   | Arranque de servicios de Windows (se ejecutan bajo una cuenta de servicio o una cuenta de usuario) | ⚠️ Solo si se ejecutan bajo una cuenta de usuario normal      |
| **NetworkCleartext**  | 8   | Red con credenciales en texto claro (IIS básica)                                                   | ✅ Se envían las credenciales por la red                       |
| **NewCredentials**    | 9   | `runas /netonly` — usa credenciales de otro usuario para acceder a un servicio red                 | ✅ Sí, en la máquina remota                                    |
| **RemoteInteractive** | 10  | RDP — escritorio remoto                                                                            | ✅ Sí, es una sesión completa                                  |
| **CachedInteractive** | 11  | Login offline con credenciales cacheadas                                                           | ✅ Sí, en la máquina local                                     |
> [!tip] Restricted Admin Mode (RDP) 
> Desde Windows 8.1 existe el modo **Restricted Admin** para RDP: la sesión se abre con un Impersonation Token en lugar de credenciales completas, evitando que queden en memoria. Se activa con `mstsc /restrictedAdmin`.

> [!warning] El problema del "Double Hop"
> El **Double Hop** es la situación en la que un usuario se conecta a un **Servidor A**, y desde ahí quiere acceder a un **Servidor B** con sus credenciales. El segundo salto es el problema. ¿Porqué? Porque al autenticarse en el servidor A, este crea el token pero no almacena las credenciales, por tanto no puede autenticarse en el servidor B.
> - Para resolver esto, existen soluciones como la delegación de kerberos o CredSSP (reenvío de credenciales) pero ambas soluciones plantean cierto riesgo.

> [!info] Impersonation 
> Un servicio puede recibir una conexión de un usuario y crear un **Impersonation Token** para actuar en su nombre sin tener sus credenciales. Esto es la base de la delegación y también del abuso de `SeImpersonatePrivilege` (Ej: Potato Attacks).

---
## 🔗 Ver también

- [[Active Directory]]
- [[Active Directory; Kerberos]]
- [[Windows; Escalada de privilegios]]