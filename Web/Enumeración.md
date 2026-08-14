
# 0. Enumeración básica

## 0.1. Enumeración activa

#### Whatweb, wappaylyzer y curl
Estas herramientas (Whatweb en cli y wappalyzer como extensión de firefox) pueden dar pistas de las tecnologías con las que está hecha la web: ¿Utiliza un CMS conocido? ¿Utiliza en cambio un framework?

```bash
$: curl -I https://inlanefreight.com

# HTTP/1.1 301 Moved Permanently
# Date: Fri, 31 May 2024 12:12:12 GMT
# Server: Apache/2.4.41 (Ubuntu)
# X-Redirect-By: WordPress
# Location: https://www.inlanefreight.com/
# Content-Type: text/html; charset=UTF-8
```

Por otro lado `whatweb`
```bash
$: whatweb app.inlanefreight.local
# http://app.inlanefreight.local [200 OK] Apache[2.4.41], Bootstrap, Cookies[72af8f2b24261272e581a49f5c56de40], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], HttpOnly[72af8f2b24261272e581a49f5c56de40], IP[10.129.99.135], JQuery, MetaGenerator[Joomla! - Open Source Content Management],....
```

----
#### Certificado en HTTPs
En HTTPs, podemos examinar el certificado SSL, que puede que contenga nombres de dominio:
```
$: openssl s_client -connect 10.10.11.129:443
CONNECTED(00000003)
depth=0 CN = research.search.htb, CN = research
```

----
#### DNS
Mediante DNS podemos obtener información con la herramienta `dig` o con `dnsenum`, tambien tratar de realizar una transferencia de zona AXFR.

Tambien viene bien utilizar `whois` para sacar información de registro de la web.

----
#### wafw00f
La herramienta de wafw00f nos permite mirar si hay algun WAF detrás de la web o no

```bash
$: wafw00f inlanefreight.com

# (...)    
# [*] Checking https://inlanefreight.com
# [+] The site https://inlanefreight.com is behind Wordfence (Defiant) WAF.
```
En este ejemplo está el WAF de Defiant detrás.


---
#### Nikto
Nikto es una herramienta de analisis y enumeración web avanzada. Con la opción `-Tuning b` podemos indicar que solo utilice los módulos de escaneo de software. 

Con este módulo podemos extraer información como:
- Direcciones IPv4 e IPv6
- Certificado SSL
- Headers: por ejemplo si falta alguno de seguridad como el de `Strict-Transport-Security` o si hay alguno extraño
- Tecnología de servidor (Ej `Apache/2.4.41 (Ubuntu)`) y CMS `WordPress`
- Information disclosure: la presencia de un `license.txt`

```bash
$: nikto -h inlanefreight.com -Tuning b
```

> [!TIP]
> Aun así Nikto solo se debe usar en entornos controlados por la cantidad masiva de peticiones que realiza y esta información se puede obtener con otras herramientas más discretas como `whatweb`. Nikto simplemente agiliza la enumeración a cambio de ruido.

