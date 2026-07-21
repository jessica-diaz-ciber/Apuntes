# 📂 Active Directory

Un Directorio Activo (Active Directory o AD) es una estructura jerárquica que permite centralizar la gestión de recursos, usuarios y servicios en una red empresarial.

Esto nos evita tener que configurar individualmente decenas de equipos, lo que sería extremadamente lento, tedioso y propenso a errores.

El AD se basa en un servidor llamado **DC (Controlador de Dominio)**, que se encarga de:

- Almacenar una base de datos llamada **Data Store** (o Directorio), que organiza de manera estructurada y jerárquica todos los recursos de la red (usuarios, sistemas, archivos, impresoras…)
- Proporcionar una serie de servicios que permiten operar en este entorno, incluyendo funciones clave

| Puerto  | Servicio     | Descripción                                                                                                      |
| ------- | ------------ | ---------------------------------------------------------------------------------------------------------------- |
| **53**  | **DNS**      | Traduce nombres de equipos y dominios a direcciones IP y es fundamental para encontrar recursos en el directorio |
| **88**  | **Kerberos** | Protocolo principal de autenticación en AD, basado en tickets para acceder a servicios de la red                 |
| **135** | **MS-RPC**   | Facilita la comunicación entre distintos servicios de Windows y AD                                               |
| **389** | **LDAP**     | Protocolo utilizado para consultar y administrar objetos del dominio, como usuarios, grupos y equipos            |
| **445** | **SMB**      | Permite el acceso a recursos compartidos críticos del dominio                                                    |

> No existe un solo DC sino varios, réplicas que comparten información a tiempo real para evitar perder el acceso a servicios si uno de ellos es atacado e inutilizado. El proceso de replicación se realiza de manera automática y se pueden gestionar como un solo DC.

---

## 🌳 Organización del directorio

El Directorio Activo se organiza de manera jerárquica.

#### Objetos

**Un objeto es cualquier elemento que existe dentro del directorio.** Por ejemplo, puede ser un usuario, un equipo, una impresora, un grupo o una carpeta compartida.

**Cada objeto tiene una serie de atributos**, es decir, datos que lo describen. Esos atributos están definidos por el **Schema**, que actúa como una plantilla. Esto permite que todos los objetos de un mismo tipo tengan la misma estructura de atributos posibles, dándole cohesión al sistema.

> _Por ejemplo, todos los objetos de tipo usuario disponen de atributos comunes como nombre, apellidos, correo electrónico o fecha de creación.

**El Data Store se encarga de asegurar que todos los objetos cumplan las reglas de este Schema.**

En el AD existen dos tipos principales de objetos:

- **Los recursos**, que representan los elementos disponibles en la red (carpetas, equipos o impresoras)
- **Las entidades o security objects**, que representan a quienes acceden a dichos recursos, como usuarios, roles y cuentas de servicio

---
#### Unidades Organizativas (OUs)

**Las Unidades Organizativas u OUs son contenedores que sirven para agrupar objetos relacionados.** Esto permite organizar mejor el directorio y administrar varios objetos de forma conjunta.

Las OUs suelen crearse siguiendo la estructura de la empresa, por ejemplo: por departamentos, sedes o funciones. Gracias a estas OU se pueden aplicar permisos, tareas o directivas a varios objetos a la vez.

> _Por ejemplo, se puede dar permiso al departamento de IT para restablecer contraseñas de los usuarios de la OU "RRHH" y "Oficina", pero no de otras OUs. Así se limita su alcance y se evita que tengan control sobre toda la empresa._

---
#### Dominio

**Un dominio es una unidad independiente dentro del AD en la que se administran usuarios, equipos y recursos bajo unas mismas reglas de seguridad.**

Dentro de un dominio, todos los objetos comparten las mismas políticas, la misma base de datos y un nombre común gestionado mediante DNS.

