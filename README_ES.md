# 🔐 Arquitectura de Red Segura y Despliegue de NGFW Sophos
### Terra Renewables — Laboratorio de Ciberseguridad Empresarial

> **Entorno de laboratorio** construido sobre Hyper-V que emula una infraestructura de red corporativa real. Diseñado para demostrar la aplicación práctica de segmentación de red, bastionado perimetral y defensa en profundidad.

---

## 📌 Descripción del Proyecto

Este proyecto documenta el diseño y despliegue de un **Firewall de Nueva Generación (NGFW)** con **Sophos XG** como perímetro de seguridad de una organización simulada llamada *Terra Renewables*. La arquitectura se fundamenta en dos principios de seguridad:

- **Defensa en Profundidad** — múltiples capas de seguridad independientes, de modo que el fallo de un control no compromete el entorno completo.
- **Principio de Mínimo Privilegio** — cada zona, usuario y servicio dispone únicamente del acceso estrictamente necesario para operar.

El objetivo fue construir una línea base de seguridad empresarial desde cero: desde el diseño de direccionamiento IP y VLANs hasta políticas de filtrado de tráfico, bastionado administrativo e integración de telemetría de endpoints.

---

## 🏗️ Entorno de Laboratorio

| Componente | Detalle |
|---|---|
| **Hipervisor** | Microsoft Hyper-V |
| **NGFW** | Sophos XG Firewall (VM) |
| **Modelo de red** | Segmentación multisona mediante interfaces lógicas |
| **Nivel de licencia** | Licencia gratuita/Home (limitaciones indicadas donde corresponde) |

La VM de Sophos XG actúa como **puerta de enlace predeterminada** para todos los segmentos, centralizando la traducción de direcciones (NAT), el enrutamiento inter-VLAN y la aplicación de políticas de seguridad en un único punto de inspección.

---

## 🗺️ Diseño de Segmentación de Red

Cada interfaz se mapea a una zona de seguridad aislada. El razonamiento detrás de cada frontera de zona se detalla a continuación.

| Interfaz | Zona | Gateway de Subred | Función | Nivel de Confianza |
|---|---|---|---|---|
| PortA | LAN | `192.168.10.1` | Usuarios corporativos y dispositivos de trabajo | Medio |
| PortC | SERVIDORES | `192.168.30.1` | Controladores de dominio y datos críticos | Alto (restringido) |
| PortD | ADMINISTRACIÓN | `192.168.40.1` | Gestión del firewall y equipo de TI | Máximo |
| PortE | HIPERVISORES | `192.168.50.1` | Capa de virtualización Proxmox VE | Alto (restringido) |
| GuestAP | WiFi | `10.255.0.1` | Acceso a Internet para invitados | No confiable |

### Por qué importa esta segmentación

La principal amenaza que mitiga esta topología es el **movimiento lateral**: si un equipo de la zona LAN es comprometido (p. ej., mediante phishing), el atacante no puede alcanzar los controladores de dominio en la zona SERVIDORES ni pivotar hacia el plano de administración, porque el firewall aplica reglas de denegación explícita entre zonas por defecto.

```
Internet
    │
 [Sophos XG]  ← punto de estrangulamiento único para todo el tráfico inter-zona
    ├── LAN           (usuarios)
    ├── SERVIDORES    (AD, bases de datos)
    ├── ADMINISTRACIÓN (acceso exclusivo para TI)
    ├── HIPERVISORES  (virtualización)
    └── WiFi          (invitados no confiables)
```

---

## 🛡️ Ingeniería de Tráfico y Política de Firewall

El conjunto de reglas se construye sobre una postura de **Denegación por Defecto**: todo el tráfico se descarta salvo que una regla lo permita explícitamente. Las reglas se ordenan de más a menos restrictiva.

### Zona LAN
- Las consultas DNS se fuerzan a través de resolvers internos únicamente → previene la exfiltración mediante **DNS tunnelling**.
- El SMTP saliente está bloqueado en el perímetro → mitiga el riesgo de que un endpoint comprometido se use como **relay de spam** o canal de exfiltración.
- Reglas de microsegmentación bloquean el tráfico LAN → ADMINISTRACIÓN → los usuarios no pueden acceder al panel administrativo ni desde dentro de la red corporativa.

### Zona SERVIDORES
- Filtrado de egreso estricto: el único tráfico saliente permitido es hacia los servidores de actualización de los fabricantes (destinos en lista blanca).
- No se permiten conexiones entrantes desde las zonas LAN o WiFi → evita que un equipo de usuario comprometido inicie conexiones hacia los controladores de dominio.
- *Razonamiento de seguridad: los servidores nunca deben iniciar contacto con zonas de menor confianza; todos los flujos legítimos son entrantes desde fuentes conocidas.*

### Zona HIPERVISORES
- Egreso limitado exclusivamente a fuentes de parches del sistema operativo e hipervisor.
- Todo el tráfico interno bloqueado → la interfaz de gestión de Proxmox no es alcanzable desde segmentos de usuarios o servidores.

### Zona ADMINISTRACIÓN
- Acceso completo a todas las zonas internas para operaciones administrativas.
- **El acceso a esta propia zona está restringido** — ver sección de bastionado a continuación.

