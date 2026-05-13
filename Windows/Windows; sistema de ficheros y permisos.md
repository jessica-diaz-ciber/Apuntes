> [!abstract] Introducción 
> Windows organiza sus archivos en una jerarquía de carpetas bajo `C:\`. El acceso a ellas está controlado por ACLs. Toda la configuración del sistema se centraliza en el **Registro de Windows**, una base de datos jerárquica que es la columna vertebral de la configuración del sistema y las aplicaciones.

## 1. 📁 Estructura de directorios

## 1.1. Los directorios principales

La raíz del sistema es `C:\`. A partir de ahí se ramifica todo:

```
C:\
├── Users\                  → perfiles de usuario
├── Windows\                → sistema operativo
│   └── System32\           → librerías y ejecutables esenciales
│       ├── config\         → registro (SAM, SYSTEM, SOFTWARE...)
│       └── repair\         → copia de seguridad del registro
├── Program Files\          → programas 64 bits
├── Program Files (x86)\    → programas 32 bits (solo en SO 64 bits)
├── ProgramData\            → [oculta] datos de programas para todos los usuarios
├── Public\                 → archivos compartidos entre usuarios
├── PerfLogs\               → reportes de crasheos y rendimiento (suele estar vacía)
├── $Recycle.Bin\           → [oculta] papelera de reciclaje
└── inetpub\wwwroot\        → raíz web IIS (solo si hay servidor web configurado)
```

## 1.2. Detalle de carpetas clave

> [!info]+ `C:\Users\<usuario>\` 
> Directorio personal de cada usuario. En sistemas antiguos era `C:\Documents and Settings\Usuario`.
> 
> ```
> \Users\Jessica\
> ├── Desktop\, Documents\, Pictures\, Downloads\...
> └── AppData\                     ← oculta
>     ├── Roaming\                 → configuraciones que siguen al usuario (ej: en dominio)
>     ├── Local\                   → datos locales, caché, archivos grandes y sensibles
>     └── LocalLow\                → como Local pero con menor integridad (ej: modo incógnito)
> ```

> [!info]+ `C:\Windows\System32\`
>  El corazón de Windows. Contiene DLLs esenciales, ejecutables del sistema (`cmd.exe`, `control.exe`, `utilman.exe`...) y dos subcarpetas críticas:
> 
> - `\config\` → las colmenas del registro en vivo: `SAM`, `SECURITY`, `SYSTEM`, `SOFTWARE`
> - `\repair\` → copia de seguridad de esas mismas colmenas

> [!info]+ `C:\ProgramData\` 
> Carpeta **oculta** con datos que los programas necesitan para ejecutarse, accesible para todos los usuarios del sistema. Distinta de `Program Files` (los binarios) y de `AppData` (datos por usuario).

> [!tip]+ Equivalencias con Linux
> 
> |Windows|Linux|
> |---|---|
> |`C:\`|`/`|
> |`C:\Users\`|`/home/`|
> |`C:\Windows\System32\`|`/bin/`, `/lib/`|
> |`C:\inetpub\wwwroot\`|`/var/www/html/`|
> |`C:\ProgramData\`|`/etc/` (aprox.)|

---
## 2. 🔒 ACL — Control de acceso a archivos

Cada archivo y carpeta tiene un **Security Descriptor** que define quién puede hacer qué. Este está formado por:

| Componente                       | Descripción                                     |
| -------------------------------- | ----------------------------------------------- |
| **Owner SID**                    | El SID del propietario                          |
| **DACL** _(Discretionary ACL)_   | Lista de quién puede hacer qué sobre el objeto  |
| **ACE** _(Access Control Entry)_ | Cada regla individual dentro de una ACL         |
| **SACL** _(System ACL)_          | Define qué accesos se registran en el Event Log |
> [!warning] Orden de evaluación
>  Windows evalúa las ACEs en orden. Las **denegaciones explícitas tienen prioridad**. Si ninguna ACE coincide, el acceso se **deniega por defecto**.

---
## 2.1. Entender el sistema de permisos

> [!tip]+ Permisos simples
> Estos permisos son una combinación de permisos avanzados. Son los que se usan para administrar el sistema de manera sencilla sin complicarse.
> 
| Sigla  | Permiso        | Descripción                                                                          |
| ------ | -------------- | ------------------------------------------------------------------------------------ |
| **N**  | No access      | No se tiene ningún permiso                                                           |
| **F**  | Full access    | Control total: leer, escribir, ejecutar, borrar y cambiar permisos                   |
| **M**  | Modify         | Leer, escribir, ejecutar y borrar                                                    |
| **RX** | Read y Execute | Leer archivos y ejecutar programas                                                   |
| **R**  | Read           | Solo se puede ver el contenido (leer un archivo o ver los contenidos de una carpeta) |
| **W**  | Write          | Permite crear o modificar archivos                                                   |
| **D**  | Delete         | Permite borrar el archivo o carpeta                                                  |

> [!tip]+ Permisos avanzados
> Los permisos avanzados son los permisos individuales que realmente usa Windows. Existen bastantes, pero vamos a dar los más importantes
> 
> | Sigla    | Permiso                        | Descripción                                                |
| -------- | ------------------------------ | ---------------------------------------------------------- |
| **DE**   | Delete                         | Borrar el archivo o carpeta                                |
| **RC**   | Read Control                   | Ver los permisos del objeto                                |
| **WDAC** | Write DAC                      | Permite cambiar los permisos                               |
| **WO**   | Write Owner                    | Convertir al usuario en propietario                        |
| **RD**   | Read data / List directory     | Permite leer archivos o listar el contenido de una carpeta |
| **WD**   | Write data / Add file          | Permite escribir datos o crear archivos                    |
| **AD**   | Append data / Add subdirectory | Permite añadir datos o crear subcarpetas                   |
| **X**    | Execute / Traverse             | Permite ejecutar archivos o atravesar carpetas             |
| **DC**   | Delete Child                   | Permite borrar archivos dentro de una carpeta              |
| **RA**   | Read Attributes                | Permite ver atributos (oculto, solo lectura, etc.)         |
| **WA**   | Write attributes               | Permite cambiar atributos                                  |
> El permiso simple F (Full access) se construye a partir de juntar un montón de permisos avanzados

> [!tip]+ Permisos heredados
> Son permisos que relacionan los contenedores (carpetas) con su contenido (subcarpeta y archivos). Son algo complejos, para lo que tenemos el siguiente ejemplo:
> ```
Carpeta_A  
├── archivo.txt
└── Carpeta_B
       └── Carpeta_C