El dominio se encarga de identificar a los usuarios (autenticación) y de decidir a qué recursos pueden acceder, funcionando como un entorno de administración centralizada y separado de otros dominios.

> _Por ejemplo, pueden existir dominios como `BILBAO.L4H.LOCAL` y `FINANZAS.L4H.LOCAL`, cada uno con sus propios usuarios, recursos y políticas._

---
#### Bosque

El bosque es la estructura más grande de Active Directory. Sirve para unir varios dominios dentro de una misma organización. Estos dominios compartirán **el mismo schema, configuración y catálogo global.**

Representa el límite máximo de seguridad y confianza, ya que todos los dominios dentro de un bosque confían entre sí de forma implícita, permitiendo la administración y el acceso a recursos de manera integrada.

**Hay que pensar en el bosque como si fuera la empresa completa, y los dominios fueran sus distintas sedes o departamentos grandes

> Tanto `BILBAO.L4H.LOCAL` como `FINANZAS.L4H.LOCAL` forman parte del mismo bosque: `L4H.LOCAL`

Por último, aunque lo más habitual es que una empresa tenga un único bosque, en algunos casos se crean múltiples bosques para aislar ciertos sectores, como sucursales ubicadas en países con normativas específicas que exigen infraestructuras separadas.

---
## Crear un AD

Para montar un directorio activo, debemos contar con una máquina Windows Server a la que asignemos una IP estática.

```bash
# 1. Primero hay que renombrar la máquina con un nombre descriptivo (y reiniciar el sistema)
Rename-Computer -NewName AD01 -Restart

# 2. Después, hay que instalar los servicios del directorio activo
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# 3. Y unir a nuestra máquina a un nuevo dominio del que será DC
Install-ADDSForest -DomainName dominio.local -DomainNetBIOSName DOMINIO
```

Al haber hecho esto, nos autenticaremos DENTRO del dominio. Esto se ve gracias al prefijo del usuario: `DOMINIO/usuario`

> Si desactivamos el firewall, desde kali podremos ver todos los servicios activos `Set NetFirewallProfile -Profile Domain,Public,Private -Enabled False`

---
## Unir una segunda máquina  

Si queremos unir un segundo equipo al dominio, lo que no debemos hacer es CLONAR la máquina virtual, porque se clonará también su SID y esto causará un conflicto. Por tanto debemos configurar una máquina windows 10 server nueva.

```bash
# 1. Primero hay que renombrar la máquina 
Rename-Computer -NewName AD02 -Restart

# 2. Luego, obtenemos el nombre del adaptador de red para configurar el DC como DNS
netsh interface ipv4 show interface # Sale "Ethernet0"
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses "<IP_DC>"

# 3. Uniremos el equipo al dominio con la contraseña del administrador del DC
Add-Computer -DomainName l4h.local -Credential L4H\Administrator -Restart
```

----

## 🧩 Tipos de objetos

Como hemos comentado, todo elemento dentro de Active Directory (AD) es un objeto, ya sea un recurso o una security entity.

Cada objeto pertenece a una clase, la cual define un conjunto de atributos comunes según lo establecido en el schema. Por lo tanto, dos objetos de la misma clase comparten los mismos atributos, aunque sus valores puedan diferir.

- Todas las security entities tienen asociado un SID (Security Identifier). El SID es el identificador interno que utiliza el sistema para reconocer a un usuario u objeto y es empleado por componentes como LSASS.
- En cambio, el nombre o samAccountName es el identificador legible que utilizamos las personas para administrar el sistema.

----
#### Leaf Objects
Son objetos que no pueden contener otros objetos en su interior.

