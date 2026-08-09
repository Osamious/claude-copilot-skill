---
name: copilot
description: Delegate a prompt to GitHub Copilot via real Agent Client Protocol (ACP) session, using acpx to drive `copilot --acp`. Preserves Copilot's own Auto model routing (only mode allowed on Free/Student plans since 2026-06-24), supports persistent multi-turn sessions. Use when the user says "ask copilot", "/copilot", "delegate to copilot", "use my copilot quota", "second opinion from copilot", or wants work done without spending Claude tokens/quota. Also use for bulk grunt work (mechanical refactors, boilerplate, log triage) that is cheap to offload.
---

# Copilot delegation (via ACP / acpx)

Drives the real `copilot --acp` agent process through the Agent Client Protocol,
using `acpx` (MIT, `openclaw/acpx` on npm) as the ACP client. This is a genuine
protocol-level bridge — not a wrapper around `copilot -p` — so Copilot's backend
negotiates its own model exactly as it would in the official CLI/IDE, including
Auto routing on Student/Free plans. Verified 2026-07-31: Auto picked
`gpt-5.4-mini` with zero `model_not_supported` errors, and a named session
correctly recalled state across two separate process invocations.

Still a subprocess handoff, not a live bridge into this conversation — Copilot
gets no view of Claude Code's context. Everything it needs must be in the
prompt (or already in a session you built up turn by turn).

**Known issue:** acpx is alpha; each invocation prints a Node `DEP0190`
deprecation warning (unescaped shell args in acpx's own subprocess spawn) to
stderr. Harmless for prompts you write yourself — filter it out of what you
show the user, don't silently pipe untrusted external text through raw.

## Preflight (once per session)

```powershell
(Get-Command copilot -ErrorAction SilentlyContinue).Source
```

Empty → tell user to install: `npm i -g @github/copilot`, then `copilot login`.
`acpx` itself needs no separate install — run via `npx acpx@latest`, which
npm caches after the first call in a session.

## Modes

Pick the least-privileged mode that can do the job. Default to `ask`.

| Mode | When | Flags |
|---|---|---|
| `ask` | questions, reviews, analysis, "second opinion". Copilot may read files, cannot write or run commands. | `--approve-reads --non-interactive-permissions deny` |
| `work` | Copilot should edit files, scoped to one directory. | `--approve-all --cwd "<target-dir>"` |
| `yolo` | only when user explicitly asks for full autonomy | `--approve-all` (no narrow `--cwd`) |

`acpx`'s permission model is coarser than raw `copilot`'s (`--allow-tool`
patterns like `shell(git:*)` aren't confirmed to work through the ACP path) —
in `work` mode the real safety boundary is `--cwd`, not tool-level denial.
Never use `work`/`yolo` unless the user asked for Copilot to change files.
Confirm before `yolo`.

## Invocation — one-shot

Always pass the prompt via a variable, never inline-quoted — prompts contain
quotes, backticks, and newlines that break PowerShell parsing.

```powershell
$sp = "<scratchpad>\copilot-prompt.txt"
Set-Content -Path $sp -Value @'
<the full prompt, verbatim, including any file paths and context Copilot needs>
'@
$p = Get-Content -Raw $sp
npx --yes acpx@latest --cwd "<target-dir>" --approve-reads --non-interactive-permissions deny copilot exec $p
```

## Invocation — persistent multi-turn session

Use when the task needs several back-and-forth turns (iterative review, a
multi-step task) and re-sending full context each call would be wasteful.
Session state lives in acpx's own store, keyed by name + cwd — not shared
with any other session on the machine.

```powershell
# once, at task start
npx --yes acpx@latest --cwd "<target-dir>" copilot sessions new -s <task-name>

# each turn
npx --yes acpx@latest --cwd "<target-dir>" --approve-reads --non-interactive-permissions deny copilot prompt -s <task-name> $p

# when the task is done
npx --yes acpx@latest --cwd "<target-dir>" copilot sessions close <task-name>
```

`prompt -s <name>` on a name that was never created via `sessions new` fails
with `No acpx session found` — always create before first prompt. Check
`copilot sessions --local` (no agent name = lists across agents) if unsure
what's alive; don't touch or close sessions you didn't create — the store is
shared machine-wide and may hold the user's own real Copilot history.

## Reporting back

Relay Copilot's answer, attributed: "Copilot says: …". Do not silently merge
its output into your own reasoning — the user is delegating specifically to
see Copilot's take, and Copilot's answer is unverified.

If Copilot edited files (`work`/`yolo`), review the diff yourself before
telling the user it's done:

```powershell
git -C "<target-dir>" diff --stat
```

## Failure modes

| Symptom | Cause | Action |
|---|---|---|
| `No acpx session found` | `prompt -s <name>` before `sessions new -s <name>` | create the session first |
| hangs, no output | a permission request has no auto-approve/deny match | widen mode by one step, or add `--non-interactive-permissions deny` |
| `not authenticated` | Copilot token expired | tell user to run `copilot login` themselves via `! copilot login` |
| `model_not_supported` | should not occur via ACP — regression if seen | fall back to `copilot -p ... --model auto` (see git history of this file) and report the acpx regression to the user |
| empty output, exit 0 | prompt too vague | restate with explicit file paths and a concrete deliverable |

Quota: Auto-mode requests on Student plan draw on included Copilot usage, not
Claude's. Premium-request billing still applies per GitHub's own metering —
see `copilot help billing`.
