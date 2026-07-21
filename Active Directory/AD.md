# 📂 Active Directory

> [!NOTE] 
> Un Directorio Activo (Active Directory o AD) es una estructura jerárquica que permite centralizar la gestión de recursos, usuarios y servicios en una red empresarial. Esto nos evita tener que configurar individualmente decenas de equipos, lo que sería extremadamente lento, tedioso y propenso a errores.

El AD se basa en un servidor llamado **DC (Controlador de Dominio)**, que se encarga de:

- Almacenar una base de datos llamada **Data Store** (o Directorio), que organiza de manera estructurada y jerárquica todos los recursos de la red (usuarios, sistemas, archivos, impresoras…)
- Proporcionar una serie de servicios que permiten operar en este entorno, incluyendo funciones clave

| Puerto          | Servicio           | Descripción                                                                                                      |
| --------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| **53**          | **DNS**            | Traduce nombres de equipos y dominios a direcciones IP y es fundamental para encontrar recursos en el directorio |
| **88**          | **Kerberos**       | Protocolo principal de autenticación en AD, basado en tickets para acceder a servicios de la red                 |
| **135**         | **MS-RPC**         | Facilita la comunicación entre distintos servicios de Windows y AD                                               |
| **389**         | **LDAP**           | Protocolo utilizado para consultar y administrar objetos del dominio, como usuarios, grupos y equipos            |
| **464**         | **kpasswd**        | Servicio de Kerberos para cambio y reseteo de contraseñas                                                        |
| **636**         | **LDAPS**          | LDAP cifrado sobre TLS                                                                                           |
| **3268 / 3269** | **Global Catalog** | Consultas al Catálogo Global (sin cifrar / con TLS)                                                              |

> 🔁 No existe un solo DC sino varios, réplicas que comparten información a tiempo real para evitar perder el acceso a servicios si uno de ellos es atacado e inutilizado. El proceso de replicación se realiza de manera automática y se pueden gestionar como un solo DC.

---
# 🗂️ 1. Organización del directorio

El Directorio Activo se organiza de manera jerárquica.

## 1.1. Tipos de Objetos
### 1.1.1 🧩 Objetos

> [!NOTE] 
> **Un objeto es cualquier elemento que existe dentro del directorio.** Por ejemplo, puede ser un usuario, un equipo, una impresora, un grupo o una carpeta compartida.

**Cada objeto tiene una serie de atributos**, es decir, datos que lo describen. Esos atributos están definidos por el **Schema**, que actúa como una plantilla. Esto permite que todos los objetos de un mismo tipo tengan la misma estructura de atributos posibles, dándole cohesión al sistema.

> _Por ejemplo, todos los objetos de tipo usuario disponen de atributos comunes como nombre, apellidos, correo electrónico o fecha de creación.

**El Data Store se encarga de asegurar que todos los objetos cumplan las reglas de este Schema.**

En el AD existen dos tipos principales de objetos:
- **Los recursos**, que representan los elementos disponibles en la red (carpetas, equipos o impresoras)
- **Las entidades o security objects**, que representan a quienes acceden a dichos recursos, como usuarios, roles y cuentas de servicio

---
### 1.1.2. 📂 Unidades Organizativas (OUs)

> [!NOTE] 
> **Las Unidades Organizativas u OUs son contenedores que sirven para agrupar objetos relacionados.** Esto permite organizar mejor el directorio y administrar varios objetos de forma conjunta.

Las OUs suelen crearse siguiendo la estructura de la empresa, por ejemplo: por departamentos, sedes o funciones. Gracias a estas OU se pueden aplicar permisos, tareas o directivas a varios objetos a la vez.

> _Por ejemplo, se puede dar permiso al departamento de IT para restablecer contraseñas de los usuarios de la OU "RRHH" y "Oficina", pero no de otras OUs. Así se limita su alcance y se evita que tengan control sobre toda la empresa._

---
### 1.1.3. 🌐 Dominio

> [!NOTE] 
> **Un dominio es una unidad independiente dentro del AD en la que se administran usuarios, equipos y recursos bajo unas mismas reglas de seguridad.**

Dentro de un dominio, todos los objetos comparten las mismas políticas, la misma base de datos y un nombre común gestionado mediante DNS.

El dominio se encarga de identificar a los usuarios (autenticación) y de decidir a qué recursos pueden acceder, funcionando como un entorno de administración centralizada y separado de otros dominios.

