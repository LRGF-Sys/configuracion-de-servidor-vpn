# Fase 4: Pruebas de Conectividad, Validación y Auditoría de Bitácoras

Esta sección documenta los procedimientos técnicos obligatorios para auditar, validar y verificar el correcto funcionamiento del servicio VPN. Las pruebas aseguran que las directivas de túnel dividido (*Split Tuning*), el rango acotado de red (`10.8.0.0/28`) y el aislamiento de procesos estén operando en estricto cumplimiento.

## 🔍 1. Protocolo de Validación en el Cliente (Windows 10/11)

Una vez que el usuario se conecta a la VPN, se deben ejecutar los siguientes comandos en la terminal (`cmd` o PowerShell) del colaborador remoto en México para certificar que su tráfico público no se está desviando a otra región.

### A. Verificación del Enrutamiento Selectivo (Split Tunneling)
Para comprobar que el tráfico de internet navega por el ISP local y solo el corporativo va por la VPN, se utiliza el comando `tracert`.

```cmd
# 1. Validar la ruta hacia un servicio público de Internet (Debe salir por el módem local en México)
tracert google.com.mx

# 2. Validar la ruta hacia un recurso interno corporativo (Debe saltar directo al Gateway de la VPN)
tracert 10.8.0.1
```
* **Resultado Esperado:** El rastreo a `google.com.mx` mostrará saltos con latencias bajas (menores a 30ms) correspondientes al proveedor local (Totalplay, Infinitum, etc.). El rastreo a `10.8.0.1` mostrará el salto directo a través de la interfaz virtual TUN hacia IONOS.

### B. Validación del DNS Condicional y Geolocalización
Se audita que las consultas públicas resuelvan de forma local para evitar pérdidas de velocidad o geobloqueos.

```cmd
# 1. Comprobar que los dominios públicos usan el DNS del ISP en México
nslookup google.com

# 2. Comprobar que los dominios internos resuelven exclusivamente mediante el sufijo inyectado
nslookup servidor.empresa.internal
```

---

## 🖥️ 2. Auditoría y Control de Calidad en Windows Server 2022

Las siguientes verificaciones deben realizarse directamente en el servidor alojado en la nube de IONOS utilizando PowerShell con privilegios de Administrador.

### A. Auditoría de Identidad del Proceso (Privilegios Mínimos)
Se comprueba que el servicio OpenVPN se está ejecutando bajo el contexto protegido del usuario local estándar `svc-openvpn` y no bajo `SYSTEM`.

```powershell
# Obtener el identificador de proceso (PID) y el propietario del servicio activo
Get-CimInstance Win32_Process -Filter "Name = 'openvpn.exe'" | Select-Object Name, ProcessId, @{n='Owner';e={\$_.GetOwner().User}}
```
* **Resultado Esperado:** La columna `Owner` debe mostrar explícitamente `svc-openvpn`. Si muestra `SYSTEM` o `Administrator`, el hardening de la Fase 1 falló.

### B. Inspección de Bitácoras de Conexión en Tiempo Real
Para diagnosticar la correcta asignación de IPs (dentro del pool `10.8.0.0/28`) y comprobar que no existan errores de negociación criptográfica TLS o Diffie-Hellman, se leen los últimos registros del archivo log.

```powershell
# Monitorear de forma continua las últimas 20 líneas del log del servicio OpenVPN
Get-Content -Path "C:\Program Files\(\OpenVPN\log\openvpn.\)log" -Tail 20 -Wait
```

### C. Monitoreo de Sesiones Activas y Direcciones IP Persistentes
Se valida qué usuarios remotos están conectados y el estado de la tabla de persistencia del direccionamiento dinámico.

```powershell
# Leer la tabla de asignación de direcciones fijas por sesión
Get-Content -Path "C:\Program Files\(\OpenVPN\log\ipp.\)txt"
```
