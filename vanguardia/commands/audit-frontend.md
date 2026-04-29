---
description: Audit completo frontend Strev post-rediseño 2026-04-28. Refresca screenshots Playwright, /work plan por vista (estructura), emil-design-eng por vista (taste), consolida en notes/audit-design-YYYY-MM-DD/. Modo full por defecto (~21 vistas, ~2-3h). Acepta `<area>` para audit acotado.
---

# /audit-frontend

Audit visual + estructural del frontend Strev usando 3 capas:

| Capa | Tool | Detecta |
|---|---|---|
| **Estructura** | `/work plan` (3 LLMs leen código) | violaciones del design system actual, dup components, hardcoded colors, jerarquía tipográfica, estados faltantes, a11y, breakpoints |
| **Visual** | Playwright `e2e/08-visual-audit.spec.js` | Screenshots reales por viewport (1440×900, 1920, mobile) en dark+light |
| **Taste** | `/emil-design-eng` por vista | densidad, microinteracciones, palette emocional, brand vs landing |

**Idioma**: respuestas al usuario en castellano. Findings raw (inglés) → traducir y resumir.

## Baseline obligatoria

El **rediseño 2026-04-28** ya está en producción (33 commits frontend, bundle `index-49U5vyGI.js`, commit `cf54dbc`). Es la fuente de verdad. Antes de cualquier finding:

1. **Leer** `projects/strev/CLAUDE.md` sección "Rediseño 2026-04-28" — fuente única para layout, tipografía, glow, branding.
2. **No reportar** issues que el rediseño YA cerró (container 1120, 3 fuentes, AmbientGlow en data-heavy, italic Instrument Serif global, etc.). Comparar contra estado actual, no contra audit del 28-abr (PRE-rediseño).
3. Solo flagear desviaciones del nuevo design system o issues nuevas.

## Reglas duras Strev (no violar nunca al sugerir fixes)

- **NO emojis** en UI/copy/email/landing — delatan IA.
- **NO `dark:` prefix** Tailwind — usar CSS vars + `html.dark`/`html.light`.
- **NO** cambiar selector global `h1, h2, h3, h4` a `var(--font-serif)` — rompe Geist.
- **NO** quitar `useForceDarkTheme()` de landing/SEO/TrainerPublicPage.
- **NO** añadir `AmbientGlow` a data-heavy views (Clients, Routines, Metrics, Dashboard).
- **NO** Instrument Serif italic en data-heavy — solo público + auth-gated.
- **NO** `Math.random` en SVG gradients — `useId()` siempre.
- **NO** banner "Recomienda Strev" en Layout global — solo Profile.

Cualquier sugerencia de fix que viole estas reglas debe descartarse.

## Argumentos

- Sin args → audit completo (21 vistas: Dashboard, Clients, ClientDetail, Routines, RoutineDetail, NewRoutine, Metrics, Profile, Notifications, Onboarding, ExcelImport, Social, LandingPage, Login, Register, ForgotPassword, ResetPassword, NotFound, Privacy, PaymentSuccess, PaymentCancel)
- `<area>` (ej. `dashboard`, `auth`, `routines`, `landing`) → audit acotado

## Flujo

### Fase A — Pre-flight

1. Confirma cwd = `~/Desktop/VanguardIA`. Si no, `cd` y reintentar.
2. Verifica fleet vivo: `agent-status`. Si alguno `✗`: `work-fleet-restart cdx-1 gem-1 kimi deepseek`.
3. Verifica beta accounts: las 3 documentadas en `projects/strev/context/beta-test-accounts.md` con pwd común `<YOUR_BETA_PASSWORD>`. Para Playwright pasar via env on-the-fly sin commit:
   ```bash
   export E2E_TRAINER_EMAIL="<TRAINER_EMAIL>"
   export E2E_TRAINER_PASSWORD="<YOUR_BETA_PASSWORD>"
   export E2E_ATHLETE_EMAIL="<ATHLETE_EMAIL>"
   export E2E_ATHLETE_PASSWORD="<YOUR_BETA_PASSWORD>"
   ```