Otras herramientas de reconocimiento completo pueden ser:  [FinalRecon](https://github.com/thewhiteh4t/FinalRecon),  [Recon-ng](https://github.com/lanmaster53/recon-ng),  [theHarvester](https://github.com/laramies/theHarvester) o  [SpiderFoot](https://github.com/smicallef/spiderfoot) y   [OSINT Framework](https://osintframework.com/) para OSINT. Al igual que pasa con Nikto, son herramientas extremadamente ruidosas.
```bash
$: finalrecon --full --url http://inlanefreight.com
```

----
## 0.2. Archivos importantes

#### Robots.txt
El archivo `robots.txt` es un archivo de texto plano ubicado en el directorio raíz de un sitio web (p. ej., `web.com/robots.txt`) e indica a los buscadores web que partes del sitio indexar y cuales no siguiendo una serie de directivas. 
```
User-agent: *
Disallow: /private/
```

Siempre hay un `User-agent` que especifica a que rastreador o bot se aplican las reglas. Si pone un wildcard (`*`) indica que se aplican a todos los bots. Luego por otro lado están las directivas, que proporcionan instrucciones específicas al agente de usuario identificado como `Disallow` para que no accedan a dicho directorio o `Sitemap` con un mapa XML del sitio

Este archivo protege contra la sobrecarga de los servidores y protefe la información sensible, pero da pistas sobre directorios ocultos que ayudan a una enumeración por parte de un atacante, por ejemplo  `/admin/`  o `/private/`.

----
#### Well known URI
El estándar `.well-known`, sirve como un directorio estandarizado dentro del dominio raíz de un sitio web, normalmente bajo la ruta `/.well-known/`. Centraliza los metadatos críticos de un sitio web, incluyendo archivos de configuración e información relacionada con sus servicios, protocolos y mecanismos de seguridad. Entre esos archivos encontramos:

- `security.txt`: Contiene información de contacto para que los investigadores de seguridad informen sobre vulnerabilidades.
- `openid-configuration`: Define los detalles en formato JSON de configuración para OpenID Connect, una capa de identidad sobre el protocolo OAuth 2.0.

> [!NOTE]
> La información obtenida del _endpoint_ `openid-configuration` proporciona múltiples oportunidades de exploración: ya que nos proprociona rutas importantes, nivel de seguridad o detalles del algoritmo, información relevante para ataques contra JWK

----
####  Crawlers
Los crawlers son algortimos que navegan automáticamente por la web, analiza su contenido y extrae todos sus enlaces. Luego, añade estos enlaces a una cola y los rastrea, repitiendo el proceso de forma iterativa.

Estos crawlers pueden obtener emails, enlaces, comentarios, urls de archivos externos o imagenes y contenido multimedia (util para posibles rutas `LFI`)

Los mas popilares son `Burp Suite Spider`, `OWASP ZAP` o `Apache nutch` y tambien tenemos la librería de python `scrapy`

Scrapy permite usar la herramienta [reconSpider](https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip)
```bash
$: python3 ReconSpider.py http://inlanefreight.com
$: cat results.json | jq .links[] | tr -d '"'    # enlaces
$: cat results.json | jq .emails[] | tr -d '"' | awk -F'@' '{print $1}' | sort -u # usuarios
```

---
## 0.2. Enumeración pasiva
Una vez identificadas las tecnologías básicas, podemos ahorrarnos mucho tiempo si ya tenemos nociones de cómo funcionan.

- Sistemas conocidos como **Wordpress** o **Apache Tomcat** tienen siempre la misma estructura de rutas ya conocidas. 
- Si es un proyecto de github, se puede buscar en el repositorio el esqueleto básico para saber como está montado.

El objetivo es buscar páginas de administración, nombres de bases de datos conocidas, rutas o archivos de configuración... etc


---
# 1. Lista de diccionarios

Diccionarios (dentro de `/usr/share`):

| Uso                                | Diccionario                                                |
| ---------------------------------- | ---------------------------------------------------------- |
| Vhosts/subdominios                 | `seclists/Discovery/DNS/subdomains-top1million-110000.txt` |
| subdirectorios comunes (Ej `.git`) | `secLists/Discovery/Web-Content/common.txt`                |
| Parámetros GET                     | `secLists/Discovery/Web-Content/burp-parameter-names.txt`  |
| Webs IIS                           | `secLists/Discovery/Web-Content/IIS.fuzz.txt`              |
| plugins wordpress                  | `secLists/Discovery/Web-Content/CMS/wp-plugins.fuzz.txt`   |
| Subdirectorios                     | `worldlists/dirbuster/directory-list-2.3-medium.txt`       |
| Crear diccionario custom           | `cewl http://web.com -w dict.txt`                          |
| Nombres de usuarios                | `secLists/Usernames/Names/names.txt`                       |
| Direcciones IP                     | `seclists/Fuzzing/IPv4-Addresses.txt`                      |
| User Agents                        | `seclists/Fuzzing/User-Agents/user_agents.txt`             |
| Extensiones                        | `seclists/Discovery/Web-Content/web-extensions.txt`        |
| Números                            | `for i in $(seq 1 1000); do echo $i >> nums.txt; done`     |


---------
# 2. Fuzzing

#### FUFF
Con ffuf, necesitamos una url, un diccionario y la palabra `FUZZ` que sustituya el parámetro que bruteforcear.

Tenemos los operadores como filtros:

| Uso                         | Evitarlo                 | Buscarlo              |
| --------------------------- | ------------------------ | --------------------- |
| Códigos de estado           | `-fc 404`                | `-mc 200,301,302`     |
| Respustas con X líneas      | `-fl 20`                 | `-ml 20`              |
| Tamaño (número de bytes)    | `-fs 320`                | `-ms 320`             |
| Tamaño (número de palabras) | `-fw 203`                | `-mw 203`             |
| Texto exacto                | `-mr "No account found"` | `-mr "Account found"` |

Aparte, existen estos parámetros:
- `-ic`: ignorar lineas de copyright


----
## 2.1. Fuzzing de distintas partes

----------
#### Fuzzing de vhosts / subdominios
Los vhost y subdominios pueden revelar rutas ocultas mal protegidas. Los vhost necesitan especificarse en la cabecera `Host`
```bash
ffuf -w .../subdomainstopmillion50000.txt -H "Host: FUZZ.web.com" -u http://web.com/

gobuster vhost -u http://web.com -w .../subdomains-top1million-110000.txt --append-domain
```

Para los subdominios es más sencillo:
```
ffuf -u https://FUZZ.web.com -w .../subdomainstopmillion50000.txt 
```

-----
#### Fuzzing de subdirectorios
Ahora, teniendo una página objetivo, vamos a tratar de encontrar sus subdirectorios y obtener una estructura interna de la web. 

```bash
ffuf -w ./directory-list-2.3-medium.txt -t 200 -u "http://web.htb/FUZZ"
ffuf -w ./directory-list-2.3-medium.txt -t 200 -u "http://web.htb/FUZZ.php"

gobuster dir -w .../directory-list-2.3-medium.txt -t 200 -u http://web.com/ 
```

Si hay SLL/TLS:
```bash
gobuster -k -w .../directory-list-2.3-medium.txt -t 200 -u http://web.com/
```

Tambien podemos activar la recursión y su profundidad, por ejemplo si queremos buscar rutas y dentro páginas php usaríamos estas opciones:
```bash
ffuf -w .../directory-list-2.3-small.txt -u http://web.com/FUZZ \
-recursion -recursion-depth 1 -e .php -v
```

----------
#### Fuzzing de extensiones
Podemos tratar de averiguar las extensiones que procesa el servidor web, aunque tengamos ciertas nociones de antemano (los IIS procesan `.asp` y `.aspx` y los apache `.php`)
```
ffuf -w .../web-extensions.txt -u http://web/blog/indexFUZZ
```

Si queremos hacer fuzzing con extensiones ya conocidas (php, aspx, html...):
```bash
ffuf -u https://web.com/FUZZ -w .../directory-list-2.3-medium.txt -e .php,.html

gobuster dir -x php -w .../directory-list-2.3-medium.txt -t 200 -u http://web.com/
```


----
#### Fuzzing de parámetros 
Para los parámetros GET
```bash
ffuf -u http://web.com/image.php?FUZZ=/etc/passwd -mc 200 \
-w .../burp-parameter-names.txt
```

Para los parámetros POST simplemente ponemos `-X POST` y el `Content-Type`
```bash
ffuf -w .../burp-parameter-names.txt:FUZZ -u http://admin.web.com/admin/admin.php \
 -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx
```


----
#### Brute Force formulario POST

```bash
ffuf -w ../rockyou.txt -X POST -d "username=admin\&password=FUZZ" \
-u https://web/login.php -fc 401 -H 'Content-Type: application/x-www-form-urlencoded'
```

Para formularios JSON
```bash
ffuf -u https://api.web.com/login -X POST -w .../rockyou.txt -fc 401 \
  -d '{"username":"admin","password":"FUZZ"}' -H 'Content-Type: application/json' 
```


---
#### Fuzzing de headers

```bash
# X-Forwarded-For bypass
ffuf -u https://web.com/admin -H 'X-Forwarded-For: FUZZ' -w .../IPv4-Addresses.txt -mc 200

# Fuzzing de header

ffuf -u https://wev.com/api/v1/users -H 'FUZZ: 127.0.0.1' -fs 0 -mc 200\
  -w /usr/share/seclists/Fuzzing/http-request-headers/http-request-headers-common.txt \
```

-----
#### Fuzzing de cookies

```bash
ffuf -u https://web.com/admin -b 'session=abc123; role=FUZZ' \
  -w /usr/share/seclists/Fuzzing/Strings/generics.txt -mc 200 -fc 403

# Brute-force a session token
ffuf -u https://web.com/dashboard -b 'session=FUZZ' -w sessions_wordlist.txt -mc 200
```

----
#### Filtrar por texto
Por ejemplo, queremos que salga la cabecera `"Acces-Control-Allow-Origin"`
```bash
ffuf -w .../subdomains_top1million-5000.txt -u http://web.com \
-H “Origin: http://FUZZ.web.com” -mr "Acces-Control-Allow-Origin" -ignorebody 
```

#### Filtrar por tiempo
Ejemplo, Fuzzing blind SQLi
```bash
ffuf -u ‘https://web.com/search.php?q=FUZZ’ \
  -w /usr/share/seclists/Fuzzing/SQLi/Generic-SQLi.txt \
  -fc 404 -mt 5000
```


---
## 2.2. Fuzzing avanzado
#### Fuzzing multiple

```bash
# pitchfork: dos posiciones (A-1 B-2 C-3)
ffuf -u https://web.com/login -X POST \
  -d ‘username=USER&password=PASS’ -w usernames.txt:USER -w passwords.txt:PASS \
  -H ‘Content-Type: application/x-www-form-urlencoded’ \
  -mode pitchfork -fc 401

# Cluster bomb: todas las configuraciones (A-1 A-2 A-3 B-1...)
ffuf -u https://target.com/login -X POST \
  -d ‘username=USER&password=PASS’ \
  -w usernames.txt:USER \
  -w passwords.txt:PASS \
  -H ‘Content-Type: application/x-www-form-urlencoded’ \
  -mode clusterbomb -fc 401
```

-----
#### Fuzzing por archivo REQ

Tenemos un archivo REQ con la request de HTTPv1 guardada
```bash
POST /login HTTP/1.1
Host: web.com
Content-Type: application/x-www-form-urlencoded

username=admin&password=FUZZ
```

Entonces podemos pasarsela a la herramienta:
```bash
ffuf -request request.txt -request-proto https -w /usr/share/wordlists/rockyou.txt -fc 401
```

Para http2 usamnos:  `-http2`

> [!CAUTION]
> **Fuzzing para explotar vulnerabilidades:** Distintas vulnerabilidades como el SQLim el OS inyection o el SQLi entre otras pueden servirse del fuzzing para encontrar payloads válidos. Por ejemplo, podemos fuzzear webroots para LFI con [esta lista de palabras para Linux](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/default-web-root-directory-linux.txt) o [esta lista de palabras para Windows](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/default-web-root-directory-windows.txt). 
