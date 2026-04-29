# RUNBOOK — Workflow GodMode + Strev

Patrón completo: audit → triage → implement → deploy verify. Para reproducir en otra máquina con el mismo "conocimiento" del setup actual.

## Setup inicial en máquina nueva

1. **Clonar este repo** y seguir [`INSTALL_MACOS.md`](./INSTALL_MACOS.md) (~20 min total con re-auth de CLIs).
2. **Restaurar memoria Claude**:
   ```bash
   # Auto-discovery del proyecto path:
   PROJECT_PATH="$HOME/.claude/projects/$(ls $HOME/.claude/projects | grep -i Desktop | head -1)/memory"
   mkdir -p "$PROJECT_PATH"
   cp ~/dev/GodModeSkill/vanguardia/memory-snapshot/*.md "$PROJECT_PATH/"
   ```
   Esto pone los memory files en el path que Claude Code descubrirá automáticamente.
3. **Reemplazar placeholders** en `vanguardia/commands/*.md` con tus valores reales:
   - `<TRAINER_EMAIL>`, `<CLIENT_EMAIL>`, `<ATHLETE_EMAIL>` → tus cuentas test
   - `<BETA_PASSWORD>` → tu password compartida
   - `<CHATGPT_ACCOUNT_EMAIL>` → tu email ChatGPT Plus
4. **Copiar slash commands a workspace target** (ej. `~/Desktop/VanguardIA/.claude/`):
   ```bash
   mkdir -p <workspace>/.claude/commands <workspace>/.claude/rules
   cp ~/dev/GodModeSkill/vanguardia/commands/*.md <workspace>/.claude/commands/
   cp ~/dev/GodModeSkill/vanguardia/rules/*.md <workspace>/.claude/rules/
   ```
5. **Test smoke**:
   ```bash
   agent-status                      # 4 agentes ✓
   work pick-agents                  # devuelve 1+1+1
   ```

## Workflow operativo (uso diario)

### A. Audit pre-cambios

```
/audit-frontend <area>          # ej. dashboard, auth, routines
```

Resultado: `notes/audit-design-YYYY-MM-DD/AUDIT.md` + `PLAN.md` + findings/ + emil/.
Auto-invoca `/triage-audit` en Fase F.

### B. Triage post-audit

`/triage-audit` aplica las 4 tablas de [`rules/godmode-triage.md`](./rules/godmode-triage.md):

1. **Tabla 4** (reglas duras del producto) — descarta sugerencias que violen.
2. **Tabla 1** (herramienta) — Direct apply / codex-delegate / `/work` / decisión humana.
3. **Tabla 3** (filtro pre-launch) — archiva si esfuerzo > valor.
4. **Tabla 2** (orden) — pondera substantives.

Output: `TRIAGE.md` con 5 listas. Lote directo se ejecuta sin preguntar.

### C. Implementación substantive

Para items que requieren `/work`:

```
/work plan <item>             # design review por 3 reviewers
/work implement <plan>        # build con review gate antes de merge
```

Para refactors mecánicos: codex-delegate con prompt diseñado por Claude.
Para typos/imports muertos: edit directo.

### D. Pre-merge gate manual (alternativa a /work implement completo)

Cuando el plan ya está fuerte y solo quieres review del DIFF:

```bash
work review --mode implement \
  --task <task> \
  --task-id <slug> \
  --round 1 \
  --diff-ref <base>..HEAD \
  --planning-doc <ruta-plan>
```

### E. Post-deploy verification

Tras `git push origin main`:

```
/post-deploy-check
```

Pollea Render API + Cloudflare Pages, corre Playwright suite contra prod, reporta pass/fail por spec.

## Decisión rápida: ¿qué herramienta para esto?

| Naturaleza del cambio | Tool |
|---|---|
| Typo, dead import, copy mecánico | Direct edit (sin ceremonia) |
| Rename masivo, extract primitivo, mover constantes | codex-delegate |
| Multi-archivo + ADR + cross-cutting | `/work plan` + `/work implement` |
| Solo review del diff antes de merge | `work review --mode implement` manual |
| A/B visual, brand naming | Preguntar al user |
| Refactor cross-cutting bajo impacto pre-launch | Archivar, post-launch |

## Anti-patrones (no hacer)

- **`/work plan` para typos** — overhead > valor.
- **Re-`/work plan` un plan ya converged** — doble-dip caro.
- **`/work implement` cuando solo quieres review del diff** — usa `work review` manual.
- **Editar código sin branch** — usar `feat/*` o `fix/*`.
- **Mergear sin pre-merge checklist** — siempre confirmación humana.
- **Pushear cambios cross-cutting sin `/post-deploy-check`** — Render+CF flakes silenciosos pasan desapercibidos.

## Limitaciones conocidas

- **`PROMPT_RE`** del orchestrator no matchea el formato nuevo de Gemini CLI ("Allow execution of [grep]?" + numbered options). Workaround: `tmux send-keys -t gem-1 "1" Enter` manual cuando se atasque. Patch pendiente en `~/.local/bin/work`.
- **Free OpenRouter models rotan rate-limits**. Si `nemotron-3-super-120b-a12b:free` falla 429, swap a `inclusionai/ling-2.6-1t:free` o `openai/gpt-oss-120b:free`. Tabla de fallbacks en `INSTALL_MACOS.md`.
- **Nemotron / GLM occasionalmente returnean empty**. Workaround: `work-fleet-restart kimi` + re-nudge manual con prompt round actual.
- **Memoria Claude no transfiere chat-history**. Solo memory files (`*.md` en `~/.claude/projects/<proj>/memory/`). Conversaciones literales no son persistibles vía git.

## Estructura de respaldo aquí

```
vanguardia/
├── commands/
│   ├── audit-frontend.md       # /audit-frontend
│   ├── post-deploy-check.md    # /post-deploy-check
│   └── triage-audit.md         # /triage-audit (auto-invocado)
├── rules/
│   └── godmode-triage.md       # 4 tablas decisión
├── memory-snapshot/
│   ├── MEMORY.md               # index
│   ├── project_godmode_setup.md
│   └── feedback_godmode_macos_port.md
├── agents.json                 # fleet config sanitized
├── opencode.json               # OpenRouter provider config
├── nudge*                      # 4 tmux nudgers
├── .env.example                # OPENROUTER_API_KEY placeholder
├── INSTALL_MACOS.md            # setup reproducible 20 min
└── RUNBOOK.md                  # este doc
```