> _Por ejemplo, pueden existir dominios como `BILBAO.L4H.LOCAL` y `FINANZAS.L4H.LOCAL`, cada uno con sus propios usuarios, recursos y políticas._

---
### 1.1.4. 🌳 Bosque

> [!NOTE] 
El bosque es la estructura más grande de Active Directory. Sirve para unir varios dominios dentro de una misma organización. Estos dominios compartirán **el mismo schema, configuración y catálogo global.** Representa el límite máximo de seguridad y confianza, ya que todos los dominios dentro de un bosque confían entre sí de forma implícita, permitiendo la administración y el acceso a recursos de manera integrada.

**Hay que pensar en el bosque como si fuera la empresa completa, y los dominios fueran sus distintas sedes o departamentos grandes.**

> Tanto `BILBAO.L4H.LOCAL` como `FINANZAS.L4H.LOCAL` forman parte del mismo bosque: `L4H.LOCAL`

Por último, aunque lo más habitual es que una empresa tenga un único bosque, en algunos casos se crean múltiples bosques para aislar ciertos sectores, como sucursales ubicadas en países con normativas específicas que exigen infraestructuras separadas.

---
### 1.1.5. 📝 Catálogo Global (Global Catalog)

> [!NOTE] 
**El Catálogo Global (GC) es un índice parcial de todos los objetos de todos los dominios de un bosque**, alojado en determinados DCs (los "servidores de catálogo global").

- Contiene **todos los objetos del bosque**, pero solo con un **subconjunto de sus atributos** (los más usados para búsquedas, como el nombre o el email), no el objeto completo.
- Permite buscar objetos de **cualquier dominio del bosque sin tener que consultar uno a uno** cada DC de cada dominio.
- Es imprescindible para el login con **UPN** (`usuario@dominio.local`) cuando el usuario y el equipo están en dominios distintos del mismo bosque, y también participa en la resolución de **membresías de grupos universales**.

> _Piensa en el GC como el índice de una biblioteca gigante: no tiene el libro entero de cada dominio, pero sabe en qué "estantería" (dominio) está y algunos datos básicos de cada uno, para no tener que recorrerla entera._

---
### 1.1.6. 🗺️ Sites (Sitios) y Subnets

> [!NOTE] 
**Un Site agrupa subredes (subnets) según su ubicación física o de red**, normalmente correspondiéndose con oficinas o centros de datos con buena conectividad interna.

Sirven principalmente para dos cosas:

- **Optimizar la autenticación**: un cliente intenta autenticarse contra el DC más cercano según su subred, evitando saltos innecesarios a través de enlaces lentos (WAN).
- **Controlar la replicación**: dentro de un mismo site, la replicación entre DCs es frecuente e inmediata; entre sites distintos se usa un **Inter-Site Topology Generator (ISTG)** que programa la replicación en intervalos para no saturar los enlaces WAN.

> _Por ejemplo, una empresa con sede en Bilbao y otra en Madrid puede definir un Site por ciudad. Un usuario que se conecta en Madrid se autenticará contra un DC de Madrid, no contra uno de Bilbao, aunque ambos pertenezcan al mismo dominio._


----
## 1.2. 🧩 Tipos de objetos

Todo elemento dentro de Active Directory (AD) es un objeto, ya sea un recurso o una security entity.

Cada objeto pertenece a una clase, la cual define un conjunto de atributos comunes según lo establecido en el schema. Por lo tanto, dos objetos de la misma clase comparten los mismos atributos, aunque sus valores puedan diferir.

----
### 1.2.1. 🌿 Leaf Objects

Son objetos que no pueden contener otros objetos en su interior.

| Nombre                     | Descripción                                                                                                            | Ejemplo                                     | ObjectClass              |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------ |
| Usuario                    | Security Principal que representa a una persona o servicio. Tiene GUID y SID; algunos se usan como cuentas de servicio | juan.cuesta                                 | user                     |
| Contacto                   | Representa usuarios externos, con menos atributos. No es Security Principal ni tiene SID                               | Contacto externo de un partner              | contact                  |
| Foreign Security Principal | Representa usuarios de otros bosques añadidos a grupos del dominio actual, conservando su SID original                 | Usuario de otro dominio agregado a un grupo | foreignSecurityPrincipal |
| Impresora                  | Recurso disponible en el AD. No es Security Principal; tiene GUID, pero no SID                                         | Impresora de red corporativa                | printQueue               |
| Computadora                | Representa un equipo unido al dominio. Es Security Principal y tiene GUID y SID                                        | PC-01                                       | computer                 |