### Zona WiFi
- Acceso a Internet permitido únicamente mediante NAT.
- Todo acceso a segmentos internos (LAN, SERVIDORES, ADMINISTRACIÓN, HIPERVISORES) bloqueado incondicionalmente → los invitados se tratan como hostiles por diseño.

---

## ⚙️ Configuración Avanzada y Bastionado

### Resolución DNS Segura
Resolvers externos configurados como **Quad9** (`9.9.9.9`) y **Cloudflare** (`1.1.1.1`). Ambos proveedores ofrecen inteligencia de amenazas a nivel DNS (bloqueo de dominios maliciosos) sin coste adicional, añadiendo un filtro en el primer salto contra callbacks de C2 y dominios de phishing.

### Filtrado Web e Inspección HTTPS
La **Inspección Profunda SSL/TLS** está habilitada para inspeccionar el tráfico cifrado — sin ella, los túneles HTTPS eluden por completo el filtrado de contenido.

Las reglas de **File Protection** bloquean extensiones de alto riesgo en la pasarela:
- `.ps1` (scripts PowerShell — vector habitual de entrega de malware)
- `.iso` (imágenes de disco — utilizadas para eludir las protecciones Mark-of-the-Web en Windows)
- `.exe`, formatos Office con macros habilitadas

*Nota: la inspección HTTPS introduce una dependencia de confianza (la CA de Sophos debe distribuirse a los endpoints). En este laboratorio, esto se gestiona mediante GPO en el dominio.*

### Sophos Security Heartbeat
Security Heartbeat establece un **canal de telemetría** entre el firewall y los endpoints gestionados (a través de Sophos Central). Cuando el estado de salud de un endpoint cambia a **Rojo** (amenaza activa detectada), el firewall automáticamente:
1. Pone en cuarentena el dispositivo bloqueando su movimiento lateral hacia otras zonas.
2. Mantiene el acceso a Internet únicamente para remediación (opcional, configurable).

Esto es un ejemplo de **respuesta automatizada a incidentes** — reduciendo el tiempo medio de contención (MTTC) sin necesidad de intervención manual.

### Configuración VPN SSL
Se implementó una política base de VPN SSL. El conjunto de cifrado previsto era **AES-256-GCM** con secreto perfecto hacia adelante (forward secrecy). *La licencia gratuita restringe algunas funciones de VPN; esto se documenta como limitación conocida del entorno de laboratorio, no como un compromiso de diseño.*

### Bastionado del Plano de Gestión (Principio de Mínimo Privilegio aplicado al propio firewall)

| Método de Acceso | LAN | SERVIDORES | HIPERVISORES | WiFi | ADMINISTRACIÓN |
|---|:---:|:---:|:---:|:---:|:---:|
| HTTPS (panel admin) | ✗ | ✗ | ✗ | ✗ | ✅ |
| SSH | ✗ | ✗ | ✗ | ✗ | ✅ |

La interfaz de gestión del propio firewall es accesible **únicamente desde la zona ADMINISTRACIÓN** (`PortD`). Esto previene el escenario en que un atacante que compromete un equipo de usuario pueda acceder al panel de administración del firewall desde ese mismo segmento.

---

## 📐 Diagrama de Arquitectura

```
                        ┌─────────────────────────────────┐
                        │         SOPHOS XG NGFW           │
                        │  (Default Gateway / IPS / WAF)   │
                        └──────────────┬──────────────────┘
                                       │
           ┌───────────┬───────────────┼───────────────┬───────────┐
           │           │               │               │           │
      ┌────▼────┐ ┌────▼────┐   ┌─────▼────┐   ┌─────▼────┐ ┌────▼────┐
      │   LAN   │ │SERVERS  │   │  ADMIN   │   │HYPERVISOR │ │  WiFi   │
      │.10.0/24 │ │.30.0/24 │   │ .40.0/24 │   │ .50.0/24  │ │10.255/24│
      │         │ │         │   │          │   │           │ │         │
      │ Usuarios│ │  AD DC  │   │ Admins TI│   │ Proxmox   │ │Invitados│
      │Equipos  │ │  Datos  │   │ Gestión FW│  │    VMs    │ │(hostil) │
      └─────────┘ └─────────┘   └──────────┘   └───────────┘ └─────────┘

      [Medio]     [Restringido]  [Máximo]       [Restringido] [No fiable]
```

---

## 💡 Decisiones de Diseño y Compromisos

| Decisión | Razonamiento | Compromiso (trade-off) |
|---|---|---|
| NGFW único como punto de estrangulamiento | Simplifica la gestión de políticas; todo el tráfico inter-zona es inspeccionable | Punto único de fallo (en producción: par HA) |
| Zona ADMINISTRACIÓN en puerto dedicado | La separación física/lógica impide el acceso admin desde VLANs de usuario | Requiere NIC dedicada o switch con soporte VLAN |
| DNS forzado a resolvers externos | Quad9/Cloudflare proporcionan filtrado por inteligencia de amenazas | Dependencia del tiempo de actividad de resolvers de terceros |
| Inspección HTTPS habilitada | Elimina el punto ciego del tráfico C2/exfiltración cifrado | Rompe el certificate pinning; requiere confianza en la CA de Sophos en los endpoints |
| Security Heartbeat para aislamiento automático | Reduce el MTTC sin intervención manual | Requiere endpoints gestionados por Sophos (no todos los dispositivos MDM) |

