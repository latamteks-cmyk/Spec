**📘 Especificación Técnica Detallada Final: `asset-management-service` (Puerto 3010) — Versión 4.0 (BUILD FREEZE READY)**  
**Metodología:** `github/spec-kit`  
**Estado:** `✅ Listo para Build Freeze`  
**Última actualización:** `2025-09-25`  
**Rol:** Sistema central para la gestión del ciclo de vida de activos (hard y soft), planificación de mantenimiento, gestión de órdenes de trabajo, proveedores e inventario, integrado con la plataforma SmartEdify para gobernanza, finanzas y operaciones.  
**Visión de Negocio:** Convertir el mantenimiento de condominios en una experiencia proactiva, transparente y basada en datos, generando valor para los propietarios (tranquilidad, ahorro de costos) y para SmartEdify (ingresos por servicios premium, fidelización, minería de datos).

---

## **🧭 1. Visión y Justificación**

El `asset-management-service` es el sistema operativo técnico del edificio inteligente. Su misión es transformar el mantenimiento reactivo y opaco en un proceso proactivo, predecible y basado en datos.

*   **Para el Cliente (Condominio):** Proporciona transparencia financiera, prevención de fallas, optimización de costos y una experiencia de usuario simple (para residentes y técnicos).
*   **Para SmartEdify:** Es la fuente primaria de datos operativos del edificio. Estos datos alimentan servicios premium (predictivo, consultoría, marketplace) que generan ingresos recurrentes y diferencian a la plataforma.

Este servicio no solo gestiona tickets; **gestiona el valor del activo físico del condominio a lo largo del tiempo**.

---

## **🏗️ 2. Arquitectura y Diseño Global**

### **2.1. Patrones Arquitectónicos Clave**

| Patrón | Implementación | Justificación |
| :--- | :--- | :--- |
| **Microservicio Autónomo** | Node.js + NestJS, PostgreSQL, Redis, Kafka. | Escalabilidad, despliegue independiente, aislamiento de fallos. |
| **Event-Driven Architecture** | Comunicación asíncrona vía Apache Kafka con patrón Outbox. | Desacoplamiento, resiliencia, consistencia eventual con otros servicios (Finanzas, RRHH, Notificaciones). |
| **Multi-Tenant (Shared DB, Shared Schema)** | Discriminador `tenant_id` + Row-Level Security (RLS) activo en todas las tablas. FK compuestas en relaciones críticas. | Eficiencia operativa, escalabilidad a miles de condominios. |
| **CQRS (Command Query Responsibility Segregation)** | Modelos separados para escritura (gestión de activos, OT) y lectura (dashboards, reportes). | Optimización de rendimiento para consultas complejas y agregadas. |
| **Saga Pattern** | Para orquestar flujos de negocio complejos (SOS → Ofertas → Adjudicación → OT → Pago → Evaluación). | Garantiza consistencia en operaciones distribuidas. |
| **Mobile-First & Offline-First** | App móvil con sincronización automática y manejo de conflictos. | Soporte para técnicos que trabajan en áreas sin conectividad. |
| **Query Side para Finanzas** | Tablas de lectura optimizadas que consumen eventos de `finance-service`. | Rendimiento ultra-rápido en reportes financieros sin sobrecargar el servicio de finanzas. |

### **2.2. Diagrama de Contexto (Mermaid)**

```mermaid
graph TD
    subgraph Frontend
        F1[Admin Web] --> G
        F2[Mobile App (Técnico)] --> G
        F3[Resident App] --> G
    end
    subgraph Gateway
        G[API Gateway <br/> Puerto 8080] --> AMS[asset-management-service <br/> Puerto 3010]
    end
    subgraph Core Dependencies
        AMS --> IS[identity-service <br/> 3001] -. Autenticación, Validación QR, DSAR .-> AMS
        AMS --> TS[tenancy-service <br/> 3003] -. Ubicaciones, Áreas .-> AMS
        AMS --> FS[finance-service <br/> 3007] -. Presupuesto, Costeo, Fondos de Reserva .-> AMS
        AMS --> PS[hr-compliance-service <br/> 3009] -. Disponibilidad de Personal, Perfiles .-> AMS
        AMS --> NS[notifications-service <br/> 3005] -. Alertas, Notificaciones, SMS .-> AMS
        AMS --> DS[documents-service <br/> 3006] -. Evidencias, Manuales, Fotos .-> AMS
        AMS --> MKT[marketplace-service <br/> 3015] -. Diagnóstico Especializado, Cotizaciones Express .-> AMS
        AMS --> GS[governance-service <br/> 3011] -. Crear Propuestas de Asamblea .-> AMS
        AMS --> ANL[analytics-service <br/> 3016] -. Simulaciones, Predicciones, Benchmarking .-> AMS
    end
    classDef frontend fill:#4A90E2,stroke:#333,color:white;
    classDef gateway fill:#50E3C2,stroke:#333,color:black;
    classDef service fill:#F5A623,stroke:#333,color:black;
    class F1,F2,F3 frontend
    class G gateway
    class AMS,IS,TS,FS,PS,NS,DS,MKT,GS,ANL service
```

