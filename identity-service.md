📘 Especificación Técnica: identity-service (Puerto 3001) — Versión 2.0

> Metodología: github/spec-kit
Estado: Listo para desarrollo
Última actualización: 2025-09-22
Ámbito: Proveedor central de identidad, autenticación, autorización y sesiones en entorno multi‑tenant con cumplimiento GDPR/ARCO y Ley Nº 29733.
Renombrado: Este servicio reemplaza a auth-service y mantiene compatibilidad retroactiva controlada.




---

🧭 1. Visión y Principios

identity-service autentica usuarios, emite y gestiona tokens, aplica autorización (RBAC/ABAC), ofrece MFA, actúa como Authorization Server OAuth 2.0 y Proveedor OpenID Connect. Se integra con users-service para atributos de perfil y con el API_GATEWAY para validación perimetral.

Principios:

Zero‑Trust + Multi‑tenant: Toda operación exige tenant_id. Aislamiento con RLS a nivel de datos.

Estándares first: OAuth 2.1 (cuando aplique), OIDC, PKCE, DPoP/MTLS, WebAuthn/FIDO2, JOSE, JWKS, Argon2id.

Seguridad adaptativa: Evaluación de riesgo y step‑up a MFA.

Criptografía gestionada: Firmas asimétricas (RS256/ES256) con rotación por HSM/Vault. Publicación /.well-known/jwks.json.

Auditoría inmutable: Bitácora con hash encadenado y exportación a SIEM.

DX clara: OpenAPI, SDKs y sandbox.



---

🏗️ 2. Arquitectura y Diseño

2.1 Patrones

Área	Decisión	Motivo

API síncrona	REST JSON sobre TLS 1.3	Interoperabilidad
AuthZ perimetral	Middleware en API_GATEWAY validando JWT vía JWKS	Rendimiento y coherencia
Estado de sesión	Refresh tokens + Redis para administración de sesiones	Revocación y logout global
Eventos	Kafka/NATS: user.created, role.changed, token.revoked	Sincronía entre servicios
Riesgo	Motor ABAC + señales de security-service	Step‑up dinámico
Claves	HSM/Vault, rotación en caliente, JWKS versionado	Seguridad operativa


2.2 Diagrama de contexto (Mermaid)

graph TD
  subgraph Clients
    W[Web] --> GW
    M[Mobile] --> GW
    S[Service to Service] --> GW
  end
  GW[API_GATEWAY:8080] --> ID[identity-service:3001]
  ID --> US[users-service:3002]
  ID --> TE[tenants-service:3003]
  ID --> SE[security-service:3004]
  ID --> CO[communications-service:3005]
  ID --> EV[(Kafka/NATS)]
  ID --> RS[(Redis)]
  ID --> DB[(PostgreSQL)]
  ID --> VA[(HSM/Vault)]

2.3 Compatibilidad y deprecación