----
### 📦 1.2.2. ContainerObject
**Pueden contener otros objetos, ya sean leaf objects u otros container objects.

| Nombre   | Descripción                                                                                             | Ejemplo                           | ObjectClass        |
| -------- | ------------------------------------------------------------------------------------------------------- | --------------------------------- | ------------------ |
| Grupo    | Security Principal que agrupa usuarios y otros grupos (nested groups) para asignar permisos compartidos | Domain Admins                     | group              |
| Built-in | Contenedor que almacena los grupos predeterminados creados automáticamente por AD                       | Builtin\Administrators            | container          |
| Dominio  | Contenedor principal que alberga usuarios, computadoras, OUs y grupos, con reglas y políticas propias   | L4H.local                         | domainDNS          |
| OU       | Contenedor lógico que agrupa objetos para facilitar la administración y aplicación de políticas         | OU=IT,  <br>DC=L4H,  <br>DC=local | organizationalUnit |

----
### 1.2.3. ⚫ Otros

| Nombre             | Descripción                                                                                                      | Ejemplo                 | ObjectClass |
| ------------------ | ---------------------------------------------------------------------------------------------------------------- | ----------------------- | ----------- |
| Domain Controller  | Computadora que proporciona servicios de autenticación y autorización dentro del dominio                         | DC01.L4H.local          | computer    |
| Carpeta compartida | Recurso compartido de una computadora, accesible según permisos definidos. Tiene GUID y no es Security Principal | \\FS01\Datos            | volume      |
| Site               | Agrupa computadoras según su ubicación de red para optimizar la replicación entre controladores de dominio       | Default-First-Site-Name | site        |


---
# 2. 🔍 LDAP y el DIT

> [!TIP]
El directorio es, en esencia, una base de datos que almacena y organiza todos estos objetos en una estructura jerárquica en forma de árbol, conocida como DIT (Directory Information Tree). Esta base de datos se accede y gestiona mediante un protocolo especializado llamado LDAP (Lightweight Directory Access Protocol), el cual permite consultar, compartir y administrar la información del directorio de forma eficiente.

En terminología LDAP, todos los objetos del árbol se denominan entries. Cada entry pertenece a una clase ("objectClass"), que determina los atributos que puede contener (Ej: mail, uid, memberOf).

---
## 2.1. Nomenclaturas

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
## 2.2. Sintaxis LDAP

LDAP emplea **filtros LDAP** para las consultas, diseñados específicamente para trabajar buscando la posición de los objetos dentro del árbol DIT. Toda búsqueda LDAP se define a partir de tres elementos principales:

1. **Base DN**: indica la ruta desde la que se inicia la búsqueda dentro del directorio
2. **Filtro LDAP**: especifica las condiciones que deben cumplir los objetos para ser devueltos (por ejemplo, tipo de objeto o valor de un atributo)
3. **Atributos a devolver**: qué datos concretos de cada entrada se quieren obtener como resultado de la búsqueda

Las consultas LDAP siempre van entre paréntesis con la sintaxis: `(<clave>=<valor>)`.

| Regla                     | Ejemplo       | Explicación del ejemplo                        |
| ------------------------- | ------------- | ---------------------------------------------- |
| **Igualdad**              | `(cn=jperez)` | Buscamos al usuario Juan Perez                 |
| **Presencia de atributo** | `(mail=*)`    | Todos los objetos con el atributo mail relleno |
| **Coincidencia parcial**  | `(cn=Juan*)`  | Todos los "Juanes"                             |
Para consultas complejas, se combinan varias consultas simples concatenadas por un operador AND u OR

| Regla   | Ejemplo                               | Explicación del ejemplo                             |
| ------- | ------------------------------------- | --------------------------------------------------- |
| **AND** | `(&(ou=contabilidad)(cn=Juan*))`      | Usuario que sea del ou contabilidad y se llame Juan |
| **OR**  | `(\|(ou=contabilidad)(ou=direccion))` | Usuarios de dirección o contabilidad                |
| **NOT** | `(!(cn=admin))`                       | Todos menos el admin                                |

Las consultas LDAP se hacen con el comando `Get-ADObject`, que admite estos parámetros:
```bash
Get-ADObject
  -LDAPFilter "(objectClass=user)"
  -SearchBase "OU=IT,DC=empresa,DC=local"
  -Properties samAccountName,mail,whenCreated
```