---

## **📦 3. Especificación Funcional Detallada Consolidada y Mejorada**

### **3.1. Gestión Maestra de Activos (Hard y Soft)**

*   **Ficha Técnica Completa (CRUD):** Crear, leer, actualizar y eliminar activos con estructura jerárquica `Sistema > Subsistema > Equipo > Componente`.
*   **Categorización Estratégica:**
    *   **Activos Técnicos (Hard):** Gestionados para **disponibilidad y funcionalidad** (ej: ascensores, bombas). Requieren plan de mantenimiento obligatorio.
    *   **Activos Espaciales (Soft):** Gestionados para **estándar de calidad y presentación** (ej: jardines, lobby). Requieren rondas de inspección obligatorias.
*   **Atributos Clave:**
    *   `marca`, `modelo`, `número de serie`, `fecha de instalación`, `estado operativo`.
    *   `garantia_hasta`: Fecha de vencimiento. El sistema muestra una **alerta prominente** si la garantía está vigente.
    *   `criticidad`: `{A (Crítico), B (Importante), C (Secundario)}`.
    *   `manual_operacion_id`, `manual_mantenimiento_id`: Referencias a `documents-service`.
    *   `fotos`: Array de URLs de fotos del activo.
    *   `metadatos`: Atributos libres personalizables.
*   **Consolidado de Costos por Área:** Proporciona un resumen financiero y operativo a nivel de área (sistema/subsistema), incluyendo activos totales, incidencias abiertas y costo histórico de mantenimiento.

### **3.2. Planificación Preventiva, Predictiva y de Servicios Generales**

*   **Planes de Mantenimiento (Hard):**
    *   Configurables por `tiempo` (frecuencia), `uso` (horas de operación) o `condición` (lecturas de sensores IoT).
    *   Generación automática de tareas y órdenes de trabajo.
    *   Consolidación de tareas para optimizar rutas.
*   **Rondas de Inspección (Soft):**
    *   Creación y gestión de tareas recurrentes y extraordinarias para áreas soft (limpieza, jardinería).
    *   Configuración de frecuencia, horarios fijos o aleatorios.
    *   Checklist digital para inspecciones. Un reporte "No Conforme" genera automáticamente una incidencia.

### **3.3. Gestión de Incidencias y Solicitudes de Servicio (SOS)**

*   **Registro de Incidencias:** Recibe eventos de falla desde app móvil, web, IoT o inspecciones.
*   **Clasificación y Triage:**
    *   **Verificación de Garantía:** Alerta visual prominente si el activo afectado tiene garantía vigente.
    *   **Vinculación de Incidencias:** Permite agrupar múltiples reportes en una "Incidencia Maestra". Notifica automáticamente a todos los reportantes sobre actualizaciones.
    *   **Asignación de Prioridad:** `{Crítico, Urgente, Programable}`.
*   **Integración con Marketplace:** Permite enviar la incidencia al `marketplace-service` para diagnóstico profesional por un técnico especializado.
*   **Creación de SOS:** Transforma la incidencia clasificada en una Solicitud de Servicio formal, definiendo alcance, activos afectados y fecha límite para ofertas.
*   **Gestión Proactiva del Presupuesto:** Antes de aprobar una SOS, consulta en tiempo real al `finance-service` para mostrar el impacto financiero (porcentaje de presupuesto utilizado, saldo restante).

### **3.4. Gestión de Ofertas y Proveedores**

*   **Catálogo de Proveedores:** Gestión de información fiscal y de contacto.
*   **Envío de Invitaciones:** Notifica a proveedores seleccionados vía `notifications-service`.
*   **Recepción y Comparación de Ofertas:** Proporciona un comparador con filtros por precio, plazo y reputación.
*   **Portal de Gestión y Calificación (Vendor Scorecard):** Dashboard con métricas consolidadas por proveedor:
    *   **Calidad:** Promedio de calificaciones de OS cerradas.
    *   **Fiabilidad:** % de OS completadas dentro del plazo.
    *   **Velocidad:** Tiempo promedio de respuesta y ejecución.
    *   **Costo:** Relación con el promedio del mercado.

