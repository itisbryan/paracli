# paracli

Draft and submit your **Parabol daily standup** from your git activity — with your approval,
never automatically. One Ruby file, stdlib only, no gems. Works driven by an agent, by cron/launchd,
or by hand.

```
gather git → draft → notify (ntfy) → you approve → submit to Parabol
```

## Install

```bash
brew install itisbryan/tap/paracli
# or, latest from main:
brew install --HEAD itisbryan/tap/paracli
```

Or just drop `bin/paracli` on your PATH (it needs only `ruby` + `git`).

The `itisbryan/tap/paracli` formula is auto-bumped when a stable tag like `v0.1.0`
is pushed (see `.github/workflows/bump-homebrew-tap.yml`). It requires a repository
secret named `TAP_TOKEN` with write access to `itisbryan/homebrew-tap` (and permission
to bypass branch protection if the tap's default branch is protected).

## Prerequisites

- `ruby` (3.x+) and `git`.
- A **Parabol personal access token** in `$PARABOL_TOKEN` (Parabol → profile settings → Tokens).
  Required scopes: `USERS_READ`, `TEAMS_READ`, `MEETINGS_READ`, `MEETINGS_WRITE`.
- For automation: the [ntfy](https://ntfy.sh) app + a secret topic; macOS for launchd.

## Quick start

```bash
export PARABOL_TOKEN="…"            # in ~/.zshrc
paracli introspect                  # verify token + scopes
paracli init                        # index the repos where you commit
$EDITOR ~/.config/paracli/config.json   # trim to the repos this standup covers
paracli gather                      # preview your activity
paracli draft                       # preview the draft (nothing sent)
paracli list                        # get the meetingId (needs an open standup)
paracli submit <meetingId> <file> --yes   # post — only with --yes
```

## The repo index — `~/.config/paracli/config.json`

paracli doesn't guess where your work is; it reads this index (build it with `paracli init`).
Override the path with `$PARACLI_CONFIG`. All keys optional:

```json
{
  "repos":    ["/abs/repoA", "/abs/repoB"],
  "roots":    ["/abs/devdir"],
  "since":    "yesterday 06:00",
  "template": "full",
  "meeting_id": "RTc7Bnxm8M",
  "authors":  ["extra@email.com"]
}
```

| Key | Meaning |
|---|---|
| `repos` | Explicit repo paths. When set, root scanning is disabled (precise; new repos need re-`init`). |
| `roots` | Dirs to scan for git repos. Auto-includes new repos. Ignored if `repos` is set. |
| `since` | Git `--since` window for "what did I do". |
| `template` | `full` (Yesterday/Today/Blockers) or `today` (single Today block). |
| `meeting_id` | Pin a Parabol meeting; unset → auto-pick the one active standup. |
| `authors` | Extra author identities beyond each repo's git `user.name`/`user.email`. |

**Precedence: env var > config file > default.** Discovery skips `node_modules`, `vendor`,
`.bundle`, `Pods`, `.terraform`, `dist`, `build`, `.cache`, `tmp`.

## Commands

| Command | Effect |
|---|---|
| `list` | Active standup meetings: `<meetingId>\t[team] name`. |
| `submit <meetingId> [file\|-] --yes` | Post text as your response (upsert). **Refuses without `--yes`** — prints a preview. |
| `introspect` | Print the submit mutation's args (schema/auth check). |
| `init` | Discover work repos, write them to the index. Idempotent. |
| `discover` | Print candidate work repos as JSON. |
| `config` | Show resolved settings + index path. |
| `gather` | My git activity grouped by repo. |
| `draft [file\|-]` | A draft. No arg = gather+draft; `-` = stdin; else read a file. |
| `notify <file>` | Push a draft to ntfy with Approve/Reject buttons. |
| `run` | Full daily flow (what launchd runs). |
| `version` / `--selftest` | — |

## The submit gate

`submit` never posts without an explicit **signal** — the `--yes` flag. The CLI does not prompt;
**asking is the caller's job**: an agent asks you in chat, or the ntfy push asks you. Whoever did
the asking passes `--yes` after you approve. Without it, `submit` prints a preview and exits 0.

## Environment

| Var | Default | Purpose |
|---|---|---|
| `PARABOL_TOKEN` | — (required) | Parabol token (4 scopes above). |
| `PARABOL_ENDPOINT` | `https://action.parabol.co/graphql` | GraphQL endpoint (self-hosted override). |
| `PARACLI_CONFIG` | `~/.config/paracli/config.json` | Path to the JSON index. |
| `PARACLI_NTFY_TOPIC` | — (for notify/run) | **Secret** ntfy topic. |
| `PARACLI_NTFY_SERVER` | `https://ntfy.sh` | ntfy base URL (self-host for privacy). |
| `PARACLI_ROOTS` | config `roots`, else `~/Desktop` | Colon-separated roots to scan. |
| `PARACLI_SINCE` | config `since`, else `yesterday 06:00` | Git `--since` window. |
| `PARACLI_AUTHORS` | config `authors` | Comma-separated extra identities. |
| `PARACLI_TEMPLATE` | config `template`, else `full` | `today` or `full`. |
| `PARACLI_MEETING_ID` | config `meeting_id`, else auto | Pin a meeting. |
| `PARACLI_DRAFT_CMD` | — (mechanical) | Agent CLI reading a prompt on stdin, printing a draft (`claude -p`, `llm`, `codex exec`, `ollama run …`). |
| `PARACLI_DRAFT_ONLY` | — | `1` = draft + notify, never submit. |
| `PARACLI_APPROVE_TIMEOUT` | `10800` | Seconds `run` waits for an ntfy tap. |

## Automation (launchd + ntfy)

`paracli run`: gather → draft → ntfy push (✅/❌) → wait for your tap (bounded) → on Approve,
submit and push a result. Reject/timeout posts nothing.

```bash
cp com.itisbryan.paracli.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.itisbryan.paracli.plist
launchctl start com.itisbryan.paracli   # run once now
```

**Privacy:** ntfy.sh is a public broker; drafts pass through it. Use an unguessable topic, or set
`PARACLI_NTFY_SERVER` to a self-hosted ntfy. macOS launchd self-heals: a job missed while asleep
runs on wake.

## Files written

- `~/.config/paracli/config.json` — the index.
- `~/.cache/paracli/draft-<date>.txt` — each run's draft.
- `~/.cache/paracli/launchd.log` — launchd output.

Nothing is written into your repos; `$PARABOL_TOKEN` is never printed or persisted.

## Develop

`ruby bin/paracli --selftest` — offline, no token. Run after edits.

MIT.
