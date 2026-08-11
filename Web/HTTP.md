> **HTTP (HyperText Transfer Protocol)** es el protocolo de comunicación cliente-servidor sobre el que se sustenta la web. El cliente envía una **petición** y el servidor devuelve una **respuesta**. Es un protocolo sin estado: cada petición es independiente y el servidor no recuerda las anteriores. **HTTPS** añade cifrado TLS sobre HTTP, protegiendo los datos en tránsito y autenticando el servidor mediante certificados digitales.

---
# 1. 🌐 El protocolo HTTP

## 1.1. Visión general — cómo carga una página

```
  1. El navegador pide el HTML:
     GET /index.html → servidor devuelve el esqueleto HTML

  2. El navegador parsea el HTML y descubre los recursos que necesita:
     GET /styles.css   → hoja de estilos
     GET /logo.png     → imagen
     GET /app.js       → código JavaScript

  3. Con todos los recursos, el navegador renderiza la página final
```

Cada recurso es una petición HTTP independiente. El HTML actúa como índice que declara qué otros archivos hay que cargar.

| Tipo de recurso              | Función                                            |
| ---------------------------- | -------------------------------------------------- |
| **HTML**                     | Estructura y contenido de la página (el esqueleto) |
| **CSS**                      | Estética: colores, fuentes, diseño visual          |
| **JavaScript**               | Interactividad dinámica sin recargar la página     |
| **Imágenes / vídeo / audio** | Contenido multimedia                               |

---
#### 🏗️ El DOM — Document Object Model
Cuando el navegador recibe el HTML, no lo muestra directamente, sino que interpreta y construye una representación **jerárquica** en memoria llamada **DOM**, el cuál es un árbol de objetos en memoria que JavaScript puede manipular en tiempo real para modificar elementos, reaccionar a eventos del usuario y actualizar la página **sin recargarla**. Eso es lo que hace posible las interfaces modernas.

*Si el  HTML es el plano de un edificio, el DOM es el edificio construido*
```
document
└── html
    ├── head
    │   └── title → "Mi tienda"
    └── body
        ├── h1 → "Bienvenido"
        └── form
            ├── input → (usuario)
            └── input → (contraseña)
```

---

## 1.2. 📨 Estructura de una petición y respuesta
HTTP sigue un modelo cliente-servidor. En una petición se mandan estos elementos

Petición del cliente
```http
GET /productos?categoria=ropa HTTP/1.1
Host: tienda.com
User-Agent: Mozilla/5.0
Accept: text/html
Cookie: session_id=abc123
```

Respuesta del servidor
```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 1234
Set-Cookie: session_id=abc123; HttpOnly; Secure

<html>...</html>
```

---

## 1.3.📋 Cabeceras HTTP
Las cabeceras son metadatos que viajan al inicio de cada mensaje. Controlan cómo interpretar y manejar el intercambio.

> [!CAUTION] 
> Una cabecera mal configurada puede exponer información sensible (`Server: Apache/2.4.1` revela versión vulnerable), permitir robo de cookies o habilitar ataques como CSRF o clickjacking.

-----
#### Cabeceras de petición (cliente → servidor)

```bash
Host: www.example.com                # Dominio al que se realiza la petición 
Referer: https://www.example.com     # URL desde la que se llegó al recurso actual
User-Agent: Mozilla/5.0 (Windows NT 10.0) # Navegador / cliente que realiza la petición
Accept: text/html, application/json  # Tipos de contenido que acepta el cliente
Accept-Encoding: gzip, deflate       # Compresiones que acepta el cliente
Accept-Language: es-ES, en;q=0.9     # Idiomas preferidos
Cookie: session_id=abc123            # Cookies que el cliente envía al servidor
Authorization: Bearer YWRtazc3dvcmQ= # Credenciales de autenticación

Content-Type: xxx-form/urlencoded    # Formato del cuerpo de la petición (en POST, PUT, PATCH)
Content-Lenght: 0                    # Tamaño en bytes de los datos enviados
```

-----
#### Cabeceras de respuesta (servidor → cliente)

```bash
Content-Type: text/html; charset=UTF-8 # Tipo y codificación del contenido devuelto
Content-Length: 1234           # Tamaño en bytes de la respuesta
Server: Apache/2.4.41 (Ubuntu) # Software del servidor (⚠️ puede revelar versión vulnerable)
Set-Cookie: session_id=abc123; HttpOnly # Establece una cookie en el navegador del cliente
Cache-Control: max-age=3600 # El cliente puede cachear esto 1 hora
Location: /index.html       # URL de redirección (en respuestas 3xx)
```


