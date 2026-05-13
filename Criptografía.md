
> [!abstract]+ Introducción
> La criptografía es una técnica que se utiliza para proteger un mensaje de terceros no autorizados, y que solo el remitente y destinatario puedan entenderlo. Consiste en tomar un mensaje legible, en texto plano, (por ejemplo “Hola Mundo”), y transformarlo en uno ilegible, cifrado (“Ubyn Zhaqb”);

- **Si lo intercepta alguien autorizado,** le daremos una clave para descifrarlo.  _En este caso le decimos que mueva cada letra del alfabeto 13 posiciones (cifrado rot13). Por ejemplo, la “a” se convierte en “n”, la “b” en “o”._
    
- **Si lo intercepta alguien no autorizado,** a quien no le hemos dado la clave, no podrá leer el mensaje a menos que rompa el cifrado.  _En este caso hemos utilizado rot13, un algoritmo tan sencillo que se puede romper anotando el abecedario en una hoja de papel. Por tanto, podemos decir que en este caso el cifrado es muy débil._

# 1. Codificación, esteganografía y cifrado

## 1.1. Codificación

> [!abstract]+ Codificación
> La codificación es el procedimiento por el cual unos datos originales son reemplazados por otros alternativos, con el objetivo de que puedan ser enviados a través de un canal de comunicación sin errores.  

**Este sistema no garantiza la confidencialidad de los datos ya que se basa en una serie de reglas públicas y cualquiera puede revertir este proceso. Por tanto es una manera alternativa de representar datos.**

Es similar a como distintos idiomas escriben los mismos sonidos con caracteres diferentes. *Por ejemplo, el código morse es un esquema de codificación para enviar mensajes en base a una serie de pitidos más largos o más cortos.*

> [!warning]+ Binario
>Para entender la codificación, tenemos que saber que el ordenador solo entiende información en binario. La unidad más pequeña de información es el byte, un número que va del 1 al 256. Cada byte se compone de 8 bits. Por tanto, toda la información en memoria, está en bruto, en "1"s y "0"s. **Para traducir esos bytes, necesitamos la codificación**

El primer esquema de codificación es ASCII, que toma 1 byte por caracter. De los 256 valores posibles, se tomaron la mitad para traducirlos a símbolos imprimibles. ASCII. Por tanto es una tabla de caracteres a la vez que un formato (que indica como representarlos en memoria)

| **Rango**   | **Tipo**               | **Ejemplo**             |
| ----------- | ---------------------- | ----------------------- |
| 0x00 - 0x1F | Códigos de control     | `CR` (retorno de carro) |
| 0x20 - 0x40 | Números y puntuación   | `123 *,_`               |
| 0x41 - 0x7F | Caracteres alfabéticos | `Hola`                  |
| 0x80 - 0xFF | nada                   |                         |
Luego se creo el "**Extended ASCII**" para símbolos de otros idiomas como el francés o el alemán. Este ocupa en memoria lo mismo que el ASCII (1 byte por carácter), pero aprovecha los 256 bytes. Hay varias versiones para cada idioma, como el 'latin-1'.

> [!warning]+ Unicode
> Aun así, esto no es suficiente para cubrir otros idiomas como el chino, que tiene miles de caracteres. Para poder representar todos los símbolos posibles del mundo (incluidos emojis), se creó un estándar llamado **Unicode**. **Este asigna a cada símbolo un número único llamado punto de código** (code point), que suele escribirse como “U+XXXX”.
> **Para guardar esos caracteres en memoria o en archivos, se usan esquemas de codificación, los UTF, que traducen esos code points en bytes**.
> 
>| Tipo       | Descripción                                                                                                                                            |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **UTF-8**  | Usa entre 1 y 4 bytes por carácter.  Para los caracteres ASCII (inglés básico) solo necesita 1 byte, por eso es muy eficiente y popular en Internet.   |
| **UTF-16** | Usa 2 bytes para la mayoría de caracteres comunes y 4 bytes para los caracteres menos frecuentes (por ejemplo algunos emojis o caracteres históricos). |
| **UTF-32** | Usa siempre 4 bytes por carácter, lo que lo hace muy simple pero poco eficiente en memoria.                                                            |
> En videojuegos o aplicaciones, usar textos con muchos caracteres complejos (como chino o emojis) puede ocupar más memoria porque cada carácter requiere más bytes. _En el videojuego "Minecraft", existen maneras de saturar la memoria de una región del mapa creando libros llenos de caracteres chinos._

## 1.2. Codificación en base64

Para el resto de archivos que no sean texto, como videos, fotos y programas compilados, existen formatos especiales como Base64 y hexadecimal, que convierten cualquier byte en caracteres legibles, lo que permite transportar e intercambiar estos datos binarios sin errores. Base64 es más eficiente que hexadecimal porque usa menos caracteres para la misma información (ocupa aproximadamente un tercio)

 Base64 toma todos los bytes, los divide en grupos de 3 bytes, que se traducen en 24 bits. Estos bits los divide a su vez en grupos de 6 bits y traduce cada grupo a un carácter de una tabla con 64 caracteres. Si sobra uno o dos bytes, se le añade relleno (0).

