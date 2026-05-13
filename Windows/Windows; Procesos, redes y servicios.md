# Índice

[[Windows; Procesos, redes y servicios#1. 🔄 Procesos]]
- [[Windows; Procesos, redes y servicios#1.1. Jerarquía de procesos]]
- [[Windows; Procesos, redes y servicios#1.2.🧵 Hilos (Threads)]]
- [[Windows; Procesos, redes y servicios#1.3. 🎟️ Tokens de acceso]]
- [[Windows; Procesos, redes y servicios#1.4. Comunicación entre procesos]]
- [[Windows; Procesos, redes y servicios#1.5. Integrity Levels (IL)]]

[[Windows; Procesos, redes y servicios#2. 🔧 Servicios]]
- [[Windows; Procesos, redes y servicios#2.1. Cuentas de servicio]]
- [[Windows; Procesos, redes y servicios#2.2. 🔐 Permisos de servicios (DACL de servicio)]]
- [[Windows; Procesos, redes y servicios#2.3. 🗺️ Visión general]]

---
> [!abstract] Introducción
> Un **proceso** es un programa en ejecución con sus propios recursos aislados. Cada proceso contiene uno o más **hilos** que ejecutan el código. Los **servicios** son procesos especiales gestionados por Windows que corren en segundo plano. Todo proceso lleva asociado un **Access Token** que determina qué puede hacer en el sistema.

# 1. 🔄 Procesos

En informática, un proceso es un programa en ejecución, junto a su contexto (memoria, archivos abiertos, permisos, etc.). Aunque un mismo programa puede ejecutarse varias veces al mismo tiempo, cada ejecución es una instancia independiente con su propio estado y recursos.

**Cada proceso se identifica mediante un PID** (identificador de proceso) **y se ejecuta bajo un usuario**, lo que significa que hereda sus permisos y privilegios (security context)

Además cuenta con estos atributos

| Atributo       | Descripción                                                                                                 |
| -------------- | ----------------------------------------------------------------------------------------------------------- |
| **PID**        | Process ID — identificador único numérico                                                                   |
| **PPID**       | Parent PID — PID del proceso que lo creó                                                                    |
| **Nombre**     | Nombre del ejecutable (`notepad.exe`)                                                                       |
| **Ruta**       | Ubicación completa del ejecutable en disco                                                                  |
| **Usuario**    | Cuenta bajo la que corre el proceso                                                                         |
| **Token**      | Access Token con identidad y privilegios, pertenece al usuario o cuenta de servicio que ejecuta el proceso. |
| **Prioridad**  | Cuánto tiempo de CPU recibe (Idle → Realtime)                                                               |
| **Handles**    | Referencias abiertas a recursos del kernel                                                                  |
| **Módulos**    | DLLs cargadas en el espacio de memoria                                                                      |
| **Memoria**    | Working Set (RAM en uso) y memoria virtual (espacio de memoria aislada y exclusiva del proceso)             |
| **Integridad** | Nivel IL (Low / Medium / High / System)                                                                     |
## 1.1. Jerarquía de procesos

En el sistema, un proceso puede generar otros procesos, llamados “procesos hijo”. Cada proceso tiene un PID único, pero también un PPID (identificador de proceso padre), que indica quién lo creó.  El propio sistema operativo es un proceso base, y de él derivan otros procesos, que a su vez pueden generar más procesos. 

Windows mantiene un árbol de procesos padre-hijo, que muestra la jerarquía hasta llegar al proceso del sistema. Si el padre muere, los hijos quedan huérfanos (los adopta `System` o siguen corriendo).

| **Proceso**      | **Descripción**                                            |
| ---------------- | ---------------------------------------------------------- |
| **smss.exe**     | Session Manager, inicializa y gestiona las sesiones        |
| **system**       | Proceso del kernel del que derivan el resto de procesos    |
| **lsass.exe**    | Gestiona la autenticación y seguridad                      |
| **explorer.exe** | Escritorio y buscador de archivos                          |
| **dwm.exe**      | Renderizado gráfico de ventanas                            |
| **wininit.exe**  | Inicia servicios críticos en el arranque                   |
| **winlogon.exe** | Maneja el inicio y cierre de sesión y la pantalla de login |
| **services.exe** | Administra los servicios del sistema                       |
| **csrss.exe**    | Client/Server Runtime                                      |
| **svchost.exe**  | Agrupa varios servicios                                    |
| **spoolsv.exe**  | Gestión de impresoras                                      |
> [!tip] ¿Por qué hay tantos `svchost.exe`? 
> `svchost.exe` es un proceso genérico que aloja múltiples servicios de Windows dentro del mismo proceso. Cada instancia puede alojar uno o varios servicios. Es completamente normal ver 10-20 instancias. Se puede ver qué servicios aloja cada uno con `tasklist /SVC`.
****
> [!warning] Procesos huérfanos y anomalías 
> En análisis forense y detección de malware, la jerarquía padre-hijo es clave. Algunos indicadores sospechosos:
> - `cmd.exe` con padre `word.exe` → posible macro maliciosa
> - `lsass.exe` con padre distinto de `wininit.exe`
> - `svchost.exe` corriendo fuera de `C:\Windows\System32\`
> - Proceso con nombre legítimo pero ruta incorrecta (masquerading)

---
## 1.2.🧵 Hilos (Threads)

Cada hilo ejecuta una sola acción u operación dentro de un proceso, siendo la unidad mínima de ejecución. **El procesador ejecuta hilos, no procesos.**
- La memoria virtual del proceso principal es compartida por todos sus hilos, esto permite que se comuniquen entre sí compartiendo datos.
- Al acceder a la memoria de su proceso, tambien accede a los mismos recursos

El hilo consta de estos atributos:

| Atributo                   | Descripción                                                           |
| -------------------------- | --------------------------------------------------------------------- |
| **TID**                    | Thread ID — identificador único                                       |
| **Estado**                 | Running / Waiting / Suspended / Terminated                            |
| **Prioridad**              | Heredada del proceso, pero ajustable                                  |
| **Stack**                  | Cada hilo tiene su propio stack de llamadas                           |
| **Token de impersonación** | Un hilo puede tener un token diferente al del proceso (impersonation) |
> [!info] Impersonation a nivel de hilo 
> Un hilo puede adoptar temporalmente la identidad de otro usuario sin que el resto del proceso lo haga. Esto se usa en servidores: el proceso corre como SYSTEM pero cada hilo atiende a un usuario concreto usando su token. Es también la base del abuso de `SeImpersonatePrivilege`.

---
## 1.3. 🎟️  Tokens de acceso 

El Access Token es el objeto del kernel que define **quién es** un proceso y **qué puede hacer**. El token representa la cuenta que ejecuta cada proceso y su security context, por tanto es una pieza clave en la **autorización**. Por cada recurso, se compara el token y el sistema decide si tiene acceso o no en base a las ACLs definidas.

 - Se crea cuando el usuario inicia sesión y se hereda por todos sus procesos.
 - Muchos procesos del sistema los ejecuta la cuenta de sistema que representa al kernel, por tanto corren con un token muy valioso, el de `NT Authority/System`

Por tanto el token se compone de:

| Categoría       | Atributos                                                                                                       |
| --------------- | --------------------------------------------------------------------------------------------------------------- |
| **Identidad**   | SID, nombre de usuario y un LUID o identificador de sesión                                                      |
| **Grupos**      | A los que pertenece el usuario y de los que hereda sus privilegios                                              |
| **Privilegios** | Heredados de los grupos, indican que acciones puede realizar el proceso a nivel de sistema                      |
| **Metadatos**   | Tipo de token (primario / impersonation y su tipo), nivel de integridad y cómo fue obtenido (NTLM, Kerberos...) |
Los tokens pueden ser de dos tipos:

> [!info] Primary Token 
> Es el token principal del proceso. Se crea al iniciar sesión y se asigna al primer proceso de la sesión. Todos los procesos que lanza el usuario heredan una copia de este token.

> [!info] Impersonation Token 
> Token temporal que un **hilo** usa para actuar como otro usuario. Tiene cuatro niveles de capacidad:
> 
> | Anonymous              | Identification                                          | Impersonation                                        | Delegation                                                                 |
| ---------------------- | ------------------------------------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------------- |
| Sin identidad asociada | Se puede identificar al usuario, pero no actuar como él | Puede actuar como el usuario en el **sistema local** | Puede actuar como el usuario en **sistemas remotos** (delegación Kerberos) |
> 

> [!warning] El token partido
> Cuando un miembro del grupo Administradores inicia sesión, recibe **dos tokens**:
> 1. El token filtrado: Para uso diario, actúa con nivel medio de integridad y se desactivan los privilegios del grupo de Admins.
> 2. El token completo: En reserva, con un nivel grande de integridad y hereda todos los permisos y privilegios de los admins
> - Al hacer "Ejecutar como administrador" → se usa el token completo
> > ⚠️ Bypass de UAC Muchas técnicas de escalada de privilegios en Windows buscan obtener el token High sin pasar por el prompt de UAC. Herramientas como `fodhelper.exe`, `eventvwr.exe` o `sdclt.exe` han sido usadas históricamente para esto porque se auto-elevan sin mostrar el prompt.

## 1.4. Comunicación entre procesos 

Como los procesos están aislados y no pueden leer la memoria de otro, necesitan mecanismos adicionales para comunicarse entre sí y enviar o recibir datos. A esto se le llama **IPC (Inter-Process Communication)**.

> [!tip]+ **COM (Component Object Model)** 
**COM (Component Object Model) es una tecnología de Microsoft que permite que distintos programas (o procesos) se comuniquen entre sí aunque estén creados en lenguajes diferentes.** 
> 
> Un objeto COM, expone una serie de métodos o funciones que pueden utilizar los programas (por ejemplo, leer o escribir en archivos). Estos están creados a partir de una clase (un "molde") identificada por un CLSID que define cómo debe crearse ese objeto, cuales son sus métodos y cómo debe accederse a ellos (interfaz). El conjunto de CLSIDs disponibles está en el registro, en: `HKEY_CLASSES_ROOT\CLSID\{CLSID}`.
> 
> Si un programa necesita de esta funcionalidad, solicita la creación de un objeto COM mediante su CLSID. El sistema COM (a través de RPCSS) localiza la clase correspondiente, activa la implementación adecuada (DLL, ejecutable o servicio) y crea el objeto. Finalmente, devuelve al programa una interfaz para acceder a sus métodos y a través de ella el programa realiza las llamadas necesarias. 
> - DCOM: Si el objeto no es local, se pueden llamar a sus métodos de manera remota por medio de RPC.

Si bajamos un nivel, COM se apoya en RPC:

> [!warning]+ RPC
> **RPC (Remote Procedure Call)**, es el mecanismo que realmente ejecuta las llamadas entre procesos.  A ojos del usuario, es como si hubiera ocurrido todo en el propio programa o la propia máquina
> Cuando un cliente invoca un método de un objeto COM, esa llamada se transforma en una petición RPC:
> 1. El proxy (lado cliente) contacta con el **RPC Endpoint Mapper** del servidor en el puerto TCP/135 para localizar el endpoint donde escucha el servidor COM. El Endpoint Mapper responde con un endpoint RPC válido (por ejemplo un puerto dinámico, una named pipe o un endpoint ALPC).
> 2. El proxy empaqueta los parámetros (marshalling) y establece una conexión RPC con el endpoint del servidor. Durante esa conexión, el cliente puede autenticarse mediante NTLM o Kerberos.
> 3. El stub (lado servidor) recibe los datos RPC, reconstruye los parámetros (unmarshalling), ejecuta el método real del objeto COM y devuelve el resultado al cliente mediante RPC.
> - En los ataques Potato, el atacante fuerza que un servicio COM privilegiado (normalmente SYSTEM) devuelva la respuesta a un endpoint RPC o una named pipe controlada por él. Gracias a `SeImpersonatePrivilege`, puede impersonar ese contexto autenticado y obtener un token SYSTEM.

Finalmente, en el nivel más bajo, RPC utiliza distintos **transportes**, y uno de los más importantes en Windows son los **Named Pipes**. 

> [!error]+ Named Pipes
**Un Named Pipe es un canal de comunicación bidireccional con nombre, que se comporta como un archivo**: se abre, se lee, se escribe y se cierra. Además pueden tener  permisos DACL como cualquier objeto de Windows. Son el equivalente a los socket files de Linux
> - Usan el formato `\\.\pipe\nombre` (local) o `\\servidor\pipe\nombre` (remoto). Ej. `\\.\pipe\lsass` es para la autenticación local de LSASS
> - El primer proceso crea el pipe y espera conexiones; el segundo se conecta a ella y envía/recibe datos. SMB por ejemplo trabaja en base a named pipes.
>
>Se pueden listar con `dir \\.\pipe\` o con `pipelist.exe` de la suite de sysinternals
> 
> > [!error] Named Pipes y escalada de privilegios 
> > Los Named Pipes son clave en varios ataques de escalada:
> > - **Printer Bug**: Fuerza al servicio de impresión del DC a autenticarse contra una máquina controlada por el atacante via `\\.\pipe\spoolss`, capturando su TGT (útil con Unconstrained Delegation).

**Así, el flujo completo sería: COM define la interacción a alto nivel, RPC implementa la llamada entre procesos, y los Named Pipes (o TCP/IP) sirven como canal real por el que viajan los datos.**

---
## 1.5. Integrity Levels (IL)

Windows asigna un nivel de integridad a cada proceso y objeto. Un proceso solo puede escribir en objetos de igual o menor integridad.

| Nivel     | Usuarios                                           |
| --------- | -------------------------------------------------- |
| SYSTEM    | kernel, drivers, lsass                             |
| High      | proceso elevado (UAC aceptado), administrador      |
| Medium    | usuario normal, la mayoría de procesos             |
| Low       | sandboxed (IE protected mode, algunos navegadores) |
| Untrusted | sin privilegios, casi sin acceso                   |
> [!example] Regla práctica de integridad
>  Un proceso Medium **no puede escribir** en archivos o claves de registro marcados como High. Por eso un proceso de usuario normal no puede tocar `C:\Windows\System32\` aunque tenga los permisos DACL correctos — la integridad lo bloquea antes.

---
# 2. 🔧 Servicios

Un servicio es un proceso especial controlado por el **SCM (Service Control Manager)**, `services.exe`. A diferencia de un proceso normal, un servicio:
- Puede arrancar **antes de que haya un usuario logueado**
- Se gestiona mediante la API de servicios (no directamente desde el proceso)
- Tiene un **tipo de inicio** configurable
- Corre bajo una **cuenta de servicio** específica

Posee estos atributos:

| Atributo                | Descripción                                                             |
| ----------------------- | ----------------------------------------------------------------------- |
| **Nombre**              | Identificador interno (ej: `wuauserv`)                                  |
| **Nombre para mostrar** | Nombre legible (ej: `Windows Update`)                                   |
| **Estado**              | Running / Stopped / Paused / Start Pending                              |
| **Tipo de inicio**      | Automatic / Manual / Disabled / Automatic (Delayed)                     |
| **Cuenta**              | Usuario bajo el que corre el servicio. Suele ser una cuenta de servicio |
| **Ruta del ejecutable** | Ruta completa con argumentos (`ImagePath`)                              |
| **Dependencias**        | Servicios que deben estar activos antes                                 |
| **Tipo de servicio**    | Own Process / Shared Process / Kernel Driver                            |
> [!example]+ Tipo de servicio
> 
> | Tipo                   | Descripción                                               |
> | ---------------------- | --------------------------------------------------------- |
> | **Own Process**        | El servicio tiene su propio proceso exclusivo             |
> | **Shared Process**     | Comparte proceso con otros servicios (como `svchost.exe`) |
> | **Kernel Driver**      | Corre en modo kernel (`sys`)                              |
> | **File System Driver** | Driver de sistema de archivos                             |
> 

> [!example]+ Tipo de arranque
> 
| Tipo                        | Descripción                                         |
| --------------------------- | --------------------------------------------------- |
| `Automatic`                 | Arranca con Windows, al inicio del boot             |
| `Automatic (Delayed Start)` | Arranca poco después del boot, para no ralentizarlo |
| `Manual`                    | Solo arranca cuando algo lo solicita                |
| `Disabled`                  | No puede arrancarse                                 |
| `Boot / System`             | Solo para drivers del kernel                        |
> 

---
## 2.1. Cuentas de servicio

Los servicios corren bajo una cuenta que determina sus privilegios:

| Cuenta                        | Privilegios                              | Uso típico                 |
| ----------------------------- | ---------------------------------------- | -------------------------- |
| `NT AUTHORITY\SYSTEM`         | Máximos (god mode)                       | Servicios críticos del SO  |
| `NT AUTHORITY\LocalService`   | Bajos, sin acceso a red                  | Servicios simples          |
| `NT AUTHORITY\NetworkService` | Bajos, con acceso a red                  | Servicios con conectividad |
| Cuenta de usuario             | Los del usuario                          | Servicios de terceros      |
| **gMSA**                      | Gestionada por AD, contraseña automática | Servicios en dominio       |
> [!danger] Servicios corriendo como SYSTEM
>  Un servicio vulnerable que corra como SYSTEM es un vector de escalada de privilegios muy habitual. Si se puede modificar el ejecutable o los argumentos, se puede ejecutar código arbitrario como SYSTEM.

> [!info] ¿Cómo sabe `svchost.exe` qué servicios alojar?
>  En el registro, bajo `HKLM\SYSTEM\CurrentControlSet\Services\<nombre>\Parameters`, cada servicio tiene su configuración. Los servicios de tipo Shared Process especifican el grupo al que pertenecen (`-k NetworkService`, `-k LocalSystemNetworkRestricted`...). El SCM agrupa los servicios del mismo grupo en la misma instancia de `svchost.exe`.

---
## 2.2. 🔐 Permisos de servicios (DACL de servicio)

Los servicios tienen su propia DACL independiente de los archivos. Controla quién puede iniciarlos, detenerlos o modificarlos.

```
sc sdshow <servicio>   → muestra el SDDL de permisos del servicio
```

> [!warning] Servicios con permisos débiles
>  Si un usuario no privilegiado tiene `SERVICE_ALL_ACCESS` o `SERVICE_CHANGE_CONFIG` sobre un servicio, puede cambiar su `ImagePath` para apuntar a un ejecutable malicioso y reiniciarlo. Esto es una escalada de privilegios clásica. Herramientas como `accesschk.exe` (Sysinternals) sirven para auditar esto:
> 
> ```
> accesschk.exe -uwcqv "Users" *   → servicios que Users puede modificar
> ```

---
## 2.3. 🗺️ Visión general

```
  DISCO                   MEMORIA                      KERNEL
  ─────                   ───────                      ──────
                                                        
  notepad.exe   ──carga──▶  PROCESO (notepad)          
                            ├── PID: 1234              
                            ├── Token de acceso  ──────▶ decide permisos
                            ├── Espacio de memoria      
                            │   ├── Código (.text)      
                            │   ├── Datos (.data)       
                            │   └── Heap / Stack        
                            ├── Handles (archivos,      
                            │   claves de registro...)  
                            └── Hilos (Threads)         
                                ├── Hilo 1 (main) ─────▶ ejecuta instrucciones
                                ├── Hilo 2        ─────▶ ejecuta instrucciones
                                └── Hilo N        ─────▶ ejecuta instrucciones
```
