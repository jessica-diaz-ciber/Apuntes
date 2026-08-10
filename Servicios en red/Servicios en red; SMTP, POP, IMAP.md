# 📧 Protocolos de Correo — SMTP, IMAP y POP3

El correo electrónico en red se gestiona mediante una familia de protocolos complementarios: **SMTP** para el envío y transporte, e **IMAP** o **POP3** para la recepción y gestión por parte del cliente. 

#### 🗺️ Visión general — qué protocolo hace qué

```
  [Remitente]                                          [Destinatario]
  Outlook / Gmail                                      Outlook / Gmail
  (MUA)                                                (MUA)
     │                                                    ▲
     │ SMTP (envío)                                       │ IMAP / POP3 (lectura)
     ▼                                                    │
  [MSA]  ──── SMTP ────▶  [MTA] ──── SMTP ────▶  [MDA] ───┘
  Validación              Transporte             Entrega al buzón
  del remitente           y enrutamiento         del destinatario
```

| Protocolo | Función                                        | Puerto estándar                                | Puerto cifrado |
| --------- | ---------------------------------------------- | ---------------------------------------------- | -------------- |
| **SMTP**  | Envío y transporte de correos entre servidores | 25 (servidor-servidor), 587 (cliente-servidor) | 465 (SMTPS)    |
| **IMAP**  | Acceso y gestión de correos en el servidor     | 143                                            | 993 (IMAPS)    |
| **POP3**  | Descarga de correos desde el servidor          | 110                                            | 995 (POP3S)    |

----
## 📤 SMTP — Simple Mail Transfer Protocol

#### Agentes que intervienen en el envío

| Agente  | Nombre                | Función                                                                                                                                                                                |
| ------- | --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **MUA** | Mail User Agent       | Cliente de correo del usuario (Outlook, Thunderbird, Gmail...) donde el usuario redacta el correo, se autentica ante el servidor SMTP                                                  |
| **MSA** | Mail Submission Agent | Recibe el correo del MUA, valida el remitente y lo pasa al MTA                                                                                                                         |
| **MTA** | Mail Transfer Agent   | Transporta el correo entre servidores y aplica filtrado antispam. Consulta el registro MX del DNS para localizar el servidor destino. Puede pasar por servidores **Relay** intermedios |
| **MDA** | Mail Delivery Agent   | Recibe el correo del MTA y lo almacena en el buzón del destinatario                                                                                                                    |

#### Comandos SMTP

|Comando|Descripción|
|---|---|
|`HELO` / `EHLO`|El cliente inicia la sesión con su nombre de host. `EHLO` activa ESMTP (extensiones)|
|`AUTH PLAIN`|Autenticación del cliente (extensión ESMTP)|
|`MAIL FROM`|Declara la dirección del remitente|
|`RCPT TO`|Declara la dirección del destinatario|
|`DATA`|Inicia el cuerpo del mensaje. Se termina con una línea que contiene solo `.`|
|`RSET`|Aborta la transacción actual pero mantiene la conexión|
|`VRFY`|Verifica si un buzón existe en el servidor|
|`EXPN`|Expande un alias de lista de correo|
|`NOOP`|Mantiene la conexión activa (evita timeout)|
|`QUIT`|Cierra la sesión|

Si nos conectamos al puerto 25 por telnet: `telnet 10.129.14.128 25`
```
220 ESMTP Server

EHLO web.com
250-mail1.ejemplo.com
250-SIZE 10240000
250-STARTTLS
250 HELP

MAIL FROM: <jefe@web.com>
250 2.1.0 Ok

RCPT TO: <empleado@web.com> NOTIFY=success,failure
250 2.1.5 Ok

DATA
354 End data with <CR><LF>.<CR><LF>

From: <jefe@web.com>
To: <empleado@web.com>
Subject: Prueba SMTP
Date: Tue, 28 Sept 2021 16:32:51 +0200

Esto es una prueba enviada manualmente.
.
250 2.0.0 Ok: queued as 6E1CF1681AB

QUIT
221 2.0.0 Bye
```

#### Configuración
Postfix (`/etc/postfix/main.cf`)