---
# 3. 🔁 Replicación

En el DC existe un archivo llamado **NTDS.dit**, que es la base de datos física de Active Directory. Dentro de él se almacenan varias particiones lógicas o _naming contexts_, entre ellas Domain, Configuration y Schema.

- 🌐 **Domain**: contiene los objetos del dominio, como usuarios, grupos, equipos, impresoras, unidades organizativas (OUs), etc. y sus atributos. Parte de esos atributos son especialmente sensibles, como los hashes NTLM de los usuarios.
- 🛠️ **Configuration**: almacena información sobre la estructura del bosque, los dominios, los sitios y los servicios.
- 🧾 **Schema**: define qué tipos de objetos existen en Active Directory y qué atributos puede tener cada uno.

> [!WARNING]
En una empresa normalmente no existe un único DC, sino varios. Todos ellos mantienen copias de estas particiones y deben permanecer sincronizados, sin discrepancias entre sí. Si cambia cualquier dato en un controlador de dominio, el resto debe recibir ese cambio para que la información sea coherente en todo el entorno. Esto es fundamental para evitar problemas de autenticación, autorización y acceso a recursos.

> _Por ejemplo, un usuario debe existir con la misma contraseña, pertenencia a grupos y resto de atributos en todos los controladores de dominio, de modo que pueda autenticarse correctamente desde cualquier sede o servicio de la organización._

Para evitar discrepancias, los DCs se sincronizan entre sí mediante un proceso llamado **replicación**. En lugar de copiar toda la base de datos cada vez, solo se replican los cambios realizados en una partición concreta, lo que reduce el tráfico y hace el proceso más eficiente. 

Esta replicación se realiza mediante el protocolo remoto de replicación de Active Directory, conocido como **Directory Replication Service Remote Protocol (MS-DRSR)**.

En este contexto entran en juego una serie de privilegios especiales de replicación:

- **Replicating Directory Changes (Get Changes)**: permite solicitar los cambios de una partición. De forma simplificada, equivale a decir: _"déjame ver qué ha cambiado"_.
- **Replicating Directory Changes All (Get Changes All)**: es el privilegio más sensible, ya que permite replicar también **datos secretos del dominio** (los hashes NTLM de los usuarios en la partición Domain). De forma simplificada, equivale a: _"déjame ver también la parte sensible, no solo lo básico"_.

> [!CAUTION]
> Estos privilegios suelen estar disponibles para cuentas altamente privilegiadas, como las de Domain Admins, o para ciertos servicios legítimos de sincronización. El problema aparece cuando una cuenta con estos derechos es comprometida, o cuando dichos permisos se delegan a usuarios o grupos que no deberían tenerlos.

## 4. Roles y trusts

## 4.1. 👑 Roles FSMO (Flexible Single Master Operations)

> [!NOTE]
Aunque AD es una base de datos **multi-maestro** (cualquier DC admite escrituras y las replica al resto), existen **5 operaciones que no pueden hacerse de forma distribuida** sin generar conflictos. Para esas operaciones, un único DC del bosque o dominio actúa como "dueño" en cada momento.

| Rol                       | Alcance                 | Función                                                                                                                                                          |
| ------------------------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Schema Master**         | Bosque (1 por bosque)   | Único DC que puede modificar el Schema (añadir/cambiar tipos de objeto o atributos)                                                                              |
| **Domain Naming Master**  | Bosque (1 por bosque)   | Controla la adición y eliminación de dominios dentro del bosque                                                                                                  |
| **RID Master**            | Dominio (1 por dominio) | Reparte bloques de RIDs (identificadores relativos) a cada DC, con los que se construyen los SID de los nuevos objetos                                           |
| **PDC Emulator**          | Dominio (1 por dominio) | Actúa como reloj maestro de tiempo, procesa cambios de contraseña urgentes y bloqueos de cuenta, y es el objetivo preferente de las Group Policies más recientes |
| **Infrastructure Master** | Dominio (1 por dominio) | Mantiene actualizadas las referencias a objetos de otros dominios (por ejemplo, miembros de grupos que pertenecen a otro dominio del bosque)                     |

> Si un DC con un rol FSMO se cae, esa operación concreta queda bloqueada hasta que se transfiera (o se fuerce mediante _seize_) el rol a otro DC; el resto del directorio sigue funcionando con normalidad gracias al modelo multi-maestro.

