# 🛡️ Suricata — IDS/IPS/NSM

> [!abstract] Resumen
>  **Suricata** es un motor de detección de intrusiones (IDS), prevención de intrusiones (IPS) y monitorización de seguridad de red (NSM) de código abierto, desarrollado por la OISF. Inspecciona el tráfico de red en tiempo real usando reglas y genera alertas en formato JSON que se pueden ingestar en un SIEM.

| Modo                 | Descripción                                                                                                  | Operación   |
| -------------------- | ------------------------------------------------------------------------------------------------------------ | ----------- |
| IDS (pasivo)         | Lee tráfico por un tap o SPAN port                                                                           | alert/pass  |
| IPS                  | Se pone en medio del flujo y puede bloquearlo con DROP/REJECT                                                | drop/reject |
| NSM (monitorización) | Registra todo el tráfico (flows, HTTP, DNS, TLS, archivos...) aunque no haya alertas eve.json siempre activo |             |

---
# 1. 📦 Introducción

## 1.1. 📦 Instalación

Paquete debian
```bash
# Añadir PPA de OISF (siempre la última versión estable)
sudo apt-get install -y software-properties-common && sudo add-apt-repository ppa:oisf/suricata-stable

# Instalar
sudo apt-get update && sudo apt-get install -y suricata

# Verificar versión y opciones compiladas
suricata --build-info
sudo systemctl status suricata
```

Compilado
```bash
# Dependencias
sudo apt -y install autoconf automake build-essential cargo cbindgen libjansson-dev libpcap-dev libpcre2-dev 
	libtool libyaml-dev make pkg-config rustc zlib1g-dev

# Compilar
tar xzvf suricata-7.x.x.tar.gz && cd suricata-7.x.x
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var
make -j$(nproc) && sudo make install-full   # instala binario + config + reglas ET
```

> [!tip] 
> `make install-full` Combina `install` + `install-conf` (crea directorios y `suricata.yaml`) + `install-rules` (descarga reglas Emerging Threats). Es la forma más rápida de tener Suricata listo para correr.

---
## 1.2. 🗂️ Estructura de archivos

```powershell
/etc/suricata/
├── suricata.yaml          # configuración principal
├── classification.config  # categorías de alertas y su prioridad
├── reference.config       # URLs de referencia para reglas (CVEs, etc.)
├── threshold.config       # supresión y umbralización de alertas
└── rules/                 # directorio de reglas
    ├── suricata.rules     # reglas gestionadas por suricata-update
    └── local.rules        # reglas personalizadas ← aquí van las tuyas

/var/lib/suricata/
└── rules/                 # reglas descargadas por suricata-update

/var/log/suricata/
├── eve.json               # log principal (alertas, flows, HTTP, DNS...)
├── fast.log               # log legible de alertas (formato simple)
├── stats.log              # estadísticas del motor
└── suricata.log           # log del propio proceso (errores, inicio...)
```

Para ver logs en tiempo real `tail -f /var/log/suricata/fast.log`

---
# 2. ⚙️ Configuración — suricata.yaml

El archivo de configuración usa formato **YAML**. Tiene miles de líneas; pero se divide en cuatro Steps o pasos

| Paso   | Uso                                    |
| ------ | -------------------------------------- |
| Step 1 | Indicarle a suricata la red            |
| Step 2 | Activar salids o ficheros de log       |
| Step 3 | Configuración de captura               |
| Step 4 | Configuración de la capa de aplicación |
## 2.1. Step 1: Red

Primero configuramos las variables de red
```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.0.0/16,10.0.0.0/8]"  # La red/redes a monitorear:
    EXTERNAL_NET: "!$HOME_NET" # Todo lo que no sea nuestra red, se considera externo
```

Luego encontraremos muchas lineas del estilo `PROTOCOLO_SERVERS:` que lo ideal esque apunten a nuestra red. Ej: `HTTP_SERVERS: "$HOME_NET"`

Despues, la sección `port-groups:`, que agrupa los puertos de cada protocolo, `PROTOCOLO_PORTS:`. Ej: `HTTP_PORTS:  "80"`

> [!warning] `HOME_NET` es crítico 
> Si `HOME_NET` no refleja tu red real, Suricata no distinguirá correctamente entre tráfico interno y externo, generando falsos positivos o perdiendo alertas. Es lo primero que hay que ajustar.