4. Verifica `npx playwright --version`. Si falta browsers: `cd src/strev-web && npx playwright install chromium`.
5. Crea branch: `git checkout -b godmode-audit-$(date +%Y-%m-%d)`. Si existe, append `-N`.
6. Crea `notes/audit-design-$(date +%Y-%m-%d)/` con subdirs `shots-trainer/`, `shots-athlete/`, `shots-landing/`, `findings/`, `emil/`.
7. Decide scope. Si user pasó `<area>`, filtrar. Confirma con user.

### Fase B — Screenshots Playwright

```bash
cd ~/Desktop/VanguardIA/src/strev-web
npm run test:e2e:local -- --grep "Visual Audit"
```

Mueve screenshots generadas → `notes/audit-design-$(date +%Y-%m-%d)/shots-*/`.

Si Playwright falla (login, cookie banner, network) → diagnose, fix, log a `findings/playwright-errors.log`. NO continuar hasta tener screenshots.

### Fase C — Audit por vista

Para cada vista en scope:

#### C.1 Estructura via `/work plan`

```
/work plan auditar vista <View>: src/strev-web/src/views/<View>.jsx
+ componentes que use (src/strev-web/src/components/...).

Baseline: rediseño 2026-04-28 (ver projects/strev/CLAUDE.md sección "Rediseño 2026-04-28").

Detectar SOLO desviaciones del baseline o issues nuevas:
- ¿Usa <PageShell width="default|wide|narrow">? Si no, debería.
- Tipografía: ¿headings con var(--font-display) = Geist? ¿algún h1-h4 con --font-serif por error?
- Italic Instrument Serif: ¿solo en público/auth-gated? ¿filtrado en data-heavy?
- AmbientGlow: ¿OFF en Clients/Routines/Metrics/Dashboard?
- Theme: ¿usa useColorScheme() + CSS vars? ¿algún dark: prefix superviviente?
- Componentes: ¿reutiliza StatCard/RoutineCard/EmptyStatePremium/Skeleton/Modal/Button/Input/RowItem o reinventa?
- Estados: loading skeleton, empty (con EmptyStatePremium variant compact), error
- Accessibility: focus visible, aria, contraste dual-mode, labels, kbd nav
- Math.random en SVG: prohibido (useId())
- Hardcoded rgba(255,255,255,X) que rompa light mode (bug crítico recurrente)
- Pricing tiles (si aplica): ¿usa PricingMini.jsx en vez de botones apilados?

Cita file:line por finding (anti-hallucination).
NO sugerir fixes que violen las "Reglas duras" del CLAUDE.md de Strev.
NO usar emojis en findings.

Output: planning/audit-<view>.md
```

→ Reviewers convergen → mover a `notes/audit-design-$(date +%Y-%m-%d)/findings/<view>-estructura.md`.

#### C.2 Taste via `/emil-design-eng`

Llamar `/emil-design-eng` con screenshots del view (desktop dark+light + mobile dark+light cuando existan). Pedirle:

```
Audit UI/UX de la vista <View>. Screenshots adjuntas.

Referencia de calidad ya aceptada: shots-landing/landing-1440-dark.png
(rediseño 2026-04-28 ya en producción).

Producto: Strev — software gestión entrenadores personales freelance.
ICP: trainer freelance 5-30 clientes activos. Premium feel sin ser frío.

Detectar:
- Densidad y respiración del layout (¿saturado? ¿vacío?)
- Jerarquía visual (¿el ojo va al CTA principal?)
- Microinteracciones implícitas (¿hover/focus states comunican?)
- Palette emocional (¿se siente premium sin sobrecarga?)
- Coherencia con landing (mismo brand mental?)
- Detalles "soso" — específico, no genérico

Restricciones inviolables del producto:
- 0 emojis en UI/copy
- Geist como font default (no IBM Plex)
- AmbientGlow OFF en data-heavy (Dashboard/Clients/Routines/Metrics)
- Instrument Serif italic SOLO público + auth-gated, no data-heavy
- Sin Tailwind dark: prefix (CSS vars + html.dark/light)

Output: lista priorizada (alta/media/baja) con sugerencia concreta por finding.
NO sugerir cambios que violen las restricciones inviolables.
```

