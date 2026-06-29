# 1. 🔒 SSH — Secure Shell

> [!abstract] Resumen **SSH (Secure Shell)** es el protocolo estándar para acceso remoto seguro en sistemas Unix/Linux. Proporciona un canal cifrado para ejecución de comandos, transferencia de archivos y tunelización de tráfico. A diferencia de Telnet o FTP, todo el tráfico (credenciales incluidas) viaja cifrado. Puerto por defecto: **22/TCP**.

## 1.1. 📐 El flujo de conexión SSH — nivel de red

La comunicación SSH implica una serie de mensajes de petición y respuesta entre cliente y servidor:

1. **Establecimiento TCP y banner**: Tras el 3 way handshake, viene un intercambio de banners, dónde se mandan las versiones de SSH que usa tanto el cliente como el servidor. Ej `SSH-2.0-OpenSSH_9.3`. Es la primera información que ve un atacante y puede revelar versiones vulnerables.
│
2. **Negociación de algoritmos (KEXINIT)**: El cliente manda un mensaje `SSH2_MSG_KEXINIT` dónde expone una lista de algoritmos que soporta para los pares de claves, el cifrado de mensajes, el HMAC y la compresión. El servidor elige el primero de la lista que soporta y ese es el que se utilizará. 
│
3.  **Autenticación del servidor**: `SSH2_MSG_KEX_ECDH_INIT`. El servidor le manda su `hostkey` y un certificado firmado con esa `hostkey`. El cliente verifica la firma con la  `hostkey` que ya tiene guardada en su `~/.ssh/known_hosts` (obtenida de previas iteraciones). Esto se hace para ver que nadie haya suplantado al servidor. 
│
4. **Intercambio de claves:** Luego se derivan la `clave de sesión` con la que cifrar la comunicación usando ECDH (Diffie Hellman). El cliente cifra esa `clave de sesión` con la pública del servidor.  Le manda una copia al servidor. El servidor descifra esa “llave de sesión” con su privada y por tanto, ambos la tienen. Utilizarán esa llave para cifrar las comunicaciones
│
5. **Autenticación del usuario**: `SSH2_MSG_USERAUTH` El usuario se autentica sin ninguna clave como sondeo inicial. El servidor le responde con los métodos soportados (clave pública, contraseña). 
│
6. **Apertura de canal y shell**: Una vez autenticado, con `SSH2_MSG_CHANNEL` se mandan los comandos del terminal cifrados con la clave de sesión. Además se usa un HMAC, que es una firma o sello matemático que verifica que los mensajes no hayan sido alterados y que provienen de una fuente autorizada, ya que dicha fuente utiliza una clave secreta para su cálculo. 

> [!info] TOFU — Trust On First Use 
> La primera vez que te conectas a un servidor SSH, este te muestra su host key fingerprint y te pregunta si confías en él. Si dices que sí, se guarda en `~/.ssh/known_hosts`. En conexiones futuras, si la host key cambia (servidor reinstalado, o ataque MITM), SSH avisa con el error `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED`.

> [!tip] Autenticación del cliente:** 
> Hemos hablado de cómo se autentica el servidor, pero ¿y el cliente? Este también tiene opciones:
> - **Opción segura:** autenticación por clave pública: el cliente también generará su par de claves y le mandará la pública al servidor, que se la guardará en el archivo  `~/.ssh/authorized_keys `. Por tanto, cada vez que se conecte, el servidor cifra un challenge con esa clave y el cliente lo tendrá que descifrar con su privada.
> - **Opción menos segura:** autenticación por contraseña: en este caso, nos autenticamos sabiendo la contraseña del usuario del servidor. Esta se manda por una comunicación ya cifrada, por lo que no puede ser interceptada fácilmente, el problema es que la pueden adivinar o hacerle fuerza bruta.
> 
> Si la autenticación es exitosa, se genera un `SSH2_MSG_USERAUTH_SUCCESS`

---
## 1.2. Generación de par de claves:

Para generar las claves usamos este comando:
```bash
ssh-keygen -t <algoritmo> -b <tamaño_en_bytes> -f <nombre> <opciones>
│
├──opciones: Ej -q -N “” → no queremos ponerle una contraseña a la clave
└──Podemos ponerle un nombre, Ej: -f llave → ~/.ssh/llave (privada) y ~/.ssh/llave.pub (pública)
```

