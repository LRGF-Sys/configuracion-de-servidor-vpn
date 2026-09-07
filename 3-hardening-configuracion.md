# Fase 3: Configuración del Servidor y Políticas de Split Tunneling

Esta sección documenta la creación e inyección del archivo de configuración principal del servicio OpenVPN (`server.ovpn`) en Windows Server 2022. El diseño prioriza la optimización del ancho de banda y la mitigación de latencia mediante directivas explícitas de túnel dividido (*Split Tunneling*).

## 🔀 1. Arquitectura de Enrutamiento y Red (Split Tunneling)

Para evitar que el tráfico general de internet de los usuarios remotos (ubicados en México) viaje innecesariamente hacia el nodo de IONOS en otra región, se omitió la directiva global `redirect-gateway`. En su lugar, se configuraron políticas de enrutamiento selectivo:

* **Enrutamiento Corporativo Exclusivo:** El servicio instruye de forma dinámica a la tabla de enrutamiento del cliente que envíe por el túnel cifrado *únicamente* los paquetes destinados al rango de red privada restringido de la organización (`10.8.0.0/28`).
* **Navegación Local y Consultas DNS:** El tráfico público (redes sociales, navegación web, streaming) se resuelve a través del proveedor de internet (ISP) local de cada colaborador, eliminando la degradación de velocidad transatlántica.

---

## 💻 2. Creación del Archivo de Configuración mediante PowerShell

Ejecute las siguientes instrucciones en una consola de PowerShell con privilegios de Administrador para generar el archivo de configuración activa con codificación estándar.

### A. Generación Automática del Archivo `server.ovpn`
El siguiente script utiliza el cmdlet `Out-File` para crear e inyectar todas las directivas de red, cifrado y seguridad directamente en el directorio restringido de producción.

```powershell
# 1. Definir el bloque de configuración con directivas corporativas
\$ServerConfig = @"
# Puerto y Protocolo de escucha perimetral (Alineado con IONOS y Windows Firewall)
port 1194
proto udp

# Interfaz de red virtual nativa de Windows
dev tun

# Referencias a la Capa Criptográfica (Generada en la Fase 2)
ca "C:\\Program Files\\OpenVPN\\config\\ca.crt"
cert "C:\\Program Files\\OpenVPN\\config\\server.crt"
key "C:\\Program Files\\OpenVPN\\config\\server.key"
dh "C:\\Program Files\\OpenVPN\\config\\dh.pem"
tls-crypt "C:\\Program Files\\OpenVPN\\config\\tls-crypt.key"

# Pool de Direccionamiento IP Privado limitado para los 10 clientes VPN asignados
server 10.8.0.0 255.255.255.240

# Mantener registro de las IPs asignadas dinámicamente a los clientes
ifconfig-pool-persist "C:\\Program Files\(\OpenVPN\log\ipp.\)txt"

# ==========================================
# 🔀 DIRECTIVAS DE SPLIT TUNNELING (NIST Compliant)
# ==========================================
# Se le "empuja" al cliente la ruta exclusiva y limitada de la red interna de la empresa
push "route 10.8.0.0 255.255.255.240"

# Evitar la fuga de DNS corporativo SIN afectar la navegación pública en México
# Reemplaza 'tuempresa.local' por el sufijo de dominio interno real de tu cliente
push "dhcp-option DOMAIN tuempresa.local"
# ==========================================

# Parámetros de Keepalive para detectar caídas de enlace de forma rápida
keepalive 10 120

# Criterios Criptográficos Robustos (ISO 27001 - Control A.8.24)
cipher AES-128-GCM
data-ciphers AES-128-GCM

# Persistencia de llaves e interfaces ante reconexiones
persist-key
persist-tun

# Ubicación de la bitácora técnica de conexiones (Modificable por el usuario svc-openvpn)
status "C:\\Program Files\(\OpenVPN\log\openvpn-\)status.log"
log-append "C:\\Program Files\(\OpenVPN\log\openvpn.\)log"

# Nivel de verbosidad en los logs (3 es el estándar para entornos productivos)
verb 3

# Máximo número de conexiones simultáneas autorizadas corporativamente
max-clients 10
"@

# 2. Escribir el contenido directamente en la ruta de configuración del servicio
\$ServerConfig | Out-File -FilePath "C:\Program Files\OpenVPN\config\server.ovpn" -Encoding ascii

# 3. Validar la creación e integridad del archivo generado
if (Test-Path "C:\Program Files\OpenVPN\config\server.ovpn") {
    Write-Host "[SUCCESS] Archivo server.ovpn generado con directivas de Split Tunneling." -ForegroundColor Green
}
```

### B. Inicialización del Servicio OpenVPN bajo Entorno Seguro
Una vez alojado el archivo `server.ovpn`, el servicio de Windows leerá de manera automática este perfil al arrancar. El comando inicializa el servicio bajo la identidad protegida del usuario `svc-openvpn` que configuramos en la Fase 1.

```powershell
# Iniciar el servicio OpenVPN de forma explícita
Start-Service -Name "OpenVPNService"

# Consultar el estatus de ejecución en tiempo real para verificar un estado "Running"
Get-Service -Name "OpenVPNService" | Select-Object Name, Status, StartType
```
