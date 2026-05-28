# 🔐 Despliegue de NGFW Sophos y Arquitectura de Red Segura

## 📌 Descripción del Proyecto

Este proyecto documenta el diseño y despliegue de un **Firewall de Nueva Generación (NGFW)** con **Sophos XG** como perímetro de seguridad de una organización simulada.

> **Entorno de laboratorio** construido sobre Hyper-V que emula una infraestructura de red corporativa real. Diseñado para demostrar la aplicación práctica de segmentación de red, bastionado perimetral y defensa en profundidad.

La arquitectura se fundamenta en dos principios de seguridad:

- **Defensa en Profundidad** — múltiples capas de seguridad independientes, de modo que el fallo de un control no compromete el entorno completo.
- **Principio de Mínimo Privilegio** — cada zona, usuario y servicio dispone únicamente del acceso estrictamente necesario para operar.

El objetivo fue construir una línea base de seguridad empresarial desde cero: desde el diseño de direccionamiento IP y VLANs hasta políticas de filtrado de tráfico, bastionado administrativo e integración de telemetría de endpoints.

---

## 🏗️ Entorno de Laboratorio

| Componente | Detalle |
|---|---|
| **Hipervisor** | Microsoft Hyper-V |
| **NGFW** | Sophos XG Firewall (VM) |
| **Modelo de red** | Segmentación multizona mediante interfaces lógicas |
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

La principal amenaza que mitiga esta topología es el **movimiento lateral**: si un equipo de la zona LAN es comprometido, el atacante no puede alcanzar los controladores de dominio en la zona SERVIDORES ni pivotar hacia el plano de administración, porque el firewall aplica denegaciones explícitas entre zonas por diseño.

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

## 🛡️ Modelo de Política de Firewall: Allowlist con Denegación Explícita

### Postura de seguridad adoptada

El conjunto de reglas implementa una arquitectura de **Allowlist con Denegación Explícita por zona**, que combina tres mecanismos complementarios:

1. **Reglas de denegación explícita** con criterios concretos para los vectores de mayor riesgo (túneles DNS, tráfico SMTP saliente, países de alto riesgo, acceso inter-zona no autorizado). Estas reglas generan logs con contexto, permitiendo saber exactamente qué flujo intentó cruzar una frontera y por qué fue bloqueado.

2. **Reglas de aceptación mínima** que permiten únicamente el tráfico estrictamente necesario para cada zona, con destinos y servicios definidos explícitamente.

