---
name: Dream skill Stop hook posiblemente sobre-disparando
description: 8 sesiones JSONL en Desktop entre 09:39-09:53 del 2026-04-29 sugieren cadencia 24h no respetada
type: feedback
originSessionId: 7274a523-7182-4dcd-9270-0b5ea9d9dcd5
---
Observación 2026-04-29: el directorio de sesiones de `-Users-alex-Desktop/` muestra 8 archivos `.jsonl` creados en una ventana de 14 minutos (09:39-09:53), varios identificables como invocaciones del propio skill `/dream` lanzadas por el Stop hook + flag `.dream-pending`.

**Why:** la cadencia documentada del skill (`should-dream.sh`) es 24h. Si el flag se recrea o no se borra correctamente tras cada dream, cada nueva sesión re-lanza `/dream` en bucle.

**How to apply:**
- Antes de re-ejecutar `/dream` por flag, verificar `~/.claude/.dream-pending` y `.last-dream` timestamp del proyecto.
- Si `.last-dream` < 24h respecto a `date +%s`, NO consolidar — solo borrar `.dream-pending`.
- Si se observa el patrón otra vez, revisar `~/.claude/skills/dream/should-dream.sh` y la lógica del Stop hook en `~/.claude/settings.json`.
- No diagnóstico definitivo — confirmar leyendo contenido de los .jsonl si vuelve a ocurrir.
