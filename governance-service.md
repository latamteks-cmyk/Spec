# 📘 **Especificación Técnica: `governance-service` (Puerto 3011) — Versión 2.0**
> **Metodología:** `github/spec-kit`  
> **Versión:** `2.0`  
> **Estado:** `Vision Global - Para inicio del desarrollo spec`  
> **Última Actualización:** `2025-04-05`  
> **Alcance Global:** Plataforma de Gobernanza Comunitaria Internacional para Asambleas Híbridas (Presencial/Virtual/Mixta) con Validación Legal Adaptativa, Moderación Inteligente, Auditoría Inmutable y Soporte para Participación Inclusiva.  
> **Visión Internacional:** Diseñar un sistema jurídicamente agnóstico que se adapte dinámicamente a cualquier marco regulatorio local (Perú, Chile, México, España, Brasil, etc.) mediante el motor de cumplimiento (`compliance-service`), garantizando transparencia, trazabilidad y validez legal universal.
---
## 🧭 **1. Visión y Justificación Global**

El `governance-service` es el **corazón democrático, legal y estratégico** de SmartEdify a nivel global. Su misión es orquestar el ciclo de vida completo de las asambleas de propietarios — desde la iniciativa de convocatoria hasta la generación del acta final — de manera **totalmente digital, legalmente válida, culturalmente inclusiva y estratégicamente escalable**.

Este servicio debe ser **completamente agnóstico a la legislación local**. En lugar de codificar leyes específicas, **delega toda la lógica normativa al `compliance-service`**, que actúa como el "cerebro legal" dinámico. Esto permite que una misma asamblea pueda celebrarse bajo reglas peruanas, españolas o brasileñas, simplemente cambiando el perfil regulatorio del tenant.

**Principios Fundamentales:**
*   **Adaptabilidad Legal:** El servicio no define quórum, mayorías ni flujos. Solo los ejecuta según las reglas proporcionadas por el `compliance-service`.
*   **Inclusión Universal:** Soporta votación digital, presencial (registrada por moderador) y asistida (biometría, SMS), garantizando que nadie quede excluido por barreras tecnológicas.
*   **Auditoría Inmutable:** Cada acción, voto y decisión queda registrada en una cadena de custodia digital (event sourcing) y vinculada criptográficamente a la grabación de video.
*   **Transparencia Radical:** Todos los propietarios pueden verificar la integridad de la grabación y el acta en cualquier momento, mediante un QR o un enlace de auditoría.
*   **Participación Proactiva:** El “Canal de Aportes” permite a la comunidad enviar sugerencias que se consolidan con IA y se presentan como “Puntos Varios”, fomentando una gobernanza más colaborativa.
*   **Estrategia CTO:** Incorpora innovaciones disruptivas (asambleas asíncronas, votación por delegación digital, simulador de impacto) y mejoras de arquitectura (Kafka, Feature Flags, Circuit Breaker) para posicionarse como líder global.

---

## 🏗️ **2. Arquitectura y Diseño Global**

### **2.1. Patrones Arquitectónicos Clave**

