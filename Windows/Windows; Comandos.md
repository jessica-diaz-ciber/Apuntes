# 1. 💻 Básico

## 1.1. Redirecciones y operadores

| Comando              | Descripción                                                             |
| -------------------- | ----------------------------------------------------------------------- |
| `cmd1 \| cmd2`       | Pipe: pasa la salida de`cmd1`como entrada de`cmd2`                      |
| `cmd > archivo.txt`  | Redirige la salida estándar a un archivo. Pero lo sobrescribe si existe |
| `cmd >> archivo.txt` | Añade el contenido de la salida al final del archivo sin sobrescribir   |
| `cmd 2> errores.txt` | Redirige la salida de **error**(stderr) a un archivo                    |
| `cmd 2>&1`           | Redirige stderr al mismo destino que stdout (fusiona salidas)           |
| `cmd1 && cmd2`       | Ejecuta `cmd2` solo si `cmd1` tuvo éxito (exit code 0)                  |
| `cmd1 \|\| cmd2`     | Ejecuta `cmd2` solo si `cmd1` falló (exit code distinto a 0)            |
| `cmd1 & cmd2`        | Ejecuta ambos comandos **siempre**, en secuencia                        |
**Referencia a directorios**
- `.` : Directorio actual
- `..` : Directorio padre

---
# 2. 📁 Gestión de ficheros

## 2.1. Listado

| Acción                          | CMD               | Powershell                               |
| ------------------------------- | ----------------- | ---------------------------------------- |
| Cambiar de directorio           | `cd <ruta>`       | `cd <ruta>` (alias de `Set-Location`)    |
| Listado                         | `dir <ruta>`      | `dir <ruta>` (alias de `Get-ChildItem`)  |
| Listar solo nombres             | `dir /b <ruta>`   |                                          |
| Listado recursivo               | `dir /s <ruta>`   | `dir -Recurse <ruta>`                    |
| Listar archivos ocultos         | `dir /a <ruta>`   | `dir -Force <ruta>`                      |
| Listar solo carpetas            | `dir /ad <ruta>`  | `dir -Directory <ruta>`                  |
| Listar solo archivos            | `dir /a-d <ruta>` | `dir -File <ruta>`                       |
| Leer el contenido de un archivo | `type <archivo>`  | `cat <archivo>` (alias de `Get-Content`) |
| Contar archivos                 |                   | `(dir <ruta>).Count `                    |

## 2.2. Crear y borrar archivos y carpetas

| Acción                             | CMD                  | Powershell                                              |
| ---------------------------------- | -------------------- | ------------------------------------------------------- |
| Crear una carpeta                  | `mkdir <carpeta>`    | `ni <nombre> -ItemType Directory` (alias de `New-Item`) |
| Eliminar un archivo                | `del <archivo>`      |                                                         |
| Eliminar una carpeta vacía         | `rmdir <carpeta>`    |                                                         |
| Eliminar una carpeta con contenido | `rmdir /s <carpeta>` | `del .\carpeta -Recurse -Force`                         |
| Vaciar una carpeta                 |                      | `del .\carpeta\* -Recurse -Force`                       |

## 2.3. Copiar y mover ficheros

| Acción                     | CMD                         | Powershell                                     |
| -------------------------- | --------------------------- | ---------------------------------------------- |
| Copiar un archivo          | `copy <origen> <destino>`   | `cp <origen> <destino>` (alias de `Copy-Item`) |
| Mover/renombrar un archivo | `move <origen> <destino>`   | `mv <origen> <destino>` (alias de `Move-Item`) |
| Copiar todo el contenido   | `copy <origen>\* <destino>` | `cp <origen>\* <destino>`                      |

## 2.4. Permisos

Los permisos se asignan con `icacls <ruta>`

---
# 3. Filtrado, propiedades y búsquedas

## 3.1. CMD

En CMD utilizamos `findstr` para los filtros

| Acción                                      | CMD                             |
| ------------------------------------------- | ------------------------------- |
| Buscar un patrón en un archivo              | `findstr <patrón> <archivo>`    |
| Ignorar mayúsculas/minúsculas               | `findstr /I <patrón> <archivo>` |
| Excluir líneas que coincidan                | `findstr /V <patrón> <archivo>` |
| Buscar usando expresión regular             | `findstr /R <patrón> <archivo>` |
| Buscar en subdirectorios de forma recursiva | `findstr /S <patrón> *.txt`     |
| Buscar un archivo de forma recursiva        | `dir /s /b \| findstr archivo`  |

## 3.1. Powershell

**PowerShell devuelve objetos, no texto plano. Por eso se puede filtrar por propiedades:**

| Acción                           | CMD                                                        |
| -------------------------------- | ---------------------------------------------------------- |
| Filtrar cadena de un texto       | `sls <patrón> <archivo>` (alias de `Select-String`)        |
| Mostrar propiedades concretas    | `select propiedad1, propiedad2` (alias de `Select-Object`) |
| Limitar resultados               | `select -First/-Last x`                                    |
| Mostrar en modo columnas / lista | `ft` (`Format-Table`) / `fl` (`Format-List`)               |
| Filtro por propiedades           | `? { $_.<propiedad> <filtro>}` (alias de `Where-Object`)   |
> [!abstract] Los filtros pueden ser 
> 
| mayor que | menor que | coincidencias exactas | oincidencias parciales | AND lógico | OR lógico |
| --------- | --------- | --------------------- | ---------------------- | ---------- | --------- |
| `-gt`     | `-lt`     | `-match`              | `-like`                | `-and`     |  `-or`    |