---
## 4.2. 🤝 Confianzas (Trusts)

> [!NOTE]
**Una confianza (trust) es una relación que permite a los usuarios de un dominio o bosque autenticarse y acceder a recursos de otro dominio o bosque**, sin necesidad de tener una cuenta duplicada en cada uno.

Se definen por dos características:

- **Dirección**: unidireccional (A confía en B, pero no al revés) o bidireccional (ambos confían entre sí)
- **Transitividad**: transitiva (la confianza se extiende a otros dominios/bosques que confíen en el segundo) o no transitiva (solo aplica entre los dos extremos definidos)

|Tipo|Descripción|
|---|---|
|**Parent-Child**|Automática y transitiva, entre un dominio y los que cuelgan de él dentro del mismo bosque|
|**Tree-Root**|Automática y transitiva, entre el dominio raíz del bosque y un nuevo árbol de dominio añadido al mismo bosque|
|**Shortcut**|Manual, para optimizar la autenticación entre dos dominios muy alejados dentro del mismo bosque, evitando recorrer toda la jerarquía|
|**Forest Trust**|Manual, transitiva dentro de cada bosque, conecta dos bosques completos entre sí|
|**External Trust**|Manual, no transitiva, conecta un dominio concreto con otro dominio de un bosque distinto (sin extenderse al resto del bosque)|
|**Realm Trust**|Conecta un dominio de AD con un entorno Kerberos no-Windows (por ejemplo, un KDC de MIT Kerberos en Linux)|

> _Por ejemplo, si `FINANZAS.L4H.LOCAL` tiene una External Trust con el dominio `PARTNER.LOCAL`, solo los usuarios de `PARTNER.LOCAL` (y no los de todo su bosque) podrán acceder a recursos autorizados de `FINANZAS.L4H.LOCAL`._

---
## 4.3. 📜 Group Policy Objects (GPO)

> [!NOTE]
**Las GPOs son el mecanismo de AD para aplicar configuración de forma centralizada** a usuarios y equipos: políticas de contraseñas, restricciones de software, scripts de inicio/cierre de sesión, configuración de seguridad, mapeo de unidades, etc.

- Se almacenan en dos partes: los metadatos en el **Data Store** (contenedor `groupPolicyContainer`) y los archivos de configuración en **SYSVOL** (`\\dominio\SYSVOL\dominio\Policies\{GUID}`), una carpeta compartida que se replica entre todos los DCs.
- Se **vinculan** (link) a un Sitio, Dominio u OU, y se aplican a todos los objetos contenidos en ese nivel.
- Siguen un orden de aplicación conocido como **LSDOU**: Local → Site → Domain → OU (de más general a más específico), donde la configuración más específica prevalece salvo que se marque **Enforced** (obligatoria, no se puede sobrescribir) o el contenedor tenga **Block Inheritance** (bloquea las políticas heredadas de niveles superiores).

> _Por ejemplo, una GPO vinculada al dominio puede forzar una política de contraseñas mínima para todos, mientras que una GPO vinculada a la OU "IT" añade software adicional solo para ese departamento._

---

## 🛡️ 4.4. Autorización

Cada objeto de AD tiene un **Security Descriptor**, una estructura que define quién puede hacer qué sobre él. Contiene, entre otras cosas:
- **Owner (propietario)**: el SID del propietario del objeto, que siempre puede modificar sus permisos
- **DACL (Discretionary ACL)**: la lista que define **qué usuarios o grupos tienen qué permisos** sobre el objeto (lectura, escritura, borrado, etc.) Estña compuesta por  **ACEs (Access Control Entries)**, cada una indicando un SID y el permiso que se le concede o deniega. 
- **SACL (System ACL)**: define qué acciones sobre el objeto se **auditan** (quedan registradas en el log de seguridad)

Algunos de los permisos más relevantes en AD:

|Permiso|Qué permite|
|---|---|
|`GenericAll`|Control total sobre el objeto (equivalente a ser su dueño)|
|`GenericWrite`|Modificar la mayoría de atributos del objeto|
|`WriteOwner`|Cambiar el propietario del objeto (y desde ahí, tomar control total)|
|`WriteDACL`|Modificar los permisos del propio objeto|
|`ForceChangePassword`|Cambiar la contraseña de un usuario sin conocer la actual|
|`AllExtendedRights`|Ejecutar todas las "acciones extendidas", incluyendo cambiar contraseñas o (si aplica) leer atributos confidenciales|

