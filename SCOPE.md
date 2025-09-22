# 📘 **SmartEdify Global — Especificación Técnica del Sistema**

> **Estado:** `Diseño Final — Listo para Implementación`  
> **Alcance:** Plataforma Global de Gobernanza y Gestión de Condominios Multi-País  
> **Visión:** Convertir a SmartEdify en el sistema operativo digital para comunidades residenciales y comerciales en Latinoamérica y Europa, garantizando cumplimiento legal local, transparencia operativa y participación comunitaria inteligente.

---

## 🧭 **1. Visión General del Sistema**

SmartEdify es una **plataforma SaaS de gobernanza comunitaria** diseñada para digitalizar, automatizar y hacer transparente la gestión de condominios, edificios corporativos y complejos residenciales en múltiples jurisdicciones.

La plataforma combina **gobernanza democrática digital** (asambleas, votaciones, propuestas), **gestión operativa inteligente** (activos, mantenimiento, reservas) y **cumplimiento normativo adaptativo** (laboral, financiero, de seguridad, protección de datos) en un solo ecosistema unificado.

Con una arquitectura de microservicios altamente modular y un motor de cumplimiento legal dinámico, SmartEdify puede adaptarse a las regulaciones locales de cada país sin reescribir el núcleo del sistema.

---

## 🏗️ **2. Arquitectura General del Sistema**

### **2.1. Patrones Arquitectónicos**

| Patrón | Implementación | Justificación |
|--------|----------------|---------------|
| **Microservicios** | 13 servicios independientes, cada uno con su propia base de datos. | Escalabilidad, despliegue independiente, aislamiento de fallos. |
| **API Gateway** | Punto de entrada único para todos los clientes. | Centralización de seguridad, enrutamiento, rate limiting. |
| **Event-Driven** | Comunicación asíncrona vía RabbitMQ. Registro de esquemas en `notifications-service`. | Desacoplamiento, resiliencia, escalabilidad horizontal. |
| **Multi-Tenant** | Modelo: *Shared Database, Shared Schema* con discriminador `condominium_id` + RLS. | Eficiencia operativa, escalabilidad a miles de tenants. |
| **Frontend Monorepo** | Aplicaciones: User Web, Admin Web, Mobile App (React/React Native). | Reutilización de código, consistencia UX, despliegue coordinado. |

### **2.2. Diagrama de Componentes (Alto Nivel)**

```
[User Web]  [Admin Web]  [Mobile App]
       \        |        /
        \       |       /
         \      |      /
          [ API GATEWAY ] (8080)
                 |
    -------------------------------
    |        |        |         |
[identity] [user-profiles] [tenancy] [notifications] ...
    |        |        |         |
[finance] [payroll] [hr-compliance] [asset-management] ...
    |        |        |         |
[governance] [reservation] [physical-security] [compliance] [documents]
```

---

## 📦 **3. Especificación de Microservicios (13 Servicios)**

Cada servicio sigue el estándar `spec-kit`: **Nombre, Puerto, Alcance, Contrato de API (resumen), Eventos Emitidos, Eventos Consumidos, Integraciones Clave.**

---

### **3.1. `gateway-service` (Puerto 8080)**

*   **Alcance:** Punto de entrada único. Enrutamiento, autenticación, rate limiting, CORS.
*   **Contrato API:** Proxy inverso. No expone lógica de negocio.
*   **Eventos:** Ninguno.
*   **Integraciones:** Todos los servicios backend. Valida JWT y extrae `tenant_id`.

---

### **3.2. `identity-service` (Puerto 3001)**

*   **Alcance:** Gestión de identidad digital. Login, registro, MFA, OAuth2/OIDC, RBAC/ABAC.
*   **Contrato API:** `POST /login`, `POST /register`, `GET /me`, `POST /logout`.
*   **Eventos Emitidos:** `user.created`, `user.logged_in`, `role.assigned`.
*   **Integraciones:** `user-profiles-service`, `compliance-service` (para consentimientos ARCO/GDPR).

---

### **3.3. `user-profiles-service` (Puerto 3002)**

*   **Alcance:** Perfiles de usuario, roles por condominio, estructura organizacional (Junta Directiva, Comités).
*   **Contrato API:** `GET /profiles/{user_id}`, `PUT /profiles/{user_id}/roles`.
*   **Eventos Emitidos:** `profile.updated`.
*   **Eventos Consumidos:** `user.created` (de `identity-service`).
*   **Integraciones:** `governance-service` (para validar votantes), `reservation-service` (para validar reservas).

---

### **3.4. `tenancy-service` (Puerto 3003)**

*   **Alcance:** Ciclo de vida de condominios. Unidades, alícuotas, onboarding, configuración dinámica.
*   **Contrato API:** `POST /tenants`, `GET /tenants/{id}/units`, `GET /tenants/{id}/config`.
*   **Eventos Emitidos:** `tenant.created`, `unit.assigned`.
*   **Integraciones:** `finance-service` (para cuotas), `governance-service` (para votación ponderada).

---

### **3.5. `physical-security-service` (Puerto 3004)**

