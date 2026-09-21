---
name: spec-qa-orchestrator
description: Orchestrate QA audit of an SDD spec — criteria, edge cases, risk, consistency
version: "1.0.0"
metadata:
  hermes:
    tags: [qa, spec, audit, orchestration, sdd]
    category: engineering
---

# Spec QA Orchestrator

## When to Use

- El usuario pide auditar un spec existente
- El usuario dice "QA sobre el spec X"
- El pipeline intake-sdd terminó y hay un spec listo para auditar
- El usuario quiere saber si un spec está listo para implementación

## Runtime note (Hermes v0.20.x)

Los bloques `delegate_task(...)` de este skill son **pseudocódigo**: expresan
la intención de delegación, no una firma literal. `delegate_task` acepta
`goal`, `context`, `tasks[]`, `role`, `output_schema`. **No existe `worktree`**
y `background` está deprecado/ignorado (el spawn ya corre en background y el
resultado vuelve solo). El ROL de cada especialista va dentro de `context` —
no hay SOUL.md por subagente.

## Procedure

### Paso 0: Preflight

Verificá que exista el artefacto del spec:

```bash
ls odd/specs/<change-name>/
# o
ls openspec/changes/<change-name>/
```

Deberías ver:
- `proposal.md` (opcional pero recomendado)
- `spec.md` (obligatorio)
- `design.md` (opcional, si el scope es grande)

Si `spec.md` no existe, reportá el gap y pedí que corra intake-sdd primero.
Si no sabés dónde está el spec, preguntá el path exacto antes de asumir.

### Paso 1: Leer el spec completo

Antes de lanzar nada, leé el spec y el design (si existe). Entendé:
- ¿Qué problema resuelve?
- ¿Cuáles son los requirements?
- ¿Cuáles son los acceptance criteria?
- ¿Qué edge cases menciona?
- ¿Qué riesgos reconoce?

### Paso 2: Decidir qué sub-agentes lanzar

Por defecto, lanzá los 4:

| Sub-agente | Cuándo lanzar |
|---|---|
| `qa-criteria` | Siempre. Es el core de la auditoría |
| `qa-edge-cases` | Siempre. QA sin edge cases no es QA |
| `qa-risk` | Siempre, salvo specs triviales (<3 requirements) |
| `qa-consistency` | Cuando hay >5 requirements o múltiples archivos de diseño |

### Paso 3: Lanzar sub-agentes en paralelo

```
delegate_task(
  tasks=[
    {
      goal: "Validar acceptance criteria del spec <change-name>",
      context: """
        SOS QA-CRITERIA SUB-AGENT.

        Tu rol: validar que CADA acceptance criterion del spec sea
        testeable, completo, y no ambiguo.

        Spec: <repo_path>/odd/specs/<change-name>/spec.md

        Para cada criterio, evaluá:
        - ¿Es testeable? (¿se puede escribir un test que lo verifique?)
        - ¿Es específico? (¿o es vago como "funciona bien"?)
        - ¿Es completo? (¿cubre el requirement?)
        - ¿Es medible? (¿hay un threshold claro?)

        Devolvé informe:
        - Criterios validados: N/M
        - Criterios con problemas (lista con severidad y fix sugerido)
      """
    },
    {
      goal: "Buscar edge cases no cubiertos en <change-name>",
      context: """
        SOS QA-EDGE-CASES SUB-AGENT.

        Tu rol: encontrar edge cases que el spec NO cubre.

        Spec: <repo_path>/odd/specs/<change-name>/spec.md

        Categorías a cubrir:
        - Inputs vacíos/nulos/malformados
        - Concurrencia y race conditions
        - Límites (max, min, off-by-one)
        - Error paths (¿qué pasa si falla la DB? ¿la red?)
        - Datos inusuales (unicode, emojis, caracteres especiales)
        - Estados inconsistentes (usuario eliminado pero sesión activa)

        Devolvé informe:
        - Edge cases agrupados por categoría
        - Top 3 más críticos con justificación
        - Scenario Given/When/Then para cada uno
      """
    },
    {
      goal: "Evaluar riesgos del spec <change-name>",
      context: """
        SOS QA-RISK SUB-AGENT.

        Tu rol: evaluar riesgos técnicos y de negocio del spec.

        Spec: <repo_path>/odd/specs/<change-name>/spec.md
        Design: <repo_path>/odd/specs/<change-name>/design.md (si existe)

        Dimensiones de riesgo:
        - Blast radius (¿qué puede romperse?)
        - Dependencias externas
        - Integridad de datos
        - Implicaciones de seguridad
        - Performance

        Devolvé informe:
        - Riesgos con severidad (alto/medio/bajo)
        - Mitigación sugerida por riesgo
        - ¿El spec reconoce este riesgo? (sí/no)
      """
    },
    {
      goal: "Verificar consistencia interna del spec <change-name>",
      context: """
        SOS QA-CONSISTENCY SUB-AGENT.

        Tu rol: verificar que el spec sea internamente consistente.

        Spec: <repo_path>/odd/specs/<change-name>/spec.md
        Proposal: <repo_path>/odd/specs/<change-name>/proposal.md
        Design: <repo_path>/odd/specs/<change-name>/design.md

        Buscá:
        - Requirements que se contradicen
        - Acceptance criteria que no derivan del requirement
        - Naming inconsistente (¿mismo concepto, distintos nombres?)
        - Design que contradice el spec
        - Proposal que promete algo que el spec no entrega
        - Numeración rota (R1, R2, R4 — falta R3)

        Devolvé informe:
        - Inconsistencias con severidad
        - Ubicación exacta (archivo:línea)
        - Fix sugerido
      """
    }
  ]
)
```

