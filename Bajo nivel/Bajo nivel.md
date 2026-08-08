
# 1. El lenguaje C y los binarios

C es un lenguaje compilado de bajo nivel, cercano al hardware, que sirve de base a gran parte del software moderno:

|Categoría|Ejemplos|
|---|---|
|Sistemas operativos|UNIX, Linux, Windows (componentes base), macOS|
|Lenguajes de programación|Intérpretes de Python, Perl, Ruby, PHP|
|Bases de datos|MySQL, PostgreSQL|
|Software de alto rendimiento|Chrome, Firefox, motores gráficos|
|Dispositivos embebidos|Microcontroladores, hardware especializado|

Un programa sencillo en C:
```c
#include <stdio.h>    // biblioteca de entrada/salida estándar
#include <string.h>   // biblioteca de manipulación de cadenas

int main(void) {
    char nombre[32];                       // buffer para una cadena de máximo 32 bytes
    printf("Nombre: ");
    fgets(nombre, sizeof(nombre), stdin);  // lee una línea desde stdin
    return 0;                              // indica que el programa terminó con éxito
}
```

A diferencia de Python, C requiere:

- **Tipado explícito**: hay que declarar el tipo de cada variable (`char`, `int`, `float`...)
- **Gestión manual de memoria**: se indica el tamaño de los buffers y cuántos bytes se leen
- **Compilación**: el código fuente no se ejecuta directamente, hay que traducirlo a lenguaje máquina


---------
## 1.2. Compilación en Linux

El compilador de Windows es gcc
```bash
gcc programa.c -o programa          # compilación estándar (enlazado dinámico)
gcc programa.c -o programa -static  # enlazado estático
```

---------
#### Proceso de compilación
El proceso de compilación tiene tres etapas:
```
programa.c   ──── Compilación ────▶   programa.o   ──── Enlazado ────▶   programa
(código C)                            (código obj,                        (ejecutable ELF)
                                       sin funciones                      listo para ejecutar)
                                       de sistema)
```

1. **Compilación**: genera un archivo objeto (`.o`) con código máquina pero con huecos para las funciones de las bibliotecas del sistema (como `printf`). El compilador deja marcadores del tipo: _"esta función se buscará en el enlazado"_.
2. **Enlazado**: combina el archivo objeto con las bibliotecas del sistema (como `glibc`) para crear el ejecutable final.
3. **Ejecución**: el dynamic loader (`/lib64/ld-linux-x86-64.so.2`) carga el ejecutable y sus bibliotecas dinámicas en memoria.

---------
#### Tipos de enlazado
Existen dos tipos:
- **Dinámico**: Las funciones de las bibliotecas no se incluyen en el binario. Se cargan en tiempo de ejecución desde los `.so` del sistema. Eso hace un ejecutable ligero, pero dependiente del entorno.
- **Estático**: Copia las funciones de las bibliotecas directamente en el binario. Es independiente del entorno pero mucho mas grande.

---------
#### glibc — la biblioteca estándar de C en Linux
La biblioteca `glibc`  implementa las funciones declaradas en las cabeceras estándar de C:

> La implementación real del código está en `libc.so.6`. Cada nueva versión de glibc incluye optimizaciones y mejoras de seguridad (protecciones contra buffer overflows, entre otras).

```bash
ldd /bin/ls
#   linux-vdso.so.1 =>  (0x00007fff03bc7000)
#   libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1
#   libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6   ← glibc

/lib/x86_64-linux-gnu/libc.so.6
# versión exacta de glibc
```

---------
#### Cómo busca el ejecutable sus bibliotecas dinámicas
En tiempo de ejecución, el sistema busca las bibliotecas `.so` en este orden de prioridad:

1. `LD_PRELOAD` — carga esta biblioteca **antes** que cualquier otra (muy usado en exploits y en análisis)
2. `LD_LIBRARY_PATH` / `LD_RUN_PATH` — rutas definidas en variables de entorno
3. `rpath` — ruta grabada en el propio ejecutable durante la compilación (`-rpath`)
4. `/usr/lib`, `/lib` y otros directorios estándar del sistema
5. Rutas definidas en `/etc/ld.so.conf`

---------
## 1.3. Compilación en Windows
El equivalente Windows usa el compilador `cl.exe` de Microsoft, que genera binarios en formato **PE (Portable Executable)** o **DLL**. Las bibliotecas estándar de C en Windows son `msvcrt.dll` y variantes.