| Patrón | Implementación | Justificación |
|--------|----------------|---------------|
| **Microservicio RESTful + WebSocket** | API síncrona para CRUD y orquestación. WebSocket para actualizaciones en tiempo real (quórum, turno de palabra, votos presenciales). | Soporta interacciones en vivo sin bloquear la API. |
| **Event-Driven Architecture** | Emite eventos a **Apache Kafka** (en lugar de RabbitMQ) para desencadenar acciones asíncronas (notificaciones, generación de actas, actualización de dashboards, consolidación de aportes). | Mayor throughput, persistencia y tolerancia a fallos para escalar globalmente. |
| **CQRS + Event Sourcing** | Separación de modelos para escritura (gestión de asambleas) y lectura (dashboards, listados). Eventos inmutables para auditoría legal. | Permite reconstruir el estado de cualquier asamblea para una auditoría forense. |
| **Saga Pattern** | Orquesta flujos complejos: aprobar iniciativa → emitir convocatoria → generar PDF → firmar → generar sello de quórum → grabar video → notificar → consolidar aportes. | Garantiza consistencia en operaciones distribuidas. |
| **Workflow Engine** | Para ejecutar los flujos de aprobación definidos por el `compliance-service` (ej: aprobación por órgano ejecutivo vs. iniciativa ciudadana). | Flexibilidad para adaptarse a cualquier reglamento interno o ley local. |
| **AI Agent Pattern** | El Protocolo de Contexto de Modelo (MCP) redacta borradores de actas mediante NLP y consolida aportes de la comunidad. | Automatiza la tarea más compleja y propensa a error. |
| **Feature Flags (LaunchDarkly)** | Gestión de funcionalidades por tenant, país o porcentaje de usuarios. | Permite despliegues progresivos, pruebas A/B y reducción de riesgos en producción. |
| **Circuit Breaker (Resilience4j)** | Protege las llamadas a servicios dependientes (compliance, documents, streaming). | Mejora la resiliencia y el SLA del sistema ante fallos de terceros. |

### **2.2. Diagrama de Contexto Global (Mermaid)**

```mermaid
graph TD
    subgraph Frontend
        F1[User Web] --> G
        F2[Admin Web] --> G
        F3[Mobile App] --> G
    end
    subgraph Gateway
        G[API Gateway<br/>Puerto 8080] --> GS[governance-service<br/>Puerto 3011]
    end
    subgraph Core Dependencies
        GS --> US[user-profiles-service<br/>3002]
        GS --> TS[tenancy-service<br/>3003]
        GS --> FS[finance-service<br/>3007]
        GS --> CS[compliance-service<br/>3012] -. Define y Valida TODAS las reglas legales .-> GS
        GS --> DS[documents-service<br/>3006]
        GS --> NS[notifications-service<br/>3005]
        GS --> SS[streaming-service<br/>3014]
        GS --> COMM[notifications-service<br/>3005]
    end
    subgraph Async Events
        GS -.-> NS
        SS -.-> NS
        DS -.-> NS
        COMM -.-> NS
    end
    classDef frontend fill:#4A90E2,stroke:#333,color:white;
    classDef gateway fill:#50E3C2,stroke:#333,color:black;
    classDef service fill:#F5A623,stroke:#333,color:black;
    classDef async fill:#D0021B,stroke:#333,color:white;
    class F1,F2,F3 frontend
    class G gateway
    class GS,US,TS,FS,CS,DS,NS,SS,COMM service
    class NS async
```

---

## 📦 **3. Especificación Funcional Detallada (Visión Global)**

### **3.1. Gestión del Ciclo de Vida de la Asamblea**

*   **Crear/Editar/Eliminar Asamblea (Solo Administrador):**
    *   Definir título, descripción, fecha/hora, modalidad (`Presencial`, `Virtual`, `Mixta`, `ASYNC`).
    *   Asignar un código único (ej: `ASM-2025-001`).
    *   Adjuntar documentos relevantes (reglamento, presupuestos).
    *   **NO asignar un moderador designado.** El moderador se elige al inicio de la reunión.
    *   **Configurar reglas de sala:** Duración máxima por intervención, número de ampliaciones, política de micrófonos. Estas reglas se definen en el momento de la creación de la asamblea y se aplican una vez que se designa el moderador.

### **3.2. Flujos de Iniciativa y Emisión de Convocatoria**

*   **Iniciativa de Convocatoria (Creada por cualquier Propietario):**
    *   El propietario crea una `AssemblyInitiative` con un orden del día estructurado:
        *   **Puntos Informativos:** Solo para información, sin votación.
        *   **Puntos a Votación:** Cada punto **debe** tener un `decisionType` (ej: `BUDGET`, `ASSET_DISPOSAL`, `BYLAW_AMENDMENT`). El `compliance-service` inyecta automáticamente el `requiredQuorumPercentage` y `requiredMajorityPercentage`.
    *   El sistema notifica a todos los propietarios para recoger **adhesiones** (no votos).
    *   Cuando se alcanza el porcentaje mínimo definido por el `compliance-service` (ej: 25% de alícuotas), el estado cambia a `“QUOTA_ACHIEVED”`.