→ Output a `notes/audit-design-$(date +%Y-%m-%d)/emil/<view>.md`.

### Fase D — Consolidación

1. Lee TODOS los `findings/<view>-estructura.md` y `emil/<view>.md`.
2. Genera `notes/audit-design-$(date +%Y-%m-%d)/AUDIT.md` con estructura tipo:
   ```
   ---
   date: YYYY-MM-DD
   owner: alex
   type: design-audit
   scope: strev-web post-rediseño-2026-04-28
   baseline-commit: cf54dbc (rediseño 2026-04-28)
   shots: ./shots-trainer · ./shots-athlete · ./shots-landing
   ---

   # Audit diseño Strev — YYYY-MM-DD

   ## Resumen ejecutivo
   <top 6-10 issues por impacto>

   ## Inventory por vista
   <tabla view → estructura findings → emil findings → priority>

   ## Diff vs rediseño 2026-04-28
   <qué se conservó bien, qué áreas se desviaron, qué descubrimos nuevo>
   ```
3. Genera `notes/audit-design-$(date +%Y-%m-%d)/PLAN.md`:
   - Backlog priorizado (P0/P1/P2)
   - Estimación de esfuerzo por item
   - Sugerencia de orden (quick wins primero)

NO usar emojis en AUDIT.md ni PLAN.md.

### Fase E — Hand-off al user

Mostrar al usuario:
1. Resumen ejecutivo (top issues)
2. Cuántos findings P0/P1/P2 hay
3. Preguntar si commitear el audit en `godmode-audit-YYYY-MM-DD`.

### Fase F — Triage automático (auto-invocar)

Tras commit del audit (o si user salta commit), invocar **automáticamente**:

```
/triage-audit notes/audit-design-$(date +%Y-%m-%d)/PLAN.md
```

`/triage-audit` aplica las reglas en `~/.claude/rules/godmode-triage.md`, produce TRIAGE.md con 5 listas (directo / codex / /work / decisión humana / archivados), ejecuta el lote directo sin preguntar, y guía al user por el resto.

Esto reduce friction post-audit — el user solo decide lo genuinamente subjetivo, no qué herramienta usar para cada finding.

## Reglas no-negociables del propio audit

1. **Solo lectura** del código fuente Strev. Cualquier cambio sale en `/work implement` posterior.
2. **No loguear secrets**. Beta password (`<YOUR_BETA_PASSWORD>`) NO va a findings, solo se usa runtime via env var.
3. Si `/work` o emil fallan en una vista: capturar error, continuar siguientes, listar fallidas al final.
4. Pack size de `/work` ≤ 800KB — si vista + componentes superan, dividir en sub-audits.
5. Si user dice "stop" o Ctrl-C: guardar parcial en `findings/`/`emil/`, skip a Fase D con lo que haya.

## Ejemplos de invocación

```
/audit-frontend                    # all 21 views
/audit-frontend dashboard          # solo Dashboard (trainer + athlete)
/audit-frontend auth               # Login + Register + ForgotPassword + ResetPassword
/audit-frontend routines           # Routines + RoutineDetail + NewRoutine
/audit-frontend payment            # PaymentSuccess + PaymentCancel + pricing en Profile
```

## Recovery

- Reviewer hung: `work status --watch` → `work-fleet-restart <name>`.
- Quota 429 OpenRouter free: pausar 60s, retry. Si persiste, swap modelo en `agents.json`.
- Playwright falla todo: posible cold start Render (75s) si tests cargan datos via API. Esperar y reintentar.
- Beta account login falla en Playwright: verificar password con `curl -X POST https://api.strev.app/api/auth/login -d '{"email":"...","password":"..."}'` antes de seguir.
