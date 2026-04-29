---
name: GodModeSkill instalado en VanguardIA workspace
description: /work multi-LLM cross-review pipeline operativo en macOS, 100% gratis (ChatGPT Plus + Gemini free + OpenRouter free)
type: project
originSessionId: a56ccfb3-1867-4813-9b38-404f6aedb0f3
---
GodModeSkill instalado y operativo el 2026-04-29.

**Por qué**: Strev pre-launch (1-jun-2026) necesita gate multi-familia para auth/Stripe/migraciones. Concepto: 3 familias revisan código antes de merge → atrapa puntos ciegos que un solo modelo se pierde.

**Cómo aplicar**:
- Usar `/work plan|implement|major-bug|fix` SOLO en rutas críticas (auth, Stripe, webhooks, migraciones DB).
- NO usar para typos, CSS, refactors mecánicos — overhead > beneficio.
- Antes de merge muestra checklist (CLAUDE.md, arquitectura, tests, consenso) — siempre confirmar.

**Setup activo**:
- Repo fork: `~/dev/GodModeSkill` (port a macOS aplicado)
- Binarios: `~/.local/bin/work`, `work-pack-build`, `work-converge`, `work-fleet-restart`, `agent-status`
- Slash command: `~/.claude/commands/work.md` (skill /work)
- Config fleet: `~/.config/agent-sessions/agents.json`
- Sesiones tmux: `cdx-1` (Codex/ChatGPT Plus), `gem-1` (Gemini free), `kimi` (Nemotron-3-Super free), `deepseek` (GLM-4.5-Air free)
- API key OpenRouter en `~/.config/opencode/.env` (mode 600). Cuenta Codex: <CHATGPT_ACCOUNT_EMAIL>.

**Cuándo NO usar**: si las sesiones tmux están muertas (ver `agent-status`), restart con `work-fleet-restart cdx-1 gem-1 kimi deepseek`. Free tier OpenRouter: 200 req/día sin créditos, 1000 con $10 one-time.

**Decisiones 2026-04-28:**
- **NO clonar** repo paralelo `VanguardIAGodMode`. GodMode se integra dentro de VanguardIA (mantiene workflows/skills/agentes existentes). Un solo repo, un solo lugar de verdad.
- **Modelos locales descartados** ("igual en mi equipo no es buena idea"). Fallback acordado = OpenRouter free models.
- Alex tiene **ChatGPT Plus** (no solo Claude Pro). Codex puede usar esa cuenta vía Plus, no requiere upgrade adicional.