| Acción                                                         | Comando                       |
| -------------------------------------------------------------- | ----------------------------- |
| Pasar datos a base64. La opción `-w0` lo pasa a una sola línea | `cat <archivo> \| base64 -w0` |
| Decodificar de base64                                          | `echo <base64> \| base64 -d`  |
 Siempre que tengamos una cadena que combine números, letras y algún símbolo y que acabe en "=" será base64. Por tanto los datos no están cifrados sino codificados.

## 1.3. Esteganografía

> [!abstract]+ Codificación
> **La esteganografía estudia cómo transmitir un mensaje de un emisor a un receptor y que solo el receptor sepa extraer dicha información, que está oculta en otros datos, como una imagen, un video o un audio.**  
> **En este caso se aplica el concepto de “seguridad por oscuridad”, ya que el atacante, no sabe que los datos interceptados esconden información oculta.**

Para ello analiza el archivo (foto, audio) para localizar los bits menos significativos (ceros a la izquierda) o componentes poco perceptibles al ojo/oído humano. En estos bits "insignificantes", distribuye los bits del mensaje secreto siguiendo un patrón pseudoaleatorio generado a partir de la contraseña.

Podemos aplicar esteganografía con `steghide`

| Acción                 | Comando                                                          |
| ---------------------- | ---------------------------------------------------------------- |
| Aplicar esteganografía | `steghide embed -cf <archivo> -ef <secreto> -sf <archivo_final>` |
| Extraer el mensaje     | `steghide extract -sf <archivo_con_Datos>`                       |

## 1.6. Funciones hash

> [!abstract]+ Hashing
>  **El hashing es un proceso que toma un conjunto de datos de cualquier longitud y genera un valor único de longitud fija, llamado hash o resumen (en ingles lo suelen llamar “digest”)**  
>  
>   Este valor actúa como una "huella digital" del contenido: si el archivo cambia, aunque sea mínimamente, el hash generado será completamente diferente. Para ello, divide la entrada en fragmentos y aplica sobre ellos una serie de transformaciones matemáticas que dispersan la información y producen un resultado aparentemente aleatorio. **El hash por tanto es como crear una tarta, si alteras un mínimo por ejemplo la cantidad de azúcar, el sabor de la tarta cambiará.**

A diferencia del cifrado, el hashing no se puede revertir para recuperar el contenido original; es un proceso unidireccional.

Esto hace que el hashing sea ideal para verificar la integridad de la información: al comparar el hash original con el hash del archivo recibido, podemos saber si el archivo ha sido modificado sin necesidad de abrirlo

1. **Son deterministas**: El mismo dato de entrada siempre debe producir siempre el mismo hash de salida. **Si cambia al menos un bit, se producirá un hash diferente.** Esto asegura que, si un archivo o mensaje no ha cambiado, su hash será siempre el mismo.
| 
2. **Todos los hashes tienen un tamaño fijo:** Cualquier hash generado por una misma función siempre tendrá el mismo tamaño sin importar el tamaño de los datos de entrada. Por ejemplo, SHA-256 siempre produce un hash de 256 bits, dando igual si le introducimos un breve documento de texto o una película de 5 horas.
| 
3. **Proceso unidireccional:** Debe ser imposible reconstruir los datos originales a partir de su hash lo que no revela información confidencial. Aun así las funciones hashing son públicas y todo el mundo podrá generar el mismo hash a partir de un texto plano.
| 
4. **Resistencia a Colisiones:** Debe ser casi imposible encontrar otro conjunto de datos diferente que produzca el mismo hash y genere por tanto, una colisión. Por ejemplo, SHA-256 y SHA-3 no generan apenas colisiones (considerándose así más seguras) mientras que MD5 o SHA-1 si pueden generan muchas más (inseguras).
| 
5. **Rapidez de Cálculo:** Los hashes deben ser rápidos y eficientes de calcular, verificando la integridad de los datos incluso en sistemas de gran escala.

Gracias a estas características, los hashes tienen varias aplicaciones:

- **Almacenamiento Seguro de Contraseñas:** cuando inicias sesión, el sistema convierte tu contraseña en un hash y lo compara con el hash almacenado. Así, tu contraseña real nunca se almacena ni se envía.
- **Firmas Digitales:** En sistemas de firma digital, se utiliza una función hash para crear un resumen del documento y después se cifra ducho hash con la clave privada del firmante. Así podemos validar la integridad y autenticidad del mensaje. Habrá una sección para las firmas digitales
- **Verificación de Transmisión de archivos:** si se manda un archivo por la red, podemos verificar que no se haya corrompido en el camino debido a fallos en la trasmisión o que un tercero lo haya manipulado. Muchas páginas por eso ofrecen el hash de archivos descargables
- **Optimizar búsquedas:** Calcular hashes para los elementos de una base de datos permite búsquedas más rápidas y eficientes que usar nombres u otros atributos, ya que las comparaciones se hacen sobre valores de longitud fija y más fáciles de indexar