| Nombre                     | Descripción                                                                                                            | Ejemplo                                     | ObjectClass              |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------ |
| Usuario                    | Security Principal que representa a una persona o servicio. Tiene GUID y SID; algunos se usan como cuentas de servicio | juan.cuesta                                 | user                     |
| Contacto                   | Representa usuarios externos, con menos atributos. No es Security Principal ni tiene SID                               | Contacto externo de un partner              | contact                  |
| Foreign Security Principal | Representa usuarios de otros bosques añadidos a grupos del dominio actual, conservando su SID original                 | Usuario de otro dominio agregado a un grupo | foreignSecurityPrincipal |
| Impresora                  | Recurso disponible en el AD. No es Security Principal; tiene GUID, pero no SID                                         | Impresora de red corporativa                | printQueue               |
| Computadora                | Representa un equipo unido al dominio. Es Security Principal y tiene GUID y SID                                        | PC-01                                       | computer                 |

----
#### ContainerObject
**Pueden contener otros objetos, ya sean leaf objects u otros container objects.

| Nombre   | Descripción                                                                                             | Ejemplo                           | ObjectClass        |
| -------- | ------------------------------------------------------------------------------------------------------- | --------------------------------- | ------------------ |
| Grupo    | Security Principal que agrupa usuarios y otros grupos (nested groups) para asignar permisos compartidos | Domain Admins                     | group              |
| Built-in | Contenedor que almacena los grupos predeterminados creados automáticamente por AD                       | Builtin\Administrators            | container          |
| Dominio  | Contenedor principal que alberga usuarios, computadoras, OUs y grupos, con reglas y políticas propias   | L4H.local                         | domainDNS          |
| OU       | Contenedor lógico que agrupa objetos para facilitar la administración y aplicación de políticas         | OU=IT,  <br>DC=L4H,  <br>DC=local | organizationalUnit |

----
#### Otros

| Nombre             | Descripción                                                                                                      | Ejemplo                 | ObjectClass |
| ------------------ | ---------------------------------------------------------------------------------------------------------------- | ----------------------- | ----------- |
| Domain Controller  | Computadora que proporciona servicios de autenticación y autorización dentro del dominio                         | DC01.L4H.local          | computer    |
| Carpeta compartida | Recurso compartido de una computadora, accesible según permisos definidos. Tiene GUID y no es Security Principal | \\FS01\Datos            | volume      |
| Site               | Agrupa computadoras según su ubicación de red para optimizar la replicación entre controladores de dominio       | Default-First-Site-Name | site        |

---
## 👤 Crear usuarios

Para crear usuarios podemos hacerlo de dos maneras.

Con la interfaz gráfica es super fácil. En el DC tenemos que abrir `Active Directory Users and Computers` y tendremos una vista de todos los usuarios y grupos del directorio. 

Podemos hacer click derecho en cada uno y veremos opciones como `Añadir a un grupo`, `Desactivar la cuenta`, `Resetear (Cambiar) la contraseña`, `Mandar Email`, `Renombrar` o `Propiedades`, dónde podremos cambiar características más avanzadas.

Mediante powershell, también podemos gestionar usuarios y grupos.

| Acción                                                       | Comando                                                                                                                                         |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Crear un nuevo usuario                                       | `New-ADUser -Name "Juan Perez" -SamAccountName jperez -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) -Enabled $true` |
| Buscar un usuario y mostrar todas sus propiedades            | `Get-ADUser jperez -Properties *`                                                                                                               |
| Hacer que la contraseña pueda caducar                        | `Set-ADUser jperez -PasswordNeverExpires $false`                                                                                                |
| Obligar al usuario a cambiar la contraseña al iniciar sesión | `Set-ADUser jperez -ChangePasswordAtLogon $true`                                                                                                |
| Crear un nuevo grupo                                         | `New-ADGroup -Name "IT" -GroupScope Global -GroupCategory Security -Path "OU=Grupos,DC=dominio,DC=local"`                                       |
| Añadir un usuario a un grupo                                 | `Add-ADGroupMember -Identity "IT" -Members jperez`                                                                                              |
| Consultar los miembros de un grupo                           | `Get-ADGroupMember IT`                                                                                                                          |
| Eliminar un usuario de un grupo                              | `Remove-ADGroupMember -Identity "IT" -Members jperez -Confirm:$false`                                                                           |