*   **Alcance:** Seguridad física del condominio. CCTV, control de accesos (huella, facial), sensores IoT, protocolos de riesgo (niños en piscina).
*   **Contrato API:** `POST /access/validate`, `GET /cameras/{id}/stream`, `POST /alerts`.
*   **Eventos Emitidos:** `security.breach`, `access.granted`, `risk.detected`.
*   **Integraciones:** `notifications-service` (alertas), `asset-management-service` (cámaras como activos).

---

### **3.6. `notifications-service` (Puerto 3005)**

*   **Alcance:** Envío de notificaciones (email, SMS, push). Registro y validación de esquemas de eventos (Event Schema Registry).
*   **Contrato API:** `POST /send`, `GET /templates`.
*   **Eventos Consumidos:** Cualquier evento del sistema que requiera notificación (ej: `assembly.scheduled`, `maintenance.completed`).
*   **Integraciones:** Gateway a Twilio, SendGrid, FCM. Core para comunicación asíncrona.

---

### **3.7. `documents-service` (Puerto 3006)**

*   **Alcance:** Gestión de documentos legales. Almacenamiento (S3), generación desde plantillas, flujos de firma electrónica (simple y calificada).
*   **Contrato API:** `POST /generate`, `POST /sign`, `GET /{doc_id}/download`.
*   **Eventos Emitidos:** `document.signed`, `document.generated`.
*   **Integraciones:** `governance-service` (actas), `compliance-service` (validación legal de contenido), Llama.pe (firma calificada).

---

### **3.8. `finance-service` (Puerto 3007)**

*   **Alcance:** Gestión financiera. Cuotas de mantenimiento, conciliación bancaria, reportes contables (PCGE, NIIF), impuestos (PLE/SIRE, DIAN, SAT).
*   **Contrato API:** `GET /fees`, `POST /payments`, `GET /reports/balance`.
*   **Eventos Emitidos:** `payment.received`, `quorum.validated`.
*   **Patrón Clave:** **Saga** para transacciones distribuidas (registro de pago → actualización de saldo → notificación → validación de quórum).
*   **Integraciones:** `governance-service`, `compliance-service` (para normas contables por país).

---

### **3.9. `payroll-service` (Puerto 3008)**

*   **Alcance:** Cálculo y procesamiento de nóminas. Generación de PLAME y formatos equivalentes por país.
*   **Contrato API:** `POST /run-payroll`, `GET /payslips/{id}`.
*   **Eventos Emitidos:** `payroll.processed`.
*   **Integraciones:** `finance-service` (costos), `hr-compliance-service` (datos del empleado).

---

### **3.10. `hr-compliance-service` (Puerto 3009)**

*   **Alcance:** Gestión del ciclo de vida del empleado y cumplimiento laboral. Contratos, evaluaciones, SST, comités, reportes de inspección.
*   **Contrato API:** `POST /employees`, `GET /employees/{id}/compliance-status`.
*   **Eventos Emitidos:** `compliance.risk`, `inspection.scheduled`.
*   **Integraciones:** `compliance-service` (para reglas laborales por país), `physical-security-service` (accesos para personal).

---

### **3.11. `asset-management-service` (Puerto 3010)**

*   **Alcance:** Inventario de activos (hard y soft). Órdenes de trabajo (preventivas y correctivas), gestión de proveedores, indicadores de mantenimiento.
*   **Contrato API:** `GET /assets`, `POST /work-orders`, `GET /assets/{id}/maintenance-log`.
*   **Eventos Emitidos:** `work-order.created`, `asset.failure`.
*   **Integraciones:** `reservation-service` (activos = áreas comunes), `finance-service` (costos de mantenimiento).

---

### **3.12. `governance-service` (Puerto 3011)**

*   **Alcance:** Ciclo completo de asambleas. Convocatoria, agenda, votación ponderada en tiempo real, generación de actas, moderación con IA.
*   **Contrato API:** `POST /assemblies`, `POST /assemblies/{id}/vote`, `GET /assemblies/{id}/results`.
*   **Eventos Emitidos:** `assembly.concluded`, `vote.cast`.
*   **Integraciones:** `compliance-service` (validación legal en tiempo real), `documents-service` (generación de actas), `finance-service` (validación de quórum).

---

### **3.13. `reservation-service` (Puerto 3013)**

*   **Alcance:** Gestión de reservas de áreas comunes. Calendario, reglas de uso (horarios, duración, capacidad), validación de conflictos.
*   **Contrato API:** `GET /areas`, `POST /reservations`, `GET /reservations/my`.
*   **Eventos Emitidos:** `reservation.confirmed`, `reservation.cancelled`.
*   **Integraciones:** `asset-management-service` (las áreas son activos), `notifications-service` (recordatorios), `finance-service` (bloqueo por mora).

---

### **3.14. `compliance-service` (Puerto 3012)**

*   **Alcance:** **Motor de Cumplimiento Normativo Global.** Valida reglas legales (financieras, laborales, de asambleas, protección de datos) basadas en el país del tenant y su reglamento interno. Usa un motor de reglas + LLM para interpretación semántica.
*   **Contrato API:** `POST /validate` (envía contexto y recibe veredicto de cumplimiento).
*   **Eventos Consumidos:** Cualquier evento que requiera validación legal (ej: `assembly.created`, `employee.hired`, `fee.calculated`).
*   **Integraciones:** **Todos los servicios.** Es el cerebro legal del sistema.

