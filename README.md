# spec-qa — Pipeline de QA sobre specs SDD

**Distribución de perfil Hermes que audita un spec antes de implementarlo. Cuatro especialistas en paralelo verifican criterios de aceptación, casos borde, riesgo y consistencia, y devuelven un veredicto.**

Un spec ambiguo no falla en la implementación: falla antes, cuando nadie notó que un criterio no era testeable. Esta distribución existe para atrapar eso en el único momento en que todavía es barato.

---

## Qué resuelve

En un flujo Specification-Driven, el spec es la fuente de verdad: si un criterio de aceptación es ambiguo, todo lo que se construya encima hereda la ambigüedad. Este profile lee el spec como lo leería un QA y responde una sola pregunta: **¿está listo para implementación?**

El orquestador **no genera specs y no implementa**: solo audita. Un auditor que además escribe deja de ser una auditoría.

## Cómo funciona

1. **Preflight** — busca el artefacto del spec en `odd/specs/<change>/` u `openspec/changes/<change>/` (`proposal.md`, `spec.md`, `design.md`). Si no hay `spec.md`, reporta el gap en lugar de suponer.
2. **Lectura completa del spec** antes de auditar cualquier parte.
3. **Cuatro QA en paralelo**, cada uno con contexto aislado y un enfoque distinto:
   - `qa-criteria` — ¿los criterios de aceptación son testeables?
   - `qa-edge-cases` — ¿qué casos borde no están cubiertos?
   - `qa-risk` — ¿qué riesgos introduce el cambio?
   - `qa-consistency` — ¿el spec se contradice o contradice lo ya existente?
4. **Veredicto** consolidado en un reporte: `PASS` · `PASS_WITH_CONCERNS` · `FAIL`, con el detalle por área.

Los cuatro corren en paralelo **porque solo leen**. Es una restricción real del runtime: dos sub-agentes que escriben en el mismo filesystem se pisarían.

## Instalación

```bash
hermes profile install github.com/LisandroCacciatore/spec-qa
```

Variables de entorno requeridas (declaradas en `distribution.yaml`):

| Variable | Para qué |
|---|---|
| `HERMES_MODEL` | Modelo que usa el perfil |
| `DEEPSEEK_API_KEY` | Credencial del proveedor |

## Qué incluye

| Componente | Rol |
|---|---|
| `SOUL.md` | Persona del orquestador: rol, alcance y lo que no puede hacer |
| `skills/spec-qa-orchestrator/` | Preflight, lectura del spec y coordinación de los 4 QA |
| `skills/qa-criteria/` | Validación de criterios de aceptación |
| `skills/qa-edge-cases/` | Búsqueda de casos borde |
| `skills/qa-risk/` | Evaluación de riesgo |
| `skills/qa-consistency/` | Consistencia interna y con el sistema existente |
| `config.yaml` | Delegación: 4 hijos en paralelo, 20 min por auditoría, sin auto-approve |
| `cron/pending-specs-digest.yaml` | Reporte diario 10:00 de specs pendientes de QA |
| `.no-bundled-skills` | Opta por no heredar el catálogo de skills de Hermes: el perfil trae los suyos |

## Notas de implementación que valen

- **Los `delegate_task(...)` de las skills son pseudocódigo de intención, no una firma literal.** El runtime documentado en `skills/spec-qa-orchestrator/SKILL.md` aclara qué acepta de verdad `delegate_task` y qué no existe (`worktree`, `background`).
- **El rol de cada especialista va en el `context`**, no en un archivo por sub-agente.
- **`subagent_auto_approve: false`** a propósito: un hijo que topa un pedido de aprobación y bloquea deadlockea al padre, porque el padre es dueño del stdin. Auto-deny es la opción segura.

## Estado

Pipeline en uso. Consume specs producidos por `intake-sdd`; si el spec no existe, no inventa: reporta el gap.