| Tipo            | Algoritmo           | Seguridad            | Recomendación                        |
| --------------- | ------------------- | -------------------- | ------------------------------------ |
| `ed25519`       | EdDSA (Curve25519)  | ✅ Muy alta           | ✅ Preferida — rápida y segura        |
| `ecdsa`         | ECDSA (NIST curves) | ✅ Alta               | ⚠️ Posibles backdoors en curvas NIST |
| `rsa` 4096 bits | RSA                 | ✅ Alta si ≥3072 bits | ⚠️ Legacy, aceptable con ≥4096       |
| `rsa` 1024/2048 | RSA                 | ❌ Débil              | ❌ No usar                            |
| `dsa`           | DSA                 | ❌ Roto               | ❌ No usar nunca                      |
Para copiar la clave pública al servidor podemos usar `ssh-copy-id -i ~/.ssh/id_ed25519.pub jessica@servidor` Aunque su equivalente manual es:
```bash
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh/
```

---
## 1.3. Configuración del servidor

La configuración del servidor se realiza en el archivo `/etc/ssh/sshd_config`.

| Acción                                            | Comando                       |
| ------------------------------------------------- | ----------------------------- |
| Aplicar cambios sin reiniciar (recarga la config) | `sudo systemctl reload sshd`  |
| Verificar sintaxis antes de aplicar               | `sshd -t` (`-T` con verboseo) |
```bash
# ── Puerto y red ─────────────────────────────────────────────────
Port 22                         # cambiar a un puerto no estándar reduce el ruido
ListenAddress 0.0.0.0           # interfaz en la que escuchar
AddressFamily inet              # solo IPv4 (inet) o cualquiera (any)

# ── Autenticación ─────────────────────────────────────────────────
PermitRootLogin no              # nunca permitir login directo de root
                                # alternativa con clave pública: PermitRootLogin prohibit-password
PasswordAuthentication no       # deshabilitar contraseñas → solo claves públicas
PubkeyAuthentication yes        # habilitar autenticación por clave pública
AuthorizedKeysFile .ssh/authorized_keys  # ruta del archivo de claves

# Si PasswordAuthentication está en no, esto también debe estarlo:
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
UsePAM yes                      # PAM gestiona la sesión aunque no la contraseña

# ── Usuarios permitidos ────────────────────────────────────────────
AllowUsers jessica carlos       # solo estos usuarios pueden conectar por SSH
AllowGroups sshusers            # o solo los miembros de este grupo
DenyUsers root deploy           # denegar explícitamente. Para grupos DenyGroups noremote

# ── Seguridad de sesión ────────────────────────────────────────────
MaxAuthTries 3                  # intentos de autenticación antes de cerrar la conexión
MaxSessions 5                   # sesiones simultáneas por conexión
LoginGraceTime 20               # segundos antes de cerrar si no hay autenticación
ClientAliveInterval 300         # enviar keepalive cada 300s
ClientAliveCountMax 2           # cerrar si no responde a 2 keepalives
TCPKeepAlive yes

# ── Restricciones de funcionalidad ────────────────────────────────
X11Forwarding no                # deshabilitar forwarding de X11 si no se usa
AllowAgentForwarding no         # deshabilitar forwarding del agente SSH
AllowTcpForwarding no           # deshabilitar port forwarding
PermitTunnel no                 # deshabilitar túneles VPN
Banner /etc/ssh/banner.txt      # mostrar aviso legal antes del login

# ── Algoritmos (hardening criptográfico) ──────────────────────────
# Usar solo algoritmos modernos (OpenSSH 8.0+)
KexAlgorithms curve25519-sha256,ecdh-sha2-nistp521
Ciphers aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# ── Logging ────────────────────────────────────────────────────────
LogLevel VERBOSE                # INFO por defecto, VERBOSE incluye fingerprints
SyslogFacility AUTH
```

Tambien podemos poner reglas específicas
```bash
# Reglas específicas para un usuario
Match User deploy
    PasswordAuthentication no
    AllowTcpForwarding no
    ForceCommand /usr/bin/git-shell   # limitar a un solo comando

# Reglas para una IP/red
Match Address 192.168.1.0/24
    PasswordAuthentication yes        # permitir contraseña solo desde la LAN

# Reglas para un grupo
Match Group sftp-only
    ForceCommand internal-sftp        # solo SFTP, sin shell
    ChrootDirectory /srv/sftp/%u      # encerrar en su propio directorio
    AllowTcpForwarding no
    X11Forwarding no
```

