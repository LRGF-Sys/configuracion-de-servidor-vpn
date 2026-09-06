# Fase 1: Despliegue del Software y Hardening del Sistema Operativo

Esta sección documenta las buenas prácticas para la obtención e instalación del servidor VPN, así como las directivas de seguridad locales aplicadas sobre Windows Server 2022 para garantizar un entorno aislado bajo el principio de privilegios mínimos.

## 📥 1. Descarga e Instalación Segura (Buenas Prácticas)

Para un despliegue en entornos de producción, la obtención e instalación del software debe seguir criterios estrictos de integridad:

* **Origen de los Binarios:** El instalador MSI de 64 bits se obtiene directamente de los servidores de distribución oficial de OpenVPN Community (`swupdate.openvpn.org`), asegurando el uso de la última versión estable (Compilación de producción con soporte a largo plazo).
* **Instalación Desatendida (Silent Install):** En servidores de infraestructura cloud como IONOS, se ejecuta una instalación silenciosa mediante la herramienta nativa de Windows `msiexec.exe` utilizando los parámetros `/qn /norestart`. Esto garantiza un despliegue limpio, sin interfaces gráficas innecesarias, y asegura que tanto el servicio del sistema como las herramientas criptográficas de Easy-RSA se registren con sus rutas estándar de la industria (`C:\Program Files\OpenVPN`).

---

## 🖥️ 2. Hardening e Infraestructura en Windows Server 2022 mediante PowerShell

Una vez que el software se encuentra alojado en el sistema, ejecute las siguientes instrucciones en una consola de PowerShell con privilegios de Administrador para configurar la seguridad local y el aislamiento del proceso.

### A. Configuración de Reglas de Entrada en Windows Defender Firewall
Se habilita de forma explícita el puerto de escucha perimetral para procesar el tráfico cifrado de la aplicación, bloqueando cualquier otro intento de escaneo en puertos de red locales no autorizados.

```powershell
New-NetFirewallRule -DisplayName "OpenVPN Server Inbound" `
    -Direction Inbound `
    -Action Allow `
    -Protocol UDP `
    -LocalPort 1194 `
    -Program "C:\Program Files\OpenVPN\bin\openvpn.exe" `
    -Profile Public, Private `
    -Description "Permite la entrada de tráfico cifrado para usuarios remotos VPN"
```

### B. Creación de Cuenta de Servicio Dedicada (Aislamiento de Procesos)
Se genera un usuario local estándar con restricciones nativas en el sistema operativo. Su única finalidad será la identidad del proceso en ejecución, evitando el uso de la cuenta administrativa global del sistema (`SYSTEM`).

```powershell
# Definir una credencial segura para la cuenta de servicio aislada
\$Password = ConvertTo-SecureString "TuContrasenaSeguraAqui123!" -AsPlainText -Force

# Instanciar el usuario estándar sin membresía en grupos administrativos
New-LocalUser -Name "svc-openvpn" `
    -Password $Password `
    -Description "Cuenta de servicio dedicada para la ejecución aislada de OpenVPN Server" `
    -PasswordNeverExpires $true `
    -UserMayChangePassword \$false
```

### C. Asignación de Derechos de Seguridad Local (Iniciar sesión como servicio)
Se inyecta el SID del nuevo usuario dentro de las directivas de seguridad local para permitir el arranque de subprocesos en segundo plano sin delegar facultades interactivas.

```powershell
# Exportar la política de seguridad actual a un archivo temporal de configuración
secedit /export /cfg c:\secpol.cfg

# Inyectar el privilegio SeServiceLogonRight al usuario dedicado
(Get-Content c:\secpol.cfg) -replace "SeServiceLogonRight = ", "SeServiceLogonRight = svc-openvpn," | Set-Content c:\secpol.cfg

# Reconfigurar y aplicar la base de datos de seguridad del sistema
secedit /configure /db \$env:windir\security\local.sdb /cfg c:\secpol.cfg /areas USER_RIGHTS

# Limpieza del archivo de configuración temporal
Remove-Item c:\secpol.cfg
```

### D. Configuración de Listas de Control de Acceso (ACLs) del Sistema de Archivos
Se restringen los privilegios sobre las carpetas de la aplicación de tal manera que la cuenta de servicio pueda únicamente leer las configuraciones y tenga permisos de modificación exclusivos en el directorio log técnico.

```powershell
# 1. Permisos de Lectura y Ejecución en el directorio principal de instalación
\$AclPrincipal = Get-Acl "C:\Program Files\OpenVPN"
\$ArPrincipal = New-Object System.Security.AccessControl.FileSystemAccessRule("svc-openvpn", "ReadAndExecute", "ContainerInherit, ObjectInherit", "None", "Allow")
\$AclPrincipal.SetAccessRule(\$ArPrincipal)
Set-Acl "C:\Program Files\OpenVPN" \$AclPrincipal

# 2. Permisos de Modificación/Escritura EXCLUSIVOS en la carpeta de trazas de conexión (Logs)
\$AclLogs = Get-Acl "C:\Program Files\(\OpenVPN\log\\)"
\$ArLogs = New-Object System.Security.AccessControl.FileSystemAccessRule("svc-openvpn", "Modify", "ContainerInherit, ObjectInherit", "None", "Allow")
\$AclLogs.SetAccessRule(\$ArLogs)
Set-Acl "C:\Program Files\(\OpenVPN\log\\)" \$AclLogs
```

### E. Migración del Contexto de Arranque del Servicio OpenVPN
Finalmente, se reasigna el servicio del demonio del sistema para que deje de operar bajo los máximos privilegios de Windows y se inicie exclusivamente bajo el perfil protegido del usuario local estándar.

```powershell
# Configurar las propiedades de inicio de sesión del servicio de OpenVPN
sc.exe config "OpenVPNService" obj= ".\svc-openvpn" password= "TuContrasenaSeguraAqui123!"

# Forzar el reinicio para validar la inicialización correcta bajo el nuevo contexto de seguridad
Restart-Service -Name "OpenVPNService" -Force
```
