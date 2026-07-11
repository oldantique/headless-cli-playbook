---
name: headless-cli
description: Call another local AI CLI headlessly from a Bash call — codex, opencode, or agy — to delegate a subtask, get a second opinion from a different model, fan out work in parallel, or run a one-shot prompt and capture the result. Use when the user asks to use/ask codex, opencode, agy, gemini, gpt, or deepseek; wants another model's take or a cross-model check; or when delegating coding/analysis to a non-Claude model is useful. Contains the exact verified flags plus the gotchas (stdin/sandbox/parallelism) that otherwise cause hung or empty calls.
user-invocable: true
---

# Headless CLIs: codex, opencode, agy

Call the three local coding CLIs non-interactively from a Bash call (the `claude -p` equivalent).
Follow the rules below — they stop a call from hanging or returning empty.
Setup, config, and the *why* behind each rule live in the repo README (pointer at the bottom).

> Models below are examples — substitute **your configured default** for each CLI. There is no fixed
> task→model mapping. Use whichever CLI/model you've set up; the rules and recipes are model-agnostic.

## Pick one

| | `codex exec` | `opencode run` | `agy -p` |
| --- | --- | --- | --- |
| Model | your ChatGPT-plan model @ your default effort | your default provider/model · `-m provider/model` to switch | your default Gemini model |
| Auth | ChatGPT sub — no env key | provider API key in env (auto-injected) | Google OAuth — no env key |
| Concurrency | parallel OK | parallel: **isolate data dir** | parallel OK |
| Files | read + edit (sandbox) | read/grep/edit — **absolute paths only** | read/edit, native |
| Structured out | `--output-schema` | `--format json` | none — ask in prompt |
| Use | **go-to peer for any problem**, esp. multi-turn (`resume`) | good supplement; parallel fan-out | good supplement; easiest file read/write |

## Invoke

```bash
# ask / explain (let them read files themselves)
codex exec "explain ./src and read ./config.toml" < /dev/null
opencode run "read /abs/path/README.md and list the headings"   # opencode: ABSOLUTE paths only
agy -p "read ./config.toml and summarize it" < /dev/null

# edit files
codex exec --sandbox workspace-write "fix the bug in ./app.py and save" < /dev/null
opencode run --dangerously-skip-permissions "add tests for utils.py"
agy --dangerously-skip-permissions -p "add tests for utils.py" < /dev/null

# pick model / effort — no fixed task→model mapping: codex is a strong go-to peer for ANY problem
# (esp. multi-turn), the others are good supplements
codex exec -c model_reasoning_effort="high" "..." < /dev/null
opencode run -m provider/model "..."                     # opencode effort flags hang — don't use
agy --model "Gemini 3.5 Flash (Low|Medium|High)" -p "..." < /dev/null  # switch within Gemini only

# structured JSON output
codex exec --output-schema schema.json -o out.json "extract X as JSON" < /dev/null
opencode run --format json "..."
# agy: no JSON flag — say "respond with only JSON: {...}" and parse loosely

# continue a session — codex keeps full context + model/effort across rounds;
# codex prints `session id: <uuid>` on STDERR (capture `2> r1.log`, then `grep -oP 'session id: \K\S+' r1.log`);
# use `resume <id>` whenever another codex call might land in between (`--last` = most recent session globally, races)
codex exec resume --last "now add error handling" < /dev/null   # or: resume <id>
opencode run -c "now add tests"                                 # or: -s <id>
agy -c "now add tests" < /dev/null                             # or: --conversation <id>
# multi-turn peer loop needs a stop condition: ask for "end with VERDICT: APPROVED or VERDICT: REVISE + reasons";
# loop `resume <id>` on REVISE, stop on APPROVED (cap ~5 rounds)

# big / multi-line prompt: file → arg for codex/agy, → STDIN for opencode
{ echo "Critique this:"; echo; cat plan.md; } > /tmp/p.txt
codex exec "$(cat /tmp/p.txt)" -o /tmp/out.md < /dev/null
cat /tmp/p.txt | opencode run > /tmp/out.md                # NOT a giant "$(...)" arg

# parallel fan-out — codex/agy as-is; opencode needs a private data dir per proc
ocp(){ local d; d=$(mktemp -d); XDG_DATA_HOME="$d/s" XDG_STATE_HOME="$d/t" opencode run "$@"; rm -rf "$d"; }
codex exec "task A" < /dev/null > a.md &
ocp "task B" > b.md &
agy -p "task C" < /dev/null > c.md &
wait
# many opencode calls, capped at 5:
printf '%s\n' "${prompts[@]}" | xargs -P 5 -I{} bash -c 'd=$(mktemp -d); XDG_DATA_HOME=$d XDG_STATE_HOME=$d opencode run "$1" > "out-$$.md"; rm -rf "$d"' _ {}

# codex built-in repo review
codex exec review < /dev/null
```

## Hard rules (break one = wasted / hung call)