> [!error] Fuerza bruta
Un atacante, si quiere adivinar esa contraseña tiene que hashear muchísimas posibles palabras, enviárselas al sistema y que este ya diga cual es la que coincide (colisiona). Para protegernos de esto, se recomienda utilizar funciones lentas como bcrypt o PBKDF2, lo que hace que esa fuerza bruta se demore demasiado.**

La manera más simple de identificar los algoritmos de hashing es por el tamaño, ya que repetimos, una función siempre generará un hash de la misma longitud dando igual el tamaño del archivo de entrada.

| **Algoritmo** | **Longitud en caracteres** | **Seguridad** |
| ------------- | -------------------------- | ------------- |
| **MD5**       | 32                         | Bajo          |
| **SHA-1**     | 40                         | Bajo          |
| **SHA-256**   | 64                         | Alto          |
| **SHA-512**   | 128                        | Muy Alto      |
| **BLAKE2s**   | 64                         | Alto          |
| **BLAKE2b**   | 128                        | Muy Alto      |

## 1.7. Checksums

**Las funciones checksum se utilizan para detectar errores en la transmisión de datos**. A menudo se confunden con funciones hash, ya que ambas generan un valor de longitud fija a partir de otros datos. Sin embargo, **los checksum son simples y rápidos de calcular, pero no garantizan la integridad del mensaje, ya que son vulnerables a colisiones** (diferentes datos pueden generar el mismo checksum).

Sin embargo, los checksum son simples y rápidos de calcular, pero no garantizan la integridad del mensaje, ya que son vulnerables a colisiones (diferentes datos pueden generar el mismo checksum).

Cuando un receptor recibe un mensaje, también recibe un checksum. Para verificar la integridad, recalcula el checksum del mensaje y lo compara con el enviado. Si coinciden, significa que no hubo errores en la transmisión.

**Esto solo detecta errores accidentales, no protege contra ataques intencionados, ya que el atacante puede interceptar y modificar tanto el mensaje como su checksum, engañando al receptor.**

## 1.7. MAC y HMAC

**Un MAC es un checksum cifrado con una clave.** Esta clave se comparte con el receptor, para que él pueda descifrar el cheksum recibido y comprobar que coincida con el cheksum que calcula sobre los datos enviados. Por tanto es un sistema de clave simétrica que garantiza la autenticidad de los usuarios y la integridad de los datos, pero no la confidencialidad. Esto soluciona el problema de que un atacante intercepte el checksum y lo modifique junto al mensaje, ya que viaja cifrado.

**El HMAC es una implementación de MAC basada en HASHes en lugar de checksums.** Su funcionamiento se basa en:

1. Ambos extremos intercambian dos claves secretas (“llave1” y “llave2”).
2. Luego el emisor coge el mensaje y lo concatena con la “llave2” y se lo pasa a la funcion hash
3. Ese hash se concatena con la clave “llave1” y se vuelve a hashear, resultando en el HMAC, que se enviará junto al mensaje original.

> Obviamente que la seguridad de esto radica en que la función de hash sea segura (en lugar de MD5 utilizar SHA-128 por ejemplo) y que no se recorte el mensaje en aras de hacerlo más ligero. El HMAC por tanto se denomina “HMAC” seguido del algoritmo, por ejemplo “hmac-sha128”

## 1.8. Cifrado

> [!abstract]+ Codificación
>  **A diferencia de la codificación, el cifrado transforma el contenido original de un mensaje en algo ilegible mediante un algoritmo de cifrado y una clave que permite revertir el proceso**  

Su objetivo es proteger los datos de accesos no autorizados (como interceptores o atacantes), de manera que solo las personas autorizadas, que posean la clave correcta, puedan descifrar el mensaje y recuperar la información original. De este modo, el cifrado asegura la confidencialidad, autenticidad, integridad y no repudio de la información.

Para ello se utiliza un criptosistema, formado por todos estos elementos que permiten cifrar y proteger un mensaje y también poder recuperarlo de vuelta:
```
[emisor]       clave de cifrado        [Interceptor]      clave de descifrado   [receptor]
 │                │                        X                    │                   │
 ▼                ▼                        ▼                    ▼                   ▼
texto plano → [algoritmo de cifrado] → texto cifrado → [algoritmo de cifrado] → texto plano
```

> [!error]+ Criptosistema roto
>  **Un criptosistema está roto, cuándo el atacante obtiene una manera de determinar el texto plano a partir del texto cifrado sin que haya recibido legítimamente la clave de cifrado.** Esto puede hacerse infiriendo la clave de descifrado o mediante alguna vulnerabilidad en los algoritmos que permita obtener el mensaje sin la clave, por ejemplo, mediante patrones repetitivos.

Hay que tener en cuenta que en todo criptosistema:

1. **El cifrado no impide la intercepción:** Un atacante siempre podrá interceptar el mensaje cifrado, por lo que este debe ser lo suficientemente robusto para evitar que el atacante lo rompa y descifre.
| 
2. **El criptosistema no garantiza la protección de extremo a extremo:** Es decir si un atacante obtiene repetidamente tanto el texto plano (robado del emisor) como el cifrado, un sistema débil permitiría inferir la clave y comprometer futuros mensajes.
| 
3. **El criptosistema suele ser público:** La seguridad no debe depender de que no se conozca cómo funciona el algoritmo ("seguridad por oscuridad"), sino de su solidez matemática. Los sistemas criptográficos más confiables son aquellos que, aun siendo públicos, resisten ataques y se convierten en estándares de la industria.
| 
4. **One time pad:** Un cifrado se considera perfectamente seguro (Perfect Secrecy) cuando un atacante no puede obtener ninguna información del texto cifrado, lo que hace imposible cualquier tipo de ataque, incluyendo fuerza bruta, análisis de frecuencia o explotación de vulnerabilidades en la implementación.

> [!example] One-Time-Pad
> Para lograr esta seguridad absoluta se utiliza el **One-Time Pad**, que exige:
> 1. **Clave igual o más larga que el mensaje.**
> 2. **Una clave distinta para cada mensaje.**
> 3. **Clave completamente aleatoria.**
> 4. **Prohibido reutilizarla, ni siquiera parcialmente.**
> 5. **Debe permanecer secreta entre emisor y receptor.**
> 
> > Imagina un bot de bolsa con dos órdenes: "COMPRAR" y "VENDER", representadas aleatoriamente por 1 o 0 en cada envío. Aunque emisor y receptor conozcan la asignación, un atacante no podría deducir ningún patrón.
> 
> Aunque ofrece seguridad absoluta, **su uso es impráctico salvo para información extremadamente sensible**, ya que requiere claves tan largas como el mensaje, totalmente aleatorias y de un solo uso. **Ni siquiera cifrados modernos como** AES-256 **cumplen estas condiciones, por lo que teóricamente son vulnerables.** Sin embargo, en la práctica, romperlos llevaría miles de millones de años, haciéndolos suficientemente seguros para cualquier uso real.

## 1.9. Resumen final

| **Técnica**        | **Seguridad que garantiza**                                            |
| ------------------ | ---------------------------------------------------------------------- |
| **Codificación**   | Ninguna, simplemente cambia el formato de los datos.                   |
| **Cifrado**        | Confidencialidad de los datos, haciéndolos ilegibles para un atacante. |
| **Funciones hash** | Integridad de los datos, como se altere un solo bit, el hash cambia.   |
| **Esteganografía** | Ocultación de los datos, es decir "security by obscurity".             |
| **Checksums**      | Verificar que no ha habido errores accidentales de transmisión.        |
| **HMAC**           | Integridad del mensaje y autenticidad del remitente.                   |
Si se quiere garantizar la triada de la CIA al completo, se puede concatenar un cifrado de la información y luego el cálculo de un HMAC sobre los datos cifrados. El receptor verifica entonces la integridad de los datos y si es correcta, es decir el HMAC es el mismo que le envió el emisor, procederá a descifrarlo.

Para automatizar esto, se usa un modo de operación llamado “aes-256-cbc-hmac-sha256”, que sirve para cifrado en bloque (que veremos más adelante).


# 2. Cifrado simétrico

## 2.1. Cifrado simétrico

**En el cifrado simétrico  se utiliza la misma clave tanto para cifrar el mensaje como para descifrarlo** . Esta clave es una contraseña que se le pide al usuario y tiene que ser una **contraseña robusta, generada y almacenada de manera segura**. 

El proceso de cifrado es este:
1. El remitente crea una llave y le manda una copia al receptor.
| 
2. Luego cifra un mensaje con la llave y se lo manda tambien al receptor
| 
3. El receptor descifra el mensaje con la copia de la llave. Esta le sirve además para poder cifrar una respuesta y continuar la comunicación

> [!error] El problema reside en que un atacante puede interceptar la clave, por tanto, descifrar el mensaje (y este deja de ser confidencial). Por tanto esa clave en muchos sistemas como el SSL/TLS, se cifrará también.

Las ventajas e inconvenientes del cifrado simétrico son estos:

| Ventajas                                                                                                                                                   | Inconvenientes                                                                                                                                 |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **✅** **Veloz y eficiente:** ya que requiere menos recursos de computación, lo que lo hace una opción ideal si queremos cifrar grandes cantidades de datos | **❌** **Difícil de escalar:** En sistemas con muchos usuarios sería muy complejo, ya que cada par de usuarios necesita su propia clave secreta |
| **✅** **Simple:** al final solo se comparte una clave, por tanto, cualquiera puede aprender a utilizarlo fácilmente                                        | **❌** **Algo inseguro:** Al requerir de una sola clave que puede ser robada, es más inseguro que uno asimétrico                                |

Por tanto, es muy útil cifrando:
- **Grandes volúmenes de datos:** discos duros, memorias USB, dispositivos móviles o bases de datos (en sectores como banca, salud y finanzas). También se aplica a los datos almacenados en la nube
- **Archivos y carpetas comprimidos:** mediante herramientas como WinZip o 7-Zip, protegiendo estos durante su transmisión y almacenamiento
- **Datos en canales seguros:** como redes empresariales protegidas o redes privadas virtuales (VPN). Ya no hay peligro de que nadie robe la clave, así que se puede enviar sin problema sin complicarse con el cifrado asimétrico

