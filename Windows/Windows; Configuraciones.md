# 1. 🗃️ El Registro de Windows

> [!info] ¿Qué es? 
> El Registro es una **base de datos jerárquica** que centraliza toda la configuración del sistema operativo y las aplicaciones. Cuando cambias algo en herramientas como `secpol.msc` o `gpedit.msc`, en realidad estás escribiendo en el registro.

### Estructura

```
HKLM\SOFTWARE\Policies\Microsoft\Windows Defender\Scan
 │      │        │          │          │             │
Hive  Subclave  Subclave  Subclave  Subclave      Valor
```

Cada entrada tiene:

- **Hive** → ámbito general (5 colmenas principales)
- **Subclaves** → categorías anidadas jerárquicamente
- **Nombre del valor** → identifica una configuración concreta
- **Dato** → el valor asignado a esa configuración

### Las 5 colmenas (Hives)

|Hive|Nombre completo|Contenido|
|---|---|---|
|**HKCR**|HKEY_CLASSES_ROOT|Asociaciones de extensiones de archivo con programas|
|**HKCU**|HKEY_CURRENT_USER|Configuración del usuario actual (idioma, fondo, preferencias)|
|**HKU**|HKEY_USERS|Configuración de todos los usuarios del sistema|
|**HKLM**|HKEY_LOCAL_MACHINE|Configuración global del sistema (drivers, servicios, SAM)|
|**HKCC**|HKEY_CURRENT_CONFIG|Hardware detectado en el arranque actual (subconjunto de HKLM)|

### Subclaves importantes

> [!example] `HKCU` — Configuración del usuario actual
> 
> |Subclave|Contenido|
> |---|---|
> |`Software\`|Preferencias de apps por usuario (historial, tamaño de ventana...)|
> |`Control Panel\`|Escritorio, ratón, pantalla, teclado|
> |`Console\`|Configuración de la CMD|
> |`Environment\`|Variables de entorno del usuario|
> |`Network\`|Unidades de red mapeadas|

> [!example] `HKLM` — Configuración global del sistema
> 
> |Subclave|Contenido|
> |---|---|
> |`SOFTWARE\`|Configuración de programas a nivel de sistema|
> |`SYSTEM\`|Drivers, servicios, configuración de arranque|
> |`HARDWARE\`|RAM, CPU, puertos (detectados en cada arranque)|
> |`SECURITY\`|Políticas de seguridad del sistema|
> |`SAM\`|Base de datos de cuentas locales (**protegida**, solo accesible como SYSTEM)|

### Tipos de datos

|Tipo|Descripción|Ejemplo|
|---|---|---|
|`REG_SZ`|Cadena de texto|`C:\Program Files\app\app.exe`|
|`REG_DWORD`|Número de 32 bits|`0x00000001`|
|`REG_QWORD`|Número de 64 bits|`0x0000000000000001`|
|`REG_MULTI_SZ`|Lista de cadenas|Múltiples rutas|
|`REG_EXPAND_SZ`|Texto con variables de entorno|`%SystemRoot%\System32`|
|`REG_BINARY`|Datos binarios en bruto|Configuración de hardware|

### Modificar el registro desde CMD

Podemos modificar el registro con la consola: 

```cmd
reg add <ruta> /v <nombre_valor> /t <tipo> /d <dato> /f

:: Ejemplo: deshabilitar Windows Defender
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiSpyware /t REG_DWORD /d 1 /f
```

|Parámetro|Descripción|
|---|---|
|`<ruta>`|Ruta completa de la clave (ej: `HKLM\SOFTWARE\...`)|
|`/v`|Nombre del valor a crear o modificar|
|`/t`|Tipo de dato (`REG_SZ`, `REG_DWORD`...)|
|`/d`|El dato a escribir|
|`/f`|Force — no pide confirmación|

> [!tip] Interfaz gráfica `Win + R` → `regedit` para navegar visualmente por el registro.

---