### **3.5. Gestión de Órdenes de Trabajo (OT) y Ejecución**

*   **Creación de OT:** Manual o automática (desde plan o SOS).
*   **Asignación:** A técnico interno (validando disponibilidad con `hr-compliance-service`) o a proveedor externo.
*   **Permisos de Trabajo de Alto Riesgo:** Bloquea el inicio de la OT hasta que un checklist de seguridad digital sea completado y firmado por el técnico y el supervisor.
*   **Experiencia del Técnico (Mobile-First & Offline):**
    *   **Descarga Proactiva:** El técnico sincroniza y descarga todas las OS asignadas para su jornada.
    *   **Trabajo Offline Completo:** Completa el formulario de cierre (con fotos, comentarios, checklist) sin conexión.
    *   **Sincronización Automática:** Al recuperar la conexión, la app sincroniza automáticamente todos los datos pendientes.
    *   **Validación de Ubicación:** El técnico valida su presencia en el área de trabajo escaneando un QR (emitido por `identity-service`) o seleccionando de una lista.
*   **Control de Calidad y Cierre:** Requiere aprobación del supervisor y del administrador. El residente puede dar feedback opcional.

### **3.6. Inventario de Repuestos y Costeo**

*   **Gestión de Kardex:** Registra entradas (compras) y salidas (usos en OTs) de repuestos.
*   **Niveles Mínimos y Reposición:** Alerta cuando el stock cae por debajo del mínimo y puede generar automáticamente una solicitud de compra.
*   **Costeo e Imputación:** Calcula el costo de los repuestos usados (FIFO o promedio) y lo imputa a la OT y al activo correspondiente, enviando el movimiento a `finance-service`.

### **3.7. Gestión de Excepciones y Emergencias**

*   **Registro de OTs de Emergencia:** Permite crear OTs que omiten el flujo estándar de SOS y adjudicación.
*   **Post-Regularización:** Exige un flujo de aprobación retroactivo para justificar y validar la OT de emergencia.
*   **Flujo de Falla Catastrófica (CAPEX):**
    *   Si un activo no es reparable y los fondos de reserva no son suficientes, el sistema guía al administrador para crear automáticamente una **propuesta de asamblea** (vía `governance-service`) para una cuota extraordinaria.
    *   Consulta al `finance-service` el estado del fondo de reserva y al `compliance-service` las políticas de uso.

---

## **🔄 4. Flujos Operativos Principales (7 Flujos)**

### **Flujo 1: Reporte y Triaje de Incidencia (Residente → Administrador)**

1.  **Residente:** Reporta un problema vía app (foto, video, voz) y selecciona un área común.
2.  **Sistema:** Crea una incidencia en estado “abierta” y notifica al administrador.
3.  **Administrador:** Abre la incidencia, ve la alerta de garantía (si aplica), asigna un activo, una criticidad y decide:
    *   **Gestionar Internamente:** Crea una tarea para personal interno.
    *   **Solicitar Diagnóstico SmartEdify:** Envía la incidencia al `marketplace-service`.
    *   **Crear SOS Directamente:** Inicia el flujo de cotización.

### **Flujo 2: Ejecución de Mantenimiento Preventivo (Sistema → Técnico)**

1.  **Sistema:** Un plan de mantenimiento se activa (por fecha, uso o sensor). Genera una OT.
2.  **Sistema:** Asigna la OT a un técnico disponible (consultando `hr-compliance-service`).
3.  **Técnico:** Recibe la OT en su app móvil, la descarga para trabajar offline, la ejecuta, llena el formulario de cierre y sincroniza.
4.  **Sistema:** Notifica al administrador para aprobación. Una vez aprobada, notifica al residente (si aplica).

### **Flujo 3: Gestión de Solicitud de Servicio (SOS) y Adjudicación (Administrador → Proveedor)**

1.  **Administrador:** Convierte una incidencia en SOS. El sistema consulta a `finance-service` el impacto presupuestario y lo muestra.
2.  **Administrador:** Envía la SOS a proveedores seleccionados.
3.  **Proveedores:** Envían sus ofertas.
4.  **Administrador:** Compara ofertas usando el Vendor Scorecard y adjudica.
5.  **Sistema:** Genera una Orden de Compra (OC) para `finance-service` y una OT técnica.

### **Flujo 4: Ejecución de Orden de Trabajo por Técnico (Técnico → Sistema)**