Existe una opción para ver dónde guardmos las reglas
```bash
default-rule-path: /etc/suricata/rules
rule-files:
  - mis_reglas.rules # En /etc/suricata/rules
```

## 2.2. Step 2: Outputs

En este paso decidimos **qué información guarda Suricata** y en qué formato.

```yaml
outputs:
  - eve-log:             # eve-log (eve.json), fast (fast.log), stats (stats.log)
      enabled: yes       # activado o desactivado
      filetype: regular  # tipo de fichero
      filename: eve.json # nombre
      types:
        - alert:
            payload: yes           # incluir payload en base64
            payload-printable: yes # y en texto plano
            metadata: yes          # capas de aplicación, flow...
        - http:
            extended: yes          # User-Agent, referer, etc.
        - dns:
            requests: no
            responses: no
        - tls:
            extended: yes          # JA3, SNI, certificado...
        - files:
            force-magic: yes       # magic bytes de archivos detectados
```

> [!warning]  Importante: no activar todo
> Suricata puede generar  muchísimo volumen de logs. Por ejemplo `flow`, `http` y `dns` pueden generar varios GB al día fácilmente. Por tanto lo ideal es empezar con pocos eventos e ir ajustando si hay necesidad

> [!tip] Separar logs para un SIEM 
> En entornos de producción es útil separar alertas de NSM en archivos distintos para ingestión diferenciada:
> 
> ```yaml
> outputs:
>   - eve-log:
>       filename: eve-alerts.json
>       types: [alert, anomaly]
>   - eve-log:
>       filename: eve-nsm.json
>       types: [http, dns, tls, flow, files]
> ```

## 2.3. Step 3: Captura de paquetes

Aquí configuramos cómo Suricata captura tráfico.  La sección cambia dependiendo del método usado: por ejemplo, se puede usar `af_packet`  que es el formato de linux, pero tambien `pcap` que es mas simple, pero con menor rendimiento

- **Interfaz de captura**
```yaml
af-packet: # Modo IDS — captura pasiva
  - interface: eth0    # La interfaz que queremos monitoree suricata
    cluster-id: 99     # ID interno, dejarlo tal cual
    defrag: yes        # Reensamblado de paquetes
    cluster-type: cluster_flow # Asegura que todo un flujo TCP vaya al mismo thread y no se desperdigue
```

- **Arrancar en modo IPS**: `suricata -c /etc/suricata/suricata.yaml -q 0`
```yaml
nfq: # Modo IPS — inline con NFQueue (Linux)
  mode: accept    # accept | repeat | route
  failed-open: yes
```

- **Reglas**
```yaml
default-rule-path: /var/lib/suricata/rules
rule-files:
  - suricata.rules                    # reglas de suricata-update (ET, etc.)
  - /etc/suricata/rules/local.rules   # tus reglas personalizadas
```

## 2.3. Step 4: Capa de aplicación

Aquí Suricata analiza protocolos de alto nivel: HTTP, DNS, TLS, SSH, SMTP, SMB, RDP, ...
```yaml
app-layer:
  protocols:
    http:
      enabled: yes
    dns:
      enabled: yes
      tcp:
        detection-ports:
          dp: 53
      udp:
        detection-ports:
          dp: 53
    tls:
      enabled: yes
      detection-ports:
        dp: 443
```

Pondremos una linea con cada uno y `enabled:yes`

---
# 3. 🔄 Gestión de reglas

## 3.1. Descargar reglas

`suricata-update` es la herramienta oficial para descargar y gestionar conjuntos de reglas.

| Uso                                                        | Comando                                                                 |
| ---------------------------------------------------------- | ----------------------------------------------------------------------- |
| Actualizar reglas (descarga y aplica fuentes configuradas) | `sudo suricata-update`                                                  |
| Listar fuentes disponibles                                 | `sudo suricata-update list-sources`                                     |
| Añadir fuente (ej: Emerging Threats Pro, abuse.ch...)      | `sudo suricata-update enable-source et/open`                            |
| Ver fuentes activas                                        | `sudo suricata-update list-enabled-sources`                             |
| Actualizar e indicar a Suricata que recargue las reglas    | `sudo suricata-update && sudo kill -USR2 $(pidof suricata)`             |
| Recargar reglas sin reiniciar                              | `sudo kill -USR2 $(pidof suricata)` o `sudo suricatasc -c reload-rules` |
- **Desactivar una regla concreta sin eliminarla**: en `/etc/suricata/disable.conf`
```bash
re:2019401                      # por regex del SID
group:emerging-web_server.rules # por grupo
```