-----
#### Cabeceras de seguridad (respuesta)

| Cabecera                          | Protege contra                                                 |
| --------------------------------- | -------------------------------------------------------------- |
| `Strict-Transport-Security`       | Fuerza HTTPS — protege contra downgrade a HTTP                 |
| `Content-Security-Policy`         | XSS — restringe desde dónde se cargan scripts y recursos       |
| `X-Frame-Options`                 | Clickjacking — impide que la página se incruste en un iframe   |
| `X-Content-Type-Options: nosniff` | MIME sniffing — el navegador respeta el Content-Type declarado |
| `Referrer-Policy`                 | Controla qué información de Referer se envía                   |
| `Access-Control-Allow-Origin`     | Política CORS — qué orígenes pueden acceder                    |

---

## 1.4. 🍪 Sesiones y cookies
HTTP es **sin estado**: cada petición es independiente y el servidor no recuerda las anteriores. Las sesiones resuelven esto.

1️⃣ El  usuario envía credenciales (POST /login)

2️⃣ El servidor valida y crea una sesión devuelve un token o ID de sesión via cookie o via authorization header

3️⃣ El cliente presenta el token en cada petición:

| Tipo          | Vía cookie (automático)                          | Via Authorization header (manual) |
| ------------- | ------------------------------------------------ | --------------------------------- |
| **Petición**  | `Set-Cookie: session_id=abc123 HttpOnly; Secure` | `{"access_token": "eyJ..."}`      |
| **Respuesta** | `Cookie: session_id=abc123`                      | `Authorization: Bearer eyJ...`    |

----
#### Auth bearer
Al autenticarse, el servidor entrega en el cuerpo de la respuesta el token
```json
{
 "access_token": "eyJhbGciOiJIUzI1NiJ9...",
 "token_type": "Bearer",
 "expires_in": 3600
}
```
Y el cliente la tiene que mandar en la cabecera `Authorization`. Ej  `Authorization: Bearer eyJ...`. **Bearer** significa que el token es suficiente para la probar la autenticación

----
#### Cookie
El servidor la entrega con "Set cookie" `Set-Cookie: access_token=eyJhbGciOiJIUzI1NiJ9`
. Entonces el navegador lo almacena y lo reenvía automáticamente en cada petición al mismo sitio, junto con el resto de cookies.

> [!NOTE]
> Las cookies son pequeños datos que el servidor envía al navegador mediante la cabecera Set-Cookie, y que el navegador almacena y reenvía automáticamente en peticiones posteriores al mismo sitio. En todas las webs, se usan las  molestas cookies publicitarias y analíticas que hay que aceptar cada vez que se visita una web. También podemos encontrar cookies de preferencias, como idioma o tema oscuro


----
#### Atributos de seguridad de las cookies

| Atributo          | Protege contra | Descripción                                                             |
| ----------------- | -------------- | ----------------------------------------------------------------------- |
| `HttpOnly`        | XSS            | JavaScript no puede leer la cookie — evita robo via `document.cookie`   |
| `Secure`          | Intercepción   | La cookie solo se envía por HTTPS                                       |
| `SameSite=Strict` | CSRF           | Solo se envía en peticiones del mismo origen                            |
| `SameSite=Lax`    | CSRF (parcial) | Se envía en navegación de primer nivel pero no en peticiones cross-site |
| `SameSite=None`   | —              | Se envía en cualquier contexto (requiere `Secure`)                      |

> [!CAUTION] 
> **Session Hijacking**: Si un atacante roba ese token, podría actuar en su lugar, suplantar su identidad y acceder al panel de administración sin necesidad de conocer la contraseña. A este tipo de ataque se le llama session hijacking o secuestro de sesión.


---
## 1.5. 🔢 Códigos de estado
Cuando el navegador envía una petición al servidor, este siempre responde con un código de estado: un número de tres cifras que indica el resultado exacto de esa operación. ¿Salió bien? ¿Hay que visitar otra página? ¿Hay suficientes permisos para hacer eso? ¿Hemos tratado de buscar una página que no existe? Los códigos están divididos en cinco familias, cada una identificada por su primer dígito:

1xx — Informativos: La petición se está procesando, el servidor avisa de que sigue en ello. Raramente visibles para el usuario final, pero presentes en comunicaciones continuas como los chats en directo.
```bash
100 Continue  # Sigue enviando datos, voy bien
101 Switching Protocols # Acepto cambiar al protocolo que pides
```

2xx — Éxito: La operación se completó correctamente.
```bash
200 OK # Respuesta estándar de éxito
201 Created # Se creó un recurso nuevo
204 No content # Éxito, pero no hay nada que devolver
```

3xx — Redirecciones: El recurso ha cambiado de ubicación. El navegador debe hacer una nueva petición.
```bash
301 Moved Permanently  # La URL cambió para siempre (SEO, migraciones)
302 Found # Redirección temporal al sitio indicado por la cabecera Location
304 Not Modified # El recurso no cambió; usa la versión en caché
```

4xx — Errores del cliente: La petición está mal formulada o no está autorizada. El problema está en quien pide.
```bash
Código de estado # Significado
400 Bad Request # La petición tiene errores de sintaxis
401 Unauthorized # Necesitas autenticarte primero
403 Forbidden # Estás autenticado, pero no tienes permiso
404 Not Found # El recurso no existe
405 Method Not Allowed # Ese verbo HTTP no está permitido aquí
429 Too Many Requests # Has superado el límite de peticiones permitidas
```
⚠️ Un atacante puede enumerar recursos 403 para encontrar rutas existentes pero a las que no tiene permiso.

5xx — Errores del servidor: La petición era válida, pero el servidor falló al procesarla. El problema está en quien responde.
```bash
500 Internal Server Error # Error genérico del servidor
502 Bad Gateway # Un servidor intermediario recibió una respuesta inválida
503 Service Unavailable # El servidor está caído o saturado
504 Gateway Timeout # El servidor intermediario no recibió respuesta a tiempo
```
⚠️ Un error 500 puede ser síntoma de que hay una posible vulnerabilidad detrás, ya que se rompe la lógica correcta del servidor y da entrada a consecuencias inesperadas


---

## 1.6. 🔧 Métodos HTTP

El método indica **qué operación** se quiere realizar sobre el recurso. Equivalen a las operaciones **CRUD** (Create, Read, Update, Delete).

---
#### `GET` — leer un recurso
Los datos van en la URL (query string). **Nunca** enviar datos sensibles en un GET.
```http
GET /productos?categoria=ropa&page=2 HTTP/1.1
Host: tienda.com
```

---
#### `POST` — crear o enviar datos
Los datos van en el **cuerpo** de la petición, no en la URL.

```http
POST /login HTTP/1.1
Host: tienda.com
Content-Type: application/x-www-form-urlencoded

username=jessica&password=secreto123
```

---
#### `PUT` — reemplazar un recurso completo
Envía una versión **completa** del recurso, sobrescribiendo la anterior.

```http
PUT /usuarios/42 HTTP/1.1
Content-Type: application/json

{"nombre": "Jessica García", "email": "j@empresa.com", "rol": "admin"}
```

> [!CAUTION] 
> **PUT sin control de acceso**: Si un atacante puede hacer PUT sin restricciones puede modificar campos como `rol` o `admin`. Siempre validar en el servidor qué campos puede modificar cada usuario.

---
#### `PATCH` — modificar parcialmente un recurso
Solo envía los campos que cambian, no el recurso completo.

```http
PATCH /usuarios/42 HTTP/1.1
Content-Type: application/json

{"email": "nuevo@empresa.com"}
```

---
#### `DELETE` — eliminar un recurso
```http
DELETE /usuarios/42 HTTP/1.1
```

> [!CAUTION] 
> **DELETE sin autenticación** Un endpoint DELETE expuesto sin autenticación ni autorización es una vulnerabilidad crítica.

---
#### Métodos auxiliares

|Método|Descripción|Uso en seguridad|
|---|---|---|
|`HEAD`|Como GET pero devuelve solo las cabeceras, sin cuerpo|Comprobar si un recurso existe sin descargarlo|
|`OPTIONS`|Devuelve los métodos permitidos para el recurso|Enumeración de métodos disponibles|
|`TRACE`|El servidor devuelve exactamente la petición recibida|Puede usarse en XST (Cross-Site Tracing) — desactivar en producción|