1.  **Técnico:** Recibe la OT, la descarga, va al lugar y valida su ubicación (QR o lista).
2.  **Si es Alto Riesgo:** Completa y firma digitalmente un checklist de seguridad.
3.  **Técnico:** Ejecuta la tarea, toma fotos “antes/después”, registra la causa raíz y sincroniza.
4.  **Sistema:** Notifica al supervisor y al administrador para aprobación.
5.  **Residente:** Recibe notificación y puede dar feedback.

### **Flujo 5: Inspección y Mantenimiento de Áreas Soft (Supervisor → Personal de Servicios)**

1.  **Sistema:** Genera una “Ronda de Inspección Digital” para un área soft según su frecuencia.
2.  **Personal de Servicios:** Abre la ronda en su app, recorre el área, completa el checklist.
3.  **Si “No Conforme”:** Toma una foto y envía. El sistema crea automáticamente una incidencia programable.
4.  **Supervisor:** Revisa las rondas completadas y aprueba.

### **Flujo 6: Manejo de Emergencias y CAPEX (Administrador → Junta Directiva)**

1.  **Trigger:** Un activo sufre una falla catastrófica.
2.  **Sistema:** Guía al administrador para crear una OT de emergencia.
3.  **Sistema:** Consulta a `finance-service` si hay fondos de reserva suficientes.
4.  **Si NO hay fondos:** El sistema ofrece crear automáticamente una propuesta de asamblea en `governance-service` para una cuota extraordinaria.
5.  **Sistema:** Envía cotizaciones express a 3 proveedores del `marketplace-service` para adjuntar a la propuesta.

### **Flujo 7: Calificación de Proveedores y Optimización (Sistema → Administrador)**

1.  **Trigger:** Una OT es cerrada y aprobada.
2.  **Sistema:** Actualiza automáticamente el “Vendor Scorecard” del proveedor (calidad, fiabilidad, velocidad, costo).
3.  **Sistema:** Si la puntuación de un proveedor cae por debajo de un umbral, lo excluye automáticamente de futuras invitaciones a SOS.
4.  **Administrador:** Recibe una notificación y puede revisar el scorecard en cualquier momento.

---

## **🔌 5. Contrato de API (Endpoints Clave - Consolidado)**

```yaml
# Activos
GET    /api/v1/assets?area={area_name}          # Listar activos por área
GET    /api/v1/assets/{id}                      # Obtener detalle completo de un activo (con costos, historial)
POST   /api/v1/assets                           # Crear un nuevo activo
PATCH  /api/v1/assets/{id}                      # Actualizar un activo

# Planes de Mantenimiento
GET    /api/v1/maintenance-plans/{plan_id}/calendar?view=monthly&start_date=... # Obtener calendario de tareas

# Incidencias
POST   /api/v1/incidents                        # Reportar nueva incidencia
GET    /api/v1/incidents/{id}                   # Obtener detalle de una incidencia
PATCH  /api/v1/incidents/{id}                   # Clasificar/Actualizar incidencia (asignar activo, criticidad, notas)
POST   /api/v1/incidents/{id}/link-master       # Vincular a una incidencia maestra
POST   /api/v1/incidents/{id}/request-smartedify-evaluation # Enviar a marketplace para diagnóstico
POST   /api/v1/incidents/{id}/generate-sos      # Convertir incidencia en SOS

# Órdenes de Trabajo (OT)
POST   /api/v1/work-orders                      # Crear una nueva OT
GET    /api/v1/work-orders/{id}                 # Obtener detalle de una OT
POST   /api/v1/work-orders/{id}/assign          # Asignar OT a técnico/proveedor
POST   /api/v1/work-orders/{id}/complete-safety-checklist # Completar checklist de seguridad (para alto riesgo)
POST   /api/v1/work-orders/{id}/start           # Iniciar OT (valida checklist de seguridad y ubicación QR)
POST   /api/v1/work-orders/{id}/complete        # Marcar OT como completada (sube evidencias)

# Planificación de Personal (Soft Services)
GET    /api/v1/soft-services/personnel/schedule?start_date=... # Obtener planificación semanal de personal

# Proveedores (Vendor Scorecard)
GET    /api/v1/providers/{id}/scorecard         # Obtener tarjeta de puntuación de un proveedor

# Integración con Asambleas (CAPEX)
POST   /api/v1/work-orders/{id}/generate-assembly-proposal # Crear propuesta de asamblea para cuota extraordinaria

# Simulación de Costos (Integración con analytics-service)
POST   /api/v1/analytics/cost-simulation        # Simular impacto financiero de un nuevo plan de mantenimiento
```

---

## **🛡️ 6. Seguridad y Cumplimiento**

