Aquí tienes el README propuesto. Está alineado con Spec-Driven Development descrito en `github/spec-kit` y adapta el flujo para trabajar sin el CLI ni plantillas de Spec-Kit, usando Codex como asistente de redacción de especificaciones. ([GitHub][1])

---

# README — Especificación y Desarrollo sin agente específico (modo “Spec-Kit manual”)

## Objetivo

Definir cómo preparar y ejecutar el proyecto bajo un enfoque de **Spec-Driven Development** sin usar el CLI de Spec-Kit. Usaremos **Codex** u otro asistente para ir generando artefactos, pero la estructura y el proceso son manuales. Nota: el README oficial de Spec-Kit indica que **Codex CLI no soporta argumentos personalizados en “slash commands”**, por eso aquí operamos sin comandos del kit. ([GitHub][1])

---

## Estructura del repositorio

```
.
├── SCOPE.md                         # alcance global y criterios de priorización
├── memory/
│   ├── constitution.md              # principios de ingeniería y calidad
│   └── constitution_update_checklist.md
├── scripts/
│   ├── create-feature.sh            # genera /specs/NNN-slug con plantillas
│   └── verify-readiness.sh          # checks previos a merge y release
├── specs/
│   └── 001-<slug>/
│       ├── spec.md                  # especificación funcional (scope por feature)
│       ├── plan.md                  # plan técnico e impacto arquitectónico
│       ├── data-model.md            # entidades, esquemas, migraciones
│       ├── contracts/
│       │   ├── openapi.yaml         # contratos HTTP
│       │   └── pact/                # contratos Pact para consumidores/proveedores
│       ├── tasks.md                 # backlog accionable del slice
│       ├── quickstart.md            # cómo ejecutar/ver el slice
│       └── research.md              # decisiones, pros/cons, referencias
├── .github/
│   └── workflows/
│       ├── pr.yml                   # lint + unit + integration + build en PR
│       ├── main-staging.yml         # entorno efímero + E2E al merge a main
│       └── deploy-prod.yml          # despliegue tras E2E ok o aprobación
└── README.md
```

---

## Fuente de verdad

* **SCOPE.md**: actualizar antes de cada ciclo. Contiene objetivos, límites, riesgos, supuestos y la lista priorizada de “vertical slices”.
* Cada feature vive en `specs/NNN-<slug>/` con artefactos propios. Evitar duplicar contenido entre `SCOPE.md` y `spec.md`.

---

## Flujo de trabajo

1. **Principios del proyecto**
   Crear o actualizar `memory/constitution.md` con estándares de código, testing, UX, performance y seguridad. Mantener `constitution_update_checklist.md`.

2. **Refinar alcance global**
   Revisar `SCOPE.md`. Confirmar criterios de aceptación globales, definición de “hecho” y el orden de slices.

3. **Crear un slice**
   Ejecutar `scripts/create-feature.sh 001 <slug>` o replicar la carpeta plantilla. Completar:

   * `spec.md`: problema, usuarios, historias, criterios de aceptación, no-objetivos.
   * `plan.md`: arquitectura, componentes, decisiones, migraciones, riesgos, rollback.
   * `data-model.md`: ERD, tablas, índices, eventos, versionado de esquemas.
   * `contracts/`: `openapi.yaml` y, si aplica, contratos **Pact**.
   * `tasks.md`: tareas atómicas derivadas del plan, ordenadas y estimadas.

4. **Implementar por Vertical Slice**
   Rama `feature/NNN-<slug>`. Entregar **backend + frontend + pruebas** del slice antes del siguiente.

5. **Flags de funcionalidad**
   Encapsular cambios detrás de **feature flags** (LaunchDarkly/Unleash/DB). Mantener toggles por entorno.

6. **Revisión y merge**
   Todo PR requiere al menos **2 revisores**. Política de calidad: checks verdes obligatorios.

7. **CI/CD**

   * En cada PR: **Lint, Unit, Integration, Build**.
   * Al merge a `main`: **staging efímero**, despliegue y **E2E**.
   * Si E2E pasan: **deploy a producción** automático o con aprobación manual.

8. **Contratos**
   Publicar y verificar contratos en CI:

   * **OpenAPI**: validación de esquema y breaking changes.
   * **Pact**: publicar pacts de consumidores y verificar en proveedores.

9. **Observabilidad desde el día 1**

   * **Logs JSON** con `trace_id`.
   * **Métricas Prometheus**: latencia, errores, throughput.
   * **Trazas OpenTelemetry** extremo a extremo.

---

## Uso de Codex para generar artefactos (sin Spec-Kit)

> Pegue estos prompts en Codex y guarde el resultado en los archivos indicados.

### 1) Generar `spec.md`

```
Actúa como analista de producto. Con base en SCOPE.md, crea spec.md para el slice "<slug>":
- Problema y objetivos
- Usuarios y casos de uso
- Historias de usuario con criterios de aceptación verificables (Gherkin opcional)
- Fuera de alcance
- Riesgos y supuestos
Formato: Markdown estructurado. No inventes endpoints ni modelos aún.
```

### 2) Generar `plan.md`