---
# 4. 👤 Gestión de usuarios y grupos

🔗 Relacionados
- [[Active Directory]]
- [[Windows; gestión de identidades]]

## 4.1. Consultar información

| Acción                                 | CMD                  | Powershell                |
| -------------------------------------- | -------------------- | ------------------------- |
| Ver el usuario actual                  | `whoami`             | `whoami`                  |
| Ver los privilegios de nuestro usuario | `whoami /priv`       |                           |
| Ver a qué grupos pertenecemos          | `whoami /groups`     |                           |
| Ver el nombre completo y SID           | `whoami /user`       |                           |
| Ver los usuarios con sesión activa     | `query user`         |                           |
| Listar usuarios                        | `net user`           | `Get-LocalUser`           |
| Ver información de otro usuario        | `net user <usuario>` | `Get-LocalUser <usuario>` |

## 4.2. Crear y eliminar usuarios

| Acción                              | CMD                              | Powershell                                      |
| ----------------------------------- | -------------------------------- | ----------------------------------------------- |
| Crear un usuario                    | `net user <usuario> <pass> /add` | `New-LocalUser -Name <usuario>`                 |
| Cambiar la contraseña de un usuario | `net user <usuario> <pass>`      | `Set-LocalUser -Name <usuario> -Password $pass` |
| Eliminar un usuario                 | `net user <usuario> /delete`     | `Remove-LocalUser -Name <usuario>`              |

## 4.3. Gestionar grupos

| Acción                            | CMD                                        | Powershell                                                 |
| --------------------------------- | ------------------------------------------ | ---------------------------------------------------------- |
| Listar grupos                     |                                            | `Get-Localgroup`                                           |
| Listar los miembros de un grupo   | `net localgroup <grupo>`                   | `Get-LocalGroupMember -Group <grupo>`                      |
| Añadir un usuario a un grupo      | `net localgroup <grupo> <usuario> /add`    | `Add-LocalGroupMember -Group <grupo> -Member <usuario>`    |
| Eliminar a un usuario de un grupo | `net localgroup <grupo> <usuario> /delete` | `Remove-LocalGroupMember -Group <grupo> -Member <usuario>` |

---
# 5. 👤 Gestión de procesos y redes

## 5.1. Gestionar procesos

| Acción                                        | CMD                                       | Powershell                                         |
| --------------------------------------------- | ----------------------------------------- | -------------------------------------------------- |
| Listar procesos                               | `tasklist`                                | `gps` (alias de `Get-Process`)                     |
| Listar procesos con usuario y más información | `tasklist /V `                            | `gps notepad \| select *`                          |
| Filtrar procesos por nombre de ejecutable     | `tasklist /FI "IMAGENAME eq notepad.exe"` | `gps notepad`                                      |
| Filtrar procesos por usuario                  | `tasklist /FI "USERNAME eq juan"`         |                                                    |
| Finalizar un proceso por PID                  | `taskkill /PID <pid> /F`                  | `kill -Id <pid> -Force` /(alias de `Stop-Process`) |
| Finalizar un proceso por nombre               | `taskkill /IM notepad.exe /F`             | `kill -Name notepad -Force`                        |
| Terminar un proceso y sus hijos               | `taskkill /PID <pid> /F /T`               |                                                    |
| Ver las DLLs que carga                        |                                           | `(gps notepad).Modules`                            |

## 5.2. Conexiones y puertos

| Acción                                          | CMD                            | Powershell                           |
| ----------------------------------------------- | ------------------------------ | ------------------------------------ |
| Conexiones activas con PID y sin resolución DNS | `netstat -ano`                 | `Get-NetTCPConnection -State Listen` |
| Filtrar conexiones por puerto concreto          | `netstat -ano \| findstr :443` | `Get-NetTCPConnection -Port 443`     |
| Ver si responde un puerto                       |                                | `Test-NetConnection <ip> -Port 443`  |
| Realizar un ping                                | `ping <ip>`                    | `ping <ip>`                          |

## 5.3. Servicios

| Acción                                                        | CMD                                                                | Powershell                                                                                     |
| ------------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Listar servicios                                              | `sc query`                                                         | `gsv` (alias de `Get-Service`)                                                                 |
| Listar servicios corriendo                                    |                                                                    | `gsv \| ? Status -eq Running`                                                                  |
| Iniciar/detener un servicio                                   | `sc start/stop <servicio>` / `net start/stop <servicio>`           | `Start/Stop/Restart-Service <nombre>`                                                          |
| Configurar arranque de servicio: auto, manual o deshabilitado | `sc config <servicio> start= auto/demand/disabled`                 | `Set-Service <nombre> -StartupType Automatic/Manual/Disabled`                                  |
| Ver información de un servicio                                | `sc qc <servicio>` y `sc queryex <servicio>`                       | `gsv  <servicio> \| select *`                                                                  |
| Cambiar la cuenta del servicio                                | `sc config "wuauserv" obj= "NT AUTHORITY\NetworkService"`          |                                                                                                |
| Crear un servicio                                             | `sc create MiServicio binPath= "C:\ruta\servicio.exe" start= auto` | `New-Service -Name "MiServicio" -BinaryPathName "C:\ruta\servicio.exe" -StartupType Automatic` |
| Eliminar un servicio                                          | `sc delete MiServicio`                                             | `Remove-Service -Name "MiServicio"`                                                            |
> [!warning] Espacio después de `=` En `sc.exe`, los parámetros requieren un espacio entre `=` y el valor: `start= auto` (con espacio). Sin ese espacio, el comando falla silenciosamente.