---
## 🔍 LDAP y el DIT

El directorio es, en esencia, una base de datos que almacena y organiza todos estos objetos en una estructura jerárquica en forma de árbol, conocida como DIT (Directory Information Tree).

Esta base de datos se accede y gestiona mediante un protocolo especializado llamado LDAP (Lightweight Directory Access Protocol), el cual permite consultar, compartir y administrar la información del directorio de forma eficiente.

En terminología LDAP, todos los objetos del árbol se denominan entries. Cada entry pertenece a una clase ("objectClass"), que determina los atributos que puede contener (Ej: mail, uid, memberOf).

Para poder identificar y localizar cada objeto dentro del árbol tenemos:

| Nombre             | Servicio                                                                    | Descripción                                                                                                                                               |
| ------------------ | --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RDN**            | Nombre relativo dentro de un contenedor                                     | **CN**: usuarios y computadoras `cn=Juan Perez`<br>**OU:** para las unidades organizativas. `ou=Usuarios`<br>**DC:** para el dominio. `dc=empresa,dc=com` |
| **DN**             | Representa su ruta completa dentro del directorio,                          | `cn=Juan Perez,ou=Usuarios,dc=empresa,dc=com`                                                                                                             |
| **FDQN**           | Es el nombre completo de un equipo dentro de la jerarquía del dominio (DNS) | `DC01.TXAPELATECH.LOCAL`                                                                                                                                  |
| **GUID**           | Identificador único e inmutable de 128 bits                                 | `550e8400-e29b-41d4-a716-446655440000`                                                                                                                    |
| **SID**            | Identificador único de cada security principal                              | `S-1-5-21-3623811015-3361044348-30300820-1107`                                                                                                            |
| **SamAccountName** | nombre de inicio de sesión tradicional en entornos Windows                  | `TXAPELATECH\jsanabria`                                                                                                                                   |
| UPN                | Otro identificador de inicio de sesión para el usuario                      | `jsanabria@txapelatech.local`                                                                                                                             |

---
LDAP emplea **filtros LDAP** para las consultas, diseñados específicamente para trabajar buscando la posición de los objetos dentro del árbol DIT. Toda búsqueda LDAP se define a partir de tres elementos principales:

1. **Base DN**: indica la ruta desde la que se inicia la búsqueda dentro del directorio
2. **Filtro LDAP**: especifica las condiciones que deben cumplir los objetos para ser devueltos (por ejemplo, tipo de objeto o valor de un atributo)
3. **Atributos a devolver**: qué datos concretos de cada entrada se quieren obtener como resultado de la búsqueda

Las consultas LDAP siempre van entre paréntesis con la sintaxis: `(<clave>=<valor>)`.

| Regla                     | Ejemplo       | Explicación del ejemplo                        |
| ------------------------- | ------------- | ---------------------------------------------- |
| **Igualdad**              | `(cn=jperez)` | Buscamos al usuario Juan Perez                 |
| **Presencia de atributo** | `(mail=*)`    | Todos los objetos con el atributo mail relleno |
| **Coincidencia parcial**  | `(cn=Juan*)`  | Todos los "Juanes"                             |

Para consultas complejas, se combinan varias consultas simples concatenadas por un operador AND u OR

| Regla   | Ejemplo                                 | Explicación del ejemplo                             |
| ------- | --------------------------------------- | --------------------------------------------------- |
| **AND** | **(&(ou=contabilidad)(cn=Juan*))**      | Usuario que sea del ou contabilidad y se llame Juan |
| **OR**  | **(\|(ou=contabilidad)(ou=direccion))** | Usuarios de dirección o contabilidad                |
| **NOT** | **(!(cn=admin))**                       | Todos menos el admin                                |