> [!tip] La sal (salt)
> A la clave que utilizamos para cifrar, se le puede añadir sal ("salt", en inglés), que es un valor aleatorio (no lo decidimos nosotros). Luego se hace un hash con ello: `hash("contraseña123 + "sal12345") = clave_real`
> 
> Con esta nueva clave se cifran los datos. Por tanto, si se cifran los mismos datos con la misma contraseña 2 veces, el resultado será diferente y un atacante creerá que son datos o contraseñas distintas a primera vista. **Este proceso de transformar una contraseña se llama "derivación de claves" y no solo se usa en cifrado sino también en hashing, por ejemplo, al almacenar contraseñas.**
> 
> | **❌** **Sin sal**                                                                                                  | **🧂****Con sal**                                                                                                                  |
| ------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| **usuario 1** : `hash("contraseña123") → abcdef123456`  <br>**usuario 2** : `hash("contraseña123") → abcdef123456` | **usuario 1** : `hash("contraseña123" + "sal1") → xyz123abcdef`<br>**usuario 2** : `hash("contraseña123" + "sal2") → 789ghi456def` |
Esta sal no necesita ser secreta. Generalmente, se almacena junto al hash o los datos cifrados, para que al descifrar o verificar una contraseña se pueda reconstruir el mismo valor. Simplemente sirve agregar aleatoriedad y evitar coincidencias fáciles de identificar.
 
## 2.2. Cifrado simétrico en flujo

**Aquí ciframos los datos bit por bit (o byte por byte) en tiempo real, a medida que se leen (de ahí streaming o “flujo continuo de datos”). Bit enviado, bit cifrado. Por tanto, es un cifrado a tiempo real que no necesita saber cuántos datos se enviarán.**

Estos algoritmos tienen un componente que se llama "**keystream generator**", que convierte una clave de cifrado en una cadena de bits llamada “keystream”.

Esta clave se va generando a la par que se lee el texto plano, por tanto ambos tendrán la misma longitud y se le pasarán simultáneamente a la función de cifrado de la que saldrá el texto cifrado.
- Para descifrar el mensaje, el receptor debe tener la misma clave y el mismo keystream generator. Si se le pasa este keystream y el texto cifrado a la función de cifrado, saldrá el texto plano original.
- Para derivar el keystream, el atacante debe tener tanto el texto cifrado como el texto plano, es decir, a partir de dos elementos cualesquiera, puede obtener el tercero. Es por eso que es muy importante utilizar claves de un solo uso cuando se aplique este cifrado.

Los algoritmos más famosos son:

|**Algoritmo**|**Descripción**|**Seguridad**|
|---|---|---|
|**XOR**|Empareja los bits del texto y el keystream y si coinciden, sale un 0, si se diferencian, sale un “1”|Es el más básico y débil (por tanto, desaconsejado), pero es la base sobre el que se construye el resto.|
|**RCy RC4**|Fueron muy utilizados en protocolos como SSL/TLS y WEP. Funcionan barajando (mezclando) bytes y aplicando XOR con el texto para cifrar|Se descubrió que son vulnerables: se puede romper si si la clave se reutiliza o si se analizan ciertos bytes iniciales predecibles|
|**Salsa20**|No es muy común en implementaciones actuales, pero sigue siendo una opción sólida. Mezcla la información muchas veces usando operaciones muy simples (sumar, girar y XOR)|Es más rápido y seguro que RC4 ya que no contiene vulnerabilidades conocidas significativas.|
|**ChaCha20**|Se utiliza en protocolos modernos como TLS para reemplazar RC4.  Google lo utiliza en Chrome para conexiones HTTPS,  en aplicaciones VPN y en dispositivos móviles ya debido a su excelente rendimiento en hardware de bajo consumo.|Es el algoritmo de flujo más moderno, rápido y seguro. Es una evolución más segura de Salsa20, utiliza valores pseudoaleatorios y más mezclas.|
Tiene estas  ventajas e inconvenientes:

| Ventajas                                                                                                                                                                              | Inconvenientes                                                                                                                                                                                                                          |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **✅** **Rápido, eficiente e Ideal para datos en tránsito (streaming):** Es decir ideal para datos que se envían en tiempo real y o que no se sepa cuándo se va a dejar de transmitir. | **❌** **Propagación de errores:** Si hay un error en la transmisión y cambia algún bit, solo habrá un error en ese bit correspondiente en el texto plano.                                                                               |
|                                                                                                                                                                                       | **❌** **Menos seguro:** Si la clave de flujo es débil (o sea corta) o se repite para mensajes distintos, se podrá romper el cifrado de todos los datos.                                                                                 |
|                                                                                                                                                                                       | **❌** **Sincronicidad:** Si se pierde por el camino algun bit del texto cifrado, a la hora de descifrar, no se emparejarán los bits correctos con los del keystream, por tanto todo el texto siguiente estará desincronizado y erróneo. |