> *Estos son precisamente los privilegios que aparecen en la tabla de "Grupos con privilegios elevados": no son mágicos, son ACEs concretas dentro de una DACL, iguales a las que un administrador podría delegar por error sobre cualquier objeto normal.*

> [!TIP]
🛡️ Existe además un mecanismo de protección especial llamado **AdminSDHolder**: es una plantilla de permisos que se aplica periódicamente (cada 60 min aprox., vía el proceso SDProp) a los miembros de grupos altamente privilegiados (Domain Admins, Enterprise Admins, etc.), sobrescribiendo cualquier permiso personalizado que se les hubiera añadido, para evitar que alguien delegue silenciosamente control sobre una cuenta protegida.


---
# 5. 👥 Grupos y privilegios

Recordamos que tenemos una serie de grupos de administración con privilegios clave. Si se compromete una cuenta que forme parte de estos grupos, tanto directa como indirectamente, podremos realizar una escalada de privilegios.

Puede que nuestro usuario, por medio de un permiso especial, pueda obtener el control de la cuenta de otro usuario con mayor nivel de acceso en la red.

## 5.1. Tipos de grupos

Existen dos tipos principales de grupos en Active Directory:

- ⚠️ **Security Groups**: se utilizan para agrupar usuarios (u otros objetos) con los mismos permisos y así facilitar la administración. Los usuarios heredan los privilegios de los grupos a los que pertenecen.
- 📩 **Distribution Groups**: se emplean únicamente para la distribución de correos electrónicos y facilitar el envío de mensajes masivos. No otorgan privilegios ni se usan para control de acceso.

Además del tipo, los grupos pueden clasificarse según su alcance, definido mediante un atributo específico:

|Grupo|Descripción|
|---|---|
|**Domain Local Groups**|Tienen alcance a nivel de dominio. Pueden contener usuarios de otros dominios, pero solo pueden otorgar permisos sobre recursos del dominio donde existen|
|**Global Groups**|Pueden otorgar acceso a recursos de otros dominios, pero solo pueden contener usuarios del dominio en el que fueron creados|
|**Universal Groups**|Pueden contener usuarios de cualquier dominio del forest y otorgar acceso a recursos en todo el forest|

## 5.2. Grupos con privilegios elevados

> [!WARNING]
Existen grupos especiales en Active Directory a los que se les asignan privilegios elevados por defecto. Por este motivo, hay que añadir muy pocos usuarios a estos grupos y que además contengan credenciales muy robustas, para evitar riesgos de seguridad y vectores de ataque.

| Grupo                           | Descripción                                                                                                                                                                                             | Privilegio                             |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| **Domain Admins**               | Pueden realizar tareas administrativas vía UAC                                                                                                                                                          | `GenericAll`                           |
| **Enterprise Admins**           | Privilegios de administración a nivel de bosque                                                                                                                                                         | `GenericAll`                           |
| **Account Operators**           | Pueden gestionar cuentas de usuario y grupos, aunque con restricciones sobre grupos altamente privilegiados como Domain Admins o Administrators                                                         | `ForceChangePassword` / `GenericWrite` |
| **Backup Operators**            | Pueden realizar copias de seguridad y restauración de archivos del sistema                                                                                                                              | `SeBackupPrivilege`                    |
| **Server Operators**            | Tienen capacidad para administrar servidores del dominio. Pueden iniciar y detener servicios, administrar recursos compartidos y, en algunos casos, lograr ejecución de código con privilegios elevados | —                                      |
| **Print Operators**             | Administran los servicios de impresión del dominio                                                                                                                                                      | `SeLoadDriverPrivilege`                |
| **DnsAdmins**                   | Administran el servicio DNS integrado en Active Directory. Pueden cargar DLLs como plugins de DNS                                                                                                       | —                                      |
| **Group Policy Creator Owners** | Pueden crear nuevas políticas de grupo                                                                                                                                                                  | —                                      |
| **Schema Admins**               | Tienen permisos para modificar el esquema de Active Directory                                                                                                                                           | —                                      |
| **Cert Publishers**             | Participan en la publicación de certificados dentro de Active Directory                                                                                                                                 | —                                      |
| **SQL Server Admins**           | Suele ser un grupo personalizado en entornos con Microsoft SQL Server. Sus miembros pueden tener privilegios administrativos sobre servidores SQL                                                       | —                                      |
> [!CAUTION]
> Estos son grupos anidados. Si un grupo pertenece a otro, sus miembros heredarán todos los permisos y privilegios que tenga el grupo madre.
> 
> _Pedro tiene la contraseña "password123" y pertenece al grupo Account Operators. Para "arreglarlo", sacan a Pedro de ese grupo, por lo que solo pertenece al grupo IT. El problema es que se añadió este grupo a Account Operators en el pasado. Resultado: Pedro sigue teniendo los mismos privilegios, solo que ahora nadie se da cuenta a simple vista._