Para el cliente podemos usar el archivo `~/.ssh/config`. Esto permite definir alias y opciones por host.
```bash
# Alias para un servidor
Host produccion # ssh produccion
    HostName 192.168.1.50
    User jessica
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_produccion
```

---
## 1.4. 🌐 Tunelización SSH (Port Forwarding)

SSH permite crear túneles cifrados para redirigir tráfico. Es una de las funciones más potentes y más abusadas.

> [!example] Local Port Forwarding
Redirige un puerto local hacia un destino accesible desde el servidor SSH. Esto permite por ejemplo acceder a un servicio interno que está abierto exclusivamente en el localhost. 
```bash
# ssh -L [IP_local:]puerto_local:host_destino:puerto_destino usuario@servidor_SSH
ssh -L 3307:localhost:3306 jessica@servidor # Ahora: mysql -h 127.0.0.1 -P 3307 → conecta al MySQL del servidor
```

> [!example] Remote Port Forwarding
Expone un puerto local del cliente como puerto del servidor. Útil para recibir conexiones desde dentro de una red restrictiva.
```bash
ssh -R 8080:localhost:80 jessica@servidor # Exponer el puerto 80 local en el puerto 8080 del servidor
# Desde el servidor: curl http://localhost:8080 → llega al puerto 80 del cliente
```

> [!example] Dynamic Port Forwarding

Crea un proxy SOCKS local que enruta todo el tráfico a través del servidor SSH.
```bash
ssh -D 1080 jessica@servidor # Crear proxy SOCKS5 en el puerto 1080
# Configurar el navegador o proxychains para usar 127.0.0.1:1080 → todo el tráfico sale desde el servidor SSH
proxychains nmap -sT 10.0.0.0/24 # Con proxychains
```

> [!warning] Port Forwarding y escalada de privilegios
El port forwarding es una herramienta de pivoting fundamental en pentesting. Si se compromete un servidor SSH con forwarding habilitado, se puede usar para alcanzar redes internas inaccesibles directamente.

---
## 1.5. 🚨 Vulnerabilidades y vectores de ataque

> [!error] Fuerza bruta de credenciales

El vector más común. Si `PasswordAuthentication yes` está activo y no hay protección:
```bash
hydra -l <usuario> -P passwords.txt ssh://<ip> -t 4
nxc <protocolo> <ip> -u <usuario> -p <contraseña> <opciones>
```

> [!error] Clave privada expuesta 
Si una clave privada está en un lugar accesible (backup, repositorio git, home mal configurado):
```bash
trufflehog git file://./repositorio
gitleaks detect --source .
```
> Para evitarlo, siempre hay que proteger la clave privada con una passfrase

Luego tenemos los CVEs de versiones antiguas de SSH

|CVE|Versión afectada|Descripción|
|---|---|---|
|**CVE-2024-6387** (regreSSHion)|OpenSSH < 4.4 y 8.5p1–9.7p1|Race condition en el handler de SIGALRM → RCE como root sin autenticación en sistemas glibc|
|**CVE-2023-38408**|OpenSSH < 9.3p2|RCE en `ssh-agent` via forwarding — librería maliciosa cargada remotamente|
|**CVE-2020-14145**|OpenSSH < 8.4|Information leak del algoritmo de claves preferido → ayuda a ataques MITM|
|**CVE-2016-0777**|OpenSSH 5.4–7.1|Memory leak del cliente que podía filtrar claves privadas al servidor|
|**CVE-2016-6515**|OpenSSH < 7.4|DoS por CPU con contraseñas muy largas en servidores con PasswordAuth|

> [!warning] Username enumeration CVE-2018-15473 
Algunas versiones de OpenSSH permitían determinar si un usuario existe en el sistema basándose en diferencias de tiempo en la respuesta de autenticación. Versiones previas a OpenSSH 7.7 dan respuestas diferentes para usuarios existentes vs no existentes
```bash
ssh-audit 192.168.1.50          # auditoría general
```

---
## 🔗 Ver también

- [[Linux; escalada de privilegios]]
- [[Servicios en red; SMB, FTP y NFS]]
- [[Suricata]]