## 2.3. Cifrado simétrico en bloque

Este cifrado toma el mensaje y lo divide en trozos de un tamaño fijo (bloques). Y cifra cada bloque por separado: dispone sus bits en una matriz o dividirse en mitades (según el algortimo) y aplica una serie de operaciones como permutaciones, sustituciones, mezclas y desplazamientos, con la clave.

Al finalizar, todos los bloques cifrados se ensamblan para formar el mensaje cifrado completo. Por ejemplo, un archivo de 2048 bits puede dividirse en 32 bloques de 64 bits cada uno (64 × 32 = 2048).

> **La seguridad de estos algoritmos depende del tamaño de clave y de bloque**. Cuando mayor sea el bloque, mayor fuerza bruta se necesitará, ya que hay mayor combinación posible de bits. No es lo mismo un bloque de 8 bits (28) que de 128 bits (2128).

Los algoritmos de cifrado en bloque, del menos seguro al más seguro, incluyen: DES, 3DES, Blowfish, TwoFish y AES.


> [!example] Bloques independientes
> Cada bloque se cifra de forma independiente, sin influir en los siguientes:
> 
> | Algoritmo                     | Descripción                                                                                                                                                                                                                      |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ECB (Electronic Codebook)** | Es el modo más simple pero menos seguro, ya que los bloques idénticos en el texto original se cifran de la misma manera, generando patrones visibles. Esto facilita ataques, porque textos similares producen cifrados similares |
| **CTR (Counter Mode)**        | Usa un contador que cambia en cada bloque y se combina con la clave. Esto evita repeticiones de patrones, incluso si hay contenido repetido en el texto original, ya que cada bloque se cifra de forma única                     |
| **GCM**                       | Combina el modo de operación CTR con una función MAC, por tanto verifica la autenticidad del origen de los datos                                                                                                                 |

> [!example] Bloques encadenados
> Aquí, cada bloque cifrado afecta al siguiente, lo que elimina patrones y mejora la seguridad. Sin embargo, un error en un bloque puede propagarse a los siguientes:
> 
| Algoritmo                       | Descripción                                                                                                                                                                                                                                               |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CBC (Cipher Block Chaining)** | Cada bloque de texto se combina con el bloque cifrado anterior antes de ser cifrado, evitando que bloques idénticos. Existe una implementación de esto para generar MACs en lugar de cifrar el mensaje en el que se usa el útlimo bloque cifrado como MAC |
| **CFB (Cipher Feedback)**       | Utiliza un vector de inicialización (un valor aleatorio) y una parte del texto cifrado anterior para cifrar el siguiente bloque, proporcionando seguridad similar a CBC                                                                                   |
| **OFB(Output Feedback)**        | Convierte el cifrado por bloques en cifrado en flujo, generando un keystream independiente del texto original, lo que evita la propagación de errores                                                                                                     |

Por tanto contamos con esta serie de ventajas e inconvenientes:

| Ventajas                                                                                                                                                                        | Inconvenientes                                                                                                                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **✅** **Adecuado para datos en reposo:** Datos con un tamaño fijo, que segmenta y cifra. Ideal para archivos, documentos, y bases de datos almacenadas                          | **❌** **Propagación de errores:** Si ocurre un error en un bit a lo largo de la transmisión, esto afectará a que se corrompa el bloque entero a la hora de descifrarlo                                                       |
| **✅** **Más seguro** Debido a técnicas como encadenamiento (CBC). Por tanto, es ideal para aplicaciones que manejan datos muy sensibles, como información financiera o de salud | **❌** **Más lento:** Especialmente con archivos grandes. Ya que lee y almacena todos los datos, los segmenta en bloques y cifra cada bloque de forma individual. Por tanto, no se recomienda para datos en tránsito          |
|                                                                                                                                                                                 | ❌** Necesita padding (relleno):** El texto plano debe adaptarse al tamaño de los bloques (ejemplo múltiplo de 256 bits), por tanto no pueden quedar bits sueltos. En ese caso el bloque debe completarse con bits de relleno |
Por tanto tenemos estos algoritmos:

|**Algoritmo**|**Descripción**|**Seguridad**|
|---|---|---|
|**DES**|“Data encryption standard” fue uno de los primero algoritmos modernos. Divide el texto en bloques de 64 bits y aplica 16 rondas de mezclas y operaciones matemáticas|Inseguro, utiliza una clave de 56 bits, que hoy en día es muy corta y fácil de romper con fuerza bruta|
|**3DES**  <br>(des-ede3 en openssl)|3DES aplica el algoritmo DES tres veces sobre los datos para aumentar la seguridad, utilizando una clave efectiva de 112 bits|Sin embargo, sigue siendo más lento que los algoritmos modernos, y con ciertas vulnerabilidades en algunos protocolos.|
|**Blowfish**  <br>(bf en openssl)|Blowfish es un algoritmo rápido y flexible, más seguro que DES y 3DES. Funciona muy similar a DES|Algo inseguro: su tamaño de bloque de 64 bits lo hace vulnerable a ciertos ataques cuando se maneja una gran cantidad de datos. Tiene una versión mejorada llamada **twofish**|
|**AES**  <br>(aes-128 y aes-256)|“Advanced Encryption Standard” es el estándar actual para el cifrado simétrico. Se recomienda utilizar claves de al menos 128 bits, aunque para mayor seguridad se prefieren claves de 256 bits.|Es muy seguro, rápido y adoptado mundialmente, incluidas organizaciones gubernamentales como el gobierno de EE.UU.|
Así que el cifrado en bloque se configura usando un algoritmo y un modo de operación, por ejemplo:

| **DES con ECB** | **Blowfish con ECB**                       | **Blowfish con CBC**               | **AES con CTR**                                          | **AES con CBC**                             |
| --------------- | ------------------------------------------ | ---------------------------------- | -------------------------------------------------------- | ------------------------------------------- |
| Muy inseguro    | Moderadamente seguro, pero expone patrones | Algo más seguro, pero no demasiado | Muy seguro y eficiente, ideal para aplicaciones modernas | El más seguro y resistente, pero algo lento |


# 3. Cifrado asimétrico

# 3.1. Cifrado asimétrico

**El cifrado asimétrica cuenta de dos claves: una para cifrar (clave pública) y otra para descifrar (clave privada)** . **Claves diferentes = asimétrico.**

El proceso de cifrado es este:
1. El receptor genera un par de claves. Se queda la privada para él y le manda la pública al remitente
 | 
2. El remitente cifra el mensaje con la clave pública. Y le manda el mensaje cifrado al receptor
 | 
3. El receptor descifra el mensaje con su clave privada

Por tanto, si un atacante roba la clave (pública) en el camino, habrá robado la clave de cifrar (publica) y no la de descifrar (privada), por tanto, de poco le sirve. Para romper la seguridad tendría que acceder al sistema del receptor y robarle la privada.

> **Matemáticamente la clave pública deriva de la privada. Por tanto, si tienes la privada puedes generar la pública y romper todo el cifrado. Pero al revés no, por eso no pasa nada si tienes la pública.**

En algoritmos como RSA, estas claves se generan mediante multiplicar números primos muy grandes (la clave es que si no tienes la clave privada no sabes que números son esos), mientras que en ECC las claves se generan usando una fórmula matemática especial llamada “curva elíptica” que permite gran seguridad con claves más cortas y de manera más rápida.

Las ventajas e inconvenientes del cifrado asimétrico son estos:

| Ventajas                                                                                                                                                                                                                                                                         | Inconvenientes                                                                                                                                                                                                               |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **✅** **Seguro**: se puede distribuir libremente la clave pública en vez de preocuparse tanto en protegerla como pasaba con el simétrico. La que hay que proteger es la que no se comparte.                                                                                      | **❌** **Más lento:** requiere de más recursos computacionales que un sistema simétrico, lo que lo hace menos adecuado para cifrar grandes volúmenes de datos.                                                                |
| **✅** **Útil como mecanismo de Autenticación**: además de cifrar, la criptografía asimétrica se puede usar para firmar digitalmente documentos. Esto ayuda a verificar que el mensaje proviene del remitente legítimo y no de alguien que lo suplante (salvando su Autenticidad) | **❌** **Más complejo:** hay que generar ambas claves, distribuir la pública y almacenar de manera segura la privada. Lo que lo hace más complejo de gestionar.                                                               |

Por tanto, es muy útil cifrando:
- **Correo electrónico seguro:** con protocolos como PGP. Un usuario distribuye su clave publica y le pueden mandar mensajes cifrados con ella
- **Firmas digitales:** que verifican la autenticidad y la integridad de un mensaje o documento, ya que solo el remitente legítimo tiene la clave privada necesaria para generarlas
- **Acceso a redes privadas y sistemas:** muchas redes empresariales y sistemas seguros utilizan claves asimétricas para autenticar a los usuarios al conectarse, sin necesidad de compartir contraseñas

Digamos que queremos cifrar datos de manera masiva. Sabemos que el cifrado asimétrico es el más seguro, pero que es lento. Así que utilizaremos cifrado simétrico, que utiliza una sola clave, a la que llamaremos clave de sesión, porque se renueva vez que nos conectemos (es de un solo uso). El problema es que, si alguien roba esa clave, la seguridad se viene abajo.

Por ello hay dos modos para que ambos obtengan esa clave de manera protegida: RSA y Diffie Hellman.

## 3.2. Cifrado asimétrico RSA

**El concepto es sencillo: dos pares de claves (cifrado asimétrico).**

1. El servidor genera un par de claves RSA: una privada que guarda y una pública que envía al cliente
    
2. El cliente entonces generará una clave de sesión y cifrará una copia de esta con la pública para luego mandársela al servidor
    
3. Este la descifra con su clave privada y entonces ambos tendrán la misma clave de sesión con la que cifrarán los datos en cifrado simétrico.

> Este par de claves RSA tienen que ser muy grandes para ser seguro: se recomienda un mínimo de 2048 bits, aunque lo ideal es utilizar claves de 3072 bits o más.