- **Modificar una regla concreta**: en `/etc/suricata/modify.conf`
```bash
2019401 "alert" "drop"   # cambiar acción
```
 
---
## 3.2. 📜 Sintaxis de reglas

Toda regla Suricata tiene estas partes:

| acción  | protocolo | src_ip      | src_port | dirección | dst_ip          | dst_port | (opciones;)                  |
| ------- | --------- | ----------- | -------- | --------- | --------------- | -------- | ---------------------------- |
| `alert` | `http`    | `$HOME_NET` | `any`    | `->`      | `$EXTERNAL_NET` | `any`    | `(msg:"..."; sid:1; rev:1;)` |

> [!example] Accion
> | Acción   | Modo      | Descripción                                 |
| -------- | --------- | ------------------------------------------- |
| `alert`  | IDS + IPS | Genera alerta y registra el paquete         |
| `pass`   | IDS + IPS | Deja pasar el paquete, no genera alerta     |
| `drop`   | Solo IPS  | Descarta el paquete y genera alerta         |
| `reject` | Solo IPS  | Envía TCP RST / ICMP unreachable y descarta |

> [!example] Protocolos
> | Capa                 | Valores                                                                                 |
| -------------------- | --------------------------------------------------------------------------------------- |
| **Red / Transporte** | `tcp`, `udp`, `icmp`, `ip` (= cualquiera)                                               |
| **Aplicación (L7)**  | `http`, `https`, `tls`, `dns`, `ftp`, `smb`, `ssh`, `smtp`, `http2`, `rdp`, `modbus`... |
>> Usar `http` en lugar de `tcp` en el header hace que Suricata solo evalúe la regla si el tráfico ha sido identificado como HTTP por el motor de detección de aplicaciones. Más preciso y más eficiente.

> [!example] Dirección
> | Capa                | Valores |
| ------------------- | ------- |
| De origen a destino | `->`    |
| Bidireccional       | `<>`    |

> [!example] IPs y puertos
| Tipo                | IPs                            | Puertos           |
| ------------------- | ------------------------------ | ----------------- |
| Cualquier IP/puerto | `any`                          | `any`             |
| Red/puerto          | `192.168.1.0/24`               | `80`              |
| Lista               | `[192.168.1.0/24, 10.0.0.0/8]` | `[80, 443, 8080]` |
| Negación            | `!192.168.1.5`                 | `!80`             |
| Rango               |                                | `1024:65535` o `1024:`      |

## 3.3. 📜 Opciones de reglas

Las opciones van entre paréntesis, separadas por `;`. La sintaxis es `configuracion: opcion1, opcion2;`. El orden es clave

