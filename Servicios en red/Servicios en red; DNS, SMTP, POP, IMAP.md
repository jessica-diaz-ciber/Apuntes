> DNS (Domain Name System) es el sistema que traduce nombres de dominio legibles (`google.com`) a direcciones IP que los equipos pueden usar (`142.250.184.46`). Es una base de datos distribuida y jerárquica: ningún servidor DNS conoce todos los dominios del mundo, pero cualquier servidor puede llegar a la respuesta correcta siguiendo la jerarquía.

Sin DNS, acceder a cualquier servicio en internet requeriría recordar direcciones IP. Con DNS, el nombre es estable aunque la IP cambie.

DNS usa **UDP puerto 53** por defecto, con **TCP puerto 53** para respuestas grandes (>512 bytes) o transferencias de zona. 

---
## La jerarquía DNS

DNS organiza todos los dominios del mundo en un árbol invertido con la raíz en la cima:
```c
                        . (raíz)
                        │
          ┌─────────────┼─────────────┐
         com            org           es  // TLDs 
          │              │
    ┌─────┴─────┐        │
  google      github   wikipedia   // dominios
    │
   mail     www    api   ...   // subdominios)
```

Cada nodo del árbol es una **zona DNS** gestionada de forma independiente. Google gestiona `google.com` y todos sus subdominios. ICANN gestiona `.com`, `.org`, etc. Nadie gestiona todo el árbol de forma centralizada.

Un nombre de dominio completo se lee de derecha a izquierda siguiendo el árbol:
```c
mail.google.com.
│    │       │  └── raíz // el punto final, normalmente implícito
│    │       └── zona com // gestionada por Verisign
│    └── zona google.com // gestionada por Google
└── subdominio // registro dentro de la zona google.com
```

---
## Tipos de servidores DNS
La cadena de peticiones DNS se conforma por este orden

1️⃣ **El navegador o aplicación hace la consulta:** `¿Cuál es la IP de www.ejemplo.com?`

2️⃣ El sistema busca en la caché local o en el archivo `hosts`. Si no lo encuentra ahí, consulta al `resolver` un componente que se encarga de reenviar las peticiones DNS

3️⃣ El resolver pregunta a un `Root Server` que le dará una lista de `TLD server`s y sus IPs `a.gtld-servers.net. A 192.5.6.30`

4️⃣ Los TLD servers le darán la lista de `Authoritative Server`s y sus IPs, `ejemplo.com. NS ns1.proveedor.com.`

5️⃣ El `Authoritative Server` es el propietario del dominio y resolverá la IP, que se cacheará en el cliente durante un tiempo limitado (segundos indicados en el "TTL")  `www.ejemplo.com. 300 IN A 93.184.216.34`

> [!INFO]
> 
> **Root server** — la raíz del sistema
> Existen 13 grupos de Root Servers, identificados con letras de la  `A` a la `M` (`a.root-servers.net` a `m.root-servers.net`). Son el punto de partida de cualquier resolución. No conocen las IPs, pero saben que servidores gestionan cada TLD del mundo. Por ejemplo, para el TLD `es` se pregunta al TLD nameserver `a.nic.es`. Fisicamente hay miles de instancias de estos servidores distribuidas por todo el mundo, mediante **anycast**: la misma IP enruta a la instancia más cercana geográficamente.

> [!INFO]
> 
> **TLD Nameserver** — los servidores de dominio de nivel superior
> Gestionan un TLD concreto (`.com`, `.org`, `.net`, `.es`...). No conocen las IPs de los dominios, pero conocen a que Authoritative Nameserver hacerle la petición para cada dominio registrado bajo ese TLD. Por ejemplo para`google serían` serían `ns1.google.com, ns2.google.com...`. Cuando se registra un dominio con por ejemplo GoDaddy, se notifica al operador del TLD los nameservers del dominio.

> [!INFO]
> 
> **Authoritative Nameservers** — los servidores del dominio
> Son los servidores que tienen la **respuesta definitiva** para un dominio concreto. Gestionan la **zona DNS** del dominio con todos sus registros (A, MX, TXT, CNAME...). Cuando un authoritative nameserver responde, incluye el flag `aa` (Authoritative Answer) en la cabecera DNS. Esa respuesta es la fuente de verdad, no viene de caché.