**codex**
- Always append `< /dev/null` (else it waits on stdin).
- `-o <file>` to capture the answer (raw stdout is noisy). `--skip-git-repo-check` outside a git repo. `--sandbox workspace-write` to edit.

**opencode**
- **Parallel → private data dir per proc:** `XDG_DATA_HOME=$(mktemp -d) XDG_STATE_HOME=$(mktemp -d) opencode run …` (else concurrent runs hit `database is locked`). Only for parallel one-shots — `-c`/`-s` continue must use the default dir.
- **Big prompts via stdin** (`cat f | opencode run`), never a large `"$(cat f)"` arg (it hangs).
- **Reads: pin ABSOLUTE paths** — "read this EXACT path, don't modify: `/abs/…`" (relative paths can get mangled by the model → exit 0, empty stdout). Files outside the repo root can't be read — paste inline. Reviewing code with intra-repo imports: don't let it read at all — paste the code inline + "do NOT read any files".
- **Verify the output — exit code is useless** (exits 0 even on a rejected read). `cat prompt.txt | timeout 300 opencode run > out.txt 2> err.txt`, then check `out.txt` non-empty; empty ⇒ retry once.
- **Non-empty ≠ complete — check the document SHAPE too.** Long structured outputs can arrive truncated: the model prints the review progressively across intermediate turns (lost) and stdout captures only the final tail. Prevent: append "deliver the COMPLETE <deliverable> as ONE single final message — your final message is the only thing captured" to the prompt. Detect: expected header/section missing (e.g. no `## Verdict`) ⇒ rerun with that instruction.
- **`inotify_add_watch … No space left on device` at startup is NON-FATAL** — it's the inotify watch limit (`fs.inotify.max_user_watches`), not disk; only the file watcher dies, the run continues. Do NOT kill/retry on this error alone — an empty out-file mid-run + this error looks like a dead run but isn't: check the process is still alive (`pgrep`, growing stderr log) before declaring it dead. (Fix: raise `fs.inotify.max_user_watches`, persist in `/etc/sysctl.conf`.)
- **Never** `-f/--file` (eats the message), `--variant`, `--thinking` (can hang). Writing needs `--dangerously-skip-permissions`; large-model runs can be slow (minutes) — background them.

**agy**
- Stdin hang **fixed in agy 1.1.1** (`-p` no longer reads stdin when the prompt is given as a flag; pre-1.1.1 hung forever on a non-closing pipe from an agent harness). Recipes keep `< /dev/null` anyway — zero cost, required on <1.1.1.
- **Gemini only** by convention: switch only within Gemini models (`… (Low|Medium|High)`, `Gemini 3.1 Pro (Low|High)`) unless you deliberately want otherwise. Set your own configured default.
- Reads/writes natively; `--dangerously-skip-permissions` to write. No JSON flag. `--print-timeout` needs a unit (`10m`). Kill stuck procs with `pkill -9 -x agy` (never `-f agy`).

## Flags

**codex** (`codex exec --help`): `-m <model>` · `-c model_reasoning_effort="low|medium|high|xhigh"` · `-s/--sandbox read-only|workspace-write|danger-full-access` · `-o <file>` · `--output-schema <file>` · `--json` · `-C/--cd <dir>` · `--add-dir <dir>` · `-p/--profile <name>` · `--skip-git-repo-check` · `exec resume --last|<id>` · `exec review` · `--dangerously-bypass-approvals-and-sandbox` (only if sandbox breaks).

**opencode** (`opencode run --help`): `-m provider/model` · `--format json` · `--dangerously-skip-permissions` · `-c/--continue` · `-s/--session <id>` · redirect stdout `> file` (no `-o`). Avoid `--variant`, `--thinking`, `-f`. Note `-p` = `--password`, not print.

**agy** (`agy --help`): `-p/--print` (headless) · `--model "Gemini 3.5 Flash (High)"` (Gemini only) · `--dangerously-skip-permissions` · `-c/--continue` · `--conversation <id>` · `--add-dir <dir>` · `--sandbox` (opt-in; OFF by default) · `--print-timeout 10m` (unit required). No JSON flag.

## Runtime notes

- **Keys:** provider API keys (e.g. `DEEPSEEK_API_KEY`) must be visible to the non-interactive call. Agent harnesses don't source `~/.bashrc`/`~/.profile`, so inject the key via your harness's env mechanism (for Claude Code: the `settings.json` `env` block) — then no per-call export is needed. Fallback if a call auth-fails: `export DEEPSEEK_API_KEY="$(grep -oP 'DEEPSEEK_API_KEY="?\K[^" ]+' ~/.bashrc | tail -1)"`.
- **Orchestrating:** wait for CLIs IN-TURN — launch, then poll in the SAME turn (`until [ -s out.md ]; do sleep 20; done`, bounded), verify → retry-once → synthesize before ending. Don't go idle expecting a wake-on-completion.
- **Setup, config, login, sandbox fix, and the *why* behind these rules → the repo README (`README.md`).**