Mantener rutas /auth/* por 90 días con avisos de deprecación.

Nuevas rutas bajo /identity/*. Feature flag por tenant.



---

📦 3. Especificación funcional

3.1 Gestión de usuarios y credenciales

Registro: validación de correo, contraseña, políticas y Términos. Alta enlaza con users-service.

Login con Argon2id y política de rehash automático.

Forgot/Reset password con tokens de un solo uso y expiración configurable.

Bloqueo por intentos fallidos y enfriamiento exponencial.


3.2 Tokens y sesiones

Access tokens JWT firmados (RS256 o ES256). TTL por defecto 15 min.

Refresh tokens opacos ligados a sesión en Redis. TTL por defecto 30 días. Rotación en cada uso.

Logout global y revocación por jti.

Publicación de JWKS y metadatos OIDC.


3.3 Autorización (RBAC + ABAC)

RBAC: roles y permisos por recurso/acción. Scopes OAuth por servicio.

ABAC: atributos de contexto (IP, geolocalización, dispositivo, nivel de MFA, riesgo). Política de step‑up.


3.4 MFA

TOTP: enrolamiento con QR, verificación y recuperación con códigos de respaldo.

WebAuthn/FIDO2: registro y autenticación sin contraseña o como segundo factor.


3.5 OAuth 2.0 / OIDC

Endpoints: /oauth/authorize, /oauth/token, /oauth/introspect, /oauth/revoke, /oauth/device, /oauth/userinfo.

Flujos: Authorization Code + PKCE, Client Credentials, Device Code, Token Exchange.

DPoP/MTLS para clientes confidenciales. Tokens DPoP‑bound.

OIDC: /.well-known/openid-configuration, /.well-known/jwks.json, id_token con sub, email, name, tenant_id.


3.6 Cliente y consentimiento

Registro de clientes: client_id, secreto o MTLS/DPoP, redirect_uris, scopes permitidos y políticas de expiración.

Pantalla de consentimiento por scopes. Registro y revocación de consentimientos.


3.7 Auditoría y observabilidad

Bitácora inmutable: login/logout, MFA, cambios de rol, fallos, revocaciones. Hash‑chain.

Métricas Prometheus y trazas OpenTelemetry.

Alertas de seguridad y anormalidades.


3.8 Cumplimiento y privacidad

ARCO/GDPR: derecho de acceso, rectificación, cancelación y oposición. Derecho al olvido por tenant.

Cifrado en tránsito y reposo. Minimización de datos en tokens. Pseudonimización donde aplique.



---

🗄️ 4. Modelo de datos (SQL)

> Nota: los perfiles de usuario residen en users-service. Aquí se almacenan credenciales, sesiones, clientes, claves y políticas.



-- Tenants
CREATE TABLE tenants (
  id UUID PRIMARY KEY,
  code TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Usuarios (referencia externa)
CREATE TABLE identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  user_ref UUID NOT NULL, -- id en users-service
  email CITEXT NOT NULL,
  email_verified BOOLEAN NOT NULL DEFAULT FALSE,
  status TEXT NOT NULL DEFAULT 'ACTIVE', -- ACTIVE, LOCKED, DISABLED
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (tenant_id, email)
);

-- Credenciales de contraseña
CREATE TABLE credentials_password (
  identity_id UUID PRIMARY KEY REFERENCES identities(id) ON DELETE CASCADE,
  password_hash TEXT NOT NULL,
  algo TEXT NOT NULL DEFAULT 'argon2id',
  params JSONB NOT NULL, -- memory, iterations, parallelism
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- MFA TOTP
CREATE TABLE mfa_totp (
  identity_id UUID PRIMARY KEY REFERENCES identities(id) ON DELETE CASCADE,
  secret_enc BYTEA NOT NULL, -- cifrado
  recovery_codes TEXT[] NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  disabled_at TIMESTAMPTZ
);

-- WebAuthn
CREATE TABLE webauthn_credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id) ON DELETE CASCADE,
  credential_id BYTEA NOT NULL,
  public_key BYTEA NOT NULL,
  sign_count BIGINT NOT NULL DEFAULT 0,
  transports TEXT[],
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (identity_id, credential_id)
);

-- Clientes OAuth/OIDC
CREATE TABLE oauth_clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  client_id TEXT UNIQUE NOT NULL,
  client_secret_hash TEXT, -- nulo si MTLS/DPoP
  token_endpoint_auth_method TEXT NOT NULL, -- client_secret_basic, tls_client_auth, private_key_jwt, dpop
  redirect_uris TEXT[] NOT NULL,
  grant_types TEXT[] NOT NULL,
  scopes TEXT[] NOT NULL,
  dpop_required BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Consentimientos
CREATE TABLE oauth_consents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id) ON DELETE CASCADE,
  client_id UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
  scopes TEXT[] NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_at TIMESTAMPTZ
);

-- Sesiones y refresh tokens
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id) ON DELETE CASCADE,
  client_id UUID REFERENCES oauth_clients(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  refresh_token TEXT UNIQUE NOT NULL,
  refresh_expires_at TIMESTAMPTZ NOT NULL,
  user_agent TEXT,
  ip INET,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_at TIMESTAMPTZ
);

-- Revocaciones por jti
CREATE TABLE token_revocations (
  jti UUID PRIMARY KEY,
  revoked_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  reason TEXT
);

-- Claves de firma (KMS/HSM)
CREATE TABLE signing_keys (
  key_id TEXT PRIMARY KEY,
  alg TEXT NOT NULL, -- RS256/ES256
  public_jwk JSONB NOT NULL,
  status TEXT NOT NULL, -- ACTIVE, RETIRING, RETIRED, REVOKED
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  retired_at TIMESTAMPTZ
);

-- Roles y permisos
CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  code TEXT NOT NULL,
  description TEXT,
  UNIQUE (tenant_id, code)
);

CREATE TABLE permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT UNIQUE NOT NULL,
  description TEXT
);

CREATE TABLE role_permissions (
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
  PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE identity_roles (
  identity_id UUID REFERENCES identities(id) ON DELETE CASCADE,
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  PRIMARY KEY (identity_id, role_id)
);

-- Políticas ABAC
CREATE TABLE abac_policies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name TEXT NOT NULL,
  expression JSONB NOT NULL, -- CEL/Rego equivalente serializado
  effect TEXT NOT NULL -- ALLOW/DENY
);

-- Auditoría encadenada
CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  tenant_id UUID,
  identity_id UUID,
  action TEXT NOT NULL,
  data JSONB,
  prev_hash BYTEA,
  curr_hash BYTEA NOT NULL
);


---

🔌 5. Contrato de API

# Registro y autenticación
POST   /identity/v1/register
POST   /identity/v1/login
POST   /identity/v1/forgot-password
POST   /identity/v1/reset-password
POST   /identity/v1/logout
POST   /identity/v1/token/revoke            # por jti

# MFA
POST   /identity/v1/mfa/totp/setup
POST   /identity/v1/mfa/totp/verify
DELETE /identity/v1/mfa/totp
POST   /identity/v1/mfa/webauthn/register/options
POST   /identity/v1/mfa/webauthn/register/finish
POST   /identity/v1/mfa/webauthn/authenticate/options
POST   /identity/v1/mfa/webauthn/authenticate/finish

# OAuth 2.0 / OIDC
GET    /.well-known/openid-configuration
GET    /.well-known/jwks.json
GET    /oauth/authorize
POST   /oauth/token
POST   /oauth/introspect
POST   /oauth/revoke
POST   /oauth/device/code
POST   /oauth/token/exchange
GET    /oauth/userinfo

# Clientes y consentimiento
POST   /identity/v1/clients                 # interno/admin
GET    /identity/v1/clients/{id}
PUT    /identity/v1/clients/{id}
DELETE /identity/v1/clients/{id}
POST   /identity/v1/consents
GET    /identity/v1/consents?client_id=...
DELETE /identity/v1/consents/{id}

# Roles/Permisos
POST   /identity/v1/roles
GET    /identity/v1/roles
POST   /identity/v1/roles/{id}/permissions
GET    /identity/v1/permissions
POST   /identity/v1/identities/{id}/roles
DELETE /identity/v1/identities/{id}/roles/{roleId}

# Políticas ABAC
POST   /identity/v1/abac/policies
GET    /identity/v1/abac/policies
PUT    /identity/v1/abac/policies/{id}
DELETE /identity/v1/abac/policies/{id}

# Utilidades
GET    /identity/v1/health
GET    /metrics

Respuestas de error estandarizadas

{ "error": "invalid_request", "error_description": "..." }


---

🛡️ 6. Seguridad

Hash de contraseñas: Argon2id. Parámetros configurables. Rehash automático.

Firmas JWT: RS256/ES256. kid obligatorio. aud, iat, exp, jti, tenant_id y scope.

Rotación de claves: Estados ACTIVE/RETIRING/RETIRED/REVOKED. Superposición de validación.

DPoP/MTLS: Tokens de acceso ligados a prueba de posesión o certificado.

Rate limiting: En API_GATEWAY. Límites por usuario/cliente/tenant.

CORS y headers de seguridad: HSTS, CSP estricta para UI de consentimiento.

Gestión de secretos: Solo vía Vault. Prohibidos secretos en variables persistentes.

Privacy by design: Minimizar claims. Evitar PII en logs. Enmascaramiento de emails/IP.



---

📈 7. Observabilidad

Métricas Prometheus: logins_total, login_fail_total, mfa_challenges_total{type=}, tokens_issued_total{type=access|refresh}, introspect_requests_total, revocations_total, latency_ms_bucket, active_sessions.

Tracing: OpenTelemetry. Propagación W3C. Spans por endpoint y validación de credenciales.

Logs: JSON con trace_id, tenant_id, identity_id, client_id y resultado.



---

🔄 8. Eventos

Emitidos: identity.user.created, identity.user.updated, identity.role.changed, identity.mfa.enabled, identity.token.revoked.

Consumidos: users.deleted, tenants.config.changed, security.risk.alert.

Esquemas: Avro/JSON con versionado semántico.



---

🧪 9. Pruebas y Criterios de Aceptación

Unitarias para hash, validación de JWT, flujos OAuth y MFA.

Integración con API_GATEWAY, Redis, Vault y DB.

E2E para login con PKCE, DPoP, TOTP y WebAuthn.

Seguridad: pruebas de fuerza bruta, replay, CSRF en authorize, downgrade de TLS, JWK spoofing.



---

🧰 10. DX y Entrega

OpenAPI autogenerado y publicado.

SDKs oficiales (Node.js, Python) con ejemplos.

Entorno sandbox con clientes y usuarios de prueba.

CI/CD con firmas reproducibles y escaneo SAST/DAST.



---

⚙️ 11. Configuración

TOKEN_TTL_ACCESS=900s, TOKEN_TTL_REFRESH=30d.

PASSWORD_HASH_ARGS={memory, iterations, parallelism}.

KEY_ROTATION_PERIOD=30d, KEY_OVERLAP=7d.

MFA_REQUIRED_ROLES=["ADMIN","PRESIDENT"].

ABAC_POLICY_ENGINE=CEL|Rego.



---

📜 12. SLA y Runbooks

SLO: 99.9% emisión de tokens ≤ 200 ms P95. Introspección ≤ 100 ms P95.

Runbooks: rotación de claves, respuesta a incidente de fuga de refresh tokens, invalidez masiva por jti.



---

🔁 13. Migración desde auth-service

Alias de rutas /auth/* → /identity/* en API_GATEWAY.

Migración de client_id, consentimientos, sesiones y claves. Validadores de integridad.

Monitoreo de errores y rollback controlado vía feature flag.



---

✅ 14. Cierre

identity-service consolida la identidad de la plataforma con estándares, seguridad reforzada, cumplimiento legal y una DX clara. La arquitectura habilita escalabilidad y control fino por tenant con trazabilidad completa.