```ini
myhostname = mail1.empresa.com        # nombre del servidor de correo
mydestination = $myhostname, localhost # dominios a los que entrega correo localmente
mynetworks = 127.0.0.0/8 10.0.0.0/16  # redes de confianza para enviar correos ⚠️
inet_protocols = ipv4                  # protocolo IP a usar
smtp_bind_address = 0.0.0.0           # interfaz de escucha (todas)
home_mailbox = Maildir/               # directorio donde almacenar los correos
smtpd_banner = ESMTP Server           # banner de bienvenida
smtpd_helo_restrictions = reject_invalid_hostname  # rechazar hostnames inválidos
smtpd_sasl_auth_enable = yes          # habilitar autenticación SASL
smtpd_tls_security_level = may        # usar TLS si el cliente lo soporta
```

---
#### Configuraciones peligrosas

Un **servidor de retransmisión (relay)** permite reenviar correos a otros servidores SMTP y, normalmente, exige autenticación para evitar usos no autorizados.

Sin embargo, una mala configuración puede convertirlo en un **Open Relay**. Esto ocurre, por ejemplo, cuando se permite el acceso desde cualquier dirección IP: `mynetworks = 0.0.0.0/0`. Con esta configuración, cualquier equipo puede utilizar el servidor para enviar correos, facilitando campañas de spam, la suplantación de identidad (_email spoofing_) y otros ataques relacionados con el correo electrónico.

Para enumerar el servicio smtp con Nmap, con los scripts por defecto con `-sCV` se pueden ver los comandos que están permitidos aunque tambien se puede usar el script `smtp-open-relay` para ver si el servidor objetivo utiliza retransmisión abierta utilizando 16 pruebas diferentes.

----
## 📥 IMAP — Internet Message Access Protocol

IMAP permite acceder y gestionar los correos **directamente en el servidor** sin descargarlos. Es el protocolo preferido para uso desde múltiples dispositivos porque mantiene todos los cambios sincronizados.

------
#### Comandos IMAP

Los comandos IMAP van precedidos de un **identificador de secuencia** (número o letra) que correlaciona cada petición con su respuesta.

|Comando|Descripción|
|---|---|
|`1 LOGIN usuario contraseña`|Autenticación en el servidor|
|`1 LIST "" *`|Lista todos los buzones disponibles|
|`1 SELECT INBOX`|Selecciona un buzón para operar sobre él|
|`1 UNSELECT INBOX`|Sale del buzón seleccionado|
|`1 FETCH <ID> all`|Obtiene todos los datos de un mensaje|
|`1 FETCH <ID> BODY[]`|Descarga el contenido completo del mensaje|
|`1 SEARCH FROM "jessica"`|Busca mensajes por criterio|
|`1 CREATE "Trabajo"`|Crea un nuevo buzón|
|`1 RENAME "Trabajo" "Empresa"`|Renombra un buzón|
|`1 DELETE "Trabajo"`|Elimina un buzón|
|`1 LSUB "" *`|Lista buzones suscritos|
|`1 CLOSE`|Elimina permanentemente los mensajes marcados como `\Deleted`|
|`1 LOGOUT`|Cierra la sesión|

#### Ejemplo de sesión IMAP

```bash
# Con openssl para IMAP cifrado
openssl s_client -connect 10.129.14.128:993 -quiet
```

```
* OK [CAPABILITY IMAP4rev1 ...] Dovecot ready.

1 LOGIN jessica p4ssw0rd
1 OK Logged in

1 LIST "" *
* LIST (\HasNoChildren) "." INBOX
* LIST (\HasNoChildren) "." Sent
* LIST (\HasNoChildren) "." Drafts
1 OK List completed

1 SELECT INBOX
* 3 EXISTS
* 0 RECENT
1 OK [READ-WRITE] Select completed

1 FETCH 1 BODY[]
* 1 FETCH (BODY[] {412}
From: jefe@empresa.com
...
)
1 OK Fetch completed

1 LOGOUT
* BYE Logging out
1 OK Logout completed
```

Si también usamos la opción verbose (-v) podemos ver la versión de TLS utilizada para el cifrado, más detalles del certificado SSL, e incluso el banner, que a menudo contendrá la versión del servidor de correo.

```
$: curl -k 'imaps://10.129.14.128' --user user:p4ssw0rd -v

* LIST (\HasNoChildren) "." Important
* LIST (\HasNoChildren) "." INBOX
```

Para interactuar con el servidor IMAP o POP3 a través de SSL, podemos usar openssl, así como ncat. 