*   **Multi-Tenant:** Aislamiento estricto mediante `tenant_id` y RLS en todas las tablas. FK compuestas en relaciones críticas.
*   **Autorización (RBAC/ABAC):** Se basa en políticas de `identity-service`. Ej: Solo `ADMIN` puede crear SOS o aprobar OTs de emergencia.
*   **Auditoría:** Todas las operaciones críticas generan un evento de auditoría publicado en Kafka y almacenado localmente con campos `created_by`, `updated_by`, `trace_id`.
*   **PII:** No almacena datos personales sensibles directamente. Hace referencia a usuarios por su `user_id`.
*   **Validación de Ubicación:** Integración con `identity-service` para emitir y validar tokens contextuales (QR) con DPoP y `ES256/EdDSA`.
*   **DSAR (Eliminación de Datos):** Orquestado por `identity-service`. El servicio responde a eventos `DataDeletionRequested` eliminando o anonimizando datos relacionados con el `user_id`.

---

## **📈 7. Observabilidad y Monitoreo**

*   **Métricas (Prometheus):**
    *   `assets_created_total{tenant}`
    *   `incidents_reported_total{origin, priority}`
    *   `sos_created_total{status}`
    *   `work_orders_completed_total{status, type}`
    *   `inventory_stock_below_min_total{item}`
    *   `vendor_scorecard_calculated_total{provider}`
    *   `api_request_duration_seconds{endpoint, method, status}`
*   **Trazas Distribuidas (OpenTelemetry):** Propaga `traceparent` y añade atributos como `tenant_id`, `user_id`, `asset_id`, `work_order_id`.
*   **Logs Estructurados (JSON):** Todos los logs incluyen `timestamp`, `level`, `message`, `trace_id`, `tenant_id`, `user_id`, y contexto relevante.

---

## **✅ 8. Criterios de Aceptación (Definition of Done - Consolidado)**

Para que el `asset-management-service` se considere **“completo” y listo para producción**, debe cumplir con:

1.  **Funcionalidad Completa:** Todas las historias de usuario del alcance (gestión de activos, planes, incidencias, SOS, OTs, inventario, proveedores, CAPEX) están implementadas y probadas (E2E).
2.  **API y Contratos:** Contrato de API definido en OpenAPI 3.0. Catálogo de eventos Kafka documentado y publicado. Todos los endpoints y eventos son idempotentes.
3.  **Pruebas:** >80% de cobertura de pruebas unitarias e integración. Pruebas E2E para flujos críticos: SOS → Oferta → OT → Cierre → Costeo → Evaluación de Proveedor.
4.  **Observabilidad:** Métricas, logs estructurados y trazas distribuidas implementadas y visibles en dashboards.
5.  **Seguridad:**
    *   RLS y FK compuestas activas y probadas.
    *   Integración con `identity-service` para autenticación, autorización y validación de QR (DPoP, `ES256`).
    *   Auditoría de seguridad completada. Sin vulnerabilidades críticas.
6.  **Experiencia de Usuario (Técnico):** La app móvil funciona en modo offline y sincroniza correctamente al recuperar la conexión. Manejo de conflictos implementado.
7.  **Integraciones:** Todas las integraciones con otros servicios (`finance-service`, `hr-compliance-service`, `marketplace-service`, `governance-service`) están funcionales y probadas.
8.  **Rendimiento:** Tiempos de respuesta P95 ≤ 2s para endpoints clave bajo carga nominal. La carga de dashboards complejos (Vendor Scorecard) ≤ 5s.
9.  **Documentación:** `README.md` con instrucciones claras de despliegue, configuración, uso y contratos de API/eventos.

---

## **🚀 9. Conclusión Final**

La **Versión 4.0** del `asset-management-service` es la especificación técnica final y completa, lista para el **build freeze**. Incorpora y consolida todas las funcionalidades de los documentos de alcance, las mejoras estratégicas y los 7 flujos operativos críticos.

Este servicio no es solo un sistema de tickets, sino una **plataforma estratégica de gestión de activos** que:

*   **Protege la inversión** mediante una gestión proactiva de garantías y presupuestos.
*   **Optimiza recursos** con la consolidación inteligente de incidencias y la planificación eficiente.
*   **Garantiza la seguridad** con checklists digitales obligatorios y validación de ubicación.
*   **Mejora la productividad** con una experiencia móvil offline-first para técnicos.
*   **Aumenta la transparencia** con el Vendor Scorecard y reportes financieros en tiempo real.
*   **Automatiza la gobernanza** al crear propuestas de asamblea para gastos mayores.

Con esta especificación, el equipo de desarrollo tiene una hoja de ruta clara, coherente y alineada con la arquitectura global de SmartEdify. **¡Procedan con total confianza al desarrollo!**