Las consultas LDAP se hacen con el comando Get-ADObject, que admite estos parámetros:
```bash
Get-ADObject
  -LDAPFilter "(objectClass=user)"
  -SearchBase "OU=IT,DC=empresa,DC=local"
  -Properties samAccountName,mail,whenCreated
```

---
## 🔁 Replicación

En el DC existe un archivo llamado **NTDS.dit**, que es la base de datos física de Active Directory. Dentro de él se almacenan varias particiones lógicas o _naming contexts_, entre ellas Domain, Configuration y Schema.

- **Domain**: contiene los objetos del dominio, como usuarios, grupos, equipos, impresoras, unidades organizativas (OUs), etc. y sus atributos. Parte de esos atributos son especialmente sensibles, como los hashes NTLM de los usuarios.
- **Configuration**: almacena información sobre la estructura del bosque, los dominios, los sitios y los servicios.
- **Schema**: define qué tipos de objetos existen en Active Directory y qué atributos puede tener cada uno.

En una empresa normalmente no existe un único DC, sino varios. Todos ellos mantienen copias de estas particiones y deben permanecer sincronizados, sin discrepancias entre sí. Si cambia cualquier dato en un controlador de dominio, el resto debe recibir ese cambio para que la información sea coherente en todo el entorno. Esto es fundamental para evitar problemas de autenticación, autorización y acceso a recursos.

> _Por ejemplo, un usuario debe existir con la misma contraseña, pertenencia a grupos y resto de atributos en todos los controladores de dominio, de modo que pueda autenticarse correctamente desde cualquier sede o servicio de la organización._

Para evitar discrepancias, los DCs se sincronizan entre sí mediante un proceso llamado **replicación**. En lugar de copiar toda la base de datos cada vez, solo se replican los cambios realizados en una partición concreta, lo que reduce el tráfico y hace el proceso más eficiente. Esta replicación se realiza mediante el protocolo remoto de replicación de Active Directory, conocido como **Directory Replication Service Remote Protocol (MS-DRSR)**.

En este contexto entran en juego una serie de privilegios especiales de replicación:

- **Replicating Directory Changes (Get Changes)**: permite solicitar los cambios de una partición. De forma simplificada, equivale a decir: _"déjame ver qué ha cambiado"_.
- **Replicating Directory Changes All (Get Changes All)**: es el privilegio más sensible, ya que permite replicar también **datos secretos del dominio** (los hashes NTLM de los usuarios en la partición Domain). De forma simplificada, equivale a: _"déjame ver también la parte sensible, no solo lo básico"_.

>  Estos privilegios suelen estar disponibles para cuentas altamente privilegiadas, como las de Domain Admins, o para ciertos servicios legítimos de sincronización. El problema aparece cuando una cuenta con estos derechos es comprometida, o cuando dichos permisos se delegan a usuarios o grupos que no deberían tenerlos.

---
## 👥 Grupos y privilegios

Recordamos que tenemos una serie de grupos de administración con privilegios clave. Si se compromete una cuenta que forme parte de estos grupos, tanto directa como indirectamente, podremos realizar una escalada de privilegios.

Puede que nuestro usuario, por medio de un permiso especial, pueda obtener el control de la cuenta de otro usuario con mayor nivel de acceso en la red.

Existen dos tipos principales de grupos en Active Directory:

- ⚠️ **Security Groups**: se utilizan para agrupar usuarios (u otros objetos) con los mismos permisos y así facilitar la administración. Los usuarios heredan los privilegios de los grupos a los que pertenecen.
- 📩 **Distribution Groups**: se emplean únicamente para la distribución de correos electrónicos y facilitar el envío de mensajes masivos. No otorgan privilegios ni se usan para control de acceso.

Además del tipo, los grupos pueden clasificarse según su alcance, definido mediante un atributo específico:

|Grupo|Descripción|
|---|---|
|**Domain Local Groups**|Tienen alcance a nivel de dominio. Pueden contener usuarios de otros dominios, pero solo pueden otorgar permisos sobre recursos del dominio donde existen|
|**Global Groups**|Pueden otorgar acceso a recursos de otros dominios, pero solo pueden contener usuarios del dominio en el que fueron creados|
|**Universal Groups**|Pueden contener usuarios de cualquier dominio del forest y otorgar acceso a recursos en todo el forest|

### Grupos con privilegios elevados

> Existen grupos especiales en Active Directory a los que se les asignan privilegios elevados por defecto. Por este motivo, hay que añadir muy pocos usuarios a estos grupos y que además contengan credenciales muy robustas, para evitar riesgos de seguridad y vectores de ataque.

|Grupo|Descripción|Privilegio|
|---|---|---|
|**Domain Admins**|Pueden realizar tareas administrativas vía UAC|`GenericAll`|
|**Enterprise Admins**|Privilegios de administración a nivel de bosque|`GenericAll`|
|**Account Operators**|Pueden gestionar cuentas de usuario y grupos, aunque con restricciones sobre grupos altamente privilegiados como Domain Admins o Administrators|`ForceChangePassword` / `GenericWrite`|
|**Backup Operators**|Pueden realizar copias de seguridad y restauración de archivos del sistema|`SeBackupPrivilege`|
|**Server Operators**|Tienen capacidad para administrar servidores del dominio. Pueden iniciar y detener servicios, administrar recursos compartidos y, en algunos casos, lograr ejecución de código con privilegios elevados|—|
|**Print Operators**|Administran los servicios de impresión del dominio|`SeLoadDriverPrivilege`|
|**DnsAdmins**|Administran el servicio DNS integrado en Active Directory. Pueden cargar DLLs como plugins de DNS|—|
|**Group Policy Creator Owners**|Pueden crear nuevas políticas de grupo|—|
|**Schema Admins**|Tienen permisos para modificar el esquema de Active Directory|—|
|**Cert Publishers**|Participan en la publicación de certificados dentro de Active Directory|—|
|**SQL Server Admins**|Suele ser un grupo personalizado en entornos con Microsoft SQL Server. Sus miembros pueden tener privilegios administrativos sobre servidores SQL|—|

> Estos son grupos anidados. Si un grupo pertenece a otro, sus miembros heredarán todos los permisos y privilegios que tenga el grupo madre.
> 
> _Pedro tiene la contraseña "password123" y pertenece al grupo Account Operators. Para "arreglarlo", sacan a Pedro de ese grupo, por lo que solo pertenece al grupo IT. El problema es que se añadió este grupo a Account Operators en el pasado. Resultado: Pedro sigue teniendo los mismos privilegios, solo que ahora nadie se da cuenta a simple vista._

# 🔐 Qué es Kerberos

> Kerberos es un **protocolo de autenticación** basado en tickets que permite implementar **Single Sign-On (SSO)** en entornos de Active Directory. En lugar de enviar la contraseña cada vez, el usuario demuestra su identidad una sola vez y recibe tickets que le abren las puertas a los distintos servicios.

**Puerto por defecto:** `88`    **Cuenta de servicio:** `KRBTGT`

Kerberos funciona como un sistema SSO con tres actores bien diferenciados:

|Actor|Nombre técnico|Función|
|---|---|---|
|👤 Usuario|Client|Quien quiere acceder a los recursos|
|🏛️ Proveedor de identidad|KDC _(Key Distribution Center)_|Emite y valida los tickets|
|🖥️ Servicio|Service Provider|El recurso al que se quiere acceder|

---
## 1.1. 🎟️ Los tickets: las llaves del sistema

En vez de usar la contraseña repetidamente, Kerberos usa dos tipos de tickets:

> **TGT — Ticket Granting Ticket**
> El **carnet de identidad** del usuario. Demuestra quién eres ante el KDC. 🔐 Está cifrado con la clave de la cuenta **KRBTGT**, por tanto solo lo puede leer el KDC-TGS.

