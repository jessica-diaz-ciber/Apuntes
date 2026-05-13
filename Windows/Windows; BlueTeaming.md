# 📋 Registro de Eventos de Windows

> [!abstract] Resumen
>  El **Event Log** registra de forma centralizada registros de seguridad, sistema y aplicaciones. Esto permiten auditar accesos, diagnosticar errores y monitorear el sistema.

## 📂 Canales de log

Windows organiza los eventos en **canales**. Cada canal es un archivo `.evtx` en `C:\Windows\System32\winevt\Logs\`.

| Canal                      | Archivo                                             | Contenido                                                                   |
| -------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------- |
| **Security**               | `Security.evtx`                                     | Auditoría de accesos, logons, cambios de cuentas, privilegios.              |
| **System**                 | `System.evtx`                                       | Mensajes del SO y drivers: servicios caídos, errores de hardware, reinicios |
| **Application**            | `Application.evtx`                                  | Errores y eventos de aplicaciones: crashes, errores de bases de datos...    |
| **Setup**                  | `Setup.evtx`                                        | Instalación y actualización del SO                                          |
| **Forwarded Events**       | `ForwardedEvents.evtx`                              | Eventos recibidos de otros equipos via WEF (Windows Event Forwarding)       |
| **PowerShell/Operational** | `Microsoft-Windows-PowerShell%4Operational.evtx`    | Comandos PowerShell ejecutados                                              |
| **Sysmon**                 | `Microsoft-Windows-Sysmon%4Operational.evtx`        | Telemetría extendida (si está instalado)                                    |
| **WMI Activity**           | `Microsoft-Windows-WMI-Activity%4Operational.evtx`  | Actividad WMI                                                               |
| **TaskScheduler**          | `Microsoft-Windows-TaskScheduler%4Operational.evtx` | Tareas programadas creadas/ejecutadas                                       |
> [!tip] Acceso rápido
>  `Win + R` → `eventvwr.msc` para la interfaz gráfica. Los archivos `.evtx` están en `C:\Windows\System32\winevt\Logs\` y se pueden copiar y analizar offline.

---
## 🔢 Event IDs clave — Security

> [!example] Autenticación y sesiones
> 
| ID       | Evento                                                       | Relevancia                                           |
| -------- | ------------------------------------------------------------ | ---------------------------------------------------- |
| **4624** | Logon exitoso                                                | Verificar tipo de logon, origen y hora               |
| **4625** | Logon fallido                                                | Detectar fuerza bruta. Muchos 4625 seguidos = ataque |
| **4634** | Logoff (sesión cerrada por el sistema)                       | Control de duración de sesiones                      |
| **4647** | Logoff iniciado por el usuario                               | Diferencia cierre voluntario de forzado              |
| **4648** | Logon con credenciales explícitas (`runas`) > (logon type 9) | Puede indicar movimiento lateral                     |
| **4672** | Privilegios especiales asignados al logon                    | Sesiones de admin/SYSTEM — siempre auditar           |
| **4778** | Sesión RDP reconectada                                       | Monitorizar acceso remoto                            |
| **4779** | Sesión RDP desconectada                                      | Correlacionar con 4778                               |
> > ⚠️ Tipo de login:  El campo **Logon Type** dentro del 4624 es crítico para entender cómo se autenticó el usuario:

> [!example] Cuentas de usuario y grupos
> 
| ID       | Evento                                          | Relevancia SOC                                    |
| -------- | ----------------------------------------------- | ------------------------------------------------- |
| **4720** | Cuenta de usuario creada                        | ¿Quién la creó? ¿A qué hora? ¿Era esperado?       |
| **4722** | Cuenta habilitada                               | Reactivación sospechosa de cuentas deshabilitadas |
| **4723** | Usuario cambia su propia contraseña             | Control normal                                    |
| **4724** | Admin cambia contraseña de otro usuario         | Auditar intervenciones administrativas            |
| **4725** | Cuenta deshabilitada                            | Control de bloqueos                               |
| **4726** | Cuenta de usuario eliminada                     | 🔴 Siempre investigar                             |
| **4728** | Usuario añadido a grupo de seguridad global     | ¿Se añadió a Admins?                              |
| **4729** | Usuario eliminado de grupo de seguridad global  | Cambios de permisos inesperados                   |
| **4732** | Usuario añadido a grupo local                   | Especial atención a Administrators                |
| **4733** | Usuario eliminado de grupo local                |                                                   |
| **4740** | Cuenta bloqueada (demasiados intentos fallidos) | Fuerza bruta en curso                             |
| **4767** | Cuenta desbloqueada                             | ¿Quién la desbloqueó y por qué?                   |

> [!example] Procesos y ejecución
> 
| ID       | Evento                           | Relevancia SOC                                  |
| -------- | -------------------------------- | ----------------------------------------------- |
| **4688** | Nuevo proceso creado             | Ver proceso, padre, usuario y línea de comandos |
| **4689** | Proceso terminado                | Correlacionar con 4688 para ver duración        |
| **4697** | Servicio instalado en el sistema | 🔴 Malware frecuentemente instala servicios     |
>> ⚠️ Para que el evento 4688 incluya los comandos que se han ejecutado en el proceso hay que habilitarlo: 
>> ```
> secpol.msc → Directivas locales → Auditoría → Auditar creación de procesos → Éxito
> gpedit.msc → Configuración del equipo → Plantillas administrativas → Sistema → Auditar creación de procesos → Incluir línea de comandos
>> ```

> [!example] Acceso a objetos y cambios de política
> 
| ID       | Evento                                       | Relevancia SOC                                         |
| -------- | -------------------------------------------- | ------------------------------------------------------ |
| **4663** | Acceso a un objeto (archivo, clave...)       | Requiere auditoría de objeto habilitada en SACL        |
| **4698** | Tarea programada creada                      | Persistencia habitual del malware                      |
| **4699** | Tarea programada eliminada                   |                                                        |
| **4700** | Tarea programada habilitada                  |                                                        |
| **4702** | Tarea programada modificada                  | 🔴 Modificación de tareas existentes para persistencia |
| **4719** | Política de auditoría del sistema modificada | Atacante intentando cegar el logging                   |
| **4964** | Grupos especiales asignados a logon          | Para grupos definidos en política de auditoría         |

> [!example] Uso de credenciales y kerberos
> 
| ID       | Evento                             | Relevancia SOC                                |
| -------- | ---------------------------------- | --------------------------------------------- |
| **4768** | TGT solicitado (AS-REQ)            | Base del flujo Kerberos                       |
| **4769** | ST solicitado (TGS-REQ)            | Muchas peticiones = posible Kerberoasting     |
| **4771** | Pre-autenticación Kerberos fallida | Equivalente Kerberos al 4625. AS-REP Roasting |
| **4776** | Intento de validación NTLM         | Autenticaciones NTLM locales                  |

## 🚨 Indicadores de compromiso comunes en logs

> [!danger] Señales de alerta en el Security Log

| Patrón                                                  | IDs involucrados | Posible amenaza                      |
| ------------------------------------------------------- | ---------------- | ------------------------------------ |
| Múltiples 4625 en segundos desde misma IP               | 4625             | Fuerza bruta                         |
| 4625 seguido de 4624 desde misma IP                     | 4625 → 4624      | Fuerza bruta exitosa                 |
| 4624 con Logon Type 3 + AuthPackage NTLM                | 4624             | Pass-the-Hash / NTLM Relay           |
| 4624 con Logon Type 9 (NewCredentials)                  | 4624             | `runas /netonly`, movimiento lateral |
| 4698 con nombre de tarea que imita servicios legítimos  | 4698             | Persistencia                         |
| 4720 + 4732 (crear usuario + añadirlo a Admins)         | 4720, 4732       | Backdoor de cuenta                   |
| 4719 (política de auditoría modificada)                 | 4719             | Atacante cegando el logging          |
| 4697 (servicio instalado) fuera de horario              | 4697             | Malware como servicio                |
| Sysmon 10 con target lsass.exe                          | Sysmon 10        | Volcado de credenciales              |
| Sysmon 3 con conexión a IP externa desde lsass/winlogon | Sysmon 3         | Beacon C2                            |
| 4769 masivo hacia muchos SPNs distintos                 | 4769             | Kerberoasting                        |
| 4771 masivo (pre-auth fallida)                          | 4771             | AS-REP Roasting o spray              |

---
## 🔵 Event IDs de Sysmon

**Sysmon** (System Monitor, Sysinternals) es un driver que enriquece enormemente la telemetría de Windows. No viene instalado por defecto pero es estándar en entornos SOC.

| ID        | Evento                         | Por qué importa                                                               |
| --------- | ------------------------------ | ----------------------------------------------------------------------------- |
| **1**     | Process Create                 | Como 4688 pero incluye hash del ejecutable, línea de comandos completa y GUID |
| **2**     | File creation time changed     | Técnica de timestomping para ocultar malware                                  |
| **3**     | Network connection             | Qué proceso hace qué conexión de red y a dónde                                |
| **5**     | Process terminated             |                                                                               |
| **6**     | Driver loaded                  | Carga de drivers — posible rootkit o BYOVD                                    |
| **7**     | Image/DLL loaded               | DLL cargada en un proceso — DLL hijacking                                     |
| **8**     | CreateRemoteThread             | Inyección de código entre procesos                                            |
| **10**    | ProcessAccess                  | Un proceso abre otro (ej: Mimikatz abre lsass)                                |
| **11**    | File created                   | Creación de archivos en disco                                                 |
| **12/13** | Registry key created/value set | Persistencia, modificación de configuración                                   |
| **15**    | File stream created (ADS)      | Alternate Data Streams — ocultación de datos                                  |
| **17/18** | Named pipe created/connected   | Comunicación entre procesos, Potato attacks                                   |
| **22**    | DNS query                      | Qué proceso resuelve qué dominio — C2 detection                               |
| **25**    | Process tampering              | Proceso alterado en memoria (process hollowing)                               |
> [!tip] Configuración de Sysmon 
> Sysmon es tan ruidoso o selectivo como se configure. La configuración de referencia más usada en blue team es la de **SwiftOnSecurity** (`sysmon-config.xml`) y la de **Olaf Hartong** (`sysmon-modular`).

---
## 💻 Consultar eventos desde PowerShell

```powershell
Get-WinEvent -LogName Security -MaxEvents 20 # Los últimos 20 eventos de seguridad

Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624,4625,4672} # Filtrar por IDs de evento