> [!NOTE]
> **Para programar en C con Visual Studio**: 
> 1. crear proyecto `Console App` 
> 2. Eliminar el archivo `.cpp` y añadir un nuevo archivo con extensión `.c` 
> 3. En `Project Properties` : `C/C++ 🡆 Advanced 🡆 Compile As 🡆 compile as C Code`.
> 
>> Para ejecutar el programa que hemos creado hacemos `Ctrol+F5`

---------
# 2. Las secciones de un binario
Un archivo no es una secuencia caótica de bytes. Tiene una **estructura interna** que permite al sistema operativo interpretarlo correctamente.

Todos los archivos se descomponen en estas partes fundamentales:

| Componente      | Descripción                                                                                                                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Magic bytes** | Primeros bytes del archivo que identifican su tipo, independientemente de la extensión. Un ELF comienza con `0x7F 45 4C 46` ("ELF"). Una imagen JPEG con `FF D8 FF`. El comando `file` usa estos bytes. |
| **Cabeceras**   | Metadatos sobre el archivo: tipo, arquitectura, dirección de inicio, tamaño de secciones...                                                                                                             |
| **Datos**       | El contenido real: código, variables, recursos, constantes...                                                                                                                                           |

> [!TIP]
> El proyecto [Corkami (Ange Albertini)](https://github.com/corkami) disecciona visualmente el formato interno de múltiples tipos de archivo (JPEG, PE, ELF...) con diagramas detallados.

---------
## 2.1. Secciones de un ejecutable
Los archivos exe se componen de estas partes:

| Sección                  | Contenido                                                           | Ejemplo                                |
| ------------------------ | ------------------------------------------------------------------- | -------------------------------------- |
| `.text`                  | Código del programa en lenguaje máquina                             | `MOV ebx, 5`                           |
| `.data`                  | Variables globales inicializadas                                    | `int x = 5;`                           |
| `.rodata` /<br>`.rdata`  | Datos de solo lectura: strings, constantes                          | `"Hola mundo"`🡆 <br>`const int y = 6` |
| `.bss`                   | Variables globales no inicializadas (valen 0 al arrancar)           | `int x;` 🡆 `x = 0`                    |
| `.idata`                 | Tabla de importaciones: bibliotecas y funciones que usa el programa | `stdio → printf`                       |
| `.rsrc`                  | Recursos: iconos, imágenes, audio de la interfaz gráfica            | `icon.png`                             |
| `.symtab` /<br>`.strtab` | Tabla de símbolos: nombres de funciones y variables globales        | `printf`, `main`...                    |

Además, hay secciones para gestionar la carga dinámica: `.dynamic`, `.got` (Global Offset Table), `.plt` (Procedure Linkage Table).

> [!TIP]
> El creador "corkami" tiene esta interesante [esta sección del exe](https://raw.githubusercontent.com/corkami/pics/master/binary/pe101/pe101es.png) dónde podemos comprobar su compleja estructura.

----------
## 2.2. Memoria virtual

Cuando el kernel ejecuta un programa, le asigna su propio espacio de memoria aislado: la **memoria virtual**. Cada proceso "cree" que está solo y tiene toda la memoria para sí. Esto se implementa mediante **tablas de paginación**, que traducen direcciones virtuales a direcciones físicas en la RAM.

Al cargar un ejecutable, no se carga solo: se cargan también las bibliotecas dinámicas, las secciones de código, las zonas de datos, etc. El sistema mantiene una tabla (**VAD — Virtual Address Descriptors**) que indica dónde empieza cada parte. *Como si fuera el índice de un libro que marca en qué página comienza cada capítulo*

| Tipo                               | Descripción                                                                                                                    | Ejemplo                                             |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------- |
| **Base Address**                   | Dirección donde empieza el ejecutable en memoria virtual. Los primeros `0x400000` suelen estar reservados para el sistema.<br> | `0x400000`                                          |
| **RVA** (Relative Virtual Address) | Desplazamiento de una sección respecto a la Base Address                                                                       | `.text` en `RVA 0x1000` 🡆 4096 bytes desde la base |
| **VA** (Virtual Address)           | Dirección absoluta en memoria 🡆 `Base Address + RVA`                                                                          | `0x400000 + 0x1000 = 0x401000`                      |
```
  Base Address   RVA          VA
  0x400000    +  0x1000   =   0x401000   ← inicio de .text
  0x400000    +  0x2000   =   0x402000   ← inicio de .data
  0x400000    +  0x3000   =   0x403000   ← inicio de .rodata
```

------
## 2.3. Análisis estático de un binario

**ELF (Executable and Linkable Format)** es el formato estándar de ejecutables en Linux. Es el resultado de compilar código C, C++, Rust, etc.

| Acción                                                 | Comando                                     |
| ------------------------------------------------------ | ------------------------------------------- |
| Tipo de archivo                                        | `file binario`                              |
| Ver arquitectura y protecciones de seguridad           | `objdump -f binario`                        |
| Listar cadenas imprimibles.                            | `strings binario`                           |
| Ver llamadas a funciones durante la ejecución          | `ltrace binario`<br>                        |
| Ver las cabeceras                                      | `readelf -h binario`                        |
| Ver dónde empieza cada sección                         | `readelf -S binario` / `objdump -h binario` |
| Ver los nombres de las funciones                       | `objdump -t binario \| grep -vE "_\|@"`     |
| Ver las funciones de las librerías (Ej `printf@glibc`) | `objdump -t binario \| grep "@"`            |
| Ver el volcado en bytes del programa                   | `objdump -s binario`                        |
| Ver el volcado en bytes de una sección                 | `objdump -s -j .rodata binario`             |
| Ver las dependencias                                   | `readelf -d binario` / `ldd binario`        |

> [!NOTE]
> En la salida de `objdump -h`, las flags `ALLOC` indican que la sección se carga en memoria, `READONLY` que no se puede escribir y `CODE` que contiene código ejecutable.


----------
# 3. Computación

## 3.1. Bits, bytes y representación numérica

Un bit tiene dos estados posibles (0 o 1). Un byte son 8 bits, lo que da `2⁸ = 256` combinaciones posibles (valores del 0 al 255).

Estos valores se representan en **hexadecimal** (base 16), donde cada dígito puede tomar 16 valores: `0 1 2 3 4 5 6 7 8 9 a b c d e f`. Dos dígitos hexadecimales representan un byte completo (`0xFF = 255`).

**Cada posición binaria tiene un peso que es una potencia de 2:**

| 2⁸  | 2⁷  | 2⁶  | 2⁵  | 2⁴  | 2³  | 2²  | 2¹  | 2⁰  |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 256 | 128 | 64  | 32  | 16  | 8   | 4   | 2   | 1   |

Ejemplo: `0b01101101` = `64 + 32 + 8 + 4 + 1` = `109` decimal = `0x6D` hexadecimal

Existen además estos tipos de números

| Tipo             | Rango (8 bits) | Representación                                                                    |
| ---------------- | -------------- | --------------------------------------------------------------------------------- |
| **Unsigned int** | 0 → 255        | El bit más significativo es parte del valor: `5 → 00000101`                       |
| **Signed int**   | -128 → 127     | Negativos en complemento a dos: invertir bits y sumar 1. `-5 → 11111011`          |
| **Float**        | —              | Estándar IEEE 754: bit de signo + exponente (potencia de 2) + mantisa (decimales) |

----------
## 3.2. Arquitecturas

La arquitectura define el ancho del bus de datos del procesador — cuántos bits puede manejar en cada operación.

|Arquitectura|Valor máximo sin signo|Rango con signo|Dispositivos representativos|
|---|---|---|---|
|**8 bits**|255|-128 → 127|Game Boy, NES, Arduino|
|**16 bits**|65.535|-32.768 → 32.767|SNES, Sega Mega Drive, Intel 8086|
|**32 bits (x86)**|4.294.967.295|~±2.147 millones|PlayStation, Windows 95–XP, ARMv7|
|**64 bits (x64)**|~1,8 × 10¹⁹|~±9,2 × 10¹⁸|PCs modernos, PS4/5, Linux, macOS actuales|

Con 64 bits, los límites de memoria son prácticamente inabarcables. Casi todo el software actual se compila para x64.

---
## 3.3. Registros y punteros

Los **registros** son pequeñas celdas de memoria dentro del propio procesador, ultrarrápidas. Todas las operaciones del procesador se realizan sobre los valores almacenados en ellos.

> [!NOTE]
> Las operaciones en lenguaje ensamblador son instrucciones simples que manipulan los valores de estos registros: sumar, restar, comparar números, entre otras. Cada una de estas operaciones se ejecuta en uno o varios ciclos de reloj, según su complejidad. 
> *Por ejemplo, una suma suele completarse en un solo ciclo, mientras que una multiplicación requiere varios, ya que, en esencia, consiste en una serie de sumas consecutivas.*
> 
> El reloj del procesador actúa como un metrónomo que marca el ritmo de estos ciclos, con un “tic-tac” constante que determina la velocidad a la que se ejecutan las instrucciones. Su frecuencia se mide en hercios (Hz), que indican cuántos ciclos puede realizar el procesador por segundo.

-----------
#### Registros principales

| Nombre              | x86                    | x64                    | Función                                                                       |
| ------------------- | ---------------------- | ---------------------- | ----------------------------------------------------------------------------- |
| Instruction Pointer | `EIP`                  | `RIP`                  | Dirección de la próxima instrucción a ejecutar                                |
| Stack Pointer       | `ESP`                  | `RSP`                  | Indica donde empieza la pila; cuando el frame está vacío, coincide con EBP    |
| Base Pointer        | `EBP`                  | `RBP`                  | Marca la base del stack frame actual                                          |
| Acumulador          | `EAX`                  | `RAX`                  | Almacena valores para operar. También guarda el valor de retorno de funciones |
| Contador            | `ECX`                  | `RCX`                  | Contador para bucles                                                          |
| Propósito general   | `EBX`, `ESI`, `EDI`... | `RBX`, `RSI`, `RDI`... | Usados para almacenar datos temporales o direcciones                          |
| Flags               | `EFLAGS`               | `RFLAGS`               | Estado de la última operación (zero flag, carry flag...)                      |

------
#### Subdivisiones de un registro — LSB
Si dividimos un registro de una arquitectura en dos mitades, obtenemos el registro equivalente en laarquitectura anterior: Un registro de 64 bits contiene al de 32, que contiene al de 16, que contiene al de 8

Siempre se toma la mitad derecha, los bits de menor peso o `LSB`  (bits menos significativos).  Esa parte es la que se utiliza con más frecuencia en las operaciones, ya que muchas veces solo interesa modificar una parte del registro y no su contenido completo.
```
RAX  (64 bits):  0x 01 23 45 67 89 ab cd ef
EAX  (32 bits):              0x 89 ab cd ef
AX   (16 bits):                    0x cd ef
AH    (8 bits):                    0x cd
AL    (8 bits):                       0x ef
```

------
#### Punteros
Un puntero es una dirección de memoria. Los corchetes `[]` indican que se opera con el **contenido** de esa dirección, no con la dirección en sí:

```bash
MOV [0x1000], 5    # guarda el valor 5 en la dirección 0x1000
MOV EBX, 0x1000    # guarda la dirección 0x1000 en EBX
MOV AL, [EBX]      # guarda en AL el valor que hay en la dirección apuntada por EBX → AL = 5

MOV EAX, [EBP - 4] # guarda en EAX el valor que está 4 bytes antes de EBP (variable local)
```

----------
#### Flags (EFLAGS)
El registro de flags agrupa bits que indican el resultado de la última operación:

| Flag | Nombre        | Significado                                                                            |
| ---- | ------------- | -------------------------------------------------------------------------------------- |
| `ZF` | Zero Flag     | 1 si el resultado de una comparación fue cero (los dos valores comparados son iguales) |
| `CF` | Carry Flag    | 1 si hubo desbordamiento en operaciones sin signo                                      |
| `OF` | Overflow Flag | 1 si hubo desbordamiento en operaciones con signo                                      |
| `SF` | Sign Flag     | 1 si el resultado es negativo                                                          |

```bash
CMP AX, BX    # compara AX con BX → si son iguales, ZF = 1
JNZ muy_mal   # salta a "muy_mal" si ZF = 0 (son distintos)
              # si son iguales (contraseña correcta), el programa continúa
```


---
# 4. La memoria

## 4.1. Stack (pila)
La pila es la zona de memoria que gestiona la ejecución de funciones. Sigue el orden **LIFO** (Last In, First Out). 

Cada vez que se llama a una función, se crea un **stack frame** que contiene sus parámetros, sus variables locales y la dirección de retorno.

```
Direcciones mas bajas
▲  0xdd00                  <--- [Stack frame de funcion1]
│  ┌─── ESP (tope actual) ─────┐ 
│  │  var A = "Hola Mundo"     │
│  │  var B = "123"            │
│  │  EBP guardado             │ ← base del frame anterior
│  │  EIP dirección de retorno │ ← a dónde volver cuando funcion1 termine
│  └── EBP del frame actual ───┘ 
│  0xdd30                  <--- [Stack frame de Main]
│
▼  0xdd70                  <--- [Stack frame de Glibc] 
Direcciones mas altas
```

> [!CAUTION]
> **Buffer Overflow**: Si se escriben más datos de los que cabe en una variable del stack, el exceso sobrescribe la memoria adyacente:
> 
> - Se sobrescriben otras variables locales
> - Se sobrescribe la **dirección de retorno** (EIP/RIP): si se rellena con basura, el programa intenta saltar a una dirección inválida → `segfault`. Si se sobrescribe con una dirección controlada por el atacante, este decide qué se ejecuta a continuación.

*Para entenderlo imaginemos la pila como una caja abierta en la que se meten platos. Primero tenemos la caja de "main" y como se llama a "funcion1", se trae una caja nueva dónde se apilarán los platos de esa función. Cuando esta termina, se quitarán los platos uno a uno desde arriba y luego la caja (para que no se caiga), volviendo a dónde estábamos, los platos de "main".*

*En el BOF; Digamos que en nuestra torre echamos demasiada comida en el plato de arriba hasta que desborde y la comida por la gravedad caiga a los platos inferiores, llenándolos y ensuciándolos.*

-------
## 4.2. Heap
El heap es una zona de memoria más grande y flexible, usada para:

- Datos cuyo tamaño no se conoce en tiempo de compilación
- Datos que deben sobrevivir más allá del ámbito de una función
- Estructuras grandes

```c
int n;
scanf("%d", &n);
int *array = malloc(n * sizeof(int));  // reserva memoria en el heap en tiempo de ejecución
free(array);                           // hay que liberarla manualmente
```

> [!WARNING]
Los bloques del heap (**chunks**) tienen un encabezado con su tamaño y un cuerpo con los datos. El mal uso del heap da lugar a:
> - **Memory leak**: No llamar a `free()` cuando ya no se necesita un bloque. El heap se llena de memoria inútil
> - **Fragmentación** : Los chunks se asignan donde haya hueco. Con el tiempo quedan pequeños huecos entre bloques que no pueden aprovecharse
> - **Heap overflow** : Escribir más datos de los que caben en un chunk, sobrescribiendo datos de chunks vecinos


---
## 4.3. Almacenamiento en memoria y endianness

Los valores numéricos y caracteres se representan en hexadecimal:
- Enteros: `12` decimal → `0x0C`
- Caracteres (ASCII): `'a'` → `0x61`, `'A'` → `0x41`
- Caracteres extendidos: UNICODE (múltiples bytes por carácter)

Los bytes de un valor multibyte no siempre se almacenan en el mismo orden:

| Tipo              | Descripción                                    | Valor `0x12345678` en memoria | Uso                                                                             |
| ----------------- | ---------------------------------------------- | ----------------------------- | ------------------------------------------------------------------------------- |
| **Big Endian**    | Byte más significativo primero (orden natural) | `12 34 56 78`                 | Se usa en protocolos de red (por eso se llama _network byte order_)             |
| **Little Endian** | Byte menos significativo (LSB) primero         | `78 56 34 12`                 | Es el estándar en CPUs x86/x64 modernas, es mas óptimo porque opera con los LSB |

Ejemplo: Convertir una palabra a su representación hexadecimal en little endian
```bash
echo -n "casa" | xxd -p | sed 's/../& /g' | tr ' ' '\n' | tac | tr '\n' ' '
# xxd -p: traduce a hexadecimal plano
# sed 's/../& /g': sustituye todas las ocurrencias de dos caracteres seguidos ".." por lo que había (&) seguido de un espacio " "
# tac -s " " : toma todos los grupos delimitados por el espacio " " y los invierte
```

---
## 4.4. Operaciones en ensamblador
Repetimos que, cada operación que puede realizar el procesador, la realiza sobre el valor que se almacene en los registros de la CPU. Estas son pequeñas celdas de memoria dentro del procesador, capaces de almacenar valores de 4 u 8 bytes (dependiendo de la [arquitectura](https://www.campus.learn4hack.com/mod/page/view.php?id=709 "Arquitectura")). En ensamblador, se manipulan estos registros para realizar operaciones como sumar, restar o mover datos entre ellos.

Hay que indicar que hay dos "idiomas" de ensamblador, uno es Intel y el otro es arm. En este caso enseñaré **Intel**, ya que es el más común, el estandar.

#### Movimiento de datos

| Instrucción              | Descripción                                                                   | Ejemplo                                      |
| ------------------------ | ----------------------------------------------------------------------------- | -------------------------------------------- |
| `MOV <destino>,<origen>` | Copia un valor de un lugar a otro                                             | `MOV EAX, [EBP - 2]`<br>`EBP-2 = 5; EAX = 5` |
| `PUSH <valor>`           | Guarda un valor en lo alto de la pila de la función actual                    | `PUSH EAX`                                   |
| `POP <destino>`          | Recupera un valor de la pila y lo guarda en el destino. <br>La pila desciende | `POP EAX`                                    |

#### Artimética

| Instrucción           | Descripción                                                                        | Ejemplo                                                                  |
| --------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| `ADD A, B / SUB A, B` | Suma/resta al primer valor el segundo                                              | `MOV EAX, 3; ADD EAX, 5 → EAX = 8`<br>`MOV EAX, 6; SUB EAX, 2 → EAX = 4` |
| `INC A / DEC A`       | Incrementa/decrementa el valor en 1                                                | `MOV EAX, 3; INC EAX → EAX += 4`<br>`MOV EAX, 3; DEC EAX → EAX -= 2`     |
| `IMUL A, B`           | Multiplica el primer valor por el segundo<br>y almacena el resultado en el primero | `MOV EAX, 3; IMUL EAX, 3 → EAX = 9`                                      |

#### Control de flujo

| Instrucción | Descripción                                                                                                                               | Ejemplo                            |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| `CMP A,B`   | Compara los dos valores completos. Si son iguales la zero flag (ZF) será “1”, si son distintos “0”                                        | `MOV EAX, 6; CMP EAX, 10 → ZF = 0` |
| `TEST A, B` | Comprueba si los dos valores son iguales bit a bit, de ser así la ZF será 1.<br>Si se compara un valor consigo mismo se forzará la ZF a 1 | `TEST EAX,EAX  → ZF = 1`           |

 Tras cada comparación, el resultado afectará a la Zero flag. A partir de esta, existen los saltos condicionales a la dirección destino, que suele representarse como "programa.Instrucción".
 
  Ej `JUMP programa.4010e5`:
 - `JMP`: salta si o sí a la instrucción indicada
 - `JE / JNE`: salta si los valores anteriores son iguales (ZF = 1) o no (ZF = 0)
 - `JZ / JNZ`: salta si el valor anterior es zero o no
 - `JG / JL`: Salta si el segundo valor es mayor / menor que el primero

> [!WARNING]
> Cuando crakea un programa, se crea una versión alternativa de este, un parche, en el que se han modificado instrucciones de salto para evitar que se verifiquen licencias o contraseñas, volviendo gratuito un programa de pago (Ej: un videojuego). Los atacantes, además suelen incluir además instrucciones maliciosas que comprometan a la víctima, actuando como un caballo de troya.

#### Operaciones con bits
En estas operaciones se comparan dos resultados bit a bit. Se emparejan el primer bit de uno con el primero del otro, el segundo con el segundo… así con todos. Si no hay los mismos bits, el más pequeño se rellena con 0s a la izquierda.

| Assembly   |                                                    | Contra si mismo                                  |
| ---------- | -------------------------------------------------- | ------------------------------------------------ |
| `AND A, B` | AND lógico, dónde sale 1 si los bits son ambos 1   |                                                  |
| `OR A, B`  | OR lógico, dónde solo sale 0 si ambos bits son 0   | `OR A,A` El valor queda intacto                  |
| `XOR A, B` | XOR lógico, dónde sale 1 si los bits son distintos | `XOR A,A` El valor se convierte en Zero → ZF = 1 |
| `SHL N`    | Desplaza bits a la izquierda N veces               |                                                  |
| `SHR N`    | Desplaza bits a la derecha N veces                 |                                                  |

#### Otros

| Operación    | Acción                 |
| ------------ | ---------------------- |
| `NOP`   | No hace nada           |
| `RET`  | Retorna de una función |
| `JUMP`  | Llama a una función.   |
Cuando se llama a una función con `JUMP`:
 - **Argumentos**: valores en la pila (32 bits) o en ciertos registros (64 bits). En Windows son (RCX, RDX, R8, R9), y en Linux (RDI, RSI, RDX, RCX, R8, R9)
 - **Valor de retorno** (**resultado**): se almacena en EAX / RAX
