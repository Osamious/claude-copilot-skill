# claude-copilot-skill

A [Claude Code](https://claude.com/claude-code) skill that delegates a prompt to **GitHub Copilot** through a real [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) session — not a wrapper around `copilot -p`, but a genuine protocol-level bridge using [`acpx`](https://www.npmjs.com/package/acpx) (MIT) to drive `copilot --acp`.

This preserves Copilot's own Auto model routing (the only mode allowed on Free/Student plans since 2026-06-24) and supports persistent multi-turn sessions.

## Why

- Offload bulk/mechanical work (refactors, boilerplate, log triage) to Copilot without spending Claude tokens/quota.
- Get a second opinion from Copilot on a review or design question.
- Copilot's backend negotiates its own model exactly as it would in the official CLI/IDE.

## Requirements

- [GitHub Copilot CLI](https://www.npmjs.com/package/@github/copilot) installed and authenticated:
  ```
  npm i -g @github/copilot
  copilot login
  ```
- Node/npm available (the skill runs `acpx` via `npx acpx@latest`, cached after first use — no separate install needed).

## Install

Copy `SKILL.md` into your Claude Code skills directory:

```
mkdir -p ~/.claude/skills/copilot
cp SKILL.md ~/.claude/skills/copilot/SKILL.md
```

## Usage

Once installed, trigger it in Claude Code with any of:

- "ask copilot ..."
- `/copilot ...`
- "delegate to copilot"
- "second opinion from copilot"
- "use my copilot quota"

Claude Code will read `SKILL.md`, spin up an ACP session against `copilot --acp`, and relay Copilot's answer back attributed as "Copilot says: ...".

## Modes

| Mode | When | Flags |
|---|---|---|
| `ask` (default) | questions, reviews, analysis | `--approve-reads --non-interactive-permissions deny` |
| `work` | Copilot should edit files, scoped to one directory | `--approve-all --cwd "<target-dir>"` |
| `yolo` | user explicitly wants full autonomy | `--approve-all` (no narrow `--cwd`) |

See `SKILL.md` for full invocation examples, persistent-session usage, and failure-mode troubleshooting.

## Known issues

- `acpx` is alpha; each invocation prints a Node `DEP0190` deprecation warning to stderr (unescaped shell args in acpx's own subprocess spawn). Harmless for prompts you write yourself.
- `acpx`'s permission model is coarser than raw `copilot`'s — in `work` mode, `--cwd` is the real safety boundary, not tool-level allow/deny patterns.

## License

MIT
