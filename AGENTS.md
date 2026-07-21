# For AI agents installing or operating this repo

You are likely a coding agent whose user pasted this repo's link and said "set this up".
Everything you need is here; the flow is: install the skill → verify the CLIs → operate by the rules.

## Install (one step)

Copy the skill into the user's Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills
cp -r skills/headless-cli ~/.claude/skills/
```

That's the whole install. The skill loads on demand in any Claude Code session (or explicitly via
`/headless-cli`). If the user runs a different harness, `skills/headless-cli/SKILL.md` still works
as a plain reference document — read it before calling any of the four CLIs.

## Verify (before first use)

Only check the CLIs the user actually intends to call:

```bash
codex --version    && codex exec --skip-git-repo-check "reply: ok" < /dev/null   # needs ChatGPT login
opencode --version && opencode run "reply: ok"                                   # needs provider key in env
agy --version      && agy -p "reply: ok" < /dev/null                             # needs Google login
kimi --version     && kimi -p "reply: ok"                                        # needs Kimi login
```

If one fails, the fix is in `README.md` → the matching "Per-CLI setup" section (auth, the Ubuntu
sandbox sysctl for codex, key injection for opencode, the `gai.conf` IPv4-preference fix if a login
dies with a bare `fetch failed:`). Don't improvise flags — the failure modes are specific and the
fixes are one-liners.

## Operate — safety rules

- **Never print or commit a real API key.** Keys live in the environment (or the harness's env
  block); config files reference them as `{env:VAR}`. If you need to confirm a key exists, test
  `${VAR:+set}` — never echo the value.
- **Verify every opencode result**: exit code 0 means nothing — check stdout is non-empty, or use
  `--format json` and look for tool parts with `state.status: "error"`.
- **Dry-run first**: for file-editing calls, run once without the write flag
  (`--sandbox workspace-write` / `--dangerously-skip-permissions`) to see what the model intends.
- **Parallel opencode needs a private data dir per process** (see the SKILL's `ocp()` helper) —
  otherwise you'll hit the shared-SQLite race. The other three parallelize as-is.
- **kimi inverts two conventions**: `kimi -p` already auto-approves tool calls (adding `-y`/`--auto`
  is a hard error), and you resume with `-S <id>`, never the `-r <id>` its own output prints — `-r`
  falls into the interactive TUI and hangs.
- When a call hangs or returns empty, consult the gotchas table in `README.md` before retrying:
  every known hang/empty case has a known cause.