*   **Emisión de la Convocatoria Formal (Obligatoria por el Administrador):**
    *   El sistema notifica al Administrador: *“Obligación de emitir la convocatoria en un plazo definido por el `compliance-service`.”*
    *   El Administrador elige la fecha/hora (respetando el plazo mínimo de anticipación definido por el `compliance-service`) y emite la `AssemblyNotice`.
    *   Se inicia la **Saga de Inmutabilidad**: generación de PDF, firma digital, hashing, notificación multicanal.
*   **Flujo de Revisión y Aprobación Validado por Compliance:**
    *   Las convocatorias pueden ser de tipo **Ordinarias (por cronograma)**, **Ordinarias (con aprobación del Presidente)**, o **Extraordinarias (por iniciativa del 25%)**.
    *   El `compliance-service` define y valida el flujo aplicable.
    *   **Loop de Revisión:** El aprobador (Presidente o Administrador) puede enviar comentarios al creador de la iniciativa. El creador puede editar y reenviar la propuesta para una nueva revisión.

### **3.3. Gestión de la Sesión Híbrida (Virtual/Mixta)**

*   **Validación de Asistencia (Múltiples Métodos):**
    *   **QR Dinámico (Optimizado):** El usuario escanea un QR desde la **misma pantalla** usando la cámara del dispositivo. Se usa `jsQR` para detección.
    *   **Biometría (Opcional):** El usuario valida su asistencia con huella dactilar o reconocimiento facial (Touch ID, Face ID, BiometricPrompt).
    *   **Código por SMS/Email (Fallback):** El sistema envía un código de 6 dígitos que el usuario ingresa manualmente.
    *   **Registro Manual por Moderador (Solo en Mixta/Presencial):** El moderador puede registrar manualmente a un asistente presencial, validando su identidad contra el `user-profiles-service`.
    *   **Solo los usuarios validados cuentan para el quórum.**
*   **Moderación Híbrida (Automática + Manual):**
    *   **Automática (Por Defecto):** Los propietarios se unen a una cola FIFO al hacer clic en “Pedir Palabra”. El sistema les da la palabra automáticamente, activando un cronómetro.
    *   **Manual (Intervención del Moderador):** El moderador puede conceder réplicas, ampliar tiempos o silenciar a un participante fuera de la cola.
    *   **Designación del Moderador:** Al inicio de la asamblea, cualquier propietario puede voluntariarse. Si hay múltiples, se elige por sorteo o votación rápida.
*   **Grabación y Sello de Auditoría:**
    *   La sesión se graba y almacena en S3.
    *   Al cerrar la votación, se genera un **“Sello de Quórum”**: un hash criptográfico del estado final de la votación, firmado digitalmente.
    *   Este sello se almacena como metadato del video.
    *   Se genera un **QR de Auditoría** que cualquier propietario puede escanear para verificar la integridad del video y el quórum registrado.

### **3.4. Gestión de Votaciones (Digital y Presencial)**

*   **Votación Digital:**
    *   Los propietarios validados pueden votar desde la app/web.
    *   El voto es ponderado por su alícuota (obtenida del `tenancy-service`).
    *   El `compliance-service` valida en tiempo real si el quórum y la mayoría se han alcanzado.
*   **Votación Presencial (Registrada por Moderador):**
    *   En asambleas mixtas, el moderador puede activar el **“Modo Presencial”**.
    *   Registra manualmente a los asistentes presenciales (validando su identidad).
    *   Para cada punto de votación, el moderador registra el voto del asistente y **adjunta una foto de la papeleta física**.
    *   Estos votos se incluyen en el cálculo de quórum y mayoría.
    *   Las fotos de las papeletas se adjuntan al acta final como evidencia legal.