---

## 1.7. 🔤 Codificaciones

### URL Encoding
Las URLs solo admiten ciertos caracteres. Los demás se reemplazan por `%` seguido de su valor hexadecimal:

```
hola mundo & café  →  hola%20mundo%20%26%20caf%C3%A9

Caracteres comunes:
  espacio → %20    & → %26    = → %3D
  / → %2F          ? → %3F    # → %23
  + → %2B          @ → %40    : → %3A
```

### Base64
Convierte cualquier dato binario en texto usando 64 caracteres seguros (`A-Z`, `a-z`, `0-9`, `+`, `/`, `=`). Permite enviar datos arbitrarios como si fueran texto.

```
hola mundo & café  →  aG9sYSBtdW5kbyAmIGNhZsOp
```

> [!WARNING] 
> **Codificación ≠ Cifrado**. Tanto URL encoding como Base64 son reversibles sin ninguna clave. Cualquiera que vea los datos codificados puede decodificarlos inmediatamente. No protegen la confidencialidad — para eso se necesita cifrado.

---
## 1.7. 📦 Formatos de envío de datos
Se definen por la cabecera `Content-Type`

#### application/x-www-form-urlencoded — formularios simples
El formato por defecto de los formularios HTML. Pares clave=valor separados por `&`:

```http
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=jessica&password=secreto&remember=true
```

----
#### multipart/form-data — subida de archivos
El cuerpo se divide en partes separadas por un `boundary`. Necesario para enviar archivos binarios:

```http
POST /perfil HTTP/1.1
Content-Type: multipart/form-data; boundary=----Boundary123

----Boundary123
Content-Disposition: form-data; name="nombre"

Jessica García
----Boundary123
Content-Disposition: form-data; name="avatar"; filename="foto.jpg"
Content-Type: image/jpeg

[bytes de la imagen...]
----Boundary123--
```

> [!CAUTION] 
> **Subida de archivos sin validación**: Si el servidor no valida correctamente el tipo y contenido del archivo, un atacante puede subir una **webshell** disfrazada de imagen (`shell.php` renombrado a `shell.jpg`). Con acceso a esa ruta, ejecuta comandos arbitrarios en el servidor.

---
#### application/json — APIs modernas
El formato estándar en APIs REST. Pares clave-valor anidados, fácil de leer y procesar:

```http
POST /api/usuarios HTTP/1.1
Content-Type: application/json

{
    "nombre": "Jessica García",
    "email": "jessica@empresa.com",
    "rol": "editor"
}
```

---
#### application/xml — entornos legacy y enterprise
Datos en etiquetas anidadas, similar a HTML. Predominante en banca, administración pública e integraciones SOAP:

```http
POST /api/transferencia HTTP/1.1
Content-Type: application/xml

<transferencia>
    <origen>ES91 2100 0418 4502 0005 1332</origen>
    <destino>ES76 0081 0166 2800 0100 2222</destino>
    <importe moneda="EUR">1500.00</importe>
</transferencia>
```

> [!CAUTION] 
> **XXE — XML External Entity**: Si el servidor procesa XML sin deshabilitar las entidades externas, un atacante puede inyectar una entidad que lea archivos del sistema (`/etc/passwd`) o haga peticiones internas a servicios no expuestos.
> 
> ```xml
> <?xml version="1.0"?>
> <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
> <transferencia><origen>&xxe;</origen></transferencia>
> ```

---

## 1.8. 🗃️ Caché
La caché almacena localmente recursos ya descargados (CSS, imágenes, JS) para reutilizarlos en peticiones posteriores sin volver a descargarlos.

```
Sin caché:  navegador → servidor (descarga completa) cada vez
Con caché:  navegador → caché local → devuelve el recurso guardado
            Solo si expiró: navegador → servidor → actualiza caché
```

|Directiva `Cache-Control`|Significado|
|---|---|
|`max-age=3600`|Cachear durante 3600 segundos (1 hora)|
|`no-cache`|Siempre revalidar con el servidor antes de usar la caché|
|`no-store`|No almacenar en caché bajo ningún concepto|
|`private`|Solo cacheable en el navegador, no en proxies intermedios|
|`public`|Cacheable en proxies y CDNs|

El código `304 Not Modified` indica que el recurso no cambió desde la última vez — el navegador puede usar la versión cacheada.

---