> [!NOTE]
> 
> **Recursive Resolver** — el intermediario
> También llamado **full-service resolver** o **recursive nameserver**. Es el servidor DNS que usan los clientes (el de tu router, `8.8.8.8` de Google, `1.1.1.1` de Cloudflare). Su trabajo es recibir la consulta del cliente, preguntar a los root servers, TLD servers y authoritative servers hasta obtener la respuesta, cachearla durante el TTL y devolver el resultado al cliente. El cliente hace **una sola consulta** al resolver y recibe la respuesta. El resolver hace todo el trabajo de navegar la jerarquía.

Por último tenemos el `Caching-Only Nameserver`, un resolver que no tiene zonas propias ni hace resolución completa, solo reenvía las consultas a otro resolver y cachea las respuestas. Muchos routers domésticos y servidores DNS corporativos funcionan así.
```c
cliente → resolver → root → TLD → authoritative → resolver → cliente // ~100-300ms
cliente → resolver con la respuesta cacheada → cliente  // <5ms
```


#### Protección DNS
Por defecto el tráfico DNS no viaja cifrado (es visible para el ISP y el resto de equipos de la red), así que tenemos varias tecnologías que permiten protegerlo

> DNSSEC añade firmas criptográficas con claves de verificación `DNSKEY` para garantizar la autenticidad e integridad de los registros DNS. Protege contra **DNS spoofing** y **cache poisoning**.

Luego tenemos

| Protocolo                | Puerto  | Descripción                                                                             |
| ------------------------ | ------- | --------------------------------------------------------------------------------------- |
| **DoT** (DNS over TLS)   | 853/TCP | DNS sobre TLS. El proveedor puede ver que se hace una consulta DNS pero no el contenido |
| **DoH** (DNS over HTTPS) | 443/TCP | DNS encapsulado en HTTPS. Idéntico al tráfico web, más difícil de bloquear              |
| **DoQ** (DNS over QUIC)  | 853/UDP | DNS sobre QUIC. Más rápido que DoT con menos latencia                                   |

------------
## Registros DNS

| **Registro DNS** | **Descripción**                                                                                                                                                                                                                      | Ejemplo                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------- |
| `A`              | Devuelve una dirección IPv4 del dominio/subdominio solicitado como resultado.                                                                                                                                                        | `mail IN A 93.184.216.35` para `mail.web.com`              |
| `AAAA`           | Devuelve una dirección IPv6 del dominio/subdominio solicitado.                                                                                                                                                                       | `@ IN AAAA 2001:db8::1` para `ejemplo.com`                 |
| `MX`             | Devuelve los servidores de correo responsables como resultado y se consultan según la prioridad (menor = preferido)                                                                                                                  | `@ IN MX 10 mail.web.com.` servidor mail y prioridad       |
| `NS`             | Devuelve los servidores DNS (Authritative namerservers) del dominio. Ej `ns1.google.com`<br>En los `Glue records` se devuelve la IP del servidor DNS mediante un registro `A`                                                        |                                                            |
| `TXT`            | Este registro puede contener diversa información: Ej certificados SSL, entradas SPF y DMARC para validar el tráfico de correo y protegerlo del spam.                                                                                 |                                                            |
| `CNAME`          | Este registro sirve como un alias para otro nombre de dominio.                                                                                                                                                                       | `ftp IN CNAME web.com.` el servidor ftp está en  `web.com` |
| `PTR`            | El registro PTR funciona al revés (búsqueda inversa o `reverse lookup`). Convierte direcciones IP en nombres de dominio válidos.                                                                                                     |                                                            |
| `SOA`            | Proporciona el servidor DNS autoritativo que opera sobre la zona (`MNAME`)  y la dirección de correo electrónico del contacto administrativo (`RNAME`). Está en formato `admin.ejemplo.com.` que se traduce como `admin@ejemplo.com` |                                                            |
| `CAA`            | Servidores CA que pueden emitir certificados válidos TLS                                                                                                                                                                             | `@ IN CAA 0 issue "letsencrypt.org"`                       |


---
## Configuración de zona DNS

Una **zona DNS** es un fragmento del árbol DNS que gestiona un servidor autoritativo. El archivo de zona define todos los registros de ese dominio en un formato estandarizado (Bind). Por ejemplo `/etc/bind/db.domain.com`
```c
$ORIGIN web.com.  // dominio base de esta zona (el punto final es obligatorio)
$TTL 3600             // TTL por defecto para todos los registros de la zona

// SOA — Start of Authority (obligatorio uno por zona) 
// Internet - SOA - MNAME (AuthNameserver) - RNAME (correo "." 🡆 "@") - (opciones)
@            IN    SOA   ns1.web.com.  admin.web.com. ( 
        // Las opciones para comprobar cambios, refrescar y sincronizar datos
        ) 
            
// Registros individuales: A, NS, TXT (DKIM, DMARC, servidores SSL...)
             IN    NS    ns1.web.com. 
server1      IN    A     10.129.14.5
ns1          IN    A     10.129.14.2
ftp          IN    CNAME server1
```