*   **Votación por Delegación Digital (eProxy):**
    *   Un propietario puede **delegar su voto** a otro propietario (o al administrador) mediante un formulario digital firmado con firma electrónica calificada.
    *   El sistema valida la identidad de ambas partes y registra el poder en la blockchain (opcional).
    *   Durante la votación, el delegado puede votar en nombre del poderdante, y el sistema muestra claramente: *“Voto emitido por [Delegado] en representación de [Poderdante]”*.

### **3.5. Canal de Aportes de la Comunidad**

*   **Envío de Aportes:**
    *   Desde la emisión de la convocatoria hasta 1 hora antes de la asamblea, los propietarios pueden enviar aportes (texto, audio, video) a través del `notifications-service`.
*   **Moderación y Consolidación:**
    *   Los aportes pasan por un filtro automático (palabras clave) y pueden ser revisados manualmente por un moderador humano.
    *   2 horas antes de la asamblea, el Protocolo de Contexto de Modelo (MCP) analiza todos los aportes, los agrupa por temas, elimina duplicados y genera un resumen estructurado.
*   **Incorporación al Orden del Día:**
    *   El `governance-service` crea automáticamente un nuevo punto en el orden del día: **“Puntos Varios: Resumen de Aportes de la Comunidad”**.
    *   Este punto es **informativo (no votable)**.
    *   El resumen se adjunta como un PDF al acta final.

### **3.6. Generación de Actas y Gamificación**

*   **Asistente IA (Protocolo de Contexto de Modelo) para Redacción de Actas:**
    *   Durante la asamblea, el Protocolo de Contexto de Modelo analiza la transcripción (de `streaming-service`) y genera un borrador del acta.
    *   El **moderador o el administrador** edita, aprueba y firma digitalmente el acta (vía `documents-service`).
    *   El acta incluye: lista de asistentes, quórum, resultados de votación, fotos de papeletas (si aplica) y el resumen de aportes de la comunidad.
*   **Gamificación:**
    *   Los usuarios ganan puntos por asistir, votar, comentar, enviar aportes.
    *   Los puntos pueden canjearse por beneficios (descuentos en cuotas, uso de áreas comunes) vía integración con `finance-service`.
    *   Se muestran insignias y rankings en el dashboard.
*   **Simulador de Impacto de Votación:**
    *   Durante la fase de votación, el sistema muestra un **“Simulador en Vivo”**:
        *   *“Si usted vota SÍ, el quórum será del 68% y la propuesta necesitará 2 votos más para aprobarse.”*
        *   *“Si usted vota NO, la propuesta será rechazada por 3 votos.”*
    *   El simulador se actualiza en tiempo real a medida que otros propietarios votan.

### **3.7. Asambleas Asíncronas (Async Governance)**

*   **Creación de Asamblea Asíncrona:**
    *   El administrador puede crear una asamblea con modalidad `ASYNC`.
    *   Se define un período de votación (ej: 72 horas).
*   **Participación Asíncrona:**
    *   Los propietarios pueden votar y comentar en cualquier momento durante el período.
    *   El Protocolo de Contexto de Modelo genera un resumen de los debates y lo incluye en el acta.
*   **Cierre Automático:**
    *   Al finalizar el período, la votación se cierra automáticamente y se genera el acta.

---

## ⚙️ **4. Modelo de Datos Completo (SQL)**