>  **ST — Service Ticket**
> El **pase de acceso** a un servicio concreto. Lo obtienes a cambio del TGT. 🔐 Está cifrado con la clave de la **cuenta que ejecuta el servicio**, por tanto solo lo puede leer el servicio de destino.

_El usuario solo puede transportar los tickets, sin leerlos ni modificarlos, ya que están cifrados con claves que solo poseen los servidores. Es como llevar un sobre sellado._

---
## 1.2. 🏛️ El KDC por dentro

El KDC tiene dos componentes que trabajan en secuencia:

- **KDC-AS (Auth Service)**: se encarga de la autenticación y emite el TGT
- **KDC-TGS (Ticket Granting Service)**: cambia un TGT por un ST para el servicio final

---
## 1.3. 🏷️ SPN — Service Principal Name

Los servicios se identifican ante Kerberos mediante el **SPN**, un atributo de la cuenta de servicio con este formato: `SERVICIO/host:puerto`.

 _Por ejemplo, `MSSQLSvc/SQL01.dominio.local:1433`: el servicio de MSSQL se ejecuta por el puerto 1433 en el host SQL01 del dominio.local._

> Cuando el usuario pide acceso a un servicio, le indica al KDC el SPN de ese servicio. El KDC busca qué cuenta tiene ese SPN y cifra el ST con su clave. Así, solo esa cuenta de servicio puede descifrar el ticket.

---

## 1.4. 🔄 El flujo completo de autenticación

El proceso son **6 mensajes en 3 fases**. La idea central es siempre la misma: el usuario demuestra que tiene la clave correcta cifrando un _timestamp_, y a cambio recibe un ticket más una clave de sesión temporal.

```
  USUARIO                  KDC-AS               KDC-TGS              SERVICIO
     │                       │                     │                     │
     │──── 1. AS-REQ ──────> │                     │                     │
     │<─── 2. AS-REP ────────│                     │                     │
     │                       │                     │                     │
     │──────────────────────── 3. TGS-REQ ────────>│                     │
     │<─────────────────────── 4. TGS-REP ─────────│                     │
     │                       │                     │                     │
     │────────────────────────────────────────────── 5. AP-REQ ─────────>│
     │<───────────────────────────────────────────── 6. AP-REP ──────────│
```

----

### Fase 1 — Autenticación inicial (AS)

**1️⃣ AS-REQ — El usuario se presenta**

El usuario envía al **KDC-AS**:

- Su **nombre de usuario**
- Un **authenticator**: timestamp cifrado con su _client key_ (derivada de su contraseña con AES o RC4)

> _El cifrado del timestamp demuestra que conoce la contraseña sin enviarla._

**2️⃣ AS-REP — El KDC responde con el TGT**

El **KDC-AS** descifra el timestamp con la copia de la _client key_ que tiene en su base de datos. Si es válido, devuelve:

- El **TGT** (cifrado con la clave de KRBTGT; el usuario no puede abrirlo)
- Una copia de la **clave de sesión `Kc.tgs`**, cifrada con la _client key_ del usuario

> _La `Kc.tgs` es la llave temporal para hablar con el KDC-TGS._

----

### Fase 2 — Solicitud de ticket de servicio (TGS)

**3️⃣ TGS-REQ — El usuario pide acceso a un servicio**

El usuario descifra `Kc.tgs` con su _client key_ y envía al **KDC-TGS**:

- El **TGT** (que no puede leer, pero lo transporta)
- Un nuevo **authenticator** cifrado con `Kc.tgs`
- El **SPN** del servicio al que quiere acceder

**4️⃣ TGS-REP — El KDC entrega el Service Ticket**

El **KDC-TGS**:

1. Descifra el TGT con la clave de KRBTGT → obtiene la identidad del usuario y `Kc.tgs`
2. Verifica el authenticator y comprueba que el ticket no haya expirado
3. Busca la cuenta asociada al SPN solicitado