Ademñas, para que el `FQDN` (dominio completo) se esuelva a partir de la IP, el servidor DNS debe tener un archivo de búsqueda inversa (registro PTR), como `/etc/bind/db.10.129.14`
```c
$ORIGIN 14.129.10.in-addr.arpa
$TTL 86400
// el resto es una copia del archivo de zona con sus mismos registros; SOA y demás
```

--------
## Dig

`dig` (Domain Information Groper) es la herramienta de línea de comandos estándar para hacer consultas DNS. Esta herramienta muestra la respuesta completa del servidor DNS

```
dig <@servidor_dns> <dominio> <tipo_de_consulta> <opciones>
```

Ejecutar `dig google.com` produce una salida estructurada en secciones bien definidas. Vamos sección a sección:
```c
<<>> DiG 9.18.12 <<>> google.com // versión del protocolo
// opcode: tipo de consulta y status: resultado, id: identificador
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345 

// flags (opciones), QUERY y ANSWER número de consultas respuestas
// AUTHORITY: 0, ADDITIONAL: 1 🡆 no es el servidor autoritativo 🡆 hay que consultar a otro
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

// pregunta 🡆 dominio consultado, clase internet (IN) y tipo de registro solicitado (A)
;; QUESTION SECTION:
;google.com.                    IN      A

// respuesta 🡆 dominio TTL clase registro valor
;; ANSWER SECTION:
google.com.             300     IN      A       142.250.184.46

// metadatos 🡆 tiempo, ip y puerto del servidor DNS, fechas y tamaño de respuesta
;; Query time: 23 msec
;; SERVER: 192.168.1.1#53(192.168.1.1) (UDP)
;; WHEN: Mon Jan 01 12:00:00 UTC 2024
;; MSG SIZE  rcvd: 55 
```

Las consultas TXT dan información adicional: SPF (anti-spam), DKIM, verificaciones de dominio y muchas otras cosas.
```bash
dig google.com TXT
# ;; ANSWER SECTION:
# google.com.  3600  IN  TXT  "v=spf1 include:_spf.google.com ~all"
# google.com.  3600  IN  TXT  "google-site-verification=wD8N7i1JTNTkezJ49swvWW..."
```

- Para poder ver todos los registros se puede usar una consulta `ANY`, el problema esque  muchos servidores modernos ignoran o limitan estas consultas por seguridad

Luego tenemos estas opciones

| **Registro DNS**      | **Descripción**                                                                              | Ejemplo                                |
| --------------------- | -------------------------------------------------------------------------------------------- | -------------------------------------- |
| `+short`              | solo el resultado                                                                            | `dig google.com +short` 🡆 solo la IP` |
| `+noall +answer`      | Más completo que `+short` porque muestra el TTL y el tipo, pero sin las estadísticas.        | `dig google.com +noall +answer`        |
| `@servidor -p puerto` | Consultar a un servidor DNS alternativo y permite usar otro puerto                           | `dig @9.9.9.9 -p 53 google.com`        |
| `+trace`              | Simula la resolución recursiva desde los root servers, o sea muestra todo el camino completo |                                        |

## Transferencia de zona (AXFR)

La `transferencia de zona` (`zone transfer`) permite sincronizar cambios en los servidores DNS de una empresa, para que todos tengan el mismo archivo de zona. En este proceso un servidor maestro propaga su archivo de zona a servidores esclavos. 

Siempre hay un servidor primario con los datos originales y servidores secundarios que distribuyen la carga y protegen al primario de ataques. El primario siempre es el maestro.

Usando una clave secreta `rndc-key`, los servidores se aseguran de que se comunican con su propio maestro o esclavo. 

Si se permite una transferencia de zona desde cualquier IP, se pueden obtener todos los registros del dominio y exponer sistemas internos. Por tanto esta transferencia debe permitirse desde las IPs de los servidores secundarios, nunca desde cualquier dirección.
`dig @ns1.ejemplo.com ejemplo.com AXFR`

Con la herramienta DNSenum, podemos enumerar subdominios válidos
```bash
dnsenum --dnsserver 10.129.14.128 --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt web.htb
```