```
Actúa como arquitecto de software. A partir de spec.md:
- Diagrama de componentes y responsabilidades
- Decisiones clave (ADR breve) con pros/cons
- Estrategia de datos y migraciones
- Contratos externos previstos (HTTP/eventos) y versionado
- Plan de pruebas: unit, integration, e2e, contrato
- Riesgos técnicos y mitigación
Salida: plan.md en Markdown con checklist de tareas derivables.
```

### 3) Generar `data-model.md`

```
Con base en plan.md, define entidades, campos, relaciones, índices, eventos de dominio y estrategias de migración/versionado. Incluye ejemplos de payload.
```

### 4) Generar `tasks.md`

```
Transforma plan.md en una lista de tareas atómicas ordenadas:
- Prefijo [BE]/[FE]/[TEST]/[OPS]
- Dependencias
- Criterio de aceptación por tarea
- Etiqueta de flag si aplica
```

### 5) Generar contratos

```
Proponer openapi.yaml mínimo viable para endpoints del slice. Estándares: versionado semántico, errores estructurados, paginación, idempotencia. Sugerir pacts para consumidores/proveedores relevantes.
```

---

## Convenciones

* **Ramas**: `feature/NNN-<slug>`, `hotfix/<id>`, `release/<x.y.z>`.
* **Commits**: Conventional Commits.
* **Issues**: reflejan ítems de `tasks.md`.
* **DoD** por slice:

  * `spec.md`, `plan.md`, `data-model.md`, `contracts/` actualizados
  * pruebas unitarias e integración verdes
  * E2E del slice cubiertos
  * métricas y trazas visibles
  * feature flag apagable
  * quickstart operativo

---

## CI/CD de referencia (GitHub Actions)

`.github/workflows/pr.yml`

```yaml
name: pr
on:
  pull_request:
    branches: [ main ]
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run lint
      - run: npm run test:unit
      - run: npm run test:integration
      - run: npm run build
```

`.github/workflows/main-staging.yml`

```yaml
name: main-staging
on:
  push:
    branches: [ main ]
jobs:
  staging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/provision-ephemeral-staging.sh
      - run: npm ci && npm run migrate
      - run: npm run start:staging &
      - run: npm run test:e2e -- --baseUrl ${{ steps.env.outputs.url }}
      - run: ./scripts/publish-contracts.sh
```

`.github/workflows/deploy-prod.yml`

```yaml
name: deploy-prod
on:
  workflow_run:
    workflows: [ main-staging ]
    types: [ completed ]
jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/deploy-prod.sh
```

---

## Observabilidad

* **Logs**: JSON, nivel por módulo, `trace_id` propagado.
* **Métricas**: exportadores Prometheus. Latencia p50/p95/p99, tasa de error, RPS.
* **Trazas**: OpenTelemetry SDK, sampling por entorno, spans FE↔BE↔DB.

---

## Pruebas

* **Unit**: aisladas y rápidas.
* **Integration**: con dependencias reales o testcontainers.
* **E2E**: sobre staging efímero.
* **Contrato (Pact)**: publicar y verificar en CI. Rompimientos bloquean merge.

---

## Flags de funcionalidad

* Estrategia: `boolean`, `per cohort`, `percentage rollouts`.
* Regla: todo código nuevo controlado por flag. Rollout seguro y reversible.

---

## Reglas de revisión

* 2 aprobaciones por PR.
* Sin `force-push` a `main`.
* Checks obligatorios: lint, unit, integration, build, contrato.

---

## Checklist de “Ready for Dev” por slice

* `spec.md` completo y aprobado
* `plan.md` y `data-model.md` revisados
* `contracts/` iniciales definidos
* `tasks.md` priorizado
* feature flag creado
* métricas y trazas planificadas

## Checklist de “Ready for Prod”

* PRs del slice integrados
* E2E verdes en staging efímero
* Pact verificado
* observabilidad activa
* rollback documentado

---

## Scripts sugeridos

`scripts/create-feature.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
n=$(printf "%03d" "${1:?num}"); slug="${2:?slug}"
dir="specs/${n}-${slug}"
mkdir -p "$dir/contracts/pact"
for f in spec plan data-model tasks quickstart research; do
  : > "${dir}/${f}.md"
done
echo "openapi: 3.0.3" > "${dir}/contracts/openapi.yaml"
echo "Creado ${dir}"
```

`scripts/verify-readiness.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
d="specs/$1-$2"
for f in spec.md plan.md data-model.md tasks.md; do
  test -s "${d}/${f}" || { echo "Falta ${f}"; exit 1; }
done
```

---

## Referencias

* Pasos núcleo de especificar → planificar → tareas → implementar en el README de `github/spec-kit`. ([GitHub][1])
* Nota de soporte de agentes y limitación de **Codex CLI** para argumentos personalizados. ([GitHub][1])

---

Si quieres, adapto los prompts a tu `SCOPE.md` real y dejo creada la primera carpeta `specs/001-...`.

[1]: https://github.com/github/spec-kit "GitHub - github/spec-kit:  Toolkit to help you get started with Spec-Driven Development"