Si todo es correcto, devuelve:

- El **ST** (cifrado con la clave del servicio destino)
- Una copia de la **clave de sesión `Kc.s`**, cifrada con `Kc.tgs`

> _La `Kc.s` es la llave temporal para hablar directamente con el servicio._

----

### Fase 3 — Acceso al servicio (AP)

**5️⃣ AP-REQ — El usuario llama a la puerta del servicio**

El usuario descifra `Kc.s` con `Kc.tgs` y envía al **servicio**:

- El **ST** (que el servicio sí puede abrir)
- Un nuevo **authenticator** cifrado con `Kc.s`

**6️⃣ AP-REP — El servicio abre la puerta** _(opcional)_

El **servicio**:

1. Descifra el ST con su propia clave → obtiene la identidad del usuario y `Kc.s`
2. Descifra el authenticator con `Kc.s` y verifica la identidad y el timestamp
3. ✅ Concede acceso

> [!NOTE] El AP-REP es **opcional** y sirve para **autenticación mutua**: el servicio demuestra al usuario que también es quien dice ser.

---

## 1.5. 🗝️ Resumen de claves y quién las conoce

|Clave|Se usa para cifrar...|La tiene...|
|---|---|---|
|_Client key_|Authenticators del usuario, copia de `Kc.tgs`|El Usuario y el KDC-AS|
|Clave de **KRBTGT**|El **TGT**|Solo el KDC|
|Clave de **cuenta de servicio**|El **ST**|El KDC y el Servicio|
|`Kc.tgs` _(sesión temporal)_|Authenticators hacia el TGS, copia de `Kc.s`|El Usuario y el KDC-TGS|
|`Kc.s` _(sesión temporal)_|Authenticators hacia el servicio|El Usuario y el Servicio|

> **La clave del diseño**
> Cada clave de sesión (`Kc.tgs`, `Kc.s`) es temporal y específica para esa comunicación. Si alguien la intercepta, no sirve para nada más. Las claves permanentes (la contraseña del usuario, la clave de KRBTGT...) nunca viajan por la red.

---

## Autenticación por Kerberos

Nuestro entorno es este:

|Rol|Hostname|FQDN|IP|
|---|---|---|---|
|DC|AD01|AD01.Dominio.local|10.10.10.01|
|Máquina unida|AD02|AD02.Dominio.local|10.10.10.02|

Por tanto estos nombres deben quedar reflejados en el archivo `/etc/hosts` para la resolución de nombres.

1️⃣ Instalar los módulos de krb5** con el paquete `krb5-user` y generar un archivo `/etc/krb5.conf`. Esto lo podemos hacer con:
```bash
nxc smb AD01.dominio.local --generate-krb5-file krb5.conf
```

El archivo tendría este aspecto:
```ini
[libdefaults]
   dns_lookup_kdc = false
   dns_lookup_realm = false
   default_realm = DOMINIO.LOCAL

[realms]
   DOMINIO.LOCAL = {
       kdc = AD01.DOMINIO.local
       admin_server = AD01.DOMINIO.local
   }

[domain_realm]
   .dominio.local = DOMINIO.LOCAL
   dominio.local = DOMINIO.LOCAL
```

2️⃣ Obtener el TGT del usuario:
```bash
kinit usuario@dominio.local
# o bien
impacket-GetTGT dominio.local/usuario:pass123
```

3️⃣ Guardar el ticket en una variable** llamada `KRB5CCNAME`:
```bash
export KRB5CCNAME=usuario.ccache
```

4️⃣ Listar los tickets en memoria
```bash
klist
```

**5️⃣Autenticarnos en las herramientas**, por ejemplo por SMB:
```bash
impacket-psexec dominio.local/usuario@DC.dominio.local -k -no-pass
```

> Para evitar problemas debemos sincronizar nuestra hora con la de la máquina, con `sudo ntpdate <ip_dc>`.
