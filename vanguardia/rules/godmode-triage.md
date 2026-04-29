# Reglas de triage post-audit GodMode

Aplica al output de `/audit-frontend` (o cualquier PLAN.md/AUDIT.md). Convierte findings sueltos en plan de ataque ejecutable, decidiendo herramienta + orden + qué archivar.

## Tabla 1 — ¿Qué herramienta usar?

Para cada finding, asignar UNA herramienta según naturaleza:

| Naturaleza del finding | Herramienta | Razón |
|---|---|---|
| Typo, dead import, plural concord, copy mecánico, hardcoded literal → token, espacios | **Direct apply** (Claude edita sin preguntar) | Riesgo cero, no compensa orquestar 3 reviewers |
| Refactor mecánico repetitivo en N archivos (rename, regex-able, extract primitivo, mover constantes) | **codex-delegate** | Codex genera código en bulk, mejor ratio tiempo/output |
| Multi-archivo + decisión arquitectónica + ADR-required | **`/work plan`** + `/work implement` | 3 reviewers critican plan antes de codear |
| Cross-cutting palette/spacing/branding shifts | **`/work plan`** + `/work implement` | Decisión cross-cutting necesita consenso |
| Decisión de marca / nombres / sentimiento subjetivo (ej. Cockpit→Panel) | **User decide** | No es problema de código, es taste |
| A/B visual entre 2 valores válidos (ej. --wide 1440 vs 1280) | **User decide** | Necesita ojo humano + screenshot comparativa |
| Migración cross-cutting de bajo impacto pre-launch (ej. PageShell 10 vistas) | **Archivar** | Esfuerzo > valor pre-launch |

## Tabla 2 — Orden de ataque entre substantives

Cuando múltiples items van a `/work plan` o codex-delegate, ordenar por peso (mayor peso = atacar primero):

| Criterio | Peso |
|---|---|
| Bloquea uso real beta (bug visible al user) | +5 |
| Bloquea futuros audits/tests (ej. modal tapa shots Playwright) | +4 |
| Cross-cutting affects vistas no auditadas | +3 |
| ADR required (estandariza decisiones reusables) | +2 |
| Quick win con impacto visible | +1 |

Empate → orden de aparición en PLAN.md.

## Tabla 3 — Filtro pre-launch (1-jun-2026)

Strev en pre-launch beta cerrada. Ventana ~5 semanas. Filtrar agresivo:

| Tipo de item | Acción |
|---|---|
| Bug visible en beta (rompe flujo, copy incorrecto, modo light desafinado) | Atacar AHORA |
| Mejora UX con efecto en retención (onboarding, jerarquía CTAs primarios) | Atacar si esfuerzo <3h |
| Refactor estructural sin user impact (extract primitivos, centralizar constantes) | Archivar — post-launch backlog |
| Brand/naming changes (Cockpit→Panel) | Solo si user lo pide explícito |
| A11y bugs reales (focus invisible, contraste WCAG fail, sin aria) | Atacar AHORA |
| A11y nice-to-have (aria-live en clock, etc.) | Si <30 min total, agrupar en lote |

## Tabla 4 — Reglas duras Strev (filtro de validez)

Toda sugerencia que viole ALGUNA de estas reglas se descarta o archiva con razón. Aplicar ANTES de Tabla 1:

- 0 emojis en UI/copy/email/landing — delatan IA
- No `dark:` prefix Tailwind — usar CSS vars + html.dark/html.light
- Selector global `h1, h2, h3, h4` con `var(--font-display)` (Geist) — NUNCA `var(--font-serif)`
- AmbientGlow OFF en data-heavy (Clients, Routines, Metrics, Dashboard)
- Instrument Serif italic solo público + auth-gated — NUNCA data-heavy
- `Math.random` prohibido en SVG gradients — `useId()` siempre
- Banner "Recomienda Strev" solo en Profile — NO Layout global
- `useForceDarkTheme()` permanece en landing/SEO/TrainerPublicPage — no quitar

## Output esperado del triage

Tras aplicar las 4 tablas, producir 5 listas:

```
LOTE DIRECTO (Claude aplica sin preguntar — 1 commit):
  - <ID>: <descripción> (<file:line>) [<esfuerzo>]
  - ...

LOTE codex-delegate (Claude propone target dir + prompt — 1 commit por refactor):
  - <ID>: <descripción> [target: <ruta>] [esfuerzo]
  - ...

LOTE /work plan + implement (Claude propone, espera confirmación user, ejecuta):
  - <ID>: <descripción> [esfuerzo] [ADR? sí/no]
  - ...

DECISIONES HUMANAS (Claude formula pregunta numbered, espera respuesta):
  - <ID>: <pregunta concreta con opciones>
  - ...

ARCHIVADOS (con razón explícita):
  - <ID>: <descripción> — razón: <pre-launch | viola regla dura | esfuerzo > valor | duplicado de <X>>
  - ...
```

## Cómo aplicarlo en el flujo

1. **Tras `/audit-frontend`**: invocar `/triage-audit <PLAN.md>` automáticamente o manual.
2. **Claude lee este rules file + el PLAN.md**, ejecuta las 4 tablas, produce las 5 listas.
3. **Claude ejecuta lote DIRECTO automáticamente** (single commit limpio, mensaje "fix(<area>): lote directo audit YYYY-MM-DD — N quick wins").
4. **Para codex-delegate**: muestra propuesta al user, espera "OK" o "skip <ID>".
5. **Para /work**: muestra propuesta + estimación coste reviewer, espera confirmación.
6. **Para decisiones humanas**: pregunta numbered, espera respuesta.
7. **Archivados**: NO preguntar, solo listar al final con razones.

Esto reduce friction: user solo decide lo genuinamente subjetivo. Resto = automático.
