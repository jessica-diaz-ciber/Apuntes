# Índice

[[Redes#1. Introducción a redes]]
- [[Redes#1.1. Red privada]]
- [[Redes#1.2. Red pública]]
- [[Redes#1.3. Protocolo NAT]]

[[Redes#2. Modelo OSI]]
- [[Redes#2.1. Modelo OSI y TCP/IP]]
- [[Redes#2.2. Capa de acceso]]
- [[Redes#2.3. Capa de internet]]
- [[Redes#2.4. Capa de transporte]]
- [[Redes#2.5. Capa de aplicación]]

[[Redes#3. Direcciones MAC]]
- [[Redes#3.1. Interfaces de red]]
- [[Redes#3.2. Tablas ARP]]

[[Redes#4. Direcciones IP]]
- [[Redes#4.1. Tipos de IP]]
- [[Redes#4.2. Anatomía de una IP]]
- [[Redes#4.3. Tamaños de redes]]
- [[Redes#4.4. Direcciones IP reservadas]]
- [[Redes#4.5. Ver información de red]]
- [[Redes#4.6. Configurar nuestra dirección IP]]

[[Redes#5. Los puertos lógicos]]
- [[Redes#5.1. Rangos de puertos]]
- [[Redes#5.2 Sockets]]


# 1. Introducción a redes

> [!abstract] Introducción 
> Una red informática es un sistema que permite conectar dos o más dispositivos electrónicos, permitiéndoles comunicarse e intercambiar información a través de aplicaciones.

Elementos de una red:

| Elemento                       | Descripción                                                                                                                                                                                                             |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Hosts**                      | Son los equipos que se conectan a la red para enviar y recibir información. Entre ellos se incluyen ordenadores, smartphones, impresoras y dispositivos IoT.                                                            |
| **Protocolos**                 | Son el conjunto de reglas que regulan cómo se establece la comunicación y cómo deben interpretarse los datos que se transmiten a través de la red.                                                                      |
| **Medio de transmisión**       | Constituyen el canal por el cual se envía la información desde el origen hasta el destino. Pueden ser medios físicos, como cables Ethernet y fibra óptica, o medios inalámbricos, como las conexiones wireless (Wi-Fi). |
| **Elementos de interconexión** | Son los dispositivos encargados de conectar los hosts a la red y asegurar que los datos lleguen al destino correcto, como adaptadores de red, switches y routers                                                        |

Hay dos tipos de redes principales: las redes redes privadas e Internet, que es una red pública global.

## 1.1. Red privada

Una red privada es una red aislada formada por dispositivos interconectados mediante un switch. Cada dispositivo está identificado por una IP privada, válida solo dentro de esa red. Este reparto puede realizarse de manera manual o automática mediante el protocolo DHCP ("Dynamic Host Configuration Protocol").
Dentro de estas redes, hay varios tipos:

| Red                                                           | Descripción                                                                                                                                                                                           |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 🏠 LAN: (Red de Área Local)                                   | Son redes domésticas o pequeñas redes de oficina que conectan dispositivos a corta distancia, generalmente con cables Ethernet. <br>Una LAN con Wi-Fi es una "WLAN" o "Wireless LAN"                  |
| 🏫 CAN : (Red de Área de Campus)                              | Se forma por varias redes LAN conectadas en un área limitada, como una universidad o una empresa.                                                                                                     |
| 📡MAN (Red de Área Metropolitana) y WAN (Red de Área amplia): |  Se forman al interconectar varias redes LAN o CAN dentro de una ciudad o zona urbana, lo que permite, por ejemplo, conectar las distintas oficinas que una empresa tiene distribuidas por la ciudad. |
> Cuando usamos datos móviles, nuestro teléfono se conecta a la antena más cercana de la compañía telefónica. Estas antenas están interconectadas mediante una MAN que cubre la ciudad (o WAN que cubra un país) y permite enviar el tráfico a Internet.

## 1.2. Red pública

A una red LAN se le puede añadir un componente fundamental: el router. Este dispositivo permite conectar nuestra red local con el exterior, es decir, con Internet.
Internet es la red pública mundial, conocida como la “red de redes”, ya que se forma mediante la interconexión de múltiples redes WAN pertenecientes a distintos proveedores y organizaciones de todo el mundo. El router se encarga de gestionar la comunicación entre la red privada (LAN) y la red pública (Internet).

En esta red, el proveedor de Internet asigna una dirección IP pública al router, y no a cada uno de los dispositivos de la red local. Esto tiene dos ventajas principales:
1. Evita que se agoten las direcciones IP disponibles, ya que no son infinitas.
2. Impide el acceso directo desde Internet a los dispositivos de la LAN, lo que mejora la seguridad.

## 1.3. Protocolo NAT

Dentro de una red privada los dispositivos se identifican por una dirección IP privada que sirve para comunicar los dispositivos entre sí.
El protocolo NAT (Network Address Translation) se encarga de traducir las direcciones IP privadas de los dispositivos dentro de una red doméstica (LAN) a una única IP pública compartida para acceder a Internet

Esta IP pública pertenece al dispositivo que actúa como puerta de enlace, el router, que funciona como un puente entre la red privada y la red pública.

- ➡️ Todas las comunicaciones que salen de la LAN hacia Internet pasan por el router y utilizan su IP pública. Este por tanto, realiza la traducción entre direcciones IP privadas y públicas. Además como todo el tráfico pasa por este punto, tiene visibilidad completa del tráfico de la red.
- ⬅️ Las respuestas desde Internet también llegan primero al router, que se encarga de redirigirlas al dispositivo correspondiente dentro de la red local.

El emisor externo no sabe a qué dispositivo concreto se están enviando los datos. Aquí se produce el enmascaramiento, ya que todos los dispositivos individuales quedan ocultos tras la IP pública del router.

> *Ej: Si en mi casa el teléfono tiene la IP privada "192.168.1.13" y el ordenador tiene la "192.168.1.15", al conectarse a internet, ambos tienen la IP pública "81.23.45.67" que es la del router. En otra casa hay una TV inteligente con la IP "192.168.1.13" pero no tiene nada que ver con nuestro teléfono, ya que hablamos de redes distintas y aisladas.*

> [!tip]+ El protocolo OSPF
> Para que dispositivos de dos redes distintas se comuniquen, la conexión tiene que pasar entre los routers intermedios. Pues bien, **este protocolo trata de encontrar la ruta mas corta entre ambos para que la conexión sea lo más rápida posible**. Esto sucede ya que los routers intercambian información de enrutamiento constantemente.


---
# 2.  Modelo OSI

Cuando enviamos datos por la red, es fundamental que lleguen de forma segura, confiable y estructurada al destinatario correcto.

Para ello, la comunicación se organiza en una serie de pasos llamados **capas de protocolo o pila de protocolos**. Cada capa cumple una función específica y añade información a los datos mediante un proceso denominado **encapsulación**.

> *Los datos por tanto son como una cebolla, están compuestos por múltiples capas que se añaden o quitan en cada paso.*

## 2.1.  Modelo OSI y TCP/IP

Existen dos modelos de referencia principales que estandarizan estas capas y facilitan la implementación y comprensión del funcionamiento de una red:

> [!info] El modelo TCP/IP
> El modelo TCP/IP describe de forma práctica y realista cómo funcionan las redes actuales, y es la base del funcionamiento de Internet.
Está formado por cuatro capas, en las que los datos se procesan y encapsulan de manera progresiva para permitir la comunicación entre dispositivos. Cada capa utiliza los servicios de la capa inferior y ofrece servicios a la capa superior.

> [!info] El modelo OSI
> El modelo OSI es un modelo conceptual de referencia que divide la comunicación en siete capas, asignando funciones más específicas y detalladas que el modelo TCP/IP. Su objetivo principal no es describir una red real, sino facilitar el estudio, el diseño y la resolución de problemas en redes, permitiendo entender mejor la función de cada protocolo y componente dentro del proceso de comunicación.

Los datos viajan a través de las capas. Cuando enviamos los datos se añaden encabezados con información necesaria para el envío, como direcciones o instrucciones. Los datos junto a sus encabezados constituyen una **PDU** o "unidad de datos de protocolo" y cada capa tiene el suyo.

Estas son las capas del modelo TCP/IP y cómo se desglosan en el modelo OSI. También vemos los protocolos y las PDU, que tienen más capas en las fases más bajas y cercanas al hardware.

| TCP/IP        | OSI                                            | PDU       |
| ------------- | ---------------------------------------------- | --------- |
| 4. Aplicación | 7. Aplicación <br>6. Presentación<br>5. Sesión | Datos     |
| 3. Transporte | 4. Transporte                                  | Segmentos |
| 2. Internet   | 3. Red                                         | Paquetes  |
| 1. Acceso     | 2. Enlace<br>1. Física                         | Tramas    |

---
## 2.2.  Capa de acceso

**Esta capa se encarga de comunicar los dispositivos de una red local entre sí**

**Su PDU son las tramas, que incluyen las direcciones MAC** (direcciones físicas) de origen y destino de cada dispositivo. Estas direcciones son únicas, no cambian y están grabadas en la tarjeta de red (NIC) que opera con los datos en binario.

Se divide en las capas físicas y de Enlace del modelo OSI

> [!example]+ Capa física
> **La capa física** **define la manera en la que se van a transmitir y recibir los datos en binario a través de medios físicos como cables y redes inalámbricas**. Gestiona aspectos físicos como la velocidad, el ancho de banda y la codificación de datos. **Esta es la capa más cercana al Hardware**
> 
> Los protocolos que maneja son:
> 
> | Protocolo        | Descripción                                                                |
| ---------------- | -------------------------------------------------------------------------- |
| **Ethernet**     | Conexiones cableadas, rápidas y confiables                                 |
| **Wi-Fi**        | Redes inalámbricas, más flexibles, pero menos veloces que Ethernet         |
| **Bluetooth**    | Redes de muy corta distancia                                               |
| **LTE/5G**       | Redes móviles para acceso a Internet desde dispositivos móviles.           |
| **Fibra óptica** | Señales lumínicas ultrarrápidas que se envían por un cable de fibra óptica |

> [!example]+ Capa de enlace
**Esta capa se encarga de la transmisión de datos entre equipos conectados directamente en una misma red (conexión de extremo a extremo).** Se encarga de reordenar los datos y realizar una detección de errores, rechazando cualquier trama que haya sido alterada.
> - Los switches operan en esta capa, interconectando los dispositivos de la red y asegurando que cada trama llegue al destino correcto dentro de la red local.
> - Opera junto a la capa de red, ayudando a definir el camino que seguirán los paquetes (enrutamiento).
>
> En una red además se utilizan estos modelos de difusión para enviar los datos:
> 
> | Protocolo     | Descripción                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------ |
| **Broadcast** | Si la MAC destino es “FF:FF:FF:FF:FF:FF”, los datos se envían a todos los dispositivos de la red |
| **Unicast**   | Se envían los datos a un solo dispositivo, al que corresponde la MAC destino                     |
| **Multicast** | Se envían datos a un grupo multicast conformado por varios dispositivos.                         |

--- 
## 2.3. Capa de internet

**Se encarga de transportar datos entre dispositivos que se encuentran en redes distintas (conexión punto a punto). También se llama capa de red en el modelo OSI**

Maneja direcciones IP (direcciones lógicas), que cambian según la red en la que esté conectado el dispositivo. **Su PDU es el paquete, que incluye un encabezado con información de enrutamiento, como la dirección IP origen y destino**.

Por cada red que pasa, los paquetes son desencapsulados y se vuelven a encapsular actualizando la IP a la del paso siguiente hasta llegar al destino final. Por tanto **no se puede saber cuál es la IP original** (la IP privada del dispositivo de su red LAN correspondiente)

**El router opera en esta capa**: interconecta redes diferentes determinando la ruta más corta hacia el destinatario, a esto se le llama **enrutamiento**

Utiliza estos protocolos:

| Protocolo | Descripción                                                                           |
| --------- | ------------------------------------------------------------------------------------- |
| IMCP      | Permite detectar y diagnosticar problemas de red, como ver si el destino es accesible |
| DHCP      | Permite asignar IPs de manera automática a cada dispositivo en una red                |
| OSPF      | Lo hemos enseñado antes, este protocolo calcula la ruta más corta entre dos routers   |

--- 
## 2.4. Capa de transporte

**Gestiona la comunicación entre aplicaciones que se ejecutan en distintos hosts, independientemente de si están en redes distintas. También se llama capa de transporte en el modelo OSI**

- Su PDU divide los datos intercambiados entre aplicaciones en **fragmentos**, que pueden ser segmentos (TCP) o datagramas (UDP). Este proceso, llamado **segmentación**, evita la saturación de la red y de los propios dispositivos.
- Los datos se etiquetan para que, aunque se mezclen durante el trayecto, puedan reordenarse y ensamblarse correctamente al llegar al destino, esto es la **multiplexación**
- El intercambio de datos entre la aplicación de un host y la de otro se denomina **conversación**. Un equipo puede mantener varias conversaciones simultáneas e independientes, cada una identificada por un puerto lógico.

Puede priorizar confiabilidad o velocidad, según el protocolo:

> [!example]+ TCP
Es un protocolo **orientado a la conexión**: **garantiza que los datos se entreguen sin errores, completos y en el mismo orden en que fueron enviados.**
> - Antes de enviar datos establece una comunicación segura mediante un proceso de tres pasos conocido como _handshake_ (SYN, SYN-ACK, ACK). Al terminar la conversación, utiliza otro (FIN-ACK).
> - Si un paquete se pierde, se retransmite.
> - Incorpora también mecanismos de control de flujo y congestión para ajustar la velocidad de envío según las condiciones de la red.
Todas estas comprobaciones hacen que sea más lento, pero mucho más fiable.

> [!example]+ UDP
Es un protocolo no orientado a la conexión, lo que significa que no realiza comprobaciones previas ni establece un canal asegurado entre los extremos antes de enviar los datos. Simplemente transmite los paquetes sin verificar su recepción, orden o integridad.
Esto lo hace mucho más rápido y eficiente, aunque menos fiable, ya que los paquetes pueden perderse o llegar desordenados sin que el emisor lo detecte.
Es ideal para streaming, dónde no importa si se corrompe algo de información, pero no para enviar datos bancarios.

## 2.5. Capa de aplicación

**Gestiona los protocolos que utilizan las aplicaciones con las que interactúa el usuario. En el modelo OSI se divide entre las capas de aplicación, presentación y sesión**

Es la capa más alta del modelo TCP/IP, donde operan las aplicaciones o servicios en red. Esta capa proporciona comunicación entre las aplicaciones y las redes.

Puede seguir uno de estos modelos:
- **Cliente-servidor**  : El servidor ofrece uno o varios recursos, y los clientes se conectan a él para acceder a dichos recursos.    
- **Peer to Peer**  : Los equipos de la red comparten y utilizan recursos entre sí, actuando simultáneamente como clientes y servidores.  Los datos que viajan del cliente al servidor se denominan upload (subida). Los que van del servidor al cliente se conocen como download (descarga).  

> [!example]+ Capa de aplicación
> Es la capa que maneja el lenguaje e interfaz con el que el usuario interactúa con la aplicación. Podemos encontrar estos protocolos:
> 
> | Protocolo                       | Ejemplos                                         |
| ------------------------------- | ------------------------------------------------ |
| **Navegación web:**             | HTTP (80), DNS (53) y HTTPS (443)                |
| **Administración de sistemas:** | Telnet (23), SSH (22), WinRM (5985) y RDP (3389) |
| **Correo electrónico:**         | POP3 (110), IMAP (143) y SMTP (25)               |
| **Transferencia de archivos:**  | FTP (21), SMB (445) y NFS (2049)                 |
> 

 > [!example]+ Capa de presentación
> Se encarga de que los datos enviados sean comprensibles para la aplicación que los recibe. Sus principales funciones son:
> - **Codificación**: Define cómo se interpretan los bytes enviados (e.g., ASCII, UTF-8 para texto, JPEG para imágenes)
> - **Compresión**: Reduce el tamaño de los datos para acelerar la transmisión
> - **Encriptación**: Protege los datos de accesos no autorizados, como ocurre con SSL/TLS, que convierte HTTP en HTTPS

 > [!example]+ Capa de sesión
> Gestiona las sesiones, es decir, las conexiones que se establecen entre aplicaciones de distintos dispositivos. **Cada sesión corresponde a un usuario o proceso distinto, por lo que esta capa asegura que no se mezclen los datos.**
> Se encarga de:
> - **Establecer, mantener y finalizar sesiones**
> - **Identificar y separar sesiones mediante identificadores únicos o tokens, que funcionan como "pasaportes" para cada conexión activa**
> - **Sincronizar la comunicación,** garantizando que, si una conexión se interrumpe, los datos puedan recuperarse o retransmitirse sin pérdidas ni corrupción

---
# 3. Direcciones MAC

Cada dispositivo que se conecta a Internet dispone de un componente de **hardware** llamado **NIC (Network Interface Card)** o **tarjeta de red**. Esta tarjeta permite al sistema **enviar y recibir datos** hacia el exterior y está **adaptada a un tipo concreto de transmisión física**. Es decir, existen distintos tipos de tarjetas de red:
- **Inalámbricas**, que utilizan **Wi-Fi**.
- **Cableadas**, que emplean **cables Ethernet**.

La **NIC** se encarga de **convertir las señales eléctricas o inalámbricas** en **bits** que el sistema puede procesar, y viceversa. Cada tarjeta de red tiene asociada una dirección MAC o dirección física, que está grabada de fábrica, es única y no cambia, a diferencia de la dirección IP, que puede variar.

 > [!info]+ Anatomía de una MAC
> La dirección MAC está compuesta por **6 bytes** (48 bits) y se representa mediante **números hexadecimales**.
> 
| OUI                                                                                                                  | Identificador                                                                     |
| -------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Los 3 primeros se denominan el OUI (“Organizationally Unique Identifier”) e identifican al fabricante de la tarjeta. | Los otros 3 son arbitrarios y diferencian a la tarjeta de otras con el mismo OUI. |
> 

Podemos identificar nuestra dirección MAC si hacemos el comando  `ifconfig`  y filtramos por la palabra "Ether". 
```powershell
ifconfig | grep "ether" # > ether AA:AA:AA:BB:BB:BB

# Conocer nuestro fabricante
macchanger --list | grep <nuestro_OUI>  # > grep "AA:AA:AA"
```

---
## 3.1. Interfaces de red

Un mismo dispositivo puede disponer de **varias tarjetas de red**, ya sea **físicas (hardware)** o **virtuales (creadas por software)**. Cada vez que el dispositivo se conecta a una red mediante una **tarjeta de red**, el sistema utiliza una **interfaz de red**, que actúa como una "puerta" de comunicación entre el dispositivo y esa red.

La interfaz de red está asociada a una **dirección MAC**, y sobre ella se **configuran una o varias direcciones IP**, que identifican al dispositivo dentro de esa red concreta.

| Interfaz | Nombre común        | Uso                                                                                                                                                                                                                                                                                                |
| -------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Loopback | Lo                  | Esta interfaz es interna al dispositivo y permite la comunicación consigo mismo. Utiliza la IP 127.0.0.1, conocida como **localhost**. Es útil para realizar pruebas o para servicios que solo deben estar accesibles dentro del mismo dispositivo                                                 |
| Ethernet | eth0, enp0s3, ens33 | Corresponde a las conexiones cableadas (Ethernet), que ofrecen alta velocidad y estabilidad. En el caso de las maquinas virtuales, se conectan con un "cable Ethernet" virtual.                                                                                                                    |
| Wi-Fi    | wlan0, wlp2s0       | Se usa para conexiones inalámbricas (Wi-Fi)                                                                                                                                                                                                                                                        |
| TUN/TAP  | tun0                | Estas interfaces son virtuales y se utilizan para redes privadas virtuales (VPN) o redes virtuales como VLANs. La interfaz tun0 es comúnmente creada por programas como OpenVPN, WireGuard o cualquier software de VPN.                                                                            |
| Otras    | docker0             | Si configuras herramientas como Docker o contenedores, pueden aparecer interfaces adicionales (por ejemplo, docker0) que gestionan redes internas o puentes virtuales entre contenedores. Cada servicio especializado que utilice redes puede generar sus propias interfaces, según sea necesario. |

---
## 3.2. Tablas ARP

El **protocolo ARP (Address Resolution Protocol)** se encarga de **relacionar direcciones IP (lógicas)** con las **direcciones MAC (físicas)** de los dispositivos dentro de una **red local**.
1. Cuando un dispositivo necesita comunicarse con otro **en la misma red**, primero debe conocer la **dirección MAC** del destino. Para ello, envía una **solicitud ARP** a la **dirección broadcast**, preguntando:  `«¿Quién tiene esta dirección IP?»`. 
2. Este mensaje se envía a **todos los dispositivos de la red**.
3. El dispositivo que posee esa **dirección IP** responde indicando su **dirección MAC**. 
4. Con esta información, el equipo emisor **actualiza su tabla ARP**, lo que permite que los **paquetes se envíen correctamente** en la **capa de enlace de datos**. 

Las **tablas ARP** almacenan estas asociaciones IP–MAC de forma **temporal**, evitando repetir el proceso continuamente.

El comando  **arp**  permite **consultar la tabla ARP** del sistema y ver todos los registros almacenados.

| Uso                                             | Comando             |
| ----------------------------------------------- | ------------------- |
| Mostrar la tabla ARP completa del equipo        | `arp -a`            |
| Eliminar una entrada ARP específica             | `arp -d <IP>`       |
| Agrega una entrada estática en la tabla ARP<br> | `arp -s <IP> <MAC>` |

---
# 4. Direcciones IP

**Las direcciones IP son identificadores únicos que se reparten por cada dispositivo conectado a una red.**

Como hemos dicho, las direcciones IP operan en la capa de Internet o Red. Este protocolo está vigente desde los 70 y es una parte fundamental de internet, independientemente del medio de transmisión; ya sea Ethernet, Wi-Fi o fibra. 

Mientras que las direcciones MAC son las "direcciones físicas", porque son las que usa el hardware, las direcciones IP se llaman "lógicas" porque las usa el software.

## 4.1. Tipos de IP

Existen estos dos tipos:

 > [!example]+ Direcciones IPv4
> Las direcciones más famosas y utilizadas.
Están compuestas por 4 bytes representados en decimal y separados por un punto “.”. A cada uno de estos bytes se le llaman **octetos** (porque cada byte son 8 bits).
Ej:  `192.168.1.4`

 > [!example]+ Direcciones IPv6
Estas utilizan 16 bytes en formato hexadecimal separados por dos puntos “:”.
Se crearon por si algún día se acaban las Ipv4, pero apenas se utilizan ya que gracias al protocolo NAT no hay que repartir una IP por host sino por red.
Ej  `fe80::37ce:2e75:e076:b0c6`

## 4.2. Anatomía de una IP

Una IPv4 se divide en dos partes:

| Parte                     | Descripción                                                                                                                                                                                                                                                                                           |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identificador de Red**  | Identifica la red a la que pertenece un dispositivo. En base a este identificador, los routers saben si el paquete a enviar está dentro de su red o no.   <br>*Por ejemplo, si en una LAN las IPs empiezan por 192.168.1 y se manda un paquete a la 10.2.56.3, se sabe que hay que salir a internet.* |
| **Identificador de Host** | Distingue un dispositivo específico dentro de una red, lo que permite a los dispositivos comunicarse dentro de la red entre sí.                                                                                                                                                                       |
¿Y cuantos octetos se le asigna a cada parte? Pues esto depende del tamaño de la red. En redes muy grandes (CAN, MAN), hay muchos dispositivos, así que necesitamos más octetos (2 o 3) para el identificador de host, en cambio en redes más pequeñas (LAN), con un octeto basta.

## 4.3. Tamaños de redes

Si elevamos 2 a los bits del identificador de host, nos dará el número de direcciones IPs posibles a repartir.

```powershell
192.168.1.14

1. En este caso hay solo un byte para el host, así que son en total 8 bits
2. Elevamos 2 a 8, esto nos da 256 (a los que le restamos dos, uno para la red (192.168.1.0), y otro para la broadcast (192.168.1.0).
3.  Por tanto es una red de pequeño tamaño (clase C)
```

Para representar el tamaño de la red tenemos dos cosas, la máscara de red y el CIDR:

 > [!example]+ Máscara de red
**Representa el tamaño mediante una IP.**
Si le restamos a 255.255.255.255 la máscara de nuestra red obtendremos el número de dispositivos que se pueden conectar a la red  quitando las direcciones reservadas
> ```
> Por ejemplo tenemos la máscara 255.255.255.0
> Por tanto se lo restamos a 255.255.255.255 y nos da 0.0.0.0, lo que significa que en nuestra red se pueden conectar 255 dispositivos
> ```

 > [!example]+ CIDR**Representa el tamaño mediante una IP.**
Representa el tamaño de manera más eficiente; con una barra ("/") y un número del 1 al 32 que indica la cantidad de bits utilizados para la red. El resto por tanto serían los bits para el dispositivo.
> ```
> Dado el CIDR /24
> Habría 24 bits para la red, es decir, 3 bytes, por tanto 3 octetos. El octeto restante es para el dispositivo, por tanto estaríamos ante una red pequeña de clase C, probablemente una LAN doméstica
> ```

Por tanto nos podemos encontrar con estos tamaños de redes

| Clase                       | Mascara de subred                                     | CIDR | Máximo de dispositivo                                                       |
| --------------------------- | ----------------------------------------------------- | ---- | --------------------------------------------------------------------------- |
| **Clase A**  <br>(MAN, WAN) | 255.0.0.0:  <br>- 8 bits de red  <br>- 24 de host     | /8   | 256 redes con 16 millones de dispositivos por cada una (224)                |
| **Clase B**  <br>(CAN, LAN) | 255.255.0.0:  <br>- 16 bits de red  <br>- 16 de host  | /16  | 65,000 dispositivos (216) máximos por red.  <br>Suelen empezar por 10.X.X.X |
| **Clase C**  <br>(LAN)      | 255.255.255.0:  <br>- 24 bits de red  <br>- 8 de host | /24  | 256 equipos por red (28).  <br>Suelen empezar por 172.X.X.X o 192.168.X.X   |

## 4.4. Direcciones IP reservadas

En una red tendríamos que prestarle especial atención a ciertas direcciones IP reservadas, es decir, que no se pueden asignar a los dispositivos:

 > [!info]+ La dirección de red
Esta dirección es la primera de todas en una red. Siempre se representa junto al CIDR, que indica por tanto el tamaño de la red. 
*Por ejemplo, en las redes LAN la dirección de red por defecto suele ser la  192.168.1.0/24 .*
> - ¿Y qué pasa si se manda un paquete a esta dirección? Pues que simplemente se descarta, no es una dirección valida.

 > [!info]+ Dirección de la puerta de enlace (gateway)
La puerta de enlace está presente en redes que tienen salida a otra red más grande y corresponde al dispositivo que hace la conversión NAT, que suele ser el router.
Este router suele tener un portal de administración web, así que si accedemos a esa dirección desde nuestro navegador, probablemente nos lo encontremos. Esta suele ser la segunda dirección IP que se reparte. 
*Por tanto en nuestra red 192.168.1.0/24, la puerta de enlace muy probablemente es la 192.168.1.1*

 > [!info]+ Dirección broadcast
Es la dirección de difusión, todo lo que se envíe a esta dirección, se retransmite a todos los miembros de la red. Esto es útil por ejemplo para crear las tablas ARP.
Siempre es la última, en el ejemplo anterior, sería la  **192.168.1.255**

## 4.5. Ver información de red

En Windows, el comando para ver la información de red es `ipconfig` y en Linux `ifconfig`

1. **Estado y capacidades de la interfaz**: Si pone **UP**, es que está activa.
    
2. **Direccion IPv4**: "**inet**" seguido de la IPv4 y "**netmask**" para la máscara de subred.
    
3. **Direccion IPv6:** "**inet6**" seguido de la IPv6
    
4. **Dirección MAC:** "**ether**" seguido de la MAC

> Para ciertos sistemas Linux dónde no exista ifconfig, está el comando `ip -a`. Este nos indica también la información de redes solo que añade distinción de colores

| Comando                                | Linux                | Windows    |
| -------------------------------------- | -------------------- | ---------- |
| Saber nuestra información de red       | `ifconfig` / `ip -a` | `ipconfig` |
| Ver solo la IP                         | `hostname -I`        |            |
| Ver el nombre del host                 | `hostname`           | `hostname` |
| Ver la puerta de enlace                | `ip route`           | `ipconfig` |
| Probar la conectividad con otro equipo | `ping <ip>`          | `ping ip`  |
 > [!tip]+ Hacer Pings
**La herramienta ping permite comprobar la conectividad entre dos dispositivos en una red, ya sea local o en Internet**. 
Funciona enviando paquetes ICMP ("Internet Control Message Protocol") tipo "echo request" al destino y esperando respuestas "echo reply". 
> - Permite medir el tiempo de ida y vuelta (latencia) y detectar posibles pérdidas de paquetes
> - Es ampliamente usada para diagnosticar problemas de red y verificar si un host o servidor está activo y accesible.
> - El TTL, o "time to live" indica el rango máximo de routers por el que puede pasar un paquete antes de ser descartado. Por cada router que pasa, disminuye en una unidad.: Windows da un TTL cercano a 128 y Linux a 64

Primero tenemos que habilitar o deshabilitar interfaces:

| Comando                                | Linux                                                                 | Windows                                               |
| -------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------------- |
| Listar interfaces                      | `ifconfig` / `ip -a`                                                  | `netsh interface show interface`                      |
| Habilitar/deshabilitar una interfaz    | `ifconfig <interfaz> up/down`<br>`ip link set dev <interfaz> up/down` | `netsh set interface <interfaz> admin=enable/disable` |
## 4.6. Configurar nuestra dirección IP

En Linux existen dos servicios principales que gestionan las redes: **NetworkManager** y **systemd-networkd.**

La puerta de enlace en las máquinas virtuales es la segunda IP a repartir. Tanto para ver cuál es como para ver el CIDR de la red, usaremos `ip route` y así nos evitamos fallos.

 > [!tip]+ System-Networkd
 Está diseñado principalmente para servidores y sistemas minimalistas dónde se requiere una configuración de forma estable y predecible. Esta configuración se realiza editando a mano un archivo, por lo que se denomina "declarativa"
Toda la configuración, la realizamos por medio del archivo  `/etc/systemd/network/eth0.network`  . 
> ```powershell
> [Match]
> name=eth0
> 
> [Network]
> Address=192.168.159.145/16
> Gateway=192.168.159.2
> DNS=1.1.1.1
> ```
> - Si queremos que vaya por DHCP, en `[Network]` solo hay que poner `DHCP=yes`
>
Luego reiniciamos el servicio con:  `sudo systemctl restart systemd-networkd`

 > [!tip]+ Network-Manager
 Se usa para sistemas de escritorio portátiles y entornos donde la red cambia con frecuencia. 
 Se encarga de configurar automáticamente las interfaces de red (Ethernet, Wi-Fi, VPN), obtener direcciones IP mediante DHCP, gestionar rutas y DNS, y reaccionar a cambios como conectarse a otra red o activar una VPN. **Es el que se usa en kali**
 > 1. Para listar interfaces podemos usar `nmcli device status`. Nos tenemos que fijar en la columna "Connection", esta es el nombre por el que configuraremos la interfaz. Por ejemplo "Conexión cableada 1"
 > 2. Luego seteamos las direcciones
 > ```
> nmcli con mod <nombre> ipv4.method manual ipv4.addresses 192.168.159.145/16 \
>                        ipv4.gateway 192.168.159.2 ipv4.dns 1.1.1.1
> ```
> 3. Finalmente reiniciamos con `nmcli con down <nombre> && nmcli con up <nombre>`
> - Para dhcp es `nmcli con mod <nombre> ipv4.method auto ipv4.ignore-auto-dns no`

> [!warning] resolv.conf
En muchos tutoriales se explica cómo modificar el DNS editando el archivo **/etc/resolv.conf** o cómo configurar la puerta de enlace utilizando el comando **ip route**. Sin embargo, estos cambios son temporales y pueden perderse al reiniciar el sistema o al reiniciar el servicio de red. Si se desea que la configuración sea permanente, debe realizarse como se ha explicado.

En cuanto a Windows

| Acción             | DHCP                                             | Manual                                                                      |
| ------------------ | ------------------------------------------------ | --------------------------------------------------------------------------- |
| Establecer el modo | `netsh interface ip set address <interfaz> dhcp` | `netsh interface ip set address <interfaz> static <ip> <mascara> <gateway>` |
| DNS                | `netsh interface ip set dns <interfaz> dhcp`     | `netsh interface ip set dns <interfaz> static <ip DNS>`                     |
> En una máquina virtual, la puerta de enlace no suele ser la primera IP de la red, ya que el acceso a la red se realiza mediante NAT gestionado por el hipervisor

---
# 5. Los puertos lógicos

Los puertos lógicos. Son identificadores que permiten saber a que aplicación enviarle los datos

Un equipo puede realizar **múltiples conexiones de red simultáneamente**. Por ejemplo, puede estar consultando varias páginas web al mismo tiempo mientras mantiene abierto un juego en línea. Entonces, ¿Cómo se diferencian todas esas comunicaciones, incluso cuando se realizan con el **mismo host** y usando **protocolos distintos**?

Para resolver esto, en la **capa de transporte** se utilizan los **puertos lógicos**. Un **puerto lógico** es un **identificador virtual** que permite al sistema operativo saber **a qué aplicación o proceso** debe entregar los datos que llegan desde la red.

Cada aplicación que se comunica por la red utiliza **uno o varios puertos**, lo que permite que varias aplicaciones compartan la misma dirección IP sin interferir entre sí. El puerto, por tanto, **no identifica una conexión completa**, sino el **punto de entrada o salida** de datos de una aplicación dentro del equipo.

Se denominan _lógicos_ porque son gestionados por **software**, igual que las direcciones IP, y no corresponden a conexiones físicas del dispositivo. Esto los diferencia de los **puertos de hardware**, como USB, HDMI o el puerto de carga.

## 5.1. Rangos de puertos 

Existen **65 535 puertos para TCP y otros 65 535 para UDP**, por lo que un puerto siempre debe ir asociado a su **protocolo de transporte**.
Por ejemplo, al decir _HTTPS utiliza el puerto 443/TCP_, se está indicando tanto el puerto como el protocolo.

Los puertos se dividen en distintos **rangos según su uso y nivel de privilegio**, lo que está estrechamente relacionado con el **modelo cliente-servidor**, donde los servidores suelen escuchar en puertos bien conocidos y los clientes utilizan puertos asignados dinámicamente.

> [!info] 1 - 1023: Puertos privilegiados
Estos puertos están reservados para ofrecer servicios en línea por protocolos comunes como HTTP o FTP.  
Es decir, los suele usar el sistema cuando cumple el rol de servidor.  
Además necesitan altos privilegios para ser utilizados, lo que aporta una capa adicional de seguridad

> [!info] 1024 - 49151: Puertos NO privilegiados
Estos también suelen utilizarse cuando el equipo hace de servidor, pero son utilizados por protocolos menos conocidos, como algunos programas concretos o videojuegos o cualquier servicio sin privilegios de administrador.  
Por ejemplo Minecraft utiliza el protocolo 25565 para montar sus servidores

> [!info] 49152 - 65535: Puertos dinámicos
Estos puertos los utiliza el cliente de manera aleatoria para conectarse al servidor. Se llaman puertos efímeros porque da igual el puerto concreto, cada vez utiliza uno diferente, solo se necesita que esté libre.

## 5.2 Sockets

Un **socket** es una **estructura o canal proporcionado por el sistema operativo** que permite a los procesos comunicarse a través de la red. Siempre está definido por tres elementos: un protocolo de transporte (TCP o UDP), una dirección IP y un puerto

Estos sockets residen en la **memoria del kernel** y cada proceso puede crearlos o utilizarlos mediante funciones especiales. Además el sistema operativo gestiona automáticamente los aspectos complejos de la comunicación como establecer la conexión, evitar que haya fallos, o que los datos se mezclen o desordenen.

**En una arquitectura cliente-servidor,** el servidor utiliza un puerto para escuchar peticiones, pero **cada cliente que se conecta genera un socket distinto**. Cada uno de estos sockets representa una **conversación** independiente, es decir, una comunicación concreta entre un cliente y un servidor.

**_Habría que entender al socket como si fuera un buzón digital que el sistema operativo pone a disposición de los procesos_**

> En Linux, siguiendo el principio de "Todo es un archivo", el socket también lo sería bajo la ruta  `/dev/tcp/<ip>/<puerto>` Por tanto todo lo que se mande a esa ruta, se mandará por tanto a ese sistema, incluso un comando.

