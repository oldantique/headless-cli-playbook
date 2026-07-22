# headless-cli-playbook

**headless-cli-playbook is the operational-hardening layer for driving four coding-agent CLIs — OpenAI's `codex`, `opencode`, Google Antigravity's `agy`, and Moonshot's `kimi` — headlessly and in parallel: the stdin deadlocks, silent empty reads, SQLite races, truncated outputs, and self-contradicting resume hints the vendor docs never warn you about, each paired with the exact one-line fix.**

It is *not* another "get a second opinion from another model" orchestrator — that space is well served already (see [Prior art & credits](#prior-art--credits)). This repo is the part nobody writes down: the dozen or so failure modes that turn a headless `codex exec` / `opencode run` / `agy -p` / `kimi -p` call into a hung terminal or an empty file, and how to make each one stop happening. The payload is one copy-installable Claude Code skill (`skills/headless-cli/SKILL.md`) that encodes every fix as a rule, plus this README explaining the *why*.

If you have ever run one of these CLIs from a script, a CI step, or an agent harness and watched it hang forever or exit `0` with nothing on stdout — this is the missing manual.

---

## The gotchas (symptom → root cause → one-line fix)

| Symptom | Root cause | One-line fix |
|---|---|---|
| `codex exec` hangs forever, no output | codex reads stdin until EOF; from a non-interactive harness stdin is an open pipe that never closes. | Append `< /dev/null` to the call. |
| `agy -p` hung forever on older versions | Same stdin-until-EOF read. **Fixed upstream in Antigravity CLI 1.1.1** (`-p` no longer reads stdin when the prompt is a flag). | On ≥1.1.1 nothing needed; recipes still append `< /dev/null` — zero cost, protects pre-1.1.1 installs. |
| Parallel `opencode run` → intermittent `database is locked` | Every run shares one SQLite session DB (WAL mode); concurrent writers race. Tracked upstream in [anomalyco/opencode#14970](https://github.com/anomalyco/opencode/issues/14970) and [#15188](https://github.com/anomalyco/opencode/issues/15188). | Give each process a private data dir: `XDG_DATA_HOME=$(mktemp -d) XDG_STATE_HOME=$(mktemp -d) opencode run …` ([verified workaround](https://github.com/anomalyco/opencode/issues/14970#issuecomment-4945987727)). |
| `opencode run` exits `0` with **empty** stdout on a file read | The model self-resolves a relative path and can mangle it (e.g. word-completing `user_doc`→`user_document`); the mangled path lands out-of-tree and is auto-rejected as external — exit code stays `0`. Reported upstream with a 5/5 repro in [anomalyco/opencode#36413](https://github.com/anomalyco/opencode/issues/36413). | Pin the **absolute** path and say "read this EXACT path, don't modify". For automation, prefer `--format json`: rejects show up as a tool part with `state.status: "error"`. |
| A long `opencode` review returns only a short tail | The model prints the body progressively across intermediate turns (which are discarded); stdout captures only the final message. | Prompt "deliver the COMPLETE deliverable as ONE single final message", then shape-check the output (expected header present?). |
| Any headless call auth-fails even though the key is in `~/.bashrc` | The Claude Code Bash tool (and most agent harnesses) source neither `~/.bashrc` nor `~/.profile`. | Put the key in Claude Code's `settings.json` `env` block so it's injected into every Bash call. |
| A *different* key in your shell rc silently turns invalid right after you add a new one | `echo 'export NEW_KEY=…' >> ~/.bashrc` appends with no leading newline. If the file didn't end in one, the new `export …` glues onto the **previous** line, absorbing itself into that line's value — the old key gets 6 extra characters and starts failing auth, while the key you just added parses fine, so nothing points at the append. | Always append with a leading newline: `printf '\n%s\n' 'export NEW_KEY=…' >> ~/.bashrc`. To check for existing damage, compare each key's `${#VAR}` against the provider's documented length. |
| `codex` refuses **every** command on Ubuntu ("needs access to create user namespaces") | AppArmor blocks unprivileged user namespaces, so codex's bubblewrap sandbox can't start. | `sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0` (persist in `/etc/sysctl.d/`). |
| `inotify_add_watch … No space left on device` at opencode startup | The inotify **watch** limit, not disk. Only the file watcher dies; the run continues. | Non-fatal — don't kill/retry. Raise `fs.inotify.max_user_watches` if it bothers you. |
| `kimi -r <session_id>` hangs forever — and the CLI itself told you to run it | Every headless `kimi -p` run ends by printing `To resume this session: kimi -r <id>`, but `-r` is not a headless flag: it drops into the interactive TUI, which then blocks on the harness's never-closing stdin. | Resume with `-S <id> -p "…"` instead. Read the id machine-readably from `--output-format stream-json` (`{"type":"session.resume_hint","session_id":…}`). |
| `kimi -p … -y` (or `--auto`) does nothing: `error: Cannot combine --prompt with --yolo` | Unlike the other three CLIs, kimi's headless prompt mode already auto-approves tool calls, so the permission flags are rejected as contradictory rather than redundant. | Drop the flag — plain `kimi -p "…"` already reads *and writes* files. |
| A CLI runs fine in your terminal but is `command not found` from your agent harness / CI | Its installer only put its bin dir on `PATH` by appending to `~/.bashrc`, which non-interactive callers never source. (`kimi` installs to `~/.kimi-code/bin` this way.) | Link the binary into a dir already on the inherited `PATH`: `ln -sf ~/.kimi-code/bin/kimi ~/.local/bin/kimi`. In-place upgrades keep the link valid. |
| A JS-runtime CLI dies at login with an empty `fetch failed:` while `curl` to the same host works | The host has no IPv6 route, but the endpoint's DNS lists IPv6 first. Node-style `fetch` honours DNS order verbatim and never falls back; `curl` retries on IPv4 by itself, which is why manual testing "proves" the network is fine. | Make the resolver prefer IPv4 host-wide: `echo 'precedence ::ffff:0:0/96  100' \| sudo tee -a /etc/gai.conf`. Verify with `getent ahosts <host>` — `getent hosts` uses a legacy path that ignores `gai.conf`. |

Every row is reproduced and fixed by the skill. The rest of this README is the setup and the recipes.

---

## What's in the box

```
headless-cli-playbook/
├── README.md                       # this file — setup + the why
├── LICENSE                         # MIT
├── .gitignore
└── skills/
    └── headless-cli/
        └── SKILL.md                # the call sheet — copy to ~/.claude/skills/
```

`SKILL.md` is a Claude Code skill: a terse "call sheet" that any Claude session loads on demand (or you invoke with `/headless-cli`). It carries the pick-one comparison table, the exact verified flags for each CLI, the copy-paste recipes (single call, multi-turn, parallel fan-out, structured JSON), and every hard rule from the table above. It is self-contained — you can also just read it as a reference even if you don't use Claude Code.

---

## 30-second install

```bash
# 1. clone (or download the skills/ dir)
git clone https://github.com/oldantique/headless-cli-playbook
# 2. copy the skill into your Claude Code skills dir
mkdir -p ~/.claude/skills
cp -r headless-cli-playbook/skills/headless-cli ~/.claude/skills/
```

That's it — the skill is now loadable in any Claude Code session (on demand via its `description`, or explicitly with `/headless-cli`). You still need whichever CLIs you actually want to call installed and authed; see below.

The skill and fixes are **verified on Linux** (see [Limits](#limits)). You do not need all four CLIs — install only the ones you'll use.

---

## Per-CLI setup

Each CLI authenticates differently. Pick the ones you need.

### `codex` — OpenAI Codex CLI (ChatGPT subscription)

```bash
npm install -g @openai/codex        # add --allow-scripts=@openai/codex if your npm blocks postinstall
codex --version
codex login --device-auth           # subscription auth, NOT an API key; prints a URL + code
codex login status
```

Set your default model / reasoning effort in `~/.codex/config.toml` (top-level keys before any `[table]`):

```toml
model = "gpt-5-codex"               # whatever your plan provides; this is your configured default
model_reasoning_effort = "high"
```

**Ubuntu sandbox fix (required).** codex runs tools in a bubblewrap sandbox; Ubuntu's AppArmor blocks unprivileged user namespaces, so the sandbox can't start and codex refuses to run anything:

```bash
echo 'kernel.apparmor_restrict_unprivileged_userns=0' | sudo tee /etc/sysctl.d/99-codex-userns.conf
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
bwrap --unshare-user --ro-bind / / true && echo "bwrap OK"     # verify
```

Smoke test: `codex exec --skip-git-repo-check "reply: ok" < /dev/null`

### `opencode` — any OpenAI-compatible provider (API key in env)

opencode is provider-agnostic: point it at **any** OpenAI-compatible endpoint and key the provider off an environment variable. `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "deepseek": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "DeepSeek",
      "options": {
        "baseURL": "https://api.deepseek.com/v1",
        "apiKey": "{env:DEEPSEEK_API_KEY}"
      },
      "models": {
        "deepseek-chat": { "name": "DeepSeek Chat" }
      }
    },
    "gateway": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My Gateway",
      "options": {
        "baseURL": "https://your-openai-compatible-gateway.example.com/v1",
        "apiKey": "{env:GATEWAY_API_KEY}"
      },
      "models": {
        "some-model": { "name": "Some Model" }
      }
    }
  },
  "model": "deepseek/deepseek-chat"
}
```

The `{env:VAR}` syntax reads the key from the environment — never hardcode it. The second `gateway` block is a template for **any** OpenAI-compatible aggregator (many such gateways expose `GET /v1/models` to enumerate what's available). As one concrete example, Tencent MaaS is one documented gateway that fronts several models behind a single key; swap in whichever provider you use.

```bash
npm install -g opencode-ai          # add --allow-scripts=opencode-ai if your npm blocks postinstall
opencode --version
```

**Key injection (the fix).** opencode reads its provider key from the environment. Interactive shells can `export` it from `~/.bashrc`, but agent harnesses (Claude Code's Bash tool included) don't source your shell rc files — so put the key where every non-interactive call will see it. For Claude Code, that's the `settings.json` `env` block:

```json
{ "env": { "DEEPSEEK_API_KEY": "sk-...", "GATEWAY_API_KEY": "sk-..." } }
```

Smoke test: `DEEPSEEK_API_KEY=sk-... opencode run "reply: ok"`

### `agy` — Google Antigravity CLI, Gemini (Google OAuth)

```bash
curl -fsSL https://antigravity.google/cli/install.sh | bash
agy --version
agy                                 # first run opens Google OAuth; on a headless box it prints a URL + code
agy models
```

Default model in `~/.gemini/antigravity-cli/settings.json` (the key takes the display name):

```json
{ "model": "Gemini 3.5 Flash (High)" }
```

Effort is the parenthetical in the model name. `agy` also lists non-Gemini options; the skill's convention is to stick to Gemini models unless you deliberately want otherwise — set your own configured default.

Smoke test: `agy -p "reply: ok" < /dev/null`

### `kimi` — Moonshot Kimi Code CLI (Kimi subscription OAuth)

```bash
curl -fsSL https://code.kimi.com/kimi-code/install.sh | bash   # single binary, no Node required
kimi --version
kimi login                          # device-code flow; prints a URL + code, works over SSH

# required for headless use: the installer only exports ~/.kimi-code/bin from ~/.bashrc, which
# agent harnesses and CI never source — link it somewhere already on the inherited PATH
ln -sf ~/.kimi-code/bin/kimi ~/.local/bin/kimi
```

Default model and reasoning effort live in `~/.kimi-code/config.toml`. There is **no CLI effort flag** — set it here:

```toml
default_model = "kimi-code/k3"

[thinking]
enabled = true
effort = "high"                     # low | high | max, per the model's support_efforts
```

> If `kimi login` fails with a bare `fetch failed:` while `curl https://auth.kimi.com/` succeeds, you've hit the IPv6 row in the gotchas table — apply the `gai.conf` fix before anything else.

Two behaviours to internalise, both the inverse of the other CLIs: `-p` **already** auto-approves tool calls (adding `-y`/`--auto` is a hard error), and you resume with **`-S <id>`**, not the `-r <id>` the CLI's own output suggests.

Smoke test: `kimi -p "reply: ok"`

---

## Recipe: the multi-turn peer teammate

The under-used mode. `codex exec resume` (and `opencode -c` / `agy -c`) keep full context, model, and effort across rounds — so instead of a one-shot "review this", you can run a many-round back-and-forth with a different model as a genuine peer teammate: propose, critique, revise, re-critique, converge. In real projects the author runs `gpt-5.6-sol` this way over many rounds and finds it an excellent second brain, not just a review gate.

> **Models the author currently runs** (a 2026-07 snapshot — these drift; any model each CLI supports works):
> codex → `gpt-5.6-sol` @ `xhigh` · opencode → DeepSeek V4 `flash` (fast) / `pro` (deep) · agy → `Gemini 3.5 Flash (High)` / `Gemini 3.1 Pro (High)` · kimi → `K3` @ `high`.

The trick to making a loop terminate is a **VERDICT** convergence signal:

```bash
# round 1 — codex prints `session id: <uuid>` on STDERR, so capture the streams separately
codex exec "Review ./plan.md. End with exactly 'VERDICT: APPROVED' or 'VERDICT: REVISE' + reasons." \
  < /dev/null > round1.txt 2> round1.log
sid=$(grep -oP 'session id: \K\S+' round1.log)

# loop: feed your revision back, stop on APPROVED (cap ~5 rounds so it can't run away)
for i in $(seq 1 5); do
  out=$(codex exec resume "$sid" "Here's my revision: <...>. Same VERDICT rule." < /dev/null)
  echo "$out"
  grep -q 'VERDICT: APPROVED' <<<"$out" && break
done
```

Use `resume <id>` (not `--last`) whenever another codex call might land in between — `--last` picks the most recent session *globally* and will race.

## Recipe: parallel fan-out

Run all four at once. `codex`, `agy`, and `kimi` parallelize as-is; `opencode` needs a private data dir per process (the SQLite race above):

```bash
# one private data dir per opencode process
ocp(){ local d; d=$(mktemp -d); XDG_DATA_HOME="$d/s" XDG_STATE_HOME="$d/t" opencode run "$@"; rm -rf "$d"; }

codex exec "task A" < /dev/null > a.md &
ocp    "task B"                   > b.md &
agy -p "task C" < /dev/null       > c.md &
kimi -p "task D"                  > d.md &
wait

# many opencode tasks, capped at 5 concurrent:
printf '%s\n' "${prompts[@]}" \
  | xargs -P 5 -I{} bash -c 'd=$(mktemp -d); XDG_DATA_HOME=$d XDG_STATE_HOME=$d opencode run "$1" > "out-$$.md"; rm -rf "$d"' _ {}
```

Because config (`~/.config`) and the model cache (`~/.cache`) stay shared, the private data dir costs nothing to re-fetch — it isolates only the session DB.

---

## Prior art & credits

This project stands on a lot of shoulders and deliberately does **not** try to be a general multi-model orchestrator — several excellent ones already exist:

- **[PAL MCP](https://github.com/BeehiveInnovations/pal-mcp-server)** — the large, mature "consult another model" MCP server. If you want a polished second-opinion tool, start there.
- **[nyldn/claude-octopus](https://github.com/nyldn/claude-octopus)** — orchestrates the same three CLIs (plus more) as parallel workers; broader fan-out framing than this repo.
- **[skills-directory/skill-codex](https://github.com/skills-directory/skill-codex)** — a widely-used Claude skill for calling codex.
- **[antonbabenko/deliberation](https://github.com/antonbabenko/deliberation)** — a subscription-auth `codex` + `agy` deliberation stack; close cousin of the peer-teammate recipe here.
- **Aseem Shrey's codex-review skill** — the `codex exec resume` multi-turn-review mechanism the peer-teammate recipe builds on; credit for demonstrating that pattern.

What this repo adds that isn't written down elsewhere: the per-process `XDG_DATA_HOME` isolation for opencode's shared-SQLite bug, the DeepSeek relative-path-mangling → silent-empty-read rule, the `< /dev/null` stdin-deadlock fix, the output-truncation shape-check, kimi's `-r`-hint-that-hangs and its `-p`-is-already-autonomous inversion, the `~/.bashrc`-only PATH install that makes a CLI invisible to every non-interactive caller, the IPv6-first DNS trap that kills JS-runtime CLIs on IPv4-only hosts, and framing the multi-turn `resume` loop as an open-ended peer *teammate* rather than only a review gate. If you're building an orchestrator, treat this as the hardening notes to fold into it.

---

## Limits

Read these before trusting a rule blindly:

- **Linux-verified only.** Every fix here was reproduced on Linux (Ubuntu). macOS likely differs (no AppArmor/bubblewrap sandbox story; different inotify equivalent); Windows is untested. PRs with platform notes welcome.
- **These CLIs move fast.** Flags, default models, and auth flows change between releases. Treat the exact flag strings as "verified at time of writing" and re-check with `--help` if something behaves differently. Version numbers in examples are illustrative, not pinned.
- **Some gotchas are model-specific.** The relative-path mangling and progressive-output truncation were observed with particular models behind `opencode`; a different model on the same CLI may not reproduce them (or may add new ones). The *shape* of the fix (absolute paths, single-final-message, output shape-check) is defensive regardless.
- **No secrets, ever.** Every key in this repo is an `sk-...` placeholder. Keep real keys in your environment / `settings.json`, never in a committed file.

## Roadmap

- macOS verification pass (sandbox + inotify equivalents).
- A `doctor` script that smoke-tests all four CLIs and prints the exact fix for each failure.
- Contributed gotchas for more OpenAI-compatible providers behind `opencode`.
- CI that re-runs the smoke tests against the latest CLI releases to catch flag drift.

Contributions of new, *reproduced* gotchas (symptom + root cause + one-line fix) are the most valuable thing you can send.

---

MIT licensed. Not affiliated with OpenAI, DeepSeek, Google, or any provider named here; CLI behaviors are documented from the author's own observation and interoperability inspection.
