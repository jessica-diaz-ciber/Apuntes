


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


---------
# 2. Fuzzing

#### FUFF
Con ffuf, necesitamos una url, un diccionario y la palabra `FUZZ` que sustituya el parámetro que bruteforcear.

Luego tenemos los operadores como filtros:

| Uso                         | Evitarlo                 | Buscarlo              |
| --------------------------- | ------------------------ | --------------------- |
| Códigos de estado           | `-fc 404`                | `-mc 200,301,302`     |
| Respustas con X líneas      | `-fl 20`                 | `-ml 20`              |
| Tamaño (número de bytes)    | `-fs 320`                | `-ms 320`             |
| Tamaño (número de palabras) | `-fw 203`                | `-mw 203`             |
| Texto exacto                | `-mr "No account found"` | `-mr "Account found"` |

----
## 2.1. Fuzzing de distintas partes

#### Fuzzing de subdirectorios

```bash
ffuf -w ./directory-list-2.3-medium.txt -t 200 -u "http://web.htb/FUZZ"
ffuf -w ./directory-list-2.3-medium.txt -t 200 -u "http://web.htb/FUZZ.php"

gobuster dir -w .../directory-list-2.3-medium.txt -t 200 -u http://web.com/ 
```

Si queremos hacer fuzzing con extensiones (php, aspx, html...):
```bash
ffuf -u https://web.com/FUZZ -w .../directory-list-2.3-medium.txt -e .php,.html

gobuster dir -x php -w .../directory-list-2.3-medium.txt -t 200 -u http://web.com/
```

Si hay SLL/TLS:
```bash
gobuster -k -w .../directory-list-2.3-medium.txt -t 200 -u http://web.com/
```

----------
#### Fuzzing de vhosts / subdominios

```bash
ffuf -w .../subdomainstopmillion50000.txt -H "Host: FUZZ.web.com" -u http://web.com/

ffuf -u https://FUZZ.web.com -w .../subdomainstopmillion50000.txt 

gobuster vhost -u http://web.com -w .../subdomains-top1million-110000.txt --append-domain
```

----
#### Fuzzing de parámetros 

```bash
ffuf -u http://web.com/image.php?FUZZ=/etc/passwd -mc 200 \
-w .../burp-parameter-names.txt
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
