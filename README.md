# README — Ingesta de documentos y actualización de especificaciones funcionales

## Propósito

Estandarizar cómo **Codex / Gemini / Copilot** tomarán un documento adjunto y, usando **SCOPE.md** y este README, **actualizarán la documentación funcional** del proyecto. Esta fase se limita a **especificaciones funcionales**; cuando estén maduras se solicitará pasar a **planificación**.

## Prerrequisitos

* `SCOPE.md` actualizado con servicios y vertical slices priorizados.
* Estructura mínima:

  ```
  .
  ├── SCOPE.md
  ├── README.md
  ├── templates/specs/{spec-template.md, plan-template.md, data-model-template.md, tasks-template.md, ...}
  ├── prompts/{spec.prompt.md, plan.prompt.md, data-model.prompt.md, tasks.prompt.md, ...}
  └── specs/
      └── NNN-<slug>/
          ├── spec.md
          ├── plan.md
          ├── data-model.md
          ├── tasks.md
          └── contracts/{openapi.yaml, pact/}
  ```
* Los slices existentes deben tener `spec.md` basado en `templates/specs/spec-template.md`.

## Comando para el IDE/Asistente

> **Úsalo tal cual en Codex/Gemini/Copilot al adjuntar un archivo fuente (PDF/DOC/MD):**
>
> **“Analiza el documento adjunto, y en base a los servicios indicados en SCOPE.md, y a las directrices del README.md, actualizar la documentación funcional que es afectada del proyecto; en caso de ambigüedad y/o contradicción con los documentos existentes realizar la consulta.”**

## Regla de oro

* **No inventar servicios ni endpoints.** Si el documento sugiere algo no contemplado en `SCOPE.md`, **proponer** cambio y **preguntar** antes de modificar.
* **No borrar información aprobada.** Si hay conflicto, **marcarlo** y solicitar validación.

## Flujo operativo (IDE)

1. **Leer contexto**: `SCOPE.md`, `README.md`, `specs/**/spec.md` relevantes.
2. **Extraer del adjunto**: requisitos, reglas, casos de uso, criterios de aceptación y supuestos.
3. **Enrutar por servicio** según `SCOPE.md`:

   * Si afecta un slice existente: **actualizar su `spec.md`**.
   * Si amerita un nuevo slice: **proponer carpeta** `specs/NNN-<slug>/` con `spec.md` inicial.
4. **Actualizar documentación funcional**:

   * Editar solo secciones funcionales: contexto, objetivos, usuarios, historias, criterios de aceptación, fuera de alcance, riesgos, métricas, flags y observabilidad.
   * Agregar **“Servicios afectados”** y **trazabilidad** indicando capítulo/página del adjunto.
5. **Generar manifiesto de corrida** en `runs/<YYYY-MM-DD>/<run-id>.yaml`:

   ```yaml
   run_id: 2025-09-22-001
   sources:
     - file: <nombre-archivo-adjunto>
       sections: [ ... ]
   routing:
     - slice: NNN-<slug>
       services: [ svc-a, svc-b ]
       files_changed: [ specs/NNN-<slug>/spec.md ]
   flags:
     require_owner_approval: [legal_impact, contract_change, data_model_change, security_impact]
   notes: ["ambigüedad en ...", "contradicción con ..."]
   ```
6. **Detección de ambigüedad/contradicción**:

   * Si el adjunto contradice `spec.md` o es incompleto: **no sobreescribir**. Insertar bloque:

     ```
     > PENDIENTE: Ambigüedad/contradicción detectada [referencia]. Requiere confirmación del Owner.
     ```
   * Crear comentario de salida para el IDE con preguntas puntuales.
7. **Salida**:

   * `specs/**/spec.md` actualizados.
   * `runs/<date>/<run-id>.yaml` + `runs/<date>/<run-id>-changelog.md` con resumen de cambios.

## Qué se actualiza en esta fase

* **Solo documentación funcional**: `spec.md`.
* **No** tocar `plan.md`, `data-model.md`, `contracts/` ni `tasks.md` salvo que el Owner lo pida explícitamente.

## Criterios de edición en `spec.md`

* Historias con **criterios de aceptación verificables**.
* **Fuera de alcance** explícito para evitar creep.
* **Riesgos y supuestos** con referencia al adjunto.
* **Métricas de éxito** simples.
* **Feature flags** y **observabilidad** mínimas descritas.
* **Servicios afectados**: lista y breve impacto funcional por servicio.

## Trigger para pasar a planificación

El Owner indicará al IDE: “continuar a planificación”. Entonces:

* Usar `prompts/plan.prompt.md` sobre el `spec.md` ya validado.
* Generar o actualizar `plan.md`, luego `data-model.md`, `contracts/`, `tasks.md`.

## Reglas de validación humana obligatoria

Si el IDE detecta alguno de estos impactos al interpretar el adjunto, **debe consultar antes de editar**:

* `legal_impact` (normativas, cumplimiento).
* `contract_change` (cambios a API o eventos).
* `data_model_change` (tablas, claves).
* `security_impact` (autorización, auditoría, cifrado).

## Estructura recomendada de `spec.md`

Usar `templates/specs/spec-template.md`. Secciones mínimas:

```
# <NNN>-<slug>: Spec
## Contexto
## Objetivos
## Usuarios y casos de uso
## Historias de usuario
### Criterios de aceptación
## Servicios afectados
## Fuera de alcance
## Supuestos y riesgos
## Métricas de éxito
## Flags y observabilidad
## Trazabilidad a documentos fuente
```

## Estándares de trazabilidad

* En cada bloque nuevo o modificado, añadir al final una cita breve:

  * `Fuente: <adjunto>, cap X.Y (p. ZZ)`
* En `runs/<run-id>-changelog.md` listar:

  * Slices tocados, secciones modificadas, referencias al adjunto y motivo.

## Ejemplo de uso paso a paso

1. Adjunta el PDF/DOC.
2. Ejecuta el comando indicado arriba.
3. Revisa preguntas del IDE si marca ambigüedades.
4. Aprobado el `spec.md`, instruye: “continuar a planificación para \<NNN-<slug>>”.

## Convenciones y estilo

* Español claro, voz impersonal.
* Sin decisiones técnicas profundas en `spec.md`.
* Criterios de aceptación con condiciones observables.
* No cambiar títulos de secciones sin necesidad.

## Mensajería de salida para el IDE

Plantilla de comentario que el IDE debe producir al finalizar:

```
Actualizaciones propuestas:
- Slice: NNN-<slug> → specs/NNN-<slug>/spec.md (secciones: Historias, CA, Riesgos)
- Servicios afectados: [svc-a, svc-b]
- Ambigüedades: [...]; Preguntas: [...]
- Manifiesto: runs/YYYY-MM-DD/<run-id>.yaml
Confirma o corrige. Si validas, indicar: “continuar a planificación”.
```

## Checklist del Owner por corrida

* [ ] Cambios funcionales coherentes con `SCOPE.md`
* [ ] Ambigüedades resueltas o marcadas
* [ ] `runs/<date>/<run-id>.yaml` generado
* [ ] OK para “continuar a planificación” o solicitar ajustes

---