## 3.3. Cifrado asimétrico DH

En este caso, cliente y servidor no comparte la clave de sesión en ningún momento sino “ingredientes” para hacerla:

1.  El emisor y el receptor generan un par de claves en base a una operación matemática llamada curva elíptica. También generan un cliente y server random. Se intercambian las claves públicas y estos randoms.
    
2. Cada uno combina su propia clave privada con la clave pública del otro para generar una **Pre-Master-Secret** mediante Diffie Hellman.
    
3. Con Pre-Master-Secret mas los randoms crean la "**Master-Secret**" mediante una función pseudoaleatoria (PRF)
    
4. A partir de la Master-Secret, ambos derivan varias claves simétricas que se usarán para cifrar y autenticar los mensajes. Entre ellas, las más importantes son **Client_Write_Key** y **Server_Write_Key.**
    
5. Verifican estas claves y si es correcto, cifran los mensajes por TLS.

> Por tanto, aunque un atacante intercepte la clave pública necesitaría la clave privada para calcular la clave de sesión.

Además, este método necesita claves más pequeñas que RSA y es más eficiente.


# 4. Ataques a cifrados

## 4.1 Clasificación de ataques

Los ataques a un criptosistema se clasifican según la información que el atacante debe conocer para romperlo. **El objetivo siempre es el mismo**, **obtener la clave de cifrado**. Estos son:

| Ataque                          | Descripción                                                                                                                                                                       | Ejemplo                                                                                                                                                                                       |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CO - Ciphertext-Only            | El atacante conoce el algoritmo de cifrado (normalmente público) y solo dispone de textos cifrados.                                                                               | El cifrado César                                                                                                                                                                              |
| KPA - Known-Plaintext-attacks   | El atacante conoce el algoritmo de cifrado y además dispone de algunos pares texto plano–texto cifrado para intentar deducir la clave o el método de descifrado.                  | Un ejemplo es con un cifrado XOR: si se tiene el texto original y el texto cifrado, se puede obtener la clave aplicando XOR.                                                                  |
| CPA - Chosen-Plaintext-attacks  | El atacante tiene acceso a un servicio de cifrado y puede elegir qué texto cifrar y ver el resultado cifrado, sin necesidad de interceptar comunicaciones.                        | Un ejemplo es una aplicación web que cifra automáticamente cookies de sesión, por ejemplo `user:nombre_de_usuario`; el atacante puede enviar valores escogidos para ver cómo se cifran.       |
| CCA - Chosen-Ciphertext-attacks | El atacante, además de tener el texto cifrado, tiene acceso a un sistema de descifrado y puede enviar textos cifrados (legítimos o modificados) para ver su resultado descifrado. | Un ejemplo son los ataques de “padding oracle”, en los que se modifica ligeramente el texto cifrado y se usan los mensajes de error para inferir información sobre la clave o el texto plano. |
Hay varias maneras de romper un cifrado, que recordemos era o infiriendo la clave de descifrado o encontrando alguna vulnerabilidad en los algoritmos como repeticiones o patrones.

ESPACIO DE CLAVES  
El espacio de claves es el número total de combinaciones posibles para una contraseña dada una longitud y un conjunto de caracteres.

En el Cifrado César, el espacio de claves es de "27" posibilidades, que son todos los desplazamientos posibles del alfabeto. En este caso da igual la longitud del texto cifrado.

Esto es una buena medida para evaluar si una contraseña es robusta o no. Esto sigue esta función matemática:
```c
                           número de caracteres posibles^longitud de la contraseña = Espacio de claves
5 dígitos alfanuméricos -> 62^5                                                    = 916.132.832 combinaciones
```

## 4.2 Clasificación de ataques

Es un método de ataque que consiste en probar claves candidatas de forma sistemática hasta encontrar la correcta. Suele aplicarse en escenarios de texto cifrado conocido (CO).

Puede haber dos modalidades:

| Ataque                | Descripción                                                                                                                                                                                                                                                |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ataque de diccionario | Se prueba con una lista de claves predefinidas probables (un diccionario). Es más rápido, pero solo funciona si la clave real está incluida en esa lista. _Ej: pass123, abc123, 000000..._                                                                 |
| Fuerza bruta pura     | Se prueban todas las combinaciones posibles de caracteres (más lento y exhaustivo). Es más costoso, pero cubre todo el espacio de claves; aunque solo funciona si recorrer ese espacio lleva un tiempo o recursos viables. _Ej: 000000, 000001, 000002..._ |
También dos variantes según el tipo de dato:
- **Sobre texto cifrado:** el atacante descifra el texto cifrado con cada clave candidata hasta obtener un resultado que tenga sentido o un formato esperado. _Por ejemplo, el texto resultante sea "Hola" en lugar de b'3_1_
- **Sobre hashes de contraseñas:** cuando solo se tiene un hash y no un texto cifrado, no se puede descifrar directamente. En su lugar se calcula el hash de cada candidato y se compara bit por bit con el hash original hasta encontrar coincidencia.