> ```
> | Sigla    | Permiso               | Descripción                                                                                       | Ejemplo                                                          |
| -------- | --------------------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **(I)**  | **Inherit**           | Los permisos son heredados de la carpeta madre                                                    | _El permiso de archivo.txt viene de Carpeta_A_                   |
| **(OI)** | **Object-Inherit**    | El permiso se copiará a los archivos dentro de la carpeta.                                        | _El permiso de Carpeta_A llega a archivo.txt_                    |
| **(CI)** | **Container-Inherit** | El permiso se copiará a las subcarpetas dentro de la carpeta.                                     | _El permiso de Carpeta_A llega a Carpeta_B y Carpeta_C_          |
| **(IO)** | **Inherit-Only**      | El permiso no afecta a esta carpeta, solo se usa para heredarlo a lo que hay dentro.              | _Carpeta_A no tiene el permiso, pero archivo.txt y Carpeta_B sí_ |
| **(NP)** | **No-Propagate**      | El permiso solo baja un nivel. Las subcarpetas lo reciben, pero no lo siguen heredando más abajo. | _El permiso de Carpeta_A solo llega a Carpeta_B_                 |

## 2.2. Consultar permisos

Para ver los permisos de un archivo utilizamos en **cmd** el comando `icacls` 

```python
PS C:\Users\Jessica> icacls archivo.txt
.pdfbox.cache NT AUTHORITY\SYSTEM:(I)(F)
              BUILTIN\Administradores:(I)(F)
              PC_Jessica\Jessica:(I)(F)
```

En **powershell** tenemos el comando `Get-Acl` al que le tenemos que pasar la ruta del archivo, luego lo formateamos con  **Format-List**  para que nos salga toda la información de manera ordenada. Tenemos que fijarnos sobre todo en los campos owner (propietario), group (grupo) y accesos (permisos para el resto de usuarios). 

 > En el campo  **sddl**  sale la misma información solo que representada en un formato menos legible, con SIDs y los permisos en siglas, es el formato que utiliza el sistema internamente.
 
```powershell
PS C:\Users\Jessica> Get-Acl archivo.txt | fl
Path   : Microsoft.PowerShell.Core\FileSystem::C:\Users\Jessica\archivo.txt
Owner  : PC_Jessica\Jessica
Group  : PC_Jessica\Ninguno
Access : NT AUTHORITY\SYSTEM Allow  FullControl
         BUILTIN\Administradores Allow  FullControl
         PC_Jessica\Jessica Allow  FullControl
Audit  :
Sddl   : O:S-1-5-21-3793951252-29971934-283417696-1001G:S-1-5-21-3793951252-29971934-283417696-513D:AI(A;ID;FA;;;SY)(A;
         ID;FA;;;BA)(A;ID;FA;;;S-1-5-21-3793951252-29971934-283417696-1001)
```

## 2.3. Cambiar propietario y permisos

Podemos cambiar el propietario con `icacls` de manera bastante sencilla con el parámetro `/setowner`

```c
icacls archivo.txt /setowner Juan
```

Para asignar permisos podemos usar icacls con le parámetro  `/grant`  seguido del nombre del usuario y el permiso, todo ello entre comillas. **Preferiblemente usaremos los permisos simples.**

| Acción                               | Comando                                           |
| ------------------------------------ | ------------------------------------------------- |
| Añadir un permiso                    | `icacls <archivo> /grant <usuario>:(<permiso>)`   |
| Asignar permisos de manera recursiva | `icacls <archivo> /grant:r <usuario>:(<permiso>)` |
| Quitar un permiso                    | `icacls <archivo> /remove <usuario>:(<permiso>)`  |
>⚠️ Lo ideal con archivos sensibles es eliminar todos los permisos y luego añadirlos de uno en uno. Así aplicaremos el principio de “Mínimo Privilegio”



## 🔗 Ver también

- [[Windows; gestión de identidades]]
- [[Active Directory]]