3. **Regla de cierre perimetral** al final de la jerarquía (`DENEGAR TODO DESDE WAN`, #8) que descarta silenciosamente cualquier tráfico entrante desde Internet no contemplado por las reglas anteriores.

Este modelo es más preciso que un Default Deny puro porque las denegaciones explícitas generan **telemetría accionable**: en lugar de un único contador genérico de tráfico descartado, cada regla de denegación registra el vector específico que intentó activarse, facilitando la detección de amenazas y el análisis forense.

### Lógica de procesamiento top-down

El firewall evalúa las reglas de arriba hacia abajo, deteniéndose en la primera coincidencia. La jerarquía está estructurada para priorizar las denegaciones explícitas de mayor riesgo antes que los permisos:

```
[Reglas VPN]                   ← túneles cifrados con prioridad máxima
[Reglas ADMINISTRACIÓN]        ← infraestructura crítica y gestión
[Reglas SERVIDORES]            ← tráfico hacia/desde zona restringida
[Denegaciones explícitas LAN]  ← bloqueo de vectores de riesgo por zona
[Reglas de aceptación LAN]     ← tráfico corporativo permitido
[Reglas HIPERVISORES]          ← capa de virtualización
[Reglas WiFi]                  ← invitados, al final por ser no confiables
[DENEGAR TODO DESDE WAN]       ← cierre perimetral
```

---

## ⚙️ Ingeniería de Tráfico por Zona

### Zona LAN

| Regla | Acción | Razonamiento |
|---|---|---|
| DENEGAR DESDE LAN HACIA DNS (#9) | Descartar | Fuerza la resolución DNS a través de los resolvers internos del firewall. Previene la exfiltración mediante DNS tunnelling. |
| BLOQUEAR PAÍSES DESDE LAN HACIA INTERNET (#10) | Descartar | Geo-restricción activa hacia regiones de alto riesgo (Asia, Middle East, Rusia…). Reduce la superficie de ataque perimetral eliminando destinos estadísticamente asociados a infraestructura C2 y phishing. |
| BLOQUEAR TRÁFICO SMTP DESDE LAN HACIA INTERNET (#14) | Descartar | Impide que un endpoint comprometido actúe como relay de spam o canal de exfiltración vía correo. Todo el correo legítimo se canaliza a través del servidor de correo corporativo. |
| DESDE LAN HACIA INTERNET (#3) | Aceptar | Acceso web corporativo permitido hacia North America y South America. La geo-restricción previa ya ha filtrado destinos de alto riesgo. |
| DESDE LAN HACIA SERVIDORES (#5) | Aceptar | Acceso controlado desde la subred de contabilidad (OR_CONTABILIDAD) hacia la zona SERVIDORES. Tráfico de datos estructurado con destino VM_DATOS. |
| DESDE LAN CONTABILIDAD HACIA ADMINISTRACIÓN (#6) | Aceptar | Acceso SMB restringido desde OR_CONTABILIDAD hacia ADMINISTRACIÓN y VM_ADMINISTRACIÓN. Único flujo LAN → ADMINISTRACIÓN permitido. |
| DENEGAR DESDE LAN HACIA EL RESTO (#4) | Descartar | Denegación explícita de todo tráfico LAN hacia cualquier zona o host no contemplado por las reglas anteriores. Bloquea el movimiento lateral genérico. |

### Zona HIPERVISORES

| Regla | Acción | Razonamiento |
|---|---|---|
| DESDE PMX HACIA INTERNET (#18) | Aceptar | Egreso limitado a tráfico de actualización del sistema operativo e hipervisor. La interfaz de gestión de Proxmox no es alcanzable desde segmentos de usuarios o servidores. |

### Zona WAN (Cierre Perimetral)

| Regla | Acción | Razonamiento |
|---|---|---|
| DENEGAR TODO DESDE WAN (#8) | Descartar | Regla de cierre. Descarta silenciosamente cualquier tráfico entrante desde Internet no autorizado explícitamente por reglas anteriores. |

### Zona WiFi
- Acceso a Internet permitido únicamente mediante NAT.
- Todo acceso a segmentos internos (LAN, SERVIDORES, ADMINISTRACIÓN, HIPERVISORES) bloqueado incondicionalmente → los invitados se tratan como hostiles por diseño.

### Zona ADMINISTRACIÓN
- Acceso completo a todas las zonas internas para operaciones administrativas.
- **El acceso a esta propia zona está restringido** — ver sección de bastionado a continuación.

---

## ⚙️ Configuración Avanzada y Bastionado

### Resolución DNS Segura
Resolvers externos configurados como **Quad9** (`9.9.9.9`) y **Cloudflare** (`1.1.1.1`). Ambos proveedores ofrecen inteligencia de amenazas a nivel DNS (bloqueo de dominios maliciosos) sin coste adicional, añadiendo un filtro en el primer salto contra callbacks de C2 y dominios de phishing.

La regla #9 fuerza toda la resolución DNS de la zona LAN a través de los resolvers internos del firewall, eliminando la posibilidad de usar resolvers alternativos para eludir el filtrado o establecer túneles DNS.

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
Se implementó una política base de VPN SSL. El conjunto de cifrado previsto era **AES-256-GCM** con secreto perfecto hacia adelante (forward secrecy). *La licencia gratuita restringe algunas funciones de VPN; esto se documenta como limitación conocida del entorno de laboratorio, no como una concesión de diseño.*

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

## 💡 Decisiones de Diseño y Concesiones

| Decisión | Razonamiento | Concesión |
|---|---|---|
| Allowlist con Denegación Explícita por zona | Las denegaciones explícitas generan logs accionables por vector; la regla de cierre WAN elimina tráfico no contemplado | Requiere mayor granularidad en el diseño de reglas que un Default Deny puro |
| NGFW único como punto de estrangulamiento | Simplifica la gestión de políticas; todo el tráfico inter-zona es inspeccionable | Punto único de fallo (en producción: par HA) |
| Zona ADMINISTRACIÓN en puerto dedicado | La separación física/lógica impide el acceso admin desde VLANs de usuario | Requiere NIC dedicada o switch con soporte VLAN |
| Geo-restricción activa en zona LAN | Reduce la superficie de ataque eliminando destinos estadísticamente de alto riesgo | Puede bloquear tráfico legítimo hacia regiones restringidas; requiere lista de excepciones |
| DNS forzado a resolvers internos + Quad9/Cloudflare | Elimina el vector de DNS tunnelling y añade filtrado por inteligencia de amenazas | Dependencia del tiempo de actividad de resolvers de terceros |
| Inspección HTTPS habilitada | Elimina el punto ciego del tráfico C2/exfiltración cifrado | Rompe el certificate pinning; requiere confianza en la CA de Sophos en los endpoints |
| Security Heartbeat para aislamiento automático | Reduce el MTTC sin intervención manual | Requiere endpoints gestionados por Sophos (no todos los dispositivos MDM) |

---

## ⚠️ Limitaciones Conocidas del Entorno de Laboratorio

Esta sección documenta las funcionalidades diseñadas en la arquitectura pero no activables con licencia gratuita, junto con cómo se implementarían en un entorno de producción real.

| Funcionalidad | Estado | Limitación | Implementación en Producción |
|---|---|---|---|
| **IPS (Intrusion Prevention System)** | ⚙️ Diseñado | Requiere suscripción **Network Protection** | Política personalizada con Drop en severidad High/Critical, Alert en Medium, aplicada sobre zonas LAN y SERVIDORES |
| **VPN SSL — AES-256-GCM** | ⚙️ Parcial | Algunas opciones de cifrado restringidas en licencia gratuita | Suite completa AES-256-GCM con forward secrecy (PFS) habilitado |

> **Nota sobre el IPS:** Las reglas de firewall están estructuradas para recibir políticas IPS sin modificación — la arquitectura contempla su integración desde el diseño. En producción se aplicaría un ruleset basado en `generalpolicy` clonado y ajustado por zona, priorizando detección de exploits, escaneos de puertos y tráfico C2.

---

## 🔬 Validación y Simulación de Amenazas

Esta sección documenta las pruebas realizadas para verificar que las políticas de seguridad funcionan correctamente en la práctica, no solo en la configuración.

### Prueba 1 — Simulación de Movimiento Lateral: LAN → ADMINISTRACIÓN

**Escenario simulado:** Un atacante compromete un endpoint de usuario en la zona LAN (`WIN10`, `192.168.10.100`, interfaz BOCA1_10) e intenta acceder al panel de administración del firewall (`192.168.40.1`) para escalar privilegios o modificar reglas. Este es uno de los vectores de ataque más comunes en redes corporativas tras una intrusión inicial.

**Resultado esperado:** Bloqueo total. La regla `DENEGAR DESDE LAN HACIA EL RESTO` (#4) debe impedir cualquier acceso desde segmentos de usuario hacia el plano de gestión.

**Resultado obtenido:** ✅ Acceso bloqueado.

![Simulación de Movimiento Lateral](screenshots/lateral-movement.png)

*La imagen muestra en paralelo ambas máquinas: a la izquierda, el endpoint WIN10 (zona LAN) recibe `ERR_CONNECTION_TIMED_OUT` al intentar acceder a `192.168.40.1`. A la derecha, la VM de Administración accede correctamente al panel de Sophos en la misma dirección. Misma IP de destino, dos zonas distintas, dos resultados distintos — validación directa del principio de mínimo privilegio y la microsegmentación.*

**Conclusión:** La arquitectura de zonas impide que un atacante con acceso a la red de usuarios pueda alcanzar el plano de gestión del firewall, eliminando uno de los vectores de escalada de privilegios más críticos en una red corporativa.

---

## 📊 Evidencias de Operación

### 🖥️ Centro de Control (Dashboard Principal)
![Estado General del Sistema](screenshots/ngfw-sophos-panelcontrol.png)
*Vista unificada del estado del NGFW, tráfico por interfaz y resumen de políticas activas. Se confirma la operatividad del entorno y el flujo de tráfico entre las zonas segmentadas.*

### 🚫 Monitoreo de Seguridad (Log Viewer: Deny)
![Tráfico Bloqueado](screenshots/log-viewer-denied.png)
*Evidencia del cumplimiento de la política de denegación explícita por zona. Cada entrada de log identifica el vector específico bloqueado (DNS, SMTP, geo-restricción, movimiento lateral).*

### ✅ Monitoreo de Flujo (Log Viewer: Allow)
![Tráfico Permitido](screenshots/log-viewer-allowed.png)
*Validación de reglas de aceptación en producción.*

### 🛡️ Jerarquía de Reglas del Firewall
![Reglas de Firewall Parte 1](screenshots/ngfw-vpn-admin-servers.png)
![Reglas de Firewall Parte 2](screenshots/ngfw-lan-hyperv.png)

*El firewall utiliza una lógica de procesamiento de arriba hacia abajo (top-down). La lista de reglas está estructurada para priorizar las denegaciones explícitas de mayor riesgo antes que los permisos:*

- **Reglas VPN:** Posicionadas al inicio para garantizar que los túneles cifrados se procesen con prioridad.
- **Reglas de Administración y Servidores:** El tráfico de infraestructura crítica se sitúa por encima de los segmentos de usuario para asegurar un acceso de gestión seguro.
- **Denegaciones explícitas LAN:** DNS tunnelling, SMTP saliente, geo-restricción y movimiento lateral se bloquean antes de evaluar los permisos.
- **Reglas de aceptación LAN e Hipervisores:** Tráfico corporativo autorizado, segmentado por zona y subred.
- **Zonas de Invitados (WiFi):** Posicionadas al final de la jerarquía por su estatus de no confiables.
- **DENEGAR TODO DESDE WAN:** Regla de cierre perimetral que descarta silenciosamente cualquier tráfico entrante no autorizado.

### 🔒 Bastionado del Plano de Gestión
![Administración y Servicios](screenshots/system-administration-hardening.png)
*Implementación del principio de mínimo privilegio en el plano de gestión. El acceso administrativo (HTTPS/SSH) se ha restringido exclusivamente a la zona `ADMINISTRACION`, eliminando la superficie de ataque desde segmentos no autorizados.*

---

## 📚 Conceptos Clave Demostrados

- **Segmentación de red y microsegmentación** mediante zonas de seguridad
- **Arquitectura de firewall con Allowlist y Denegación Explícita por zona**
- **Geo-restricción activa** como reducción de superficie de ataque perimetral
- **Prevención de movimiento lateral** mediante denegaciones explícitas inter-zona
- **Seguridad DNS** (resolución interna forzada, anti-tunnelling)
- **Inspección SSL/TLS** y análisis de tráfico cifrado
- **Respuesta automatizada a incidentes** mediante telemetría Security Heartbeat
- **Bastionado del plano de gestión** y principio de mínimo privilegio
- **Reducción de superficie de ataque** mediante filtrado de egreso y bloqueo de tipos de archivo
- **Validación activa de políticas** mediante simulación de movimiento lateral

---

## 🚀 Despliegue y Replicabilidad

La copia de seguridad completa de la configuración está disponible en el directorio `/backup` de este repositorio.

### Instrucciones de Restauración

1. En el panel de administración de Sophos, ir a **Backup & firmware → Restaurar**.
2. Hacer clic en **Examinar** y seleccionarlo.
3. El backup está **cifrado con AES**. Para obtener la contraseña de descifrado, contactar conmigo a través de GitHub o a través de Linkedin [https://www.linkedin.com/in/miguel-reguero/](https://www.linkedin.com/in/miguel-reguero/)
4. Hacer clic en **Cargar y restaurar**. El sistema aplicará la configuración y se reiniciará.

### Importación Selectiva
Los conjuntos de reglas o configuraciones ACL individuales pueden exportarse/importarse mediante la pestaña **Importar/Exportar** del mismo menú — útil para migrar políticas específicas a otra instancia de Sophos.

---

## 🎓 Contexto

Este proyecto se ha desarrollado con el objetivo de aplicar principios teóricos de seguridad a una infraestructura práctica de laboratorio. La configuración está diseñada para ser auditable y reproducible, sirviendo tanto como artefacto de aprendizaje como demostración de habilidades aplicadas de seguridad de red para mi portfolio profesional.

---

## 📬 Contacto

Preguntas sobre la arquitectura, la contraseña de restauración o detalles de implementación → abrir un issue o contactar a través de Linkedin [https://www.linkedin.com/in/miguel-reguero/](https://www.linkedin.com/in/miguel-reguero/)