```sql
-- Entidad: Assembly (Asamblea)
CREATE TABLE assemblies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    code TEXT UNIQUE NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    modality TEXT NOT NULL, -- 'PRESENCIAL', 'VIRTUAL', 'MIXTA'
    status TEXT NOT NULL, -- 'DRAFT', 'SCHEDULED', 'IN_PROGRESS', 'CONCLUDED'
    created_by UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: AssemblyInitiative (Iniciativa de Convocatoria)
CREATE TABLE assembly_initiatives (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assembly_id UUID NOT NULL REFERENCES assemblies(id),
    proposed_by UUID NOT NULL REFERENCES users(id),
    status TEXT NOT NULL, -- 'DRAFT', 'COLLECTING_ADHESIONS', 'QUOTA_ACHIEVED', 'NOTICE_EMITTED'
    required_adhesion_percentage NUMERIC NOT NULL, -- Definido por compliance-service
    current_adhesion_percentage NUMERIC NOT NULL DEFAULT 0.0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: AssemblyNotice (Convocatoria Formal)
CREATE TABLE assembly_notices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    initiative_id UUID NOT NULL REFERENCES assembly_initiatives(id),
    issued_by UUID NOT NULL REFERENCES users(id), -- Administrador
    scheduled_date TIMESTAMPTZ NOT NULL,
    pdf_url TEXT,
    hash_sha256 TEXT,
    status TEXT NOT NULL, -- 'DRAFT', 'EMITTED', 'INMUTABLE'
    emitted_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: Proposal (Propuesta a Votación)
CREATE TABLE proposals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assembly_id UUID NOT NULL REFERENCES assemblies(id),
    title TEXT NOT NULL,
    description TEXT,
    decision_type TEXT NOT NULL, -- 'BUDGET', 'ASSET_DISPOSAL', etc. (Validado por compliance-service)
    required_quorum_percentage NUMERIC NOT NULL, -- Inyectado por compliance-service
    required_majority_percentage NUMERIC NOT NULL, -- Inyectado por compliance-service
    status TEXT NOT NULL, -- 'DRAFT', 'IN_VOTING', 'APPROVED', 'REJECTED'
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: DigitalVote (Voto Digital)
CREATE TABLE digital_votes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    proposal_id UUID NOT NULL REFERENCES proposals(id),
    user_id UUID NOT NULL REFERENCES users(id),
    weight NUMERIC NOT NULL, -- Alícuota
    choice TEXT NOT NULL, -- 'YES', 'NO', 'ABSTAIN'
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: ManualVote (Voto Presencial Registrado por Moderador) — ¡NUEVO!
CREATE TABLE manual_votes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    proposal_id UUID NOT NULL REFERENCES proposals(id),
    moderator_id UUID NOT NULL REFERENCES users(id),
    owner_id UUID NOT NULL REFERENCES users(id),
    choice TEXT NOT NULL,
    ballot_photo_url TEXT, -- Foto de la papeleta en S3
    registered_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: AssemblySession (Sesión Virtual/Mixta)
CREATE TABLE assembly_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assembly_id UUID NOT NULL REFERENCES assemblies(id),
    video_conference_link TEXT,
    recording_url TEXT,
    recording_hash_sha256 TEXT,
    quorum_seal TEXT, -- Hash firmado del estado de quórum al cerrar votación
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: SessionAttendee (Asistente Validado) — ¡ACTUALIZADO!
CREATE TABLE session_attendees (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES assembly_sessions(id),
    user_id UUID NOT NULL REFERENCES users(id),
    validation_method TEXT NOT NULL, -- 'QR', 'BIOMETRIC', 'SMS', 'EMAIL', 'MANUAL'
    validation_code TEXT, -- Para métodos SMS/Email
    validated_at TIMESTAMPTZ NOT NULL,
    is_present BOOLEAN NOT NULL DEFAULT true
);

-- Entidad: SpeechRequest (Solicitud de Palabra)
CREATE TABLE speech_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES assembly_sessions(id),
    user_id UUID NOT NULL REFERENCES users(id),
    status TEXT NOT NULL DEFAULT 'PENDING', -- 'PENDING', 'GRANTED', 'COMPLETED'
    requested_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: CommunityContribution (Aporte de la Comunidad)
CREATE TABLE community_contributions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assembly_id UUID NOT NULL REFERENCES assemblies(id),
    user_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    media_type TEXT NOT NULL, -- 'text', 'audio', 'video'
    status TEXT NOT NULL DEFAULT 'PENDING', -- PENDING, APPROVED, REJECTED
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: ContributionSummary (Resumen de Aportes)
CREATE TABLE contribution_summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assembly_id UUID NOT NULL REFERENCES assemblies(id),
    summary_text TEXT NOT NULL,
    topics JSONB, -- Lista de temas con resúmenes
    pdf_url TEXT, -- URL del PDF generado
    generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Entidad: ProxyVote (Votación por Delegación) — ¡NUEVO!
CREATE TABLE proxy_votes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assembly_id UUID NOT NULL REFERENCES assemblies(id),
    grantor_id UUID NOT NULL REFERENCES users(id), -- Propietario que delega
    grantee_id UUID NOT NULL REFERENCES users(id), -- Propietario que recibe el voto
    document_url TEXT, -- PDF del poder firmado
    status TEXT NOT NULL DEFAULT 'ACTIVE', -- ACTIVE, REVOKED, EXPIRED
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL
);

-- Entidad: AsyncAssemblySession (Para Asambleas Asíncronas) — ¡NUEVO!
CREATE TABLE async_assembly_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assembly_id UUID NOT NULL REFERENCES assemblies(id),
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true
);
```