```
openssl s_client -connect 10.129.14.128:pop3s
openssl s_client -connect 10.129.14.128:imaps
```

---
## 📬 POP3 — Post Office Protocol v3

POP3 está diseñado para **descargar** los correos al dispositivo local y (generalmente) borrarlos del servidor. Es más simple que IMAP y está pensado para uso desde un único dispositivo sin necesidad de sincronización.

#### IMAP vs POP3

| Característica                   | IMAP                | POP3                    |
| -------------------------------- | ------------------- | ----------------------- |
| Correos en el servidor           | ✅ Permanecen        | ❌ Se descargan y borran |
| Sincronización multi-dispositivo | ✅ Sí                | ❌ No                    |
| Carpetas remotas                 | ✅ Sí                | ❌ No                    |
| Acceso offline                   | ⚠️ Con copia local  | ✅ Todo descargado       |
| Complejidad                      | Mayor               | Menor                   |
| Uso recomendado                  | Varios dispositivos | Un solo dispositivo     |

#### Comandos POP3

| Comando           | Descripción                                                |
| ----------------- | ---------------------------------------------------------- |
| `USER usuario`    | Identifica al usuario                                      |
| `PASS contraseña` | Autenticación                                              |
| `STAT`            | Número de mensajes y tamaño total del buzón                |
| `LIST`            | Lista todos los mensajes con su ID y tamaño                |
| `RETR <id>`       | Descarga el mensaje con ese ID                             |
| `DELE <id>`       | Marca el mensaje para eliminación (se borra al hacer QUIT) |
| `RSET`            | Cancela las eliminaciones marcadas                         |
| `CAPA`            | Lista las capacidades del servidor                         |
| `QUIT`            | Aplica las eliminaciones pendientes y cierra la sesión     |

---
## 🔐 Seguridad

### Problemas inherentes de los protocolos de correo

- Tráfico en texto claro Sin cifrado, cualquier persona con acceso a la red puede ver los comandos, las credenciales y el contenido de los correos con herramientas como Wireshark o `tcpdump`.
- Sin verificación de remitente en SMTP básico SMTP no verifica que el remitente declarado en `MAIL FROM` sea legítimo. Cualquiera puede declarar cualquier remitente, lo que facilita el **email spoofing**.
- Open Relay Si `mynetworks = 0.0.0.0/0` en la configuración de Postfix, **cualquier equipo del mundo** puede usar el servidor como relay para enviar correos, facilitando spam masivo y spoofing.

#### Extensiones de autenticación y cifrado

|Mecanismo|Qué protege|Cómo funciona|
|---|---|---|
|**STARTTLS**|La conexión SMTP/IMAP/POP3|Negocia TLS sobre la misma conexión después de establecerla en claro|
|**SMTPS / IMAPS / POP3S**|La conexión completa|TLS desde el inicio de la conexión (puertos 465, 993, 995)|
|**SPF**|Que solo servidores autorizados envíen desde el dominio|Registro TXT en DNS que lista las IPs autorizadas|
|**DKIM**|La integridad del mensaje|Firma criptográfica del servidor en cada correo|
|**DMARC**|Política de qué hacer si falla SPF o DKIM|Registro TXT en DNS que indica rechazar/cuarentena/nada|
|**SMTP-Auth**|La autenticación del cliente al enviar|Autenticación obligatoria en el puerto 587 (submission)|

> [!NOTE] 
> SPF + DKIM + DMARC juntos Los tres se complementan: SPF verifica el servidor de envío, DKIM verifica que el mensaje no fue alterado, y DMARC define qué hacer cuando alguno falla. Un dominio bien configurado tiene los tres.

#### Enumeración del servicio (pentesting)

```bash
# Escaneo con nmap — scripts por defecto
nmap -sCV -p 25,110,143,465,587,993,995 10.129.14.128

# Verificar si el servidor es un open relay
nmap --script smtp-open-relay -p 25 10.129.14.128

# Enumerar usuarios via VRFY
smtp-user-enum -M VRFY -U /usr/share/wordlists/users.txt -t 10.129.14.128

# Interacción manual SMTP
telnet 10.129.14.128 25
nc -nv 10.129.14.128 25

# Interacción con IMAP/POP3 cifrado
openssl s_client -connect 10.129.14.128:993    # IMAPS
openssl s_client -connect 10.129.14.128:995    # POP3S
openssl s_client -connect 10.129.14.128:465    # SMTPS

# curl para listar buzones IMAP
curl -k 'imaps://10.129.14.128' --user jessica:p4ssw0rd
curl -k 'imaps://10.129.14.128/INBOX' --user jessica:p4ssw0rd -v
```