---

# 6. 🔐 Qué es Kerberos

> [!NOTE]
> Kerberos es un **protocolo de autenticación** basado en tickets que permite implementar **Single Sign-On (SSO)** en entornos de Active Directory. En lugar de enviar la contraseña cada vez, el usuario demuestra su identidad una sola vez y recibe tickets que le abren las puertas a los distintos servicios.

**Puerto por defecto:** `88`    **Cuenta de servicio:** `KRBTGT`

Kerberos funciona como un sistema SSO con tres actores bien diferenciados:

| Actor                      | Nombre técnico                  | Función                             |
| -------------------------- | ------------------------------- | ----------------------------------- |
| 👤 Usuario                 | Client                          | Quien quiere acceder a los recursos |
| 🏛️ Proveedor de identidad | KDC _(Key Distribution Center)_ | Emite y valida los tickets          |
| 🖥️ Servicio               | Service Provider                | El recurso al que se quiere acceder |

---
## 1.1. 🎟️ Los tickets: las llaves del sistema

En vez de usar la contraseña repetidamente, Kerberos usa dos tipos de tickets:

| Ticket 🎟️                       | Uso                                                                         | Protección 🔐                                                                                                            |
| -------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **TGT — Ticket Granting Ticket** | El **carnet de identidad** del usuario. Demuestra su identidad ante el KDC. | Está cifrado con la clave de la cuenta **KRBTGT**, por tanto solo lo puede leer el KDC-TGS.                              |
| **ST — Service Ticket**          | El **pase de acceso** a un servicio concreto. Lo obtienes a cambio del TGT  | Está cifrado con la clave de la **cuenta que ejecuta el servicio**, por tanto solo lo puede leer el servicio de destino. |

> _El usuario solo puede transportar los tickets, sin leerlos ni modificarlos, ya que están cifrados con claves que solo poseen los servidores. Es como llevar un sobre sellado._

El KDC tiene dos componentes que trabajan en secuencia:

- **KDC-AS (Auth Service)**: se encarga de la autenticación y emite el TGT
- **KDC-TGS (Ticket Granting Service)**: cambia un TGT por un ST para el servicio final

---
## 1.2. 🏷️ SPN — Service Principal Name

Los servicios se identifican ante Kerberos mediante el **SPN**, un atributo de la cuenta de servicio con este formato: `SERVICIO/host:puerto`.

 > _Por ejemplo, `MSSQLSvc/SQL01.dominio.local:1433`: el servicio de MSSQL se ejecuta por el puerto 1433 en el host SQL01 del dominio.local._

Cuando el usuario pide acceso a un servicio, le indica al KDC el SPN de ese servicio. El KDC busca qué cuenta tiene ese SPN y cifra el ST con su clave. Así, solo esa cuenta de servicio puede descifrar el ticket.

---

## 1.3. 🔄 El flujo completo de autenticación

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

### 1.3.1. Fase 1 — Autenticación inicial (AS)

**1️⃣ AS-REQ — El usuario se presenta**

El usuario envía al **KDC-AS**:

- Su **nombre de usuario**
- Un **authenticator**: timestamp cifrado con su _client key_ (derivada de su contraseña con AES o RC4)

> _El cifrado del timestamp demuestra que conoce la contraseña sin enviarla._

---

**2️⃣ AS-REP — El KDC responde con el TGT**

El **KDC-AS** descifra el timestamp con la copia de la _client key_ que tiene en su base de datos. Si es válido, devuelve:

- El **TGT** (cifrado con la clave de KRBTGT; el usuario no puede abrirlo)
- Una copia de la **clave de sesión `Kc.tgs`**, cifrada con la _client key_ del usuario

> _La `Kc.tgs` es la llave temporal para hablar con el KDC-TGS._

----
### 1.3.2. Fase 2 — Solicitud de ticket de servicio (TGS)