---

## 🔌 **5. Contrato de API Completo (Endpoints Clave)**

```plaintext
# Iniciativas de Convocatoria
POST   /api/v1/initiatives                          # Crear nueva iniciativa
GET    /api/v1/initiatives/{id}                     # Obtener detalles
POST   /api/v1/initiatives/{id}/adhere              # Propietario adhiere a la iniciativa
# Convocatorias Formales
POST   /api/v1/initiatives/{id}/emit-notice         # Administrador emite convocatoria formal
GET    /api/v1/notices/{id}                         # Obtener convocatoria
# Asambleas y Sesiones
POST   /api/v1/assemblies/{id}/start-session        # Iniciar sesión híbrida
GET    /api/v1/sessions/{session_id}/validate-methods # Obtener métodos de validación disponibles
POST   /api/v1/sessions/{session_id}/validate-attendance # Validar asistencia (QR, Biometría, SMS, etc.)
POST   /api/v1/sessions/{session_id}/volunteer-moderator # Voluntariarse como moderador
POST   /api/v1/sessions/{session_id}/elect-moderator # Elegir moderador (solo admin)
# Moderación y Participación
POST   /api/v1/sessions/{session_id}/request-speech # Solicitar palabra (entra en cola FIFO)
POST   /api/v1/sessions/{session_id}/grant-speech/{request_id} # Moderador concede palabra (manual)
POST   /api/v1/sessions/{session_id}/grant-replica/{user_id} # Moderador concede réplica
# Votaciones
POST   /api/v1/proposals/{id}/vote                  # Voto digital
POST   /api/v1/proposals/{id}/manual-vote           # Moderador registra voto presencial (con foto de papeleta) — ¡NUEVO!
GET    /api/v1/proposals/{id}/results               # Obtener resultados en tiempo real
# Canal de Aportes
POST   /api/v1/assembly/{assembly_id}/contributions # Enviar aporte (texto, audio, video)
GET    /api/v1/assembly/{assembly_id}/contributions # Listar aportes (solo admin/moderador)
# Actas y Auditoría
POST   /api/v1/assemblies/{id}/generate-draft       # Generar borrador de acta con IA (MCP)
POST   /api/v1/assemblies/{id}/generate-minutes     # Generar acta final (requiere firma)
GET    /api/v1/sessions/{session_id}/audit-qr       # Obtener QR de auditoría para la sesión
GET    /api/v1/sessions/verify-recording            # Endpoint público para verificar integridad del video
# Votación por Delegación (eProxy) — ¡NUEVO!
POST   /api/v1/proxy-votes                      # Crear un poder de voto
GET    /api/v1/proxy-votes?assembly_id={id}     # Listar poderes para una asamblea
DELETE /api/v1/proxy-votes/{id}                 # Revocar un poder
# Asambleas Asíncronas — ¡NUEVO!
POST   /api/v1/assemblies/{id}/start-async      # Iniciar período de votación asíncrona
GET    /api/v1/assemblies/{id}/async-status     # Obtener estado y tiempo restante
```

---

## 🛡️ **6. Seguridad y Cumplimiento Global**