---
# 📶 SNMP

> **SNMP** es el protocolo estándar para monitorizar y gestionar dispositivos de red: routers, switches, servidores, impresoras, UPS... Un sistema central (**manager**) consulta a los dispositivos (**agents**) datos como uso de CPU, tráfico de red o temperatura, y los agentes responden con valores estructurados de su base de datos local (**MIB**). Solo **SNMPv3** ofrece autenticación y cifrado reales. Las versiones anteriores transmiten todo en texto claro, incluyendo las contraseñas.

```
  [Manager SNMP]                               [Agent SNMP]
  (servidor de monitorización)                 (router, switch, servidor...)
         │                                            │
         │── GET / GETNEXT / GETBULK ── UDP 161 ─────▶│
         │   "dame el valor del OID X"                │
         │                                            │ busca OID en MIB
         │◀─ RESPONSE ────────────────────────────────│
         │   valor del OID X                          │
         │                                            │
         │◀─ TRAP / INFORM ──────────── UDP 162 ──────│
         "ha ocurrido un evento" (el agente avisa sin que le pregunten)
```

**Puertos:**

- `UDP 161` → consultas del manager al agente
- `UDP 162` → traps/informs del agente al manager

SNMP usa UDP porque prioriza el rendimiento sobre la fiabilidad. Un sistema de monitorización hace miles de consultas por minuto a decenas de dispositivos: el overhead de establecer conexiones TCP haría el sistema lento. Las pérdidas ocasionales de paquetes son aceptables en monitorización. Para los INFORM (que sí requieren confirmación), el propio protocolo implementa retransmisión a nivel de aplicación.

----
## Componentes

#### MIB — Management Information Base
Una MIB es un archivo de texto ASCII que describe los objetos SNMP consultables de un dispositivo, indicando el nombre de cada variable y su ubicación dentro de una jerarquía de árbol estandarizada. Las MIB no contienen datos, sino que indican dónde encontrar la información, cómo está estructurada, qué valores puede devolver cada OID y qué tipo de dato utiliza.

#### OID — Object Identifier
Contiene al menos un Identificador de Objeto único (`OID`), el cual, además de la dirección única necesaria y un nombre, también provee información sobre el tipo, los derechos de acceso y una descripción del objeto respectivo. Los OIDs se pueden escribir de forma numérica (`1.3.6.1.2.1.1.1.0`) o con nombre (`SNMPv2-MIB::sysDescr.0`). Ambas formas identifican la misma variable.

----
## 📦 Tipos de mensajes SNMP

| Mensaje    | Dirección       | Descripción                                                          |
| ---------- | --------------- | -------------------------------------------------------------------- |
| `GET`      | Manager → Agent | Solicita el valor de uno o varios OIDs concretos                     |
| `GETNEXT`  | Manager → Agent | Solicita el siguiente OID en el árbol (para iterar la MIB)           |
| `GETBULK`  | Manager → Agent | Solicita múltiples OIDs en una sola petición (v2c/v3, más eficiente) |
| `SET`      | Manager → Agent | Modifica el valor de un OID en el agente                             |
| `RESPONSE` | Agent → Manager | Respuesta a GET, GETNEXT, GETBULK o SET                              |
| `TRAP`     | Agent → Manager | Notificación asíncrona de un evento (no confirmada, v1/v2c)          |
| `INFORM`   | Agent → Manager | Notificación asíncrona confirmada (el manager debe responder)        |


----
## 📦 Versiones SNMP

#### SNMPv1
La versión 1 de SNMP (`SNMPv1`) se utiliza para la gestión y monitorización de redes.Es la primera versión del protocolo y todavía se usa en muchas redes pequeñas. Soporta la obtención de información de dispositivos de red, permite la configuración de dispositivos y proporciona `traps`, que son notificaciones de eventos.

> ⚠️ No tiene un mecanismo de autenticación integrado lo que significa que cualquiera que acceda a la red puede leer y modificar datos de la red, además no soporta cifrado, lo que significa que todos los datos se envían en texto plano y pueden ser fácilmente interceptados.