---

## 🌐 **4. Estrategia Multi-País y Localización**

### **4.1. Motor de Cumplimiento (`compliance-service`)**

*   **Perfiles Regulatorios:** Cada tenant tiene un perfil que define:
    *   País y región.
    *   Tipo de propiedad (residencial, comercial, mixto).
    *   Marco legal aplicable (Ley de Propiedad Horizontal, Código Civil, etc.).
    *   Reglamento Interno cargado como documento estructurado.
*   **Motor de Reglas:** Basado en **Drools**. Las reglas se definen en archivos `.drl` y se almacenan en un repositorio versionado.
*   **Validación con LLM:** Para casos ambiguos o no cubiertos por reglas, se consulta a un LLM (Llama 3, GPT-4) con el contexto legal y el reglamento interno. El LLM devuelve un análisis que el motor interpreta.

### **4.2. Localización de Contenidos**

*   **Traducción de UI:** Usando `i18next` en los frontends.
*   **Formatos de Fecha/Moneda:** Configurables por tenant.
*   **Documentos Legales:** Plantillas de actas, contratos y comunicaciones adaptadas por jurisdicción, generadas por `documents-service`.

---

## 🛡️ **5. Seguridad y Cumplimiento**

### **5.1. Seguridad de la Información**

*   **Autenticación:** OAuth2/OIDC + MFA (TOTP/WebAuthn).
*   **Autorización:** RBAC/ABAC con políticas dinámicas.
*   **Cifrado:** AES-256 en reposo, TLS 1.3 en tránsito.
*   **PII:** Minimización de datos. Consentimientos explícitos gestionados por `compliance-service`.

### **5.2. Cumplimiento Normativo**

*   **Protección de Datos:** Cumple con GDPR (Europa), LGPD (Brasil), Ley 29733 (Perú) y equivalentes. Derechos ARCO implementados.
*   **Firma Electrónica:** Integración con proveedores locales acreditados (Llama.pe, DocuSign, etc.) para validez legal.
*   **Auditoría:** Trazas inmutables de todas las operaciones críticas (event sourcing).

---

## 🚀 **6. Infraestructura y Operaciones**

### **6.1. Stack Tecnológico**

*   **Backend:** Node.js + NestJS (TypeScript).
*   **Frontend:** React + React Native + TypeScript.
*   **Base de Datos:** PostgreSQL (por servicio) + RLS.
*   **Mensajería:** RabbitMQ.
*   **Almacenamiento:** AWS S3.
*   **Infraestructura:** Docker + Kubernetes (EKS/GKE) + AWS.
*   **Observabilidad:** Prometheus + Grafana + OpenTelemetry + ELK.

### **6.2. Despliegue y CI/CD**

*   **GitOps:** ArgoCD para despliegues automáticos.
*   **Feature Flags:** LaunchDarkly para activar funcionalidades por tenant o país.
*   **Canary Releases:** Despliegues graduales.
*   **Contrato de Observabilidad:** Todos los servicios deben emitir métricas, trazas y logs estructurados con `trace_id`.

---

## 📈 **7. Métricas de Éxito y KPIs**

| Área | KPI | Objetivo |
|------|-----|----------|
| **Adopción** | % de propietarios activos mensuales | > 70% |
| **Gobernanza** | % de asambleas realizadas digitalmente | > 90% |
| **Operaciones** | Tiempo promedio de cierre de órdenes de trabajo | < 48h |
| **Cumplimiento** | % de reglas legales validadas automáticamente | > 95% |
| **Satisfacción** | NPS de residentes y administradores | > 50 |

---

## 📅 **8. Hoja de Ruta (Roadmap) — Visión Global**

*   **Fase 1:** Lanzamiento  (MVP: `governance`, `reservation`, `asset-management` + `compliance` básico), con los soportes necesarios minimos
*   **Fase 2:** Completar `compliance-service` y `finance-service`.
*   **Fase 3:** Desarrollar 'payroll-service', 'hr-compliance-service'
*   **Fase 4:** Implementar el 100% de los microservicios
*   **Fase 5:** Retroalimentacion y mejoras

---

## ✅ **9. Conclusión**

Esta especificación, alineada con la metodología `spec-kit`, define a SmartEdify como una **plataforma global, resiliente y legalmente adaptable**. La arquitectura de 13 microservicios, coronada por el `compliance-service`, permite una expansión ágil y segura a nuevos mercados, convirtiendo los desafíos regulatorios en una ventaja competitiva insuperable.
La plataforma no solo digitaliza procesos; **reinventa la forma en que las comunidades se gobiernan, operan y cumplen con la ley en un mundo multi-jurisdiccional.**

---

**© 2025 SmartEdify Global. Todos los derechos reservados.**  
*Documento generado automáticamente a partir de la especificación técnica. Última actualización: 2025-04-05.*