*   **Delegación Legal:** Toda lógica de quórum, mayoría, plazos y flujos es proporcionada y validada en tiempo real por el `compliance-service`.
*   **Consentimiento para Grabación:** El usuario debe aceptar explícitamente que será grabado, independientemente del método de validación de asistencia usado. Este consentimiento se registra con `timestamp`, `IP` y `device`.
*   **Firma Digital:** Las actas y convocatorias formales se firman digitalmente por usuarios con roles específicos (`ADMIN`, `PRESIDENT`, `SECRETARY`) mediante integración con proveedores locales (Llama.pe, DocuSign, etc.).
*   **Cifrado:** Datos en tránsito (TLS 1.3) y en reposo (AES-256 para videos y documentos en S3).
*   **Auditoría:** Todos los eventos (iniciativa creada, asistencia validada, voto registrado, acta firmada) se almacenan como eventos inmutables para reconstrucción forense.
*   **Feature Flags:** Uso de LaunchDarkly para activar/desactivar funcionalidades por tenant, país o porcentaje de usuarios.
*   **Circuit Breaker:** Implementación de Resilience4j para proteger llamadas a servicios externos.

---

## 📈 **7. Observabilidad y Monitoreo**

*   **Métricas Clave (Prometheus):**
    *   `initiative_created_total`
    *   `notice_emitted_total`
    *   `vote_cast_total` (separar `digital`, `manual`, `proxy`)
    *   `quorum_achieved_ratio`
    *   `attendance_validation_method{method="QR|BIOMETRIC|SMS|EMAIL|MANUAL"}`
    *   `minutes_generated_total`
    *   `contributions_submitted_total`
    *   `proxy_votes_created_total`
    *   `async_assemblies_total`
*   **Trazas Distribuidas (OpenTelemetry):** Para rastrear desde la creación de la iniciativa hasta la generación del acta.
*   **Logs Estructurados (JSON):** Cada log incluye `trace_id`, `user_id`, `tenant_id`, `assembly_id`, `action`.

---

## 💼 **8. Estrategia de Producto y Monetización (Nivel CTO)**

*   **Marketplace de Servicios (`marketplace-service`, Puerto 3015):**
    *   Integrar un **“Marketplace”** donde los administradores puedan contratar servicios legales, de mantenimiento, asesoría, etc.
    *   **Revisión de Actas por Abogado:** Un abogado certificado revisa el acta generada por el Protocolo de Contexto de Modelo y emite un certificado de validez legal.
    *   **Asesoría Legal en Vivo:** Durante la asamblea, un abogado puede unirse como “observador legal” y dar consejos en tiempo real.
    *   **Servicios de Mantenimiento:** Conexión con proveedores de mantenimiento para cotizaciones y gestión de órdenes de trabajo.
*   **SmartEdify Insights (`analytics-service`, Puerto 3016):**
    *   Crear un **dashboard de “Insights”** para administradores y juntas directivas:
        *   *“Tasa de participación por tipo de propietario (residente vs. no residente).”*
        *   *“Temas más votados y su correlación con la satisfacción del propietario.”*
        *   *“Predicción de quórum para la próxima asamblea basada en tendencias históricas.”*
    *   Ofrecer este dashboard como un **módulo premium**.

---

## ✅ **9. Conclusión**

Esta **Versión 4.0.0** del `governance-service` establece las bases para un sistema de gobernanza comunitaria **verdaderamente global, inclusivo, legalmente robusto y estratégicamente avanzado**. Al externalizar toda la lógica normativa al `compliance-service` y al diseñar mecanismos de participación flexibles (digital, presencial, biométrica, asíncrona, por delegación), el servicio está preparado para operar en cualquier jurisdicción del mundo.

La arquitectura prioriza la **trazabilidad absoluta** (event sourcing, sellos criptográficos), la **experiencia de usuario inclusiva** (múltiples métodos de validación, votación asistida, canal de aportes con IA) y la **innovación estratégica** (asambleas asíncronas, marketplace legal, productos de datos), convirtiendo a SmartEdify en la plataforma de referencia para la democracia digital en comunidades residenciales y comerciales a nivel internacional.

---

**© 2025 SmartEdify Global. Todos los derechos reservados.**  
*Documento generado automáticamente a partir de la especificación técnica.*