Get-WinEvent -LogName Security | ? { $_.Message -match "jessica" } # Buscar palabra clave en el mensaje del evento

# Contar logons fallidos por origen en las últimas 24h. Propierties[19] -> Ips
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddHours(-24)} |
    % { $_.Properties[19].Value } |  Group-Object | Sort-Object Count -Descending

# Ver eventos de Sysmon — accesos a lsass
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" | ? { $_.Id -eq 10 -and $_.Message -match "lsass" }

Get-WinEvent -Path "C:\evidencia\Security.evtx" -FilterXPath "*[System[EventID=4624]]" # Leer un .evtx offline 
```

---
## 🛡️ Configurar la auditoría correctamente

Por defecto, Windows **no registra muchos eventos importantes**. Hay que habilitarlos explícitamente.

| Descripción                                             | Comando                                                                                |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Ver configuración actual                                | `auditpol /get /category:*`                                                            |
| Habilitar auditoría de creación de procesos (para 4688) | `auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable`        |
| Habilitar auditoría de logon                            | `auditpol /set /subcategory:"Logon" /success:enable /failure:enable`                   |
| Habilitar auditoría de gestión de cuentas               | `auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable` |
| Habilitar auditoría de uso de privilegios               | `auditpol /set /subcategory:"Sensitive Privilege Use" /success:enable /failure:enable` |
Para el tamaño de los logs:

| Descripción                                                 | Comando                               |
| ----------------------------------------------------------- | ------------------------------------- |
| Ver tamaño actual del log de seguridad                      | `wevtutil gl Security`                |
| Ampliar tamaño máximo del log de seguridad a 1 GB           | `wevtutil sl Security /ms:1073741824` |
| Configurar que sobrescriba eventos antiguos cuando se llene | `wevtutil sl Security /rt:false`      |
> [!warning] Tamaño por defecto insuficiente 
> El log de Security tiene por defecto **20 MB**. En un entorno activo, se rota en horas. Para investigación forense o detección, lo mínimo recomendado es **1-4 GB

---
## 📡 Windows Event Forwarding (WEF)

WEF permite centralizar los eventos de múltiples máquinas en un **colector** sin instalar agentes adicionales. Es la alternativa nativa de Windows al SIEM para entornos pequeños.

```
  Máquinas fuente                     Colector WEF
  ──────────────                      ─────────────
  PC-01  ──────┐
  PC-02  ──────┤── WinRM / HTTPS ───▶  WEC-01
  PC-03  ──────┤                       └── Forwarded Events
  SRV-01 ──────┘                           (canal centralizado)
```

powershell
1. En el colector: habilitar el servicio de recolección `wecutil qc`
 |
2. En las máquinas fuente: configurar via GPO: `Computer Configuration → Administrative Templates → Windows Components → Event Forwarding → Configure target subscription manager → Server=http://<WEC>:5985/wsman/SubscriptionManager/WEC`
 |
3. Ver suscripciones activas en el colector: `wecutil es`
 |
4. Ver estado de una suscripción: `wecutil gs <nombre_suscripcion>`

---

## 🔗 Ver también

- [[Windows; gestión de identidades]]
- [[Windows; Procesos, redes y servicios]]
- [[Active Directory; Kerberos]]
- [[Windows; Escalada de privilegios]]
- [[Active Directory]]
