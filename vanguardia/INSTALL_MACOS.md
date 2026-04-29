# GodModeSkill — VanguardIA macOS Port

Personal fork of [99xAgency/GodModeSkill](https://github.com/99xAgency/GodModeSkill) with macOS Darwin compatibility patches and 100% free-tier provider config (ChatGPT Plus + Gemini free + OpenRouter free models).

## What's different from upstream

| Patch | File | Why |
|---|---|---|
| Cross-platform fs watcher | `skill/work` | Replace `inotifywait` (Linux) with portable `wait_for_fs_event()` that uses `fswatch -1` on macOS |
| Bash 5+ shebangs | `skill/work`, copied binaries | macOS ships bash 3.2 which lacks `declare -A` (assoc arrays) — pin to `/opt/homebrew/bin/bash` |
| Memory dir auto-discovery | `skill/work-pack-build` | Hardcoded `/home/ubuntu/...` replaced with discovery of most-recent `~/.claude/projects/*/memory/` (env override `CLAUDE_MEMORY_DIR`) |
| Drop sudo chown ubuntu | `skill/work` | Linux-only ownership chown removed |
| `vanguardia/` configs | new dir | Sanitized fleet config + nudgers + free-tier OpenRouter setup |

## Reproducible install (macOS, ARM)

### 1. System tools

```bash
brew install bash tmux fswatch jq
npm install -g @google/gemini-cli opencode-ai
```

Verify bash 5+:
```bash
/opt/homebrew/bin/bash --version  # must be 5.x
```

### 2. Clone this fork

```bash
git clone https://github.com/ElRaxy/VanguardIA-GodMode.git ~/dev/GodModeSkill
```

### 3. Auth providers

- **Codex (ChatGPT Plus)**: `codex` once, login flow.
- **Gemini (free)**: `gemini` once, OAuth Google AI Studio.
- **OpenRouter**: signup at https://openrouter.ai/keys, copy `vanguardia/.env.example` to `~/.config/opencode/.env`, paste key, `chmod 600 ~/.config/opencode/.env`.

### 4. Install binaries + slash command

```bash
mkdir -p ~/.local/bin ~/.claude/commands ~/.config/agent-sessions ~/.config/opencode
cp ~/dev/GodModeSkill/skill/work* ~/.local/bin/
cp ~/dev/GodModeSkill/skill/agent-status ~/.local/bin/
chmod +x ~/.local/bin/work* ~/.local/bin/agent-status

cp ~/dev/GodModeSkill/skill/commands/work.md ~/.claude/commands/work.md
cp ~/dev/GodModeSkill/vanguardia/agents.json ~/.config/agent-sessions/agents.json
cp ~/dev/GodModeSkill/vanguardia/opencode.json ~/.config/opencode/opencode.json
```

### 5. Tmux nudgers

```bash
mkdir -p ~/dev/codex-bridge ~/dev/kimi-session ~/dev/deepseek-session
cp ~/dev/GodModeSkill/vanguardia/nudge* ~/dev/codex-bridge/
chmod +x ~/dev/codex-bridge/nudge*
cp ~/dev/GodModeSkill/skill/examples/AGENTS.md ~/dev/kimi-session/
cp ~/dev/GodModeSkill/skill/examples/AGENTS.md ~/dev/deepseek-session/
```

### 6. Spawn fleet

```bash
work-fleet-restart cdx-1 gem-1 kimi deepseek
agent-status   # all 4 should show ✓
work pick-agents   # should print {"codex":"cdx-1","gemini":"gem-1","opencode":"kimi"}
```

### 7. Trust dialog Codex (first run only)

`tmux attach -t cdx-1` → press `1` to trust the directory → `Ctrl-B D` to detach.

## Provider mix (free)

| Lineage | Provider | Model | Cost |
|---|---|---|---|
| Codex | ChatGPT Plus subscription | gpt-5.5 | $0 incremental |
| Gemini | Google AI Studio free | gemini-2.5-pro (1500 req/day) | $0 |
| Opencode kimi-slot | OpenRouter free | nvidia/nemotron-3-super-120b-a12b:free | $0 |
| Opencode deepseek-slot | OpenRouter free | z-ai/glm-4.5-air:free | $0 |

OpenRouter free: 200 req/day (1000 with $10 one-time credits).

## Free OpenRouter models — availability rotates

`qwen/qwen3-coder:free` and `meta-llama/llama-3.3-70b-instruct:free` were upstream rate-limited (429) on 2026-04-29. If yours fail, list current free options:

```bash
curl -s -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  https://openrouter.ai/api/v1/models | \
  jq -r '.data[] | select(.id | endswith(":free")) | "\(.id)  ctx=\(.context_length)"'
```

Pick 2 from different families (avoid `openai/*` and `google/*` to preserve lineage diversity vs Codex/Gemini). Update `agents.json` model field + `work-fleet-restart` launch command.

## Pulling upstream updates

```bash
cd ~/dev/GodModeSkill
git fetch upstream
git merge upstream/main   # may need to resolve conflicts in patched files
```

If upstream rewrote `work` or `work-pack-build`, re-apply the patches listed at the top.

## License

MIT (inherited from upstream 99xAgency/GodModeSkill).