#### SNMPv2
SNMPv2 existió en diferentes versiones. La versión que todavía existe hoy es `v2c`, y la extensión `c` significa SNMP basado en comunidad. En cuanto a la seguridad, SNMPv2 está a la par con SNMPv1 y ha sido extendido con funciones adicionales del SNMP basado en partes que ya no está en uso. 

En SNMPv1 y v2c, el control de acceso se basa en **community strings**: cadenas de texto plano que actúan como contraseñas compartidas entre el manager y el agente.
- `public` — Read-only - (RO) — Permite leer toda la MIB
- `private` — Read-Write - (RW) — Permite leer y **modificar** la configuración del dispositivo

> ⚠️ El peligro esque viajan sin cifrar y que muchos dispositivos usan los valores por defecto como "public" o "private"

#### SNMPv3
La seguridad se ha incrementado enormemente para `SNMPv3` mediante características de seguridad como la `autenticación` usando nombre de usuario y contraseña y el `cifrado` de la transmisión (vía `clave precompartida` o `pre-shared key`) de los datos. Sin embargo, la complejidad también aumenta en la misma medida, con significativamente más opciones de configuración que `v2c`.

SNMPv3 define tres niveles de seguridad combinables:

| securityLevel  | Autenticación | Cifrado   | Uso                                                   |
| -------------- | ------------- | --------- | ----------------------------------------------------- |
| `noAuthNoPriv` | ❌             | ❌         | Equivalente a v1/v2c en seguridad — solo para testing |
| `authNoPriv`   | ✅ SHA/MD5     | ❌         | Verifica identidad pero el tráfico es legible         |
| `authPriv`     | ✅ SHA/MD5     | ✅ AES/DES | El nivel recomendado en producción                    |

Es importante notar que muchas organizaciones todavía están usando `SNMPv2`, ya que la transición a `SNMPv3` puede ser muy compleja, pero los servicios aún necesitan permanecer activos. 

----
## ⚙️ Configuración

Linux (`/etc/snmp/snmpd.conf`)

```bash
# ── SNMPv1/v2c — community strings ───────────────────────────────────────────
# rocommunity <community> [red/IP]
rocommunity  public    127.0.0.1           # solo lectura desde localhost
rocommunity  monit     192.168.1.0/24      # solo lectura desde la LAN
rwcommunity  private   127.0.0.1           # lectura/escritura solo desde localhost

# ── SNMPv3 — usuarios ─────────────────────────────────────────────────────────
# createUser <usuario> <authProtocol> <authPass> <privProtocol> <privPass>
createUser  jessica  SHA  "auth_password_123"  AES  "priv_password_456"

# Dar acceso al usuario (rwuser = lect/escrit, rouser = solo lectura)
rouser jessica authPriv

# ── Información del sistema ───────────────────────────────────────────────────
sysLocation "CPD Planta 2, Rack 3"
sysContact  "sysadmin@empresa.com"
sysName     "servidor-prod-01"

# ── Restringir acceso a interfaces concretas ───────────────────────────────────
agentAddress udp:161,udp6:[::1]:161        # escuchar solo en UDP

# ── Qué OIDs exponer (vistas) ─────────────────────────────────────────────────
view systemonly included .1.3.6.1.2.1.1   # solo el árbol system
view systemonly included .1.3.6.1.2.1.25  # y hrSystem (host resources)
```

Para aplciar cambios `sudo systemctl restart snmpd` y para verificar que está escuchando `ss -ulnp | grep 161`

Si queremos crear un usuario debe ser con el servicio parado: 
```bash
sudo net-snmp-create-v3-user -ro -A "auth_password_123" -a SHA -X "pass123" -x AES jessica
```

----
## 💻 Comandos de consulta

Tenemos varias maneras de autenticarnos

| Descripción         | Comando                                                                                 |
| ------------------- | --------------------------------------------------------------------------------------- |
| SNMPv2              | `snmpget -v2c -c <community_string> <ip>`                                               |
| SNMPv3 noAuthNoPriv | `snmpget -v3 -l noAuthNoPriv -u <user> <ip>`                                            |
| SNMPv3 authNoPriv   | `snmpget -v3 -l authNoPriv -u <user> -a SHA -A <auth_pass> <ip>`                        |
| SNMPv3 AuthPriv     | `snmpwalk -v3 -l authPriv -u <user> -a SHA -A <auth_pass> -x AES -X <cipher_pass> <ip>` |


