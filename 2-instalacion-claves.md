# Fase 2: Infraestructura de Clave Pública (PKI) y Gestión Criptográfica conforme a ISO/NIST

Esta sección documenta la inicialización de la Autoridad de Certificación (CA) interna, la generación de llaves del servidor y la creación de mecanismos de autenticación de mensajes utilizando **Easy-RSA** y **OpenVPN**. El diseño se alinea con los controles de cifrado de la norma **ISO 27001:2022 (Control A.8.24)** y las directrices de seguridad del **NIST SP 800-77**.

## 🔑 1. Criterios de Seguridad Criptográfica y Cumplimiento

* **Protección de Llaves Raíz (ISO 27001):** La llave privada de la CA (`ca.key`) representa el núcleo de confianza de la infraestructura. En estricto cumplimiento normativo, esta llave se genera **protegida por una frase de contraseña robusta (Passphrase)**. Queda estrictamente prohibido el uso del parámetro `nopass` en la entidad raíz para mitigar riesgos de filtración o duplicación no autorizada del certificado maestro.
* **Criptografía Asimétrica Mínima (NIST SP 800-57):** Se establece una longitud mínima de clave RSA de **2048 bits** para la CA y el servidor, garantizando una resistencia adecuada contra vectores de ataque contemporáneos.
* **Mitigación de Ataques DoS y Escaneos (NIST SP 800-77):** Se incorpora la generación de una clave secreta de autenticación de canal TLS (`tls-crypt`) para firmar digitalmente el saludo inicial (*handshake*), permitiendo que el servidor descarte de forma inmediata paquetes corruptos o escaneos de puertos no autenticados en el perímetro.

---

## 💻 2. Inicialización y Generación Criptográfica mediante PowerShell y Easy-RSA

Ejecute las siguientes instrucciones en una consola de PowerShell con privilegios de Administrador.

### A. Inicialización del Entorno PKI
Se accede al directorio de Easy-RSA y se limpia cualquier estructura residual para garantizar un entorno criptográfico base de alta integridad.

```powershell
# 1. Acceder al directorio nativo de scripts
cd "C:\Program Files\OpenVPN\easy-rsa"

# 2. Inicializar los archivos de plantilla de configuración
.\easyrsa-init.bat

# 3. Construir la estructura limpia del directorio PKI
.\easyrsa.bat init-pki
```

### B. Construcción de la Autoridad de Certificación (CA) Protegida
Se genera el certificado maestro de la organización. **Nota obligatoria de cumplimiento:** El sistema solicitará definir una frase de contraseña para la CA; esta debe ser almacenada bajo políticas estrictas de gestión de credenciales corporativas.

```powershell
# Generar la CA interna (Establecer una passphrase robusta cuando el prompt lo solicite)
.\easyrsa.bat build-ca
```

### C. Generación de Parámetros Diffie-Hellman (DH) y Llave del Servidor
Se configuran los parámetros de intercambio seguro y los certificados del dominio del servidor. Para el servidor VPN, se autoriza el uso de `nopass` exclusivamente para permitir la disponibilidad continua e inicialización automática del servicio en Windows ante reinicios programados de la instancia en IONOS.

```powershell
# 1. Generar los parámetros Diffie-Hellman (Intercambio seguro de llaves de sesión)
.\easyrsa.bat gen-dh

# 2. Generar la solicitud de certificado y llave privada para el servidor VPN (Aislado sin contraseña local)
.\easyrsa.bat gen-req server nopass

# 3. Firmar el certificado del servidor utilizando la CA interna (Solicitará la contraseña de la CA definida en el paso B)
.\easyrsa.bat sign-req server server
```

### D. Generación de Llave de Autenticación de Capa TLS (`tls-crypt`)
Se invoca directamente al binario de OpenVPN para forzar la creación de un token estático de cifrado simétrico que robustece el canal de control.

```powershell
# Generar la clave tls-crypt para la protección del saludo inicial y mitigación de DoS
& "C:\Program Files\OpenVPN\bin\openvpn.exe" --genkey secret .\pki\tls-crypt.key
```

### E. Centralización de Activos Criptográficos bajo Permisos Restringidos
Los archivos públicos y privados indispensables son migrados a la carpeta de configuración activa, la cual se encuentra protegida bajo las listas de control de acceso (ACLs) asignadas en la Fase 1 a la cuenta de servicio local estándar.

```powershell
# Definir ruta destino de configuración del servicio
\$ConfigPath = "C:\Program Files\OpenVPN\config"

# Copiar el pool de llaves y certificados requeridos
Copy-Item ".\pki\ca.crt" \$ConfigPath
Copy-Item ".\pki\dh.pem" \$ConfigPath
Copy-Item ".\pki\tls-crypt.key" \$ConfigPath
Copy-Item ".\pki\issued\server.crt" \$ConfigPath
Copy-Item ".\pki\private\server.key" \$ConfigPath

# Validar la existencia de los 5 archivos indispensables en el directorio de producción
Get-ChildItem \$ConfigPath | Select-Object Name
```
