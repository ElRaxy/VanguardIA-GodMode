---
name: GodModeSkill — patches críticos para macOS
description: Pinned brew bash 5+, fswatch en lugar de inotifywait, modelos free rotativos en OpenRouter
type: feedback
originSessionId: a56ccfb3-1867-4813-9b38-404f6aedb0f3
---
Patches aplicados al fork `~/dev/GodModeSkill` para correr en macOS Darwin (no upstream).

**Why**: Repo upstream asume Linux ubuntu. Sin patches, `work pick-agents` falla con `declare: -A: invalid option` (bash 3.2 macOS no soporta arrays asociativos), `work review` falla por falta de `inotifywait`, y rutas de memory hardcodean `/home/ubuntu/`.

**How to apply**:
1. Shebangs absolutos a `#!/opt/homebrew/bin/bash` (no `/usr/bin/env bash`) en `work`, `work-fleet-restart`, `agent-status` — porque `/bin/bash` 3.2 viene antes de `/opt/homebrew/bin/bash` en PATH macOS.
2. Función portable `wait_for_fs_event()` añadida en `work` que usa `inotifywait` si existe (Linux) o `fswatch -1` con timeout vía background+kill (macOS). Reemplaza línea ~436.
3. `work-pack-build` linea 26: `MEMORY_DIR` ahora se autodescubre via `_discover_memory_dir()` — busca `~/.claude/projects/*/memory/` por mtime. Soporta env override `CLAUDE_MEMORY_DIR`.
4. Modelos free OpenRouter ROTAN disponibilidad. `qwen/qwen3-coder:free` y `meta-llama/llama-3.3-70b-instruct:free` estaban rate-limited el 2026-04-29. Funcionaban: `nvidia/nemotron-3-super-120b-a12b:free`, `z-ai/glm-4.5-air:free`, `inclusionai/ling-2.6-1t:free`, `openai/gpt-oss-120b:free`. Verificar con `curl https://openrouter.ai/api/v1/models` antes de cambiar.

Si en futuro upstream actualiza, NO copiar files crudos a `~/.local/bin/` — re-aplicar estos patches o el setup romperá.