### Paso 4: Recolectar resultados

Esperá a que los 4 sub-agentes terminen. Hermes te notifica.

Por cada resultado:
- Verificá que tenga hallazgos con severidad y evidencia
- Si falta evidencia, pedí que la agregue (una vez)
- Si el sub-agente no devolvió nada útil, anotá "no se pudo auditar <área>"

### Paso 5: Integrar y producir veredicto

```
qa_report = {
  "change_name": "<change-name>",
  "verdict": "PASS" | "PASS_WITH_CONCERNS" | "FAIL",
  "coverage": {
    "criteria_validated": "N/M",
    "edge_cases_found": N,
    "risks_identified": N,
    "inconsistencies_found": N
  },
  "findings": {
    "critical": [...],
    "high": [...],
    "medium": [...],
    "low": [...]
  },
  "recommendations": [...]
}
```

**Reglas del veredicto:**

| Veredicto | Cuándo |
|---|---|
| **PASS** | 0 críticos, 0 altos. Criterios completos. Spec listo |
| **PASS_WITH_CONCERNS** | 0 críticos, ≥1 altos. Se puede implementar con caveats |
| **FAIL** | ≥1 críticos, o criterios incompletos, o inconsistencias graves |

### Paso 6: Persistir el reporte

Guardá el reporte en el mismo directorio del spec:

```
odd/specs/<change-name>/qa-report.md
```

### Paso 7: Reportar al usuario

```
## QA Report: <change-name>

### Veredicto: PASS_WITH_CONCERNS

### Cobertura
- Criterios validados: 7/9
- Edge cases identificados: 12
- Riesgos identificados: 3
- Inconsistencias: 1

### Hallazgos críticos
(ninguno)

### Hallazgos altos
1. Criterio R2.1 no es testeable — qa-criteria
   "El sistema debe ser rápido" — no hay threshold.
   Fix: "El endpoint debe responder en <200ms p95"

2. Sin manejo de timeout en llamada externa — qa-edge-cases
   El spec no define qué pasa si la API externa no responde.
   Fix: agregar scenario con timeout de 5s y fallback.

### Hallazgos medios
...

### Recomendaciones
1. Corregir R2.1 antes de implementar
2. Agregar scenario de timeout

### Próximo paso
- Si PASS: listo para implementación (Equipo 3)
- Si PASS_WITH_CONCERNS: corregir altos, después implementar
- Si FAIL: volver a intake-sdd con estos hallazgos
```

## Pitfalls

- Auditar código en vez del spec → fuera de scope
- Inventar criterios que no están en el spec → reportá como gap, no lo agregues
- Veredicto sin evidencia → cada hallazgo necesita cita del spec
- Ignorar el design → si existe, hay que verificarlo contra el spec
- Dejar pasar inconsistencias "porque son menores" → se acumulan

## Verification

- [ ] Preflight completado (spec.md existe)
- [ ] Los 4 sub-agentes corrieron
- [ ] Veredicto asignado según las reglas
- [ ] Reporte persistido en `qa-report.md`
- [ ] Usuario recibió resumen integrado