Documentación oficial:  [doc](https://docs.suricata.io/en/suricata-8.0.4/rules/meta.html)

> [!example] Metadatos (obligatorios)
> | Opción      | Descripción                                         | Ejemplo                           |
| ----------- | --------------------------------------------------- | --------------------------------- |
| `msg`       | Mensaje descriptivo de la alerta                    | `msg:"ET SCAN Nmap SYN Scan";`    |
| `sid`       | ID única de la regla (≥1000000 para custom)         | `sid:1000001;`                    |
| `rev`       | Revisión de la regla                                | `rev:1;`                          |
| `classtype` | Categoría de la alerta (de `classification.config`) | `classtype:trojan-activity;`      |
| `reference` | URL o CVE de referencia para solucionar el problema       | `reference:cve,CVE-2014-1234;`      |
| `metadata`  | Metadatos adicionales (fecha, autor...)             | `metadata:created_at 2024_01_01;` |
| `priority`  | Prioridad (1=alta, 4=baja) — sobreescribe classtype | `priority:1;`                     |

> [!example] Buffer de protocolo
| Descripción                              | Ejemplo                                  |
| ---------------------------------------- | ---------------------------------------- |
| `http.method `                           | método (GET, POST...)                    |
| `http.uri`                               | URI sin parámetros                       |
| `http.request_line`                      | Línea completa de request                |
| `http.user_agent`                        | User-Agent                               |
| `http.host`                              | Host header                              |
| `http.cookie`                            | Cookie header                            |
| `http.request_body`/`http.response_body` | Body de la petición / respuesta          |
| `http.stat_code` / `http.stat_msg`       | código / mensaje de estado (200, 404...) |
> > Para otros protocolos tenemos por ejemplo `dns.query` para el nombre del dominio consultado en DNS o `ssh.proto` para la versión de SSH empleada

> [!example] Detección de contenido
> | Opción         | Descripción                                        | Ejemplo                  |
| -------------- | -------------------------------------------------- | ------------------------ |
| `content`      | Buscar cadena literal en el payload                | `content:"cmd.exe";`     |
| `nocase`       | Ignora mayúsculas/minúsculas                       | `content:"GET"; nocase;` |
| `offset`       | Posición de inicio en el payload                   | `offset:4;`              |
| `depth`        | Hasta qué posición buscar                          | `depth:20;`              |
| `distance`     | Posición relativa al match anterior                | `distance:0;`            |
| `within`       | Máximo N bytes desde el match anterior             | `within:10;`             |
| `pcre`         | Expresión regular (Perl-compatible)                | `pcre:"/\bpasswd\b/i";`  |
| `fast_pattern` | Marca este content como patrón de pre-filtrado MPM | `fast_pattern;`          |

> [!example] Flow y estado
>| Opción                       | Descripción                                                      |
| ---------------------------- | ---------------------------------------------------------------- |
| `flow:established,to_server` | Solo tráfico establecido, hacia el servidor (cliente → servidor) |
| `flow:established,to_client` | Solo respuestas del servidor                                     |
| `flow:stateless`             | No requiere tracking de estado                                   |
| `flowbits:set,nombre`        | Establece un flag para correlacionar con otra regla              |
| `flowbits:isset,nombre`      | Condición: el flag debe estar activo                             |
| `flowbits:unset,nombre`      | Limpia el flag                                                   |

> [!example] Threshold (control de frecuencia)
> | Descripción                                              | Opción                                                          |
| -------------------------------------------------------- | --------------------------------------------------------------- |
| Alertar solo si ocurre N veces en T segundos (por track) | `threshold: type threshold, track by_src, count 5, seconds 60;` |
| Alertar solo la primera vez                              | `threshold: type limit, track by_src, count 1, seconds 3600;`   |
| Alertar si ocurre más de N veces (both: combina los dos) | `threshold: type both, track by_src, count 5, seconds 60;`      |

---
## 3.4. ✍️ Reglas personalizadas — ejemplos prácticos

Las reglas propias van en `/etc/suricata/rules/local.rules`. Los SIDs personalizados deben ser superiores a `1000000` (millón).

- Detectar ping (ICMP Echo Request). El campo `itype` es el tipo de ping. Por ejemplo el `itype:8;` es un echo.
```
alert icmp any any -> $HOME_NET any ( msg:"ICMP Echo Request entrante"; itype:8; sid:1000001; rev:1; )
```

- Detectar escaneo de puertos (SYN sin ACK): Para el resto simplemente se cambia la flag, por ejemplo la `F` para FIN scan
```bash
alert tcp any any -> $HOME_NET !22 (msg:"Nmap SYN scan";flags:S; flow:stateless; # Escaneo SYN sin tracking de estado
    classtype:attempted-recon; sid:1000002; rev:1;)
```

- Detectar petición HTTP con User-Agent sospechoso
```
alert http $HOME_NET any -> $EXTERNAL_NET any (
    msg:"HTTP User-Agent sospechoso — curl/wget saliente"; flow:established,to_server;
    http.user_agent; pcre:"/^(curl|wget|python-requests|go-http-client)\//i";
    classtype:policy-violation; sid:1000003; rev:1;)
```

- Detectar descarga de ejecutable por HTTP
```bash
alert http $EXTERNAL_NET any -> $HOME_NET any (
    msg:"Descarga de ejecutable PE via HTTP"; flow:established,to_client;
    http.response_body; content:"MZ"; offset:0; depth:2; fast_pattern; # firma PE: MZ header al prinicpio
    classtype:trojan-activity; sid:1000004; rev:1;)
```

- Detectar consulta DNS a dominio sospechoso
```bash
alert dns $HOME_NET any -> any 53 (
    msg:"Consulta DNS a dominio con muchos subdominios (posible DNS tunneling)";
    dns.query; pcre:"/^[a-z0-9]{20,}\./i"; classtype:bad-unknown;  # subdominio aleatorio largo
    sid:1000005; rev:1; )
```

- Correlación con flowbits — detectar login + acción privilegiada
```bash
alert ssh any any -> $HOME_NET 22 ( # Regla 1: marcar el flujo cuando se detecta login SSH
    msg:"SSH login detectado";  flow:established,to_server;
    flowbits:set,ssh.login; flowbits:noalert;   # no genera alerta, solo marca el flag
    sid:1000006; rev:1; )

alert tcp any any -> $HOME_NET 22 ( # Regla 2: alertar si hay actividad POST-login y el flag está activo
    msg:"Actividad SSH tras login — posible ejecución de comandos";
    flow:established,to_server; flowbits:isset,ssh.login; content:"|00 00 00|";  # cualquier payload SSH
    classtype:successful-admin; sid:1000007; rev:1; )
```

- Detectar PowerShell encoded command (C2 común)
```bash
alert http $HOME_NET any -> $EXTERNAL_NET any (
    msg:"PowerShell EncodedCommand en HTTP — posible C2";
    flow:established,to_server; http.uri;
    content:"powershell"; nocase; content:"-enc"; nocase; distance:0;
    fast_pattern; classtype:trojan-activity; sid:1000008; rev:1;)
```

---
## 3.5. 🧪 Probar y validar reglas

| Accion                                                 | Comando                                                                     |
| ------------------------------------------------------ | --------------------------------------------------------------------------- |
| Verificar que la config y las reglas no tienen errores | `suricata -T -c /etc/suricata/suricata.yaml -v`                             |
| Probar una regla contra un pcap offline                | `suricata -c /etc/suricata/suricata.yaml -r captura.pcap -l /tmp/test/`     |
| Ver las alertas generadas                              | `cat /tmp/test/fast.log` / `cat /tmp/test/eve.json \| jq '.alert'`          |
| Listar todas las reglas cargadas                       | `suricata --dump-config \| grep rule`                                       |
| Ver estadísticas de reglas que han generado alertas    | `suricatasc -c dump-counters \| jq '.message.detect'`                       |
| Probar la regla de un archivo                          | `suricata -c <suricata.yaml> -s /etc/suricata/mi_regla.rules -i <interfaz>` |

---

# 4. 📊 Consultar `eve.json`

`eve.json` tiene una línea JSON por evento. Con `jq` se puede filtrar eficientemente:

```bash
# Ver solo alertas
cat eve.json | jq 'select(.event_type=="alert")'

# Ver alertas de un SID concreto
cat eve.json | jq 'select(.alert.signature_id==2019401)'

# Ver todas las consultas DNS
cat eve.json | jq 'select(.event_type=="dns") | .dns.rrname'

# Ver conexiones HTTP con código 200
cat eve.json | jq 'select(.event_type=="http" and .http.status==200)'

# Top 10 IPs con más alertas
cat eve.json | jq -r 'select(.event_type=="alert") | .src_ip' | sort | uniq -c | sort -rn | head

# Correlacionar por flow_id (todos los eventos del mismo flujo)
cat eve.json | jq 'select(.flow_id==1234567890)'

# Ver alertas de las últimas 24h (requiere jq con fecha)
cat eve.json | jq 'select(.event_type=="alert") | select(.timestamp > "'$(date -d '24 hours ago' -Iseconds)'")'
```

---
# 5. Modo IPS

Ponemos `suricata --build-info` y vemos si está habilitado NFQueue

Luego habilitamos las reglas iptables `iptables -I INPUT -J NFQUEUE` y `iptables -I OUTPUT -J NFQUEUE` 

Las reglas que tenian en modo alert, las ponemos en modo drop.

## 🔧 Comandos de gestión

```bash
# Iniciar / detener / reiniciar
sudo systemctl start suricata
sudo systemctl stop suricata
sudo systemctl restart suricata
sudo systemctl enable suricata    # activar en el arranque

# Ver logs del proceso
sudo journalctl -u suricata -f
sudo tail -f /var/log/suricata/suricata.log

# Recargar reglas sin reiniciar
sudo kill -USR2 $(pidof suricata)

# Modo captura manual en interfaz concreta
sudo suricata -c /etc/suricata/suricata.yaml -i eth0

# Analizar pcap offline
sudo suricata -c /etc/suricata/suricata.yaml -r archivo.pcap

# Socket de control
sudo suricatasc -c version
sudo suricatasc -c reload-rules
sudo suricatasc -c dump-counters
sudo suricatasc -c uptime
```

---

## 🔗 Ver también

- [[Windows; BlueTeaming]]
- [[Wireshark]]
