# Multi-LLM Tmux Fleet

Multi-LLM agents running side-by-side in tmux, driven by the `/work` Claude Code command. This README is the operating manual.

---

## Primary entry point: `/work`

For real workflows (planning a feature, implementing, fixing major bugs), use the **`/work`** slash command in Claude Code.

`/work` handles:
- Lineage-weighted quorum (1 codex + 1 gemini + 1 opencode must all agree)
- LRU rotation across all agents in `agents.json` (extra agents auto-eligible when added)
- Auto `/clear` between different PRs (preserves context within same task)
- XML packs with auto-discovered context (architecture docs, memory, journals)
- Event-driven wait via `inotifywait` (zero token burn while reviewers think)
- Disagreement loop with `--prior-rounds-file` (max 3 rounds)
- Auto git ops (branch, commit, push, PR) — but ALWAYS asks before merge

Modes: `plan` | `implement` | `major-bug` | `minor-bug` (no review).

---

## Priority order

Plan-priced (subscription-based) reviewers are picked first. Pay-per-token reviewers are fallback.

| Tier | Agents | Why |
|------|--------|-----|
| Primary  | cdx-* (Codex) | Plan-priced via ChatGPT subscription, top-tier code reasoning |
| Primary  | gem-* (Gemini) | Plan-priced via Google AI Pro, best second opinion vs Codex |
| Primary  | kimi, deepseek (via OpenCode Go) | Plan-priced via OpenCode Go subscription |
| Fallback | kimi, deepseek (via direct API key) | Pay-per-token, only if Go plan unavailable |

---

## Golden rule for second opinions

**Always pick a DIFFERENT model lineage.** Two Codex sessions = same model = same blind spots. The whole point of multi-reviewer fan-out is reviewer-lineage diversity. `/work` enforces this automatically via `work pick-agents` (1 codex + 1 gemini + 1 opencode).

---

## Common commands

- `/work` — primary multi-LLM workflow command (plan / implement / major-bug / minor-bug)
- `agent-status` — show table with alive / reset times for all agents
- `work pick-agents` — print the LRU-rotated picks
- `work-fleet-restart` — kill + respawn LLM tmux sessions with the correct flags
- `tmux attach -t <session>` — attach to any session
- `work clean` — purge old `/tmp/work-*` dirs

---

## Files

- `~/.config/agent-sessions/agents.json` — fleet config (one entry per agent)
- `~/.config/agent-sessions/status.json` — generated live status
- `~/.local/bin/work` — main bash orchestrator
- `~/.local/bin/work-pack-build` — XML pack assembler
- `~/.local/bin/work-converge` — XML response parser + lineage quorum
- `~/.local/bin/work-fleet-restart` — restart helper
- `~/.local/bin/agent-status` — fleet status tool
- `~/.work/state.json` — last task_id (drives `/clear` decisions)
- `~/.work/agent-lru.json` — per-agent LRU rotation
- `~/.claude/commands/work.md` — `/work` slash command spec
