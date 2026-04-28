# Reviewer guard-rails (auto-loaded by gemini)

You may be invoked as a **code reviewer** by the `/work` orchestrator. When the orchestrator asks you to read an XML pack and write findings:

## Hard rules for orchestrated runs

1. **Do NOT use `run_shell_command`** to write files (no `cat`, `tee`, `echo`, `printf > file`). Those need permission prompts and BLOCK the orchestrator.
   - Use the `write_file` / `replace` tools instead — those are auto-approved by `auto_edit` mode.
2. **Do NOT** attempt destructive shell ops — they're hard-blocked by the policy file at `~/.gemini/policies/00-deny-destructive.toml` (rm, sudo, chmod, force-push, etc.).
3. When writing review findings: use `write_file` with `file_path` and `content` arguments.
4. **Always end findings files with a single line: `## DONE`** — otherwise the orchestrator hangs forever waiting for the marker.

## Approval mode

This Gemini instance runs in **auto_edit** mode (NOT yolo — yolo has wiped repos for others; we deliberately keep one prompt-line of human/orchestrator oversight on shell). File-write tools are auto-approved. Shell commands prompt — the `work` orchestrator catches the prompt + auto-approves safe ones (e.g. `cat findings.md`) within ~15s. Destructive shell is blocked outright by the policy file.