**3️⃣ TGS-REQ — El usuario pide acceso a un servicio**

El usuario descifra `Kc.tgs` con su _client key_ y envía al **KDC-TGS**:
- El **TGT** (que no puede leer, pero lo transporta)
- Un nuevo **authenticator** cifrado con `Kc.tgs`
- El **SPN** del servicio al que quiere acceder

---

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
### 1.3.3. Fase 3 — Acceso al servicio (AP)

**5️⃣ AP-REQ — El usuario llama a la puerta del servicio**

El usuario descifra `Kc.s` con `Kc.tgs` y envía al **servicio**:
- El **ST** (que el servicio sí puede abrir)
- Un nuevo **authenticator** cifrado con `Kc.s`

---
**6️⃣ AP-REP — El servicio abre la puerta** _(opcional)_

El **servicio**:
1. Descifra el ST con su propia clave → obtiene la identidad del usuario y `Kc.s`
2. Descifra el authenticator con `Kc.s` y verifica la identidad y el timestamp
3. ✅ Concede acceso

> [!NOTE] El AP-REP es **opcional** y sirve para **autenticación mutua**: el servicio demuestra al usuario que también es quien dice ser.

---

## 1.4. 🗝️ Resumen de claves y quién las conoce

| Clave                           | Se usa para cifrar...                         | La tiene...              |
| ------------------------------- | --------------------------------------------- | ------------------------ |
| Client key                      | Authenticators del usuario, copia de `Kc.tgs` | El Usuario y el KDC-AS   |
| Clave de **KRBTGT**             | El **TGT**                                    | Solo el KDC              |
| Clave de **cuenta de servicio** | El **ST**                                     | El KDC y el Servicio     |
| `Kc.tgs` _(sesión temporal)_    | Authenticators hacia el TGS, copia de `Kc.s`  | El Usuario y el KDC-TGS  |
| `Kc.s` _(sesión temporal)_      | Authenticators hacia el servicio              | El Usuario y el Servicio |

> [!IMPORTANT]
> **La clave del diseño**: Cada clave de sesión (`Kc.tgs`, `Kc.s`) es temporal y específica para esa comunicación. Si alguien la intercepta, no sirve para nada más. Las claves permanentes (la contraseña del usuario, la clave de KRBTGT...) nunca viajan por la red.

---
## 1.5. 🎭 Delegación en Kerberos

La delegación permite que un **servicio actúe en nombre de un usuario** frente a un segundo servicio (por ejemplo, un servidor web que consulta una base de datos con la identidad del usuario que hizo login, no con su propia identidad).

|Tipo|Descripción|Riesgo|
|---|---|---|
|**Unconstrained Delegation**|El servicio recibe una copia del **TGT completo** del usuario y puede usarlo para autenticarse como él ante **cualquier otro servicio**|Muy alto: si el servidor se compromete, se puede extraer el TGT de cualquier usuario que se haya autenticado en él|
|**Constrained Delegation**|El servicio solo puede impersonar al usuario ante una **lista concreta de SPNs** definida en su atributo `msDS-AllowedToDelegateTo`|Medio: el abuso se limita a los servicios listados|
|**Resource-Based Constrained Delegation (RBCD)**|El control se define **en el recurso destino**, no en el servicio origen: el recurso decide qué cuentas pueden delegar hacia él (`msDS-AllowedToActOnBehalfOfOtherIdentity`)|Depende de quién puede modificar ese atributo del recurso|

> _La diferencia clave entre Constrained y RBCD es **quién configura el permiso**: en Constrained lo hace un admin sobre la cuenta del servicio origen; en RBCD lo hace quien tiene control sobre el objeto destino._

----
## 1.6. 🔑 NTLM: la alternativa a Kerberos

**NTLM (NT LAN Manager) es un protocolo de autenticación más antiguo que Kerberos**, basado en un esquema de reto-respuesta (challenge-response) en lugar de tickets. AD lo mantiene activo por compatibilidad, aunque Kerberos es el protocolo preferente cuando está disponible.

Se usa típicamente cuando:
- El cliente se conecta directamente por IP (sin poder resolver un SPN/nombre)
- El servicio o el sistema operativo no soporta Kerberos
- Se accede a un recurso fuera del dominio o sin confianza Kerberos

> A diferencia de Kerberos, en NTLM **no hay verificación mutua por defecto** ni tickets con caducidad, lo que históricamente lo ha hecho más débil frente a ataques de repetición y relay.