---

## ⚠️ Limitaciones Conocidas del Entorno de Laboratorio

Esta sección documenta las funcionalidades diseñadas en la arquitectura pero no activables con licencia gratuita, junto con cómo se implementarían en un entorno de producción real.

| Funcionalidad | Estado | Limitación | Implementación en Producción |
|---|:---:|---|---|
| **IPS (Intrusion Prevention System)** | ⚙️ Diseñado | Requiere suscripción **Network Protection** | Política personalizada con Drop en severidad High/Critical, Alert en Medium, aplicada sobre zonas LAN y SERVIDORES |
| **VPN SSL — AES-256-GCM** | ⚙️ Parcial | Algunas opciones de cifrado restringidas en licencia gratuita | Suite completa AES-256-GCM con forward secrecy (PFS) habilitado |

> **Nota sobre el IPS:** Las reglas de firewall están estructuradas para recibir políticas IPS sin modificación — la arquitectura contempla su integración desde el diseño. En producción se aplicaría un ruleset basado en `generalpolicy` clonado y ajustado por zona, priorizando detección de exploits, escaneos de puertos y tráfico C2.

---

## 🚀 Despliegue y Replicabilidad

La copia de seguridad completa de la configuración está disponible en el directorio `/backup` de este repositorio.

### Instrucciones de Restauración

1. En el panel de administración de Sophos, ir a **Backup & firmware → Restaurar**.
2. Hacer clic en **Examinar** y seleccionar `TerraRenewables_Sophos_Core.backup`.
3. El backup está **cifrado con AES**. Para obtener la contraseña de descifrado, contactar a través de GitHub o correo electrónico.
4. Hacer clic en **Cargar y restaurar**. El sistema aplicará la configuración y se reiniciará.

### Importación Selectiva
Los conjuntos de reglas o configuraciones ACL individuales pueden exportarse/importarse mediante la pestaña **Importar/Exportar** del mismo menú — útil para migrar políticas específicas a otra instancia de Sophos.

---

## 📚 Conceptos Clave Demostrados

- **Segmentación de red y microsegmentación** mediante zonas de seguridad
- **Arquitectura de firewall con Denegación por Defecto** (Default Deny)
- **Prevención de movimiento lateral** mediante ACLs inter-zona
- **Seguridad DNS** (resolución interna forzada, anti-tunnelling)
- **Inspección SSL/TLS** y análisis de tráfico cifrado
- **Respuesta automatizada a incidentes** mediante telemetría Security Heartbeat
- **Bastionado del plano de gestión** y principio de mínimo privilegio
- **Reducción de superficie de ataque** mediante filtrado de egreso y bloqueo de tipos de archivo

---

## 📊 Evidencias de Operación

### 🖥️ Centro de Control (Dashboard Principal)
![Estado General del Sistema](screenshots/ngfw-sophos-panelcontrol.png)
*Vista unificada del estado del NGFW, tráfico por interfaz y resumen de políticas activas. Se confirma la operatividad del entorno y el flujo de tráfico entre las zonas segmentadas.*

### 🚫 Monitoreo de Seguridad (Log Viewer: Deny)
![Tráfico Bloqueado](screenshots/log-viewer-denied.png)

### ✅ Monitoreo de Flujo (Log Viewer: Allow)
![Tráfico Permitido](screenshots/log-viewer-allowed.png)

### 🛡️ Firewall Rule Hierarchy
![Firewall Ruleset Part 1](screenshots/ngfw-vpn-admin-servers.png)
![Firewall Ruleset Part 2](screenshots/ngfw-lan-hyperv.png)

*The firewall utilizes a top-down processing logic. The rulebase is structured to prioritize trust and management access:*

* **VPN Rules:** Positioned at the top to ensure encrypted tunnels are processed with priority.
* **Administrative & Server Rules:** Critical infrastructure traffic is placed above user segments to ensure low-latency and secure management access.
* **LAN & Hypervisor Rules:** Standardized corporate traffic follows, segmented by VLAN.
* **Guest Zones:** Positioned lower in the hierarchy due to their 'untrusted' status.
* **Default Deny (Drop All):** The final rule, ensuring that any traffic not explicitly permitted is silently dropped, fulfilling the *Default Deny* architectural requirement.

## 🎓 Contexto

Este proyecto se ha desarrollado en el marco de un **Máster en Ciberseguridad**, con el objetivo de aplicar principios teóricos de seguridad a una infraestructura práctica de laboratorio. La configuración está diseñada para ser auditable y reproducible, sirviendo tanto como artefacto de aprendizaje como demostración de habilidades aplicadas de seguridad de red para un portfolio profesional.

---

## 📬 Contacto

Preguntas sobre la arquitectura, la contraseña de restauración o detalles de implementación → abrir un issue o contactar a través de la información de perfil de GitHub.
