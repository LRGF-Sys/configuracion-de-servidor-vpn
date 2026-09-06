# Implementación de Servidor VPN Corporativo con Túnel Dividido

Este repositorio contiene la documentación técnica modular para la instalación, endurecimiento (hardening) y puesta en marcha de un servidor VPN empresarial. La solución permite el acceso seguro a los recursos internos de la organización, optimizando el ancho de banda mediante la implementación de políticas de Split Tunneling.

## 🛠️ Arquitectura de la Solución
* **Infraestructura Cloud:** IONOS Cloud Compute
* **Sistema Operativo:** Windows Server 2022 Standard
* **Software VPN:** OpenVPN Server
* **Algoritmo de Cifrado:** AES-128-GCM

## 📖 Índice de Documentación Técnica
Para facilitar la auditoría y revisión técnica del proyecto, la documentación se divide en los siguientes módulos:

1. [Fase 1: Configuración de Red, Perímetro e Infraestructura Cloud](1-preparacion-servidor.md)
2. [Fase 2: Instalación de OpenVPN y Gestión de Claves/CA](2-instalacion-claves.md)
3. [Fase 3: Configuración del Servidor y Políticas de Split Tunneling](3-hardening-configuracion.md)
4. [Fase 4: Pruebas de Conectividad y Validación](4-pruebas-validacion.md)

## 📊 Impacto y Resultados

* **Mitigación de Latencia y Geobloqueos:** Mediante el despliegue de Split Tunneling, el tráfico público de los colaboradores navega a través de su ISP local (México). Esto eliminó la degradación de velocidad transatlántica y evitó los bloqueos regionales generados por la ubicación geográfica del nodo de IONOS en Europa.

* **Resolución Eficiente de Nombres (DNS):** Se optimizó el enrutamiento de consultas DNS, permitiendo que la navegación general se resuelva de forma local para mejorar el rendimiento, mientras que el tráfico estrictamente corporativo es dirigido de manera segura.

* **Optimización de Infraestructura Cloud:** Al canalizar únicamente el tráfico de negocio por el túnel, se redujo drásticamente el consumo de ancho de banda y la carga de procesamiento en el servidor Windows Server 2022, maximizando el rendimiento de la instancia en IONOS.

* **Seguridad Perimetral y Defensa en Profundidad:** Control de acceso restringido en dos capas independientes: a nivel externo mediante filtrado dinámico (DDNS) en el panel de IONOS, y a nivel interno mediante el firewall de Windows Server con una cuenta de servicio aislada bajo privilegios mínimos.
