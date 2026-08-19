# 1. Crear un AD

## 1.1. Instalación
Para montar un directorio activo, debemos contar con una máquina Windows Server a la que asignemos una IP estática.

```bash
# 1. Primero hay que renombrar la máquina con un nombre descriptivo (y reiniciar el sistema)
Rename-Computer -NewName AD01 -Restart

# 2. Después, hay que instalar los servicios del directorio activo
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# 3. Y unir a nuestra máquina a un nuevo dominio del que será DC
Install-ADDSForest -DomainName dominio.local -DomainNetBIOSName DOMINIO
```

Al haber hecho esto, nos autenticaremos DENTRO del dominio. Esto se ve gracias al prefijo del usuario: `DOMINIO/usuario`

> Si desactivamos el firewall, desde kali podremos ver todos los servicios activos `Set NetFirewallProfile -Profile Domain,Public,Private -Enabled False`

---
## 1.2. Unir una segunda máquina  

Si queremos unir un segundo equipo al dominio, lo que no debemos hacer es CLONAR la máquina virtual, porque se clonará también su SID y esto causará un conflicto. Por tanto debemos configurar una máquina windows 10 server nueva.

```bash
# 1. Primero hay que renombrar la máquina 
Rename-Computer -NewName AD02 -Restart

# 2. Luego, obtenemos el nombre del adaptador de red para configurar el DC como DNS
netsh interface ipv4 show interface # Sale "Ethernet0"
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses "<IP_DC>"

# 3. Uniremos el equipo al dominio con la contraseña del administrador del DC
Add-Computer -DomainName l4h.local -Credential L4H\Administrator -Restart
```

---
# 2. Gestión

## 2.1. 👤 Crear usuarios
Para crear usuarios podemos hacerlo de dos maneras.

Con la interfaz gráfica es super fácil. En el DC tenemos que abrir `Active Directory Users and Computers` y tendremos una vista de todos los usuarios y grupos del directorio. 

Podemos hacer click derecho en cada uno y veremos opciones como `Añadir a un grupo`, `Desactivar la cuenta`, `Resetear (Cambiar) la contraseña`, `Mandar Email`, `Renombrar` o `Propiedades`, dónde podremos cambiar características más avanzadas.

Mediante powershell, también podemos gestionar usuarios y grupos.

| Acción                                                       | Comando                                                                                                                                         |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Crear un nuevo usuario                                       | `New-ADUser -Name "Juan Perez" -SamAccountName jperez -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) -Enabled $true` |
| Buscar un usuario y mostrar todas sus propiedades            | `Get-ADUser jperez -Properties *`                                                                                                               |
| Hacer que la contraseña pueda caducar                        | `Set-ADUser jperez -PasswordNeverExpires $false`                                                                                                |
| Obligar al usuario a cambiar la contraseña al iniciar sesión | `Set-ADUser jperez -ChangePasswordAtLogon $true`                                                                                                |
| Crear un nuevo grupo                                         | `New-ADGroup -Name "IT" -GroupScope Global -GroupCategory Security -Path "OU=Grupos,DC=dominio,DC=local"`                                       |
| Añadir un usuario a un grupo                                 | `Add-ADGroupMember -Identity "IT" -Members jperez`                                                                                              |
| Consultar los miembros de un grupo                           | `Get-ADGroupMember IT`                                                                                                                          |
| Eliminar un usuario de un grupo                              | `Remove-ADGroupMember -Identity "IT" -Members jperez -Confirm:$false`                                                                           |


## 2.2. Autenticación por Kerberos
Nuestro entorno es este:

| Rol           | Hostname | FQDN           | IP          |
| ------------- | -------- | -------------- | ----------- |
| DC            | AD01     | AD01.web.local | 10.10.10.01 |
| Máquina unida | AD02     | AD02.web.local | 10.10.10.02 |

Por tanto, estos nombres deben quedar reflejados en el archivo `/etc/hosts` para la resolución de nombres.

1️⃣ **Configurar Kerberos:** Instalamos los módulos de Kerberos con el paquete `krb5-user` y generamos un archivo `/etc/krb5.conf`.
```bash
nxc smb AD01.dominio.local --generate-krb5-file krb5.conf
```

El archivo tendría este aspecto:
```ini
[libdefaults]
   dns_lookup_kdc = false
   dns_lookup_realm = false
   default_realm = WEB.LOCAL

[realms]
   DOMINIO.LOCAL = {
       kdc = AD01.WEB.local
       admin_server = AD01.WEB.local
   }

[domain_realm]
   .web.local = WEB.LOCAL
   web.local = WEB.LOCAL
```
> Kerberos es especialmente sensible a la **resolución DNS/nombres** y a la **hora del sistema**.

2️⃣ Obtener el TGT del usuario: Podemos obtener un TGT utilizando la contraseña del usuario:
```bash
kinit usuario@web.local
impacket-GetTGT web.local/usuario:pass123 # o asi, obteniendo un archivo .ccache
```
Nos solicitará la contraseña y almacenará el ticket en la caché de credenciales de Kerberos.

3️⃣ Si hemos obtenido el ticket `ccaché`, podemos indicar a las herramientas dónde encontrarlo mediante la variable `KRB5CCNAME` y comprobamos que se haya cargado en memoria
```bash
export KRB5CCNAME=usuario.ccache
klist
```

> [!NOTE]
> Otra forma de autenticarnos mediante Kerberos es utilizar un **keytab**. Un keytab es un archivo que contiene **claves asociadas a una cuenta Kerberos y que permiten obtener un TGT**. Los archivos tienen extensiones `keytab`/`kt`
> 
> El keytab puede utilizarse directamente con `kinit` : `kinit DC01@web.LOCAL -k -t /ruta/svc_workstations.kt` creando el tgt

4️⃣ Listar los tickets en memoria
```bash
klist
```

**5️⃣Autenticarnos en las herramientas**, por ejemplo por SMB:
```bash
impacket-psexec dominio.local/usuario@DC.dominio.local -k -no-pass
```

> Para evitar problemas debemos sincronizar nuestra hora con la de la máquina, con `sudo ntpdate <ip_dc>`.