Si dejamos esto, recorredemos toda la MIB (walk), Si no, debemos especificar el OID tras todo el comando, por ejemplo
```bash
snmpget -v2c -c public 192.168.1.1 sysDescr # en forma texto
snmpget -v2c -c public 192.168.1.1 1.3.6.1.2.1.1.1.0 # en forma numérica
```

Despues de eso, podemos enumerar:

| OID                    | Nombre              | Descripción                                       |
| ---------------------- | ------------------- | ------------------------------------------------- |
| `1.3.6.1.2.1.1.1.0`    | `sysDescr`          | Descripción del sistema (SO, versión...)          |
| `1.3.6.1.2.1.1.3.0`    | `sysUpTime`         | Tiempo activo desde el último reinicio            |
| `1.3.6.1.2.1.1.5.0`    | `sysName`           | Nombre del dispositivo (hostname)                 |
| `1.3.6.1.2.1.1.6.0`    | `sysLocation`       | Ubicación física del dispositivo                  |
| `1.3.6.1.2.1.2.1.0`    | `ifNumber`          | Número de interfaces de red                       |
| `1.3.6.1.2.1.2.2.1.2`  | `ifDescr`           | Descripción de cada interfaz                      |
| `1.3.6.1.2.1.4.20.1.1` | `ipAdEntAddr`       | Tabla de IPs del dispositivo                      |
| `1.3.6.1.2.1.6.13.1.3` | `tcpConnTable`      | Conexiones TCP activas                            |
| `1.3.6.1.4.1`          | —                   | Rama de OIDs privados/propietarios de fabricantes |
|                        | `hrSWInstalledName` | Software instalado                                |
|                        | `ipRouteTable`      | Tabla de rutas                                    |
|                        | `hrSWRunName`       | Procesos en ejecución (en servidores con agente)  |


Si tenemos permisos de escritura, podemos modificar un atributo: `snmpset (...) sysName.0 s "nuevo-nombre"`

----
## Vulnerabilidades y fallos de seguridad

> [!CAUTION]
> **Community strings por defecto**
> 
> Los dispositivos pueden enviarse del fabricante con `public` o `private` como community string. Un atacante en la red puede leer toda la MIB sin ninguna credencial adicional. Si `public` funciona queda espuesta la información completa del sistema, si `private` funciona, el atacante puede modificar la configuración del dispositivo.

> [!CAUTION]
> **Community strings interceptables**
> 
> Cualquier atacante con un sniffer puede leer la community string con facilidad con `sudo tcpdump -i eth0 -w snmp.pcap port 161`. Una vez obtenida, puede suplantar la IP del manager e interactuar con la red.

> [!CAUTION]
> **SNMPv3 con noAuthNoPriv — falsa seguridad**
> 
> Un dispositivo configurado con SNMPv3 pero usando el nivel `noAuthNoPriv` ofrece exactamente la misma seguridad que v1/v2c: sin autenticación real y sin cifrado.

> [!CAUTION]
> **SNMP expuesto a internet / redes no confiables**
> 
> SNMP debería estar accesible **únicamente** desde la red de gestión. Expuesto a internet, permite a un atacante enumerar la topología de la red, obtener información del sistema y versiones de software y como última instancia, si tiene permisos de escritura, modificar configuraciones

#### 🔍 Enumeración SNMP

```bash
# Descubrir dispositivos SNMP en la red
nmap -sU -p 161 --open 192.168.1.0/24

# Probar community strings comunes
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt 192.168.1.1

# Fuerza bruta de community strings
hydra -P /usr/share/wordlists/rockyou.txt -v 192.168.1.1 snmp

# Enumeración completa con snmp-check
snmp-check 192.168.1.1 -c public

# Walk completo guardando a archivo
snmpwalk -v2c -c public 192.168.1.1 > snmp_full_walk.txt

# Módulo de Metasploit para enumeración SNMP
msf> use auxiliary/scanner/snmp/snmp_enum
msf> use auxiliary/scanner/snmp/snmp_enumusers
msf> use auxiliary/scanner/snmp/snmp_login     # brute force community strings

# Enumeración de un OID con braa
braa public@10.129.14.128:.1.3.6.*
```

---
