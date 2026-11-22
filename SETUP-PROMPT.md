# Claude Code team setup for any project

One prompt that gives a project a guarded team of Claude Code agents: house rules read from the code, four read-only
specialist agents (reviewer, security reviewer, review verifier, offline QA), a `/cr` code-review skill, a PreToolUse
guard that fences the specialists, helper skills taken from [ECC](https://github.com/affaan-m/ECC), and one Claude
session per agent.

**How to use it.** Open the project folder in Claude Code and start a new session. Fill in the fields below or
leave them blank. Then either send this whole file as your message, or save it in the project (for example as
`SETUP-PROMPT.md`) and send: `Read SETUP-PROMPT.md and follow it.`

**What it needs.** bash and python3 3.8 or newer. The offline QA sandbox needs Linux (`unshare -rn`); on other
systems QA is limited to static checks. One session per agent needs the Claude desktop app (Code tab); elsewhere the
setup prints the session briefs for you to paste. The only network access is one `git clone` of ECC.

---

## Fields

- PREFIX:
- TEAM:
- LANGUAGE:
- HOSTS:
- HUMANIZER (optional path to a humanizer-style writing skill folder):

Blank fields: PREFIX is the first word of the project folder name, lowercased, letters and digits only (`acme-api`
gives `acme`). PREFIX followed by `-` must not be the start of any agent type the Agent tool already lists, because
the guard refuses every unregistered agent with that start: `web` would block `web-fetch`, `code` would block
`code-simplifier`, and `claude`, `general` or `statusline` collide the same way. On a collision, join the first two
words of the folder name (`web-app` gives `webapp`). TEAM is PREFIX with a capital first letter. LANGUAGE is the
language of any words I wrote around this prompt, or English when there are none. HOSTS are the external hosts the
code talks to (APIs, CDNs, databases, third-party sites), found in step 1. HUMANIZER, when blank, is
`~/.claude/skills/humanizer` if it exists, otherwise none. The final report says what you chose.

## Your task

You are the setup session. Build everything below in this project in one go, then report. The design is decided: do
not brainstorm, do not offer a list of proposals, do not ask me to approve a plan. This prompt is my explicit yes to
every step below, including starting the five team sessions in step 5. Do not stop between steps.

Stop and ask me only when:

- a step needs network access other than the ECC clone in step 2;
- a step needs `git init`, a commit or a push;
- you would do something this prompt does not list;
- the preflight in step 0 fails (no bash or no python3).

Talk to me in LANGUAGE. Keep identifiers, paths and commands as they are.

## What you build

| Piece | Path | Purpose |
|---|---|---|
| Root rules | `CLAUDE.md` | language and writing rules; graphify and humanizer sections when available |
| Work rules | `.claude/CLAUDE.md` | do only what the user said, no servers, git rule, review and QA rules, agents, guard, team |
| Code rules | `AGENTS.md` | the project's real code standards, read from the code |
| House skill | `.claude/skills/@@prefix@@-@@area@@/SKILL.md` | facts of this project: what runs where, contracts, data, network, known weaknesses |
| Helper skills | `.claude/skills/<ecc-skill>/` | verbatim from ECC, each with ECC's `LICENSE` |
| Agents | `.claude/agents/@@prefix@@-{reviewer,security-reviewer,review-verifier,qa}.md` | read-only specialists, `model: inherit` |
| Review skill | `.claude/skills/cr/SKILL.md`, `scripts/lint.sh`, `scripts/qa-sandbox.sh` | `/cr`: lint, parallel reviewers, verifier, one report |
| Guard | `.claude/hooks/agent-guard.py`, `.claude/hooks/agent-guard-cases.py` | PreToolUse allowlist for the specialists, with its tests |
| Team | `.claude/team/register.py`, `.claude/hooks/team-session-start.py`, `.claude/skills/@@prefix@@-team/SKILL.md` | one session per agent, registered and fenced |
| Hooks | `.claude/settings.json` | launchers for the guard (hybrid failure mode) and the session-start reminder |
| Logs | `docs/team/LOG.md`, `docs/code-review/` | the user's decisions; review reports |
| Ignores | `.claudeignore`, `.graphifyignore` | keep binaries, dependencies, build output and `.claude/` out of context and graphs |

## Decisions already made

Do not ask about these.

1. ECC is not installed: no `/plugin install`, no `install.sh`, none of its agents, hooks or `model: sonnet`. Its
   70+ agents with "MUST BE USED / PROACTIVELY" descriptions would run outside the guard. Take only the skills that
   fit this project, verbatim with ECC's `LICENSE`, as helper skills. Good ideas from ECC agents are rewritten into
   the house agents. Where a helper skill disagrees with a house rule, the house rule wins.
2. Specialists are read-only. Only the main session (the lead) changes code, config and git.
3. The guard is an allowlist. An agent whose name starts with `@@prefix@@-` but is not in `SPECIALISTS` is refused
   outright, so every new agent has to be registered.
4. The hook failure mode is hybrid. If the guard file or python3 is missing, the launcher blocks the specialists and
   lets the main session through with a warning. An error inside the guard refuses the specialist. The main session
   is never locked out.
5. No `git init`, commit or push. Under git, `/cr` reviews the diff; without git, the files the user names.
6. Review reports go to `docs/code-review/`, written in LANGUAGE. The whole project is reviewed only when the user
   asks for it in words.
7. QA is offline. Code runs only through the sandbox script with the network cut off, and QA writes only to
   `/tmp/@@prefix@@-qa`.
8. Optional ECC skills that add little to reviewing this project (for example `accessibility`, `click-path-audit`)
   are left out and listed as proposals.
9. Graphify gets a conditional rule in `CLAUDE.md` and nothing more: no hooks, and you do not build a graph.
10. One session per agent, fenced by the same guard: the lead «@@LEAD@@» plus four specialist sessions. The lead is
    pinned, and every team session is detached from this setup session, so archiving the setup session later does not
    take the team with it.
11. The guard's `TOOLS` keeps `SubagentHandback`, which a subagent uses to return its report (without it `/cr` gets
    no answers), and `advisor`.
12. A whole-project review or test (the user asked for the whole project in words) runs in the team sessions, not
    as `/cr` subagents. The lead sends one task per message to «@@TEAM@@ reviewer», «@@TEAM@@ security reviewer» and
    «@@TEAM@@ QA», then the blocking findings to «@@TEAM@@ review verifier», and writes the one report itself. With
    no registered team sessions it falls back to `/cr` subagents. A review of a change stays with `/cr` subagents.
13. Nothing else from other setups: no session guard, no "no wait loops" hook, no QA database.

## Step 0: preflight (write nothing)

- `command -v bash python3 git`, `python3 --version` (3.8+), and `unshare -rn true`. If `unshare` fails, the QA
  sandbox will refuse to run and QA is static only; say so in the report.
- `printenv CLAUDE_CODE_SESSION_ID`. If it is empty, team sessions cannot register: build everything, skip step 5 and
  say so.
- Is the project under git, and does it have a commit? You need this for `.claude/CLAUDE.md` and `/cr`.
- What exists already: `CLAUDE.md`, `.claude/`, `AGENTS.md`, `.claude/settings.json`, ignore files. Merge into
  existing files; never overwrite them or drop what is there. Existing hooks in `settings.json` stay, and the new ones
  are added beside them. Agents already in `.claude/agents/` without the prefix stay as they are and outside the
  guard; list them in the report.
- The agent types the Agent tool lists, for the PREFIX collision rule in the fields.
- Whether `graphify` is on the PATH, and whether the HUMANIZER folder has a `SKILL.md`.

## Step 1: read the project

Read the code itself and trust it over the README. Find the language and runtime, the entry points, how the parts
talk to each other (HTTP routes, queues, messages, CLI), data and storage, the external hosts (they fill HOSTS),
secrets (`.env`, keys, tokens) and where they are read, build, test and lint commands, dependencies, and the code
style as it is actually written (indentation, quotes, naming, error handling, logging, comments). In a large project,
read the entry points and the main paths, and list what you did not read. Pick AREA: one word for the house skill,
such as `backend`, `extension`, `web` or `cli`.

## Step 2: ECC helper skills

This is the only network access allowed:

```bash
git clone --depth 1 https://github.com/affaan-m/ECC <your scratchpad>/ecc-src/ECC
git -C <your scratchpad>/ecc-src/ECC log -1 --format=%h
```

Read `agents/` and `skills/`. Take a skill when its content applies to this stack and to reviewing or testing it.
Leave out ECC agents as agents, anything that writes code, skills that need the network (`npx`, `npm install`, live
scans), skills that bring their own hooks (`gateguard`, `safety-guard`, `plankton-code-quality` and similar), and
skills for another language or framework. Always take `security-review`, and `agent-security-hardening`, which the
security reviewer uses to review the guard. Typical additions are `error-handling`, `change-impact`, `e2e-testing`,
`browser-qa` for a UI, and the pattern skills for the stack's language. If a skill you need is not in the clone, do
not fetch it from anywhere else; list it in the report. Copy each chosen skill folder verbatim and put ECC's
`LICENSE` in it.

## Step 3: write the files

### Placeholders

Appendices A to F and templates T1 to T6 use these tokens. Replace every one of them.

| Token | Value |
|---|---|
| `@@prefix@@` | PREFIX, lowercase |
| `@@PREFIX@@` | PREFIX in capitals |
| `@@TEAM@@` | TEAM |
| `@@LEAD@@` | TEAM in capitals followed by ` - Coding` (for example `ACME - Coding`) |
| `@@PROJECT@@` | the product or project name, from the code |
| `@@area@@` | AREA |
| `@@DATE@@` | today, YYYY-MM-DD |
| `@@LANGUAGE@@` | LANGUAGE |
| `@@HOSTS@@` | HOSTS, comma separated, or `external services` when there are none |
| `@@code_file@@`, `@@subdir@@`, `@@ext@@` | Appendix B only: a real source file and a real subdirectory (both relative to the project root), and the main source extension |

### Copy from the appendices

- A to `.claude/hooks/agent-guard.py`. Change only the placeholders and the constants marked `STACK`.
- B to `.claude/hooks/agent-guard-cases.py`. Fill `CODE`, `SUBDIR` and `EXT`. Add cases for whatever you add to
  `READ_ONLY`, `SCRIPTS`, `SECRET_NAME` or `RUNTIME_CHECK`, and never delete an existing refusal case.
- C to `.claude/team/register.py`.
- D to `.claude/hooks/team-session-start.py`.
- E to `.claude/settings.json`. If the file exists, add the two hook entries to it.
- F to `.claude/skills/cr/scripts/qa-sandbox.sh` when the project runs on Node. For another runtime, write the
  equivalent with the same contract: run under `unshare -rn` and refuse to run when that fails; run only scripts
  under `/tmp/@@prefix@@-qa`; read the project; write only `/tmp/@@prefix@@-qa` (use the runtime's own permission
  flags where it has them); never download packages (for example `GOFLAGS=-mod=mod GOPROXY=off`, `PIP_NO_INDEX=1`).

Make the scripts executable.

### Write from the templates

T1 is `CLAUDE.md`, T2 `.claude/CLAUDE.md`, T3 `.claude/skills/@@prefix@@-team/SKILL.md`, T4
`.claude/skills/cr/SKILL.md`, T5 the four agents, T6 `docs/team/LOG.md`. Text in `<angle brackets>` inside a
template is an instruction to you; replace it with real content for this project.

### Derive from the code

- `AGENTS.md`, with these sections: 1 Stack (mandatory); 2 Code style; 3 the project's patterns (APIs, messages,
  errors, data, network); 4 UI patterns, if there is a UI; 5 Behavioral rules (surgical changes, verify with
  `lint.sh`, never contact HOSTS or any server, no commit or push unless asked); 6 User preferences and dynamic
  overrides, empty, with the line "Record here every code style or workflow preference the user states, with the
  date." Every rule must be visible in the code; the top of the file says it was read from the code on @@DATE@@.
- The house skill `.claude/skills/@@prefix@@-@@area@@/SKILL.md`. Frontmatter `name` and `description`; the
  description says what the project is and "Load it first whenever you review, test or change <the main paths>; the
  @@prefix@@-* agents load it before any helper skill. Where a helper skill disagrees, this skill and AGENTS.md win."
  Then the NEVER CONNECT header from T5 and numbered sections: what runs where; contracts (APIs, messages, events);
  data and storage; network and external hosts; the main pipelines; known weaknesses as of @@DATE@@; what offline QA
  can and cannot cover.
- `.claude/skills/cr/scripts/lint.sh`: offline syntax and consistency checks for this stack, on the files given or,
  without arguments, on the whole project. No network, no writes, exit 1 on any failure, `WARN` lines for things
  that are not failures. Use only tools that are already installed; leave out any checker that would download
  something.
- The `STACK` constants of the guard: `RUNTIME_CHECK` (only commands that neither write nor reach the network; empty
  when unsure), extra programs in `READ_ONLY` only if they are truly read-only, extra project scripts in `SCRIPTS`,
  extra secret file patterns in `SECRET_NAME` when the stack has them (such as `*.tfvars` or `credentials.json`).
- `.claudeignore` and `.graphifyignore`: dependencies, build output, binaries, IDE folders, test reports,
  `__pycache__`. `.graphifyignore` also lists `.claude/` and `docs/code-review/`. If graphify is used,
  `.claudeignore` hides `graphify-out/` except `graphify-out/wiki/index.md`.

### Never

- Do not write `sessions.json` or `specialists.txt` yourself; `register.py` creates them.
- Do not leave a placeholder or a template instruction in any file.
- Do not substitute `general-purpose` for a house agent; it runs without the guard. Agent files load when a session
  starts, so this setup session cannot run the new agents at all. The lead tests them in step 5.

### Traps

- The marker `@@PREFIX@@_AGENT_GUARD` and the `@@prefix@@-` pattern appear both in the guard and inside the one-line
  launcher in `settings.json`. If they differ, the launcher quietly treats the guard as missing and falls back to
  hybrid mode.
- `/tmp/@@prefix@@-qa` appears in the guard's `SPECIALISTS`, in the sandbox script, in the QA agent and in the cases.
- `@@PREFIX@@_TEAM_DIR` appears in the guard, in the cases and in `team-session-start.py`.
- A new agent goes into `SPECIALISTS` in the guard, `ROLES` in `register.py` and the table in the team skill, and
  then the cases are run again.

## Step 4: verify offline

All of these must hold before step 5.

- `python3 .claude/hooks/agent-guard-cases.py` ends with `all passed`.
- `.claude/skills/cr/scripts/lint.sh` runs on the whole project. Real problems in the code go into the report; do not
  fix code.
- If `unshare -rn` works: a trivial passing test in `/tmp/@@prefix@@-qa/` passes through the sandbox, and a script
  that tries the network cannot reach it. Remove `/tmp/@@prefix@@-qa/` afterwards.
- Every JSON file parses, every agent and skill frontmatter parses, and every `.claude/agents/*.md` is in
  `SPECIALISTS` and in `ROLES`.
- No placeholder is left in the files you wrote. The copied ECC skills stay out of this check, because they can
  contain `@@` legitimately:

  ```bash
  grep -rn '@@' CLAUDE.md AGENTS.md docs/team .claude/CLAUDE.md .claude/settings.json .claude/hooks .claude/team .claude/agents .claude/skills/cr .claude/skills/@@prefix@@-*
  ```

- Every path named in `CLAUDE.md`, `.claude/CLAUDE.md`, the agents and the skills exists.

## Step 5: team sessions

Skip this step if `CLAUDE_CODE_SESSION_ID` was empty.

If you have the Claude desktop app's session tools (starting, detaching and pinning sessions), start five sessions
with `initiation: "user_asked"`, `context: "fresh"`, `use_worktree: false` and exactly these titles: «@@LEAD@@»,
«@@TEAM@@ reviewer», «@@TEAM@@ security reviewer», «@@TEAM@@ review verifier», «@@TEAM@@ QA». In each `background`,
write one sentence about the project, one about the team (the lead is the only session that changes code, the four
specialists are fenced by the guard, messages go by SendMessage as the team skill describes), and that the user set it
up with this prompt. Without those tools, print the five briefs and ask me to open five sessions in this folder with
those titles and paste one brief into each.

The lead's brief (the live probe is here because a fresh session has the new agents loaded):

> You are the lead session «@@LEAD@@» of the @@PROJECT@@ project (role `lead`). This first turn is setup and a live
> guard check only: do not change any project file, do not commit, do not contact any server.
>
> 1. Run `ListAgents` and take this session's name exactly as it prints it ("This session is …").
> 2. Register: `python3 .claude/team/register.py lead "<that name>"`.
> 3. Load the skill `@@prefix@@-team`; read `AGENTS.md`, `.claude/CLAUDE.md` and `docs/team/LOG.md`.
> 4. Live guard check. Confirm that the Agent tool lists `@@prefix@@-reviewer`, `@@prefix@@-security-reviewer`,
>    `@@prefix@@-review-verifier` and `@@prefix@@-qa`; if not, report it and stop. Launch `@@prefix@@-reviewer` with
>    these steps, each a separate tool call, no retries: (a) Read the first 5 lines of a project file; (b) Bash
>    `curl --version`; (c) Bash `sed -n '1,3p' <a project file>`; (d) Bash `ls ~`; (e) Write
>    `/tmp/@@prefix@@-guard-probe.txt`. Expected: a and c allowed; b and d refused by `.claude/hooks/agent-guard.py`;
>    e refused or the tool not available; and the agent's report reaches you through `SubagentHandback` on the first
>    try. Launch `@@prefix@@-qa`: write a trivial passing test to `/tmp/@@prefix@@-qa/` and run it through
>    `.claude/skills/cr/scripts/qa-sandbox.sh`. Expected: both allowed, the test passes. Then remove
>    `/tmp/@@prefix@@-guard-probe.txt` (if it exists, the guard did not fire) and `/tmp/@@prefix@@-qa/`, and run
>    `python3 .claude/hooks/agent-guard-cases.py`.
> 5. Run `python3 .claude/team/register.py --show`.
> 6. If your system prompt names a shared notes folder, write the probe as a table (step, expected, got, first line
>    of each refusal) to `guard-live-probe-@@DATE@@.md` there.
> 7. Reply in @@LANGUAGE@@. The first line is `PROBE PASS` or `PROBE FAIL` and the file name, because only the first
>    400 characters of your reply reach the session that started you. Then your registered name and role, who else
>    is registered so far (the four specialists register in parallel, so some may be missing), the probe table if
>    you had no notes folder, and that you are waiting for the user's task. Then stop.

A specialist's brief, once for each role (`reviewer`, `security-reviewer`, `review-verifier`, `qa`):

> You are the team session «@@TEAM@@ <role name>» of the @@PROJECT@@ project (role `@@prefix@@-<role>`). This
> first turn is setup only: do not review, verify or test anything yet.
>
> 1. Run `ListAgents` and take this session's name exactly as it prints it ("This session is …").
> 2. Register: `python3 .claude/team/register.py @@prefix@@-<role> "<that name>"`. From now on
>    `.claude/hooks/agent-guard.py` fences this session as a specialist.
> 3. Load the skill `@@prefix@@-team`, read `.claude/agents/@@prefix@@-<role>.md`, and load the skill
>    `@@prefix@@-@@area@@`.
> 4. Run `python3 .claude/team/register.py --show`.
> 5. Reply in @@LANGUAGE@@ in two or three lines: your registered name and role, and that you are waiting for tasks
>    from «@@LEAD@@» or the user. Then stop; do not start work of your own.

For QA, point 2 also says that it writes only to `/tmp/@@prefix@@-qa` and runs code only through
`.claude/skills/cr/scripts/qa-sandbox.sh`. In every brief, point 1 adds: if `ListAgents` is not available, use the
session's title as its name.

When all five have answered:

- `python3 .claude/team/register.py --show` lists five sessions.
- Read the lead's probe file in the shared notes folder. If the probe failed, fix the guard (this setup session is
  outside the team and may change it), run the cases again, and tell the lead in one line.
- Pin the lead and detach all five sessions from this one, one detach call each. Check with the session list that
  they are no longer marked as started by this session. If pinning or detaching is refused, do not retry; tell me the
  manual step (right-click the session in the sidebar, then "Detach to top level" or "Pin").
- Do not archive yourself.

## Step 6: final report, in LANGUAGE

- The PREFIX, TEAM, LANGUAGE, HOSTS and AREA you chose, if any field was blank.
- The ECC commit; the skills you took and why; what you left out and why, briefly.
- The files you created and the files you changed.
- Check results: the number of passing cases, lint, the sandbox, the lead's live probe.
- What was not checked (a real browser, live services) and the manual steps for it.
- Proposals you did not act on: extra specialists, optional skills, real problems lint found.
- That the setup session can now be archived and the team will stay.

---

## T1. `CLAUDE.md`

Add these sections to an existing `CLAUDE.md`, or create it.

````markdown
## Language and writing

- Answer the user in @@LANGUAGE@@, with correct spelling. Keep identifiers, paths and commands exactly as they are.
- Write plainly and to the point, like a colleague: no boilerplate openings, inflated claims or repetition.

<Only if graphify is on the PATH; otherwise leave this section out.>
## graphify

This project may have a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file
relationships.

Rules:
- For codebase questions, when graphify-out/graph.json exists, first run `graphify query "<question>"`. Use
  `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface
  enough context.
- After modifying code, run `graphify update .` to keep the graph current, only when graphify-out/ exists.
- If graphify-out/ does not exist or the `graphify` command fails, say so once and use Read/Grep. The project has
  about <N> lines; do not build a graph unless the user asks.

<Only if the HUMANIZER folder has a SKILL.md: copy that folder to .claude/skills/humanizer/ with its LICENSE first.
Otherwise leave this section out.>
## Humanizer

- Before the first response in each new session, read `.claude/skills/humanizer/SKILL.md`.
- Apply its guidance automatically to replies, explanations, messages and review reports.
- Return only the finished text by default. Preserve facts, amounts, dates, IDs and technical meaning; keep code and
  commands exact.
````

## T2. `.claude/CLAUDE.md`

````markdown
# Rules of work (user, setup prompt, @@DATE@@)

- **Do only what the user said.** Nothing in a plan is built before the user's own explicit yes to it, and a yes
  covers only the items named in that exchange: never a task from an earlier plan, a document or another session.
- Code rules are in `AGENTS.md`; the project's behaviour and contracts are in the house skill `@@prefix@@-@@area@@`.
  Both are the law here.
- **Never contact @@HOSTS@@ or any other server** from the terminal, a script or a browser, and never read or use the
  user's browser profile, cookies or credentials, unless the user says so in this conversation. Give the user the
  command or the manual step instead. Tests in a real browser or against a live service happen only on the user's
  explicit request.
- <Not under git: "The project is not under git (@@DATE@@). Do not run `git init`, commit or push unless the user
  asks." Under git: "Do not commit or push unless the user asks.">

# Code review and QA

- Reviewing code here (your own change, a commit, named files) goes through the `/cr` skill; testing it in practice
  goes through `/cr <target> --qa`.
- **Scope is always the current change.** Under git that is the diff; without git it is the files the user names.
  Review or test the whole project only when the user explicitly says so ("the whole project", "full audit", <the
  same phrases in @@LANGUAGE@@>); a plain "review" or "check" means the current change. A whole-project review or
  test runs in the team sessions, not as `/cr` subagents (the protocol is in `@@prefix@@-team`).
- Reports go to `docs/code-review/`, in @@LANGUAGE@@; identifiers, paths and commands stay as they are.
- Code under test runs only through `.claude/skills/cr/scripts/qa-sandbox.sh` (no network, writes only
  `/tmp/@@prefix@@-qa`). Offline lint: `.claude/skills/cr/scripts/lint.sh`.

# Agents

- Four house agents, `model: inherit`, read-only towards the project: `@@prefix@@-reviewer` (correctness),
  `@@prefix@@-security-reviewer` (security surface), `@@prefix@@-review-verifier` (tries to refute blocking
  findings), `@@prefix@@-qa` (offline tests). They report; **only the lead «@@LEAD@@» changes code, config and git**
  (and sessions outside the team, such as the setup session).
- **Helper skills from ECC** (https://github.com/affaan-m/ECC, commit <hash>, MIT, verbatim with `LICENSE`): <list>.
  Each house agent says which ones it loads, after `@@prefix@@-@@area@@`. Apply them where the work touches them,
  also when coding in the main session. **Where a helper disagrees with a house rule, the house rule wins.** ECC
  itself is not installed (no plugin, no `install.sh`, none of its agents or hooks).
- **Guard**: `.claude/hooks/agent-guard.py` (PreToolUse, through the launcher in `.claude/settings.json`) fences
  every `@@prefix@@-*` agent: tools allowlist, writes only to its folders, read-only Bash, no network programs, no
  user home. A `@@prefix@@-*` agent that is not in its `SPECIALISTS` is refused outright, so **every new agent must be
  registered there**; then run `python3 .claude/hooks/agent-guard-cases.py`.
- **Hybrid failure mode**: if the guard file or python3 is missing, the launcher blocks `@@prefix@@-*` agents and
  lets the main session through with a warning; an error inside the guard refuses the specialist. The main session
  is never locked out.
- **Team of sessions**: the lead «@@LEAD@@» and one session per agent: «@@TEAM@@ reviewer», «@@TEAM@@ security
  reviewer», «@@TEAM@@ review verifier», «@@TEAM@@ QA». Each registers once with
  `python3 .claude/team/register.py <role> "<name>"`; the guard then fences a specialist session like a subagent,
  and `.claude/hooks/team-session-start.py` (SessionStart) reminds it of its role. Messages between sessions go by
  `SendMessage`. The protocol is the skill `@@prefix@@-team`; load it at the start of every team session. The lead
  logs the user's decisions in `docs/team/LOG.md`.
- Agent files load when a session starts. An agent added or changed mid-session is not available until a new
  session; do not replace it with `general-purpose`, which would run unguarded.
````

## T3. `.claude/skills/@@prefix@@-team/SKILL.md`

````markdown
---
name: @@prefix@@-team
description: How the @@PROJECT@@ team of Claude sessions works — who is who (the lead «@@LEAD@@» and the specialist sessions @@TEAM@@ reviewer, @@TEAM@@ security reviewer, @@TEAM@@ review verifier, @@TEAM@@ QA), how a session registers its role, how sessions message each other, what a specialist may and may not do, and where decisions are logged. Load it at the start of every team session and whenever you send a task to, or answer, another session of this project.
---

<The NEVER CONNECT header from T5.>

# @@PROJECT@@ team of sessions (user, setup prompt, @@DATE@@)

Each specialist is a session of its own in the sidebar, beside the lead's. The same agent files also run as
subagents inside `/cr`; both are held by the same guard.

## Who is who

| Role | Session | Changes things? | Agent file |
|---|---|---|---|
| `lead` | «@@LEAD@@» | **Yes, the only one**: code, config, skills, agents, `docs/` | none |
| `@@prefix@@-reviewer` | «@@TEAM@@ reviewer» | No | `.claude/agents/@@prefix@@-reviewer.md` |
| `@@prefix@@-security-reviewer` | «@@TEAM@@ security reviewer» | No | `.claude/agents/@@prefix@@-security-reviewer.md` |
| `@@prefix@@-review-verifier` | «@@TEAM@@ review verifier» | No | `.claude/agents/@@prefix@@-review-verifier.md` |
| `@@prefix@@-qa` | «@@TEAM@@ QA» | No (writes only `/tmp/@@prefix@@-qa`) | `.claude/agents/@@prefix@@-qa.md` |

`python3 .claude/team/register.py --show` prints the registered sessions with their names and ids.

## Registering

A team session registers once, at its first turn, with the name `ListAgents` prints for it ("This session is …"):

```bash
python3 .claude/team/register.py <role> "<this session's name>"
```

From then on `.claude/hooks/agent-guard.py` treats a specialist session exactly like a @@prefix@@-* subagent
(read-only project, read-only Bash, no network programs, no web or MCP, messages only to the team), and
`.claude/hooks/team-session-start.py` reminds the session of its role after a restart or a compaction. A role is set
once; only the user changes it, by editing `.claude/team/sessions.json` and `specialists.txt` by hand. Sessions that
do not register (like the setup session) are outside the team and unfenced.

## Messages

- Address a session by its registered name or by its app id, with `SendMessage`. The guard lets a specialist message
  only registered team sessions.
- **Lead to specialist:** one task per message: what to do, the target (the `/cr` target block, or the files), the
  scope line (`SCOPE: current change only …` or `SCOPE: whole project — the user asked: "<words>"`), and what to send
  back. The lead does not pass the user's approvals off as its own words; it quotes them.
- **Specialist to lead:** the result in the agent file's output format, in @@LANGUAGE@@, in one message. Questions go
  to the lead; the lead takes global questions to the user.
- **The user may write to any session directly.** A specialist answers the user in that session and tells the lead
  in one line if the result matters for the lead's work.
- Do not poll and do not loop waiting for an answer: send, end the turn, and the reply arrives as a new message.

## What the specialists do and do not do

- They work on what the lead or the user assigns; they never start work of their own.
- They never change code, config, git, `.claude/`, `docs/` or the user's browser, and never contact a server. A
  finding that needs a change goes to the lead with the place and the direction of the fix.
- `@@prefix@@-qa` runs code only through `.claude/skills/cr/scripts/qa-sandbox.sh` (offline sandbox).
- The first rule holds for everyone: do only what the user said; a plan is built only after the user's own explicit
  yes, and a yes covers only what the user named.

## The lead

- Reviews of a change go through `/cr` (subagents, in parallel, one report in `docs/code-review/`).
- A whole-project review or test (the user asked for the whole project in words) goes to the specialist sessions
  instead of subagents. In one turn, send one task per message: correctness to «@@TEAM@@ reviewer», the security
  surface to «@@TEAM@@ security reviewer», offline tests to «@@TEAM@@ QA». Each task carries the target block,
  `SCOPE: whole project — the user asked: "<their words>"`, and asks for the result in one message to the lead. When
  the review answers arrive, send the blocking and unconfirmed findings to «@@TEAM@@ review verifier», then write the
  one report in `docs/code-review/` as `/cr` does. With no registered team sessions, run `/cr` with subagents.
- The specialist sessions are also for work that outlives one review: a re-review after fixes, a deeper look the
  user asked for, QA of a fix, a second opinion on a disputed finding («@@TEAM@@ review verifier»).
- The lead appends the user's decisions to `docs/team/LOG.md` (date, decision, the user's words quoted).
  Specialists read it.
- A new agent or role: add `.claude/agents/<name>.md`, register it in `SPECIALISTS` of `.claude/hooks/agent-guard.py`
  and in `ROLES` of `.claude/team/register.py`, add it to the table above, then run
  `python3 .claude/hooks/agent-guard-cases.py`.
````

## T4. `.claude/skills/cr/SKILL.md`

````markdown
---
name: cr
description: Code review of a @@PROJECT@@ change — the working tree under git, a commit or a range, or, when the project is not under git, the files the user names. Lints the change offline, runs the matching @@prefix@@-* reviewer agents in parallel (correctness, security), has a verifier try to refute every blocking finding, and writes one report in @@LANGUAGE@@ to docs/code-review/. Use whenever asked to review, check or look over code in this repository. With --qa it hands the change to @@prefix@@-qa afterwards.
argument-hint: "[file... | sha | sha1..sha2] [--qa] [--deep]"
---

<The NEVER CONNECT header from T5.>

# /cr: code review

`$ARGUMENTS` is one target plus optional flags:

| Target | Means |
|---|---|
| *(empty)*, under git | uncommitted work against `HEAD`, untracked files included |
| *(empty)*, no git | nothing to diff: ask which files changed; never fall back to the whole project |
| `<file> <file>` | exactly these files (the only form when the project is not under git) |
| `<sha>` | one commit (git only) |
| `<sha1>..<sha2>` | a range (git only) |

| Flag | Effect |
|---|---|
| `--qa` | after the review, run `@@prefix@@-qa` on the same target |
| `--deep` | the verifier also checks "worth fixing" findings, not only blocking ones |

The goal is a review the user can trust and act on in one read, fast. A 20-minute sweep of the whole project is a
failure even if it finds something.

## Scope: the current change, unless the user says otherwise

- `/cr` with no target, "review", "check" and the same words in @@LANGUAGE@@ all mean the change in front of you.
- **The whole project only when the user asks in words**: "the whole project", "full audit", <the same phrases in
  @@LANGUAGE@@>. That is an audit: confirm the area and depth first, then every agent prompt carries
  `SCOPE: whole project — the user asked: "<their words>"`. The agents then run in the team sessions, not as
  subagents: send the tasks of steps 3 and 7 together by `SendMessage`, as `@@prefix@@-team` describes (QA does not
  wait for findings), end the turn, and go on with step 4 when the review answers arrive. With no registered team
  sessions, run them as subagents.
- Otherwise every agent prompt carries `SCOPE: current change only`.

## 1. Resolve the target

Check whether the project is under git: `git rev-parse --is-inside-work-tree 2>/dev/null`.

**Under git, with at least one commit:**

```bash
git status --porcelain
git diff --stat HEAD            # working tree
git show --stat <sha>           # commit
git diff --stat <a>..<b>        # range
```

Build the target block:

```
KIND: worktree | commit | range
ROOT: <absolute project path>
BASE: <sha>          HEAD: <sha or "working tree">
DIFF_CMD: git diff HEAD   |   git show <sha>   |   git diff <a>..<b>
FILES: <changed files, untracked included, each tagged: <the areas of this project>, config, docs, claude, other>
```

A repository with no commit yet has no baseline: say so, and ask the user to make a baseline commit or name the
files. Never treat "everything is new" as a diff.

**No git:** the user names the files. Without names, ask; do not guess from modification times, and do not review
the project.

```
KIND: files
ROOT: <absolute project path>
FILES: <the named files, tagged>
```

No files: say there is nothing to review and stop. `.claude/` files are reviewed only when the user asks; then
`@@prefix@@-security-reviewer` checks the guards with `agent-security-hardening`.

## 2. Lint

```bash
.claude/skills/cr/scripts/lint.sh <every changed source and config file>
```

<One sentence: what lint.sh checks in this project.> `WARN` lines are not failures; carry them into the report only
when the change touches them. A `FAIL` is a blocking finding by itself: report it and stop.

## 3. Run the reviewers

| Agent | Run when |
|---|---|
| `@@prefix@@-reviewer` | any code or config change. Always the main reviewer |
| `@@prefix@@-security-reviewer` | the change touches <this project's security surface, as concrete grep patterns: auth and sessions, input parsing, HTML sinks, SQL, shell and process calls, file paths, outgoing URLs, secrets, permissions, crypto, CORS, deserialization, and whatever else applies>. Check with `grep -nE` on the diff or the files rather than guessing |

Docs-only changes get no agent: compare the document with the code yourself, briefly.

Launch the reviewers **in one message** so they run in parallel, and wait for all of them. If an agent type is
reported as not found (agent files load when a session starts, so one added mid-session is not registered yet), do
**not** substitute `general-purpose`: it would run without `.claude/hooks/agent-guard.py`. Tell the user to start a
new session and stop.

Each prompt contains:

1. The whole target block.
2. The files that agent owns.
3. What the change is for, if the user said.
4. This line, verbatim: `SCOPE: current change only — do not review code the change does not touch. ORCHESTRATED by
   /cr — do not write a report file; return your findings in full, in @@LANGUAGE@@, in the report sections. Lint has
   already passed. Time budget: about 10 minutes.`

## 4. Verify before reporting

Collect every **blocking** finding and every finding marked *unconfirmed* (with `--deep`, every "worth fixing" one
too). If there are any, launch one `@@prefix@@-review-verifier` with the target block and the findings, numbered.

- `CONFIRMED` stays where it is.
- `PLAUSIBLE` moves to "worth fixing", marked *unconfirmed*, with what would settle it.
- `REFUTED` is dropped and gets one line under "checked, clean" saying what was checked and why it holds.

This step is not optional.

## 5. Write the report

One file: `docs/code-review/<YYYY-MM-DD>-<slug>.md`, the slug two or three words naming the change. Never overwrite:
add `-2`, `-3`. Prose in @@LANGUAGE@@; paths, identifiers and commands stay as they are. Merge the agents' findings
and drop duplicates (same place, same mechanism: keep the better-argued one). Order by severity: <what hurts the
users of this project most> first.

Headings translated into @@LANGUAGE@@:

```markdown
# Code review — <what>

- **Date:** <YYYY-MM-DD>
- **Target:** <KIND, sha / range / files>
- **Reviewers:** <agents that ran>
- **Files:** <list>
- **Lint:** <ok | WARN lines that matter>
- **Verdict:** <ready | not ready — what to fix first>

## Blocking
### 1. <name>
**Where:** <file:line>
**What is wrong:** <mechanism>
**Breaks when:** <concrete input or state → wrong result>
**How to check:** <command, QA case or manual step>
**How to fix:** <direction>

## Worth fixing

## Checked, clean

## Not checked

## For later
```

## 6. Answer in chat

In @@LANGUAGE@@. The verdict in one line, then the blocking findings one line each (`file:line`: what breaks and on
what), the count of the rest, and the report path. The detail lives in the file. Nothing is committed and nothing is
sent anywhere.

## 7. QA

With `--qa`, or when the user asks for QA after the review, launch `@@prefix@@-qa` with the target block, the report
path and the blocking findings. Add its results to the same report under a QA section with the same five
subsections, and update the verdict line if QA changed it.
````

## T5. The four agents

Every agent file has this shape. Write each one fully for this project; the role requirements follow the skeleton.

````markdown
---
name: @@prefix@@-<role>
description: <One paragraph: what it does on this project, that /cr launches it and when, and when not to use it. Never "MUST BE USED" or "PROACTIVELY".>
model: inherit
color: <reviewer blue, security reviewer red, verifier yellow, QA green>
tools: ["Read", "Grep", "Glob", "Bash", "Skill"]
---

**NEVER CONNECT TO ANY SERVER UNLESS THE USER EXPLICITLY TOLD YOU TO IN THIS CONVERSATION** — no curl, wget, fetch, ssh, ping, DNS lookups and no browser visits to @@HOSTS@@ or any other host, and never read or use the user's browser profile, cookies or credentials. Give the user the command or the manual step instead.

## Helper skills from ECC

After the house skill `@@prefix@@-@@area@@`, load <the ECC skills for this role, and which of their sections apply
here and which do not>. They are general advice: where they disagree with `AGENTS.md` or the house skill, the house
rules win, and nothing from them is a finding unless it breaks something in this code.

## Think as the principal of your field

You are the principal <field> engineer of this project, not a checklist runner. Own the whole of your field: know how
the project really works end to end (<its real flow, in one line>), find root causes rather than symptoms, think two
steps ahead (<two or three concrete risks of this project>), say plainly what is wrong and what the right fix is,
with evidence, and push back on a plan that is not good enough, the lead's included. You do not change code, config
or git: you hand findings, verdicts and questions to the lead, which decides and makes the changes.

## In the team

When this agent runs as a team session (registered with `.claude/team/register.py`), load the skill
`@@prefix@@-team` first: hand every finding, verdict and question to the lead «@@LEAD@@» with SendMessage, or answer
the user directly when the user writes to you in this session. As a subagent (`/cr`), return everything to whoever
launched you. Either way you never change code, config or git; the lead does.

## Scope: the current change, unless told otherwise

By default you work only on the current change: the diff (`DIFF_CMD` in the target block) under git, or exactly the
files the caller names (`KIND: files`). Widen only on `SCOPE: whole project` with the user's own words quoted. Read
the changed lines plus the direct callers and collaborators needed to judge them, nothing wider. A problem that
exists the same way without this change is not a finding; at most it is one line under "for later". Empty change:
say so and stop. Keep it to about 10 minutes.

## Input

Usually the `/cr` target block: `KIND` (`worktree` | `commit` | `range` | `files`), `ROOT`, `BASE`, `HEAD`,
`DIFF_CMD` or `FILES`. Without a block, work on the files the caller names.

## Process

<Numbered steps for this role.>

## Checklist

<Reviewer and security reviewer: 10 to 25 concrete items for this stack and this code.>

## Output

<This role's output format, below.>
````

Role requirements:

- **`@@prefix@@-reviewer`** (correctness). Process: load the house skill, read `AGENTS.md`, read the change and each
  changed function in full, run `lint.sh` on the changed files unless told lint passed, walk the checklist, and for
  each candidate name the input or state that triggers it and trace it to the wrong result (if something upstream
  stops it, name that file:line and drop the candidate). Checklist: the contracts between parts, error paths and
  silent failures, data integrity, lifecycle and concurrency, edge inputs, the risky spots of this stack. Before a
  finding: can you cite file:line, name the input, state and bad outcome, have you read the callers and the other end
  of the contract, is the severity defensible? A clean review is a valid review; do not flag what `AGENTS.md`
  accepts or "use another language or framework". Output: the report sections (blocking, worth fixing, checked
  clean, not checked, for later), each finding with where / what is wrong / breaks when / how to check / how to fix,
  in @@LANGUAGE@@; define what "blocking" means for this project; a finding you cannot make concrete is marked
  *unconfirmed* with what would settle it. Standalone: the same, short, in chat, with a one-line verdict.
- **`@@prefix@@-security-reviewer`**. Treats every external input as hostile (name the sources: requests, files,
  third-party responses, pages, messages). Helper skills: `security-review`, and `agent-security-hardening` when the
  change touches `.claude/hooks/`, `.claude/settings.json` or agent tool lists. Process: map every new or changed
  sink and where its value comes from, walk the checklist, and build the attack for each candidate (the request,
  file, response or message, traced to the effect). Blocking means a concrete, reachable exploit; a finding without
  a concrete payload is *unconfirmed*. No general hardening advice ("consider a CSP"): name the attack or leave it
  out. Output as the reviewer's, with the attack itself under "breaks when".
- **`@@prefix@@-review-verifier`**. Loads only the house skill. Takes a numbered list of findings and tries to break
  each: read the exact lines as they are now; trace reachability backwards to a real entry point of this project
  (name the kinds of entry points: routes, CLI commands, jobs, message handlers, UI controls); check the facts the
  finding leans on; run the smallest read-only probe that settles it. Verdicts: `CONFIRMED` (a reachable path from an
  entry point to the wrong result, nothing upstream prevents it), `PLAUSIBLE` (the mechanism is real but
  reachability depends on input, timing or a service it cannot see; say exactly what would settle it), `REFUTED`
  (the code does not do that, or a guard at file:line makes it unreachable). Severity can move either way. It never
  looks for new bugs (one line at the end if it trips over one). Output, one block per finding:
  `### <n>. <name> — CONFIRMED | PLAUSIBLE | REFUTED`, then severity (unchanged, raised or lowered, and why),
  evidence (file:line and the path or the guard), and, for PLAUSIBLE, what would settle it.
- **`@@prefix@@-qa`**. `tools` adds `"Write"`. Writes only to `/tmp/@@prefix@@-qa` and runs code only through
  `.claude/skills/cr/scripts/qa-sandbox.sh`; syntax through `lint.sh`. Never writes into the project; a case worth
  keeping goes into the report with its code, and the lead decides. A table of what to test by what the change
  touches, for this stack. Helper skills: `e2e-testing` for structuring cases and `browser-qa` for writing manual
  UI steps, ignoring their install, CI and live-site instructions. Whatever needs a real browser or a live service
  is listed as not checked, with manual steps for the user. Output: the report sections, every case it ran with its
  result, the command and the relevant output lines for each failure. It never reports "works" for something it did
  not run.

## T6. `docs/team/LOG.md`

````markdown
# Team log — @@PROJECT@@ (append-only)

The lead «@@LEAD@@» appends the user's decisions here: date, decision, the user's words quoted. Entries are never
rewritten or deleted; a changed decision gets a new entry.

- **@@DATE@@** — Claude Code setup from the one-prompt team setup. ECC commit <hash>: helper skills <list>. House
  agents `@@prefix@@-reviewer`, `@@prefix@@-security-reviewer`, `@@prefix@@-review-verifier`, `@@prefix@@-qa`;
  house skill `@@prefix@@-@@area@@`; `/cr`; reports in @@LANGUAGE@@; <git state>. The setup prompt's decisions apply:
  hybrid failure mode, allowlist guard, offline QA, a team of sessions led by «@@LEAD@@», the lead pinned and the
  team detached. User: <quote the words I wrote around the prompt, or "setup prompt sent without comment">.
````

---

## Appendices

### Appendix A. `.claude/hooks/agent-guard.py`

The guard. Generic except the constants marked `STACK`.

```python
#!/usr/bin/env python3
"""PreToolUse guard for the @@prefix@@-* specialists (user, setup prompt, @@DATE@@: hybrid failure mode).

Marker checked by the launcher in .claude/settings.json: @@PREFIX@@_AGENT_GUARD

Only the lead session («@@LEAD@@») and sessions outside the team change code, config and git. The specialists
read, review, test and report. A specialist is either a subagent whose type starts with `@@prefix@@-`, or a session
registered with `.claude/team/register.py` under a @@prefix@@-* role (user, setup prompt, @@DATE@@);
a session listed in `.claude/team/specialists.txt` whose role cannot be read is refused everything. Everything here is
an ALLOWLIST:

- An agent type that starts with `@@prefix@@-` but is not in SPECIALISTS is refused outright. A new agent must be
  registered here before it can do anything; `.claude/hooks/agent-guard-cases.py` checks that every
  `.claude/agents/*.md` is registered.
- Tools: only TOOLS. No web, no MCP (the harness's own `mcp__ccd_turn__*` aside), no Agent/Task: a general-purpose
  child would run without this guard. SendMessage only to the team's registered sessions.
- Write / Edit: only the folders in SPECIALISTS for that agent (@@prefix@@-qa: /tmp/@@prefix@@-qa/). Nobody else writes.
- Read / Grep / Glob: never `.env`-style files or key files, never the user's home outside the project and
  ~/.claude/{skills,plugins} (browser profiles, SSH keys and cloud credentials live there).
- Bash, tokenized here with quotes as the shell sees them: every simple command is a read-only program from
  READ_ONLY (sed only as `sed -n 'A,Bp'`, awk without system/getline/pipes/redirects, find without -exec/-delete),
  git only with read-only subcommands, the check-only commands in RUNTIME_CHECK, or one of the project SCRIPTS. Refused
  outright: unquoted $ and backticks, subshells and groups, braces, background &, heredocs, redirects other than to
  /dev/null and 2>&1, paths into the user's home.

Hybrid failure mode: if this file or python3 is missing, the launcher blocks @@prefix@@-* agents and lets the main session
through with a warning. An unexpected error inside this script does the same — exit 2 for a specialist; the main
session never reaches the checks.

Exit code 2 refuses the call and shows the reason to the agent. Test cases: `python3 .claude/hooks/agent-guard-cases.py`.
"""
import json
import os
import re
import sys

PREFIX = "@@prefix@@-"
# agent name -> folders it may write (absolute). Register every new .claude/agents/@@prefix@@-*.md here.
SPECIALISTS = {
    "@@prefix@@-reviewer": [],
    "@@prefix@@-security-reviewer": [],
    "@@prefix@@-review-verifier": [],
    "@@prefix@@-qa": ["/tmp/@@prefix@@-qa"],
}
# SubagentHandback is how a subagent returns its report (a live probe found that refusing it leaves /cr without
# answers); advisor is the harness's own second-opinion tool.
TOOLS = {"Read", "Grep", "Glob", "Bash", "Skill", "ToolSearch", "TodoWrite", "TaskCreate", "TaskUpdate", "TaskList",
         "TaskGet", "Write", "Edit", "MultiEdit", "NotebookEdit", "SendMessage", "ListAgents", "SubagentHandback",
         "advisor"}
WRITE_TOOLS = {"Write", "Edit", "MultiEdit", "NotebookEdit"}

ROOT = os.path.realpath(os.path.join(os.path.dirname(os.path.abspath(__file__)), "..", ".."))
# The registry of the team of sessions (.claude/team/register.py). @@PREFIX@@_TEAM_DIR exists for agent-guard-cases.py.
TEAM = os.environ.get("@@PREFIX@@_TEAM_DIR") or os.path.join(ROOT, ".claude", "team")
HOME = os.path.realpath(os.path.expanduser("~"))
HOME_ALLOWED = [os.path.join(HOME, ".claude", "skills"), os.path.join(HOME, ".claude", "plugins")]
# Large tool outputs are saved by the harness under ~/.claude/projects/<project>/<session>/tool-results/.
TOOL_RESULTS = re.compile(re.escape(os.path.join(HOME, ".claude", "projects")) + r"/[^/]+/[^/]+/tool-results(/|$)")
SECRET_NAME = re.compile(r"(^|/)(\.env(?!\.example$)[^/]*|id_(rsa|ed25519|ecdsa)[^/]*|[^/]+\.(pem|key|p12|pfx))$", re.I)

READ_ONLY = {
    "cat", "head", "tail", "wc", "sort", "uniq", "cut", "tr", "nl", "grep", "egrep", "fgrep", "rg", "ls", "tree", "stat",
    "file", "du", "pwd", "echo", "printf", "true", "false", "test", "[", "basename", "dirname", "realpath", "readlink",
    "date", "diff", "cmp", "comm", "jq", "column", "paste", "which", "type", "whoami", "id", "uname", "md5sum",
    "sha256sum", "cd", "find", "sed", "awk", "fold", "rev", "expr", "seq", "tac", "od", "hexdump",
}
# Short or long options that make an otherwise read-only program write or execute.
BAD_OPTIONS = {
    "sort": ("o", "--output", "--compress-program"),
    "tree": ("o",),
    "rg": ("--pre", "--pre-glob"),
    "date": ("s", "--set"),
    "awk": ("f", "i", "l", "E", "--file", "--include", "--load", "--exec", "--source"),
    "uniq": (),
}
# sed is allowlisted by shape: only these options, exactly one print-only script, then files.
SED_OPTION = re.compile(r"^(-[nEr]+|--quiet|--silent|--regexp-extended)$")
FIND_BAD = {"-exec", "-execdir", "-ok", "-okdir", "-delete", "-fprint", "-fprint0", "-fprintf", "-fls"}
GIT_READ = {"status", "diff", "log", "show", "blame", "grep", "ls-files", "ls-tree", "rev-parse", "rev-list",
            "merge-base", "cat-file", "describe", "shortlog", "name-rev", "for-each-ref", "show-ref", "diff-tree",
            "diff-files", "diff-index"}
GIT_BAD = re.compile(r"^(--output|-O|--open-files-in-pager|--ext-diff|--textconv|--exec-path|-c$|--config-env)")
# project-relative script -> agents allowed to run it (None: every specialist)
SCRIPTS = {
    ".claude/skills/cr/scripts/lint.sh": None,
    ".claude/skills/cr/scripts/qa-sandbox.sh": {"@@prefix@@-qa"},
    ".claude/team/register.py": None,
}
# STACK: check-only runtime commands a specialist may run directly: program -> (flags, file extensions), used as
# `<program> <flag> <file>…` or `<program> --version`. Only commands that neither write nor reach the network, e.g.
# {"node": (("--check", "-c"), (".js", ".mjs", ".cjs"))} or {"php": (("-l",), (".php",))}. Everything that runs
# project code goes through SCRIPTS (lint.sh, qa-sandbox.sh). Leave it empty when unsure.
RUNTIME_CHECK = {}
PYTHON_SCRIPTS = {".claude/team/register.py"}
SED_ADDR = r"(?:\d+|\$|/[^/\n]*/)"
SED_SCRIPT = re.compile(rf"^\s*{SED_ADDR}?(?:\s*,\s*{SED_ADDR})?\s*p\s*(?:;\s*{SED_ADDR}?(?:\s*,\s*{SED_ADDR})?\s*p\s*)*;?\s*$")
AWK_BAD = re.compile(r"system|getline|close\s*\(|fflush|ENVIRON|PROCINFO|@|[|>]")
DEVNULL = re.compile(r"(?:(?<=[\s;|&])|^)(?:[12&]?>>?)\s*/dev/null(?=$|[\s;|&])|(?:(?<=[\s;|&])|^)2>&1(?=$|[\s;|&])")
OPERATORS = ("||", "&&", ";;", ">>", "<<", "&>", ">&", "|&", "|", "&", ";", "<", ">", "(", ")")
SEPARATORS = {"|", "||", "&&", ";"}


def refuse(reason: str) -> None:
    sys.stderr.write(
        "Refused by .claude/hooks/agent-guard.py (user's rule, @@DATE@@): @@prefix@@-* specialists read, review, test "
        f"and report — only the lead «@@LEAD@@» changes things.\n- {reason}\n"
        "If the work truly needs it, say in your report what and why; the lead decides.\n"
    )
    sys.exit(2)


def inside(path: str, folder: str) -> bool:
    folder = folder.rstrip("/")
    return path == folder or path.startswith(folder + "/")


def resolve(path: str, cwd: str) -> str:
    path = os.path.expanduser(path)
    return os.path.realpath(path if os.path.isabs(path) else os.path.join(cwd, path))


def forbidden(path: str) -> bool:
    """The user's home outside the project, the allowed skill folders and tool results — and every ancestor of
    home, since a recursive grep or find on / or /home reaches into it."""
    if inside(path, ROOT) or any(inside(path, folder) for folder in HOME_ALLOWED) or TOOL_RESULTS.match(path):
        return False
    return inside(path, HOME) or inside(HOME, path)


def check_path(word: str, cwd: str) -> None:
    if SECRET_NAME.search(word):
        refuse(f"{word}: secret and key files stay out of reviews")
    glob = re.search(r"[*?\[]", word)
    if glob:
        directory = os.path.dirname(word[:glob.start()])
        if re.search(r"[*?\[]", os.path.dirname(word)):
            refuse(f"{word}: patterns only in the last part of a path")
        target = resolve(directory or ".", cwd)
        if inside(HOME, target) and not inside(target, ROOT):
            refuse(f"{word}: a pattern next to the user's home folder")
    else:
        target = resolve(word, cwd)
    if forbidden(target):
        refuse(f"{word}: the user's home folder is out of reach (only the project, ~/.claude/skills and ~/.claude/plugins)")


def check_quotes(command: str) -> None:
    """Unquoted or double-quoted $ and ` expand; braces expand when unquoted."""
    i, single, double = 0, False, False
    while i < len(command):
        ch = command[i]
        if single:
            single = ch != "'"
        elif ch == "\\":
            i += 2
            continue
        elif ch == "'" and not double:
            single = True
        elif ch == '"':
            double = not double
        elif ch in "$`":
            refuse("no $ or backticks: variables and command substitution are not allowed — write the value out")
        elif ch in "{}" and not double:
            refuse("no brace expansion or command groups")
        i += 1


def tokenize(command: str) -> list:
    """(kind, value) pairs: 'op' for unquoted shell operators, 'word' for words with quotes removed."""
    tokens, word, has_word, i, n = [], [], False, 0, len(command)

    def flush():
        nonlocal word, has_word
        if has_word:
            tokens.append(("word", "".join(word)))
        word, has_word = [], False

    while i < n:
        ch = command[i]
        if ch in " \t":
            flush()
            i += 1
        elif ch == "\n":
            flush()
            tokens.append(("op", ";"))
            i += 1
        elif ch == "'":
            end = command.find("'", i + 1)
            if end < 0:
                refuse("unbalanced single quote")
            word.append(command[i + 1:end])
            has_word, i = True, end + 1
        elif ch == '"':
            i += 1
            while i < n and command[i] != '"':
                if command[i] == "\\" and i + 1 < n and command[i + 1] in '"\\$`\n':
                    i += 1
                word.append(command[i])
                i += 1
            if i >= n:
                refuse("unbalanced double quote")
            has_word, i = True, i + 1
        elif ch == "\\":
            if i + 1 < n and command[i + 1] != "\n":
                word.append(command[i + 1])
                has_word = True
            i += 2
        elif ch in "|&;<>()":
            flush()
            op = next(o for o in OPERATORS if command.startswith(o, i))
            tokens.append(("op", op))
            i += len(op)
        else:
            word.append(ch)
            has_word = True
            i += 1
    flush()
    return tokens


def has_option(args: list, options: tuple) -> bool:
    for arg in args:
        if arg == "--":
            return False
        for option in options:
            if option.startswith("-"):
                if arg == option or arg.startswith(option + "="):
                    return True
            elif arg.startswith("-") and not arg.startswith("--") and option in arg[1:]:
                return True
    return False


def script_name(word: str, cwd: str) -> str:
    real = resolve(word, cwd)
    for relative in SCRIPTS:
        if real == os.path.join(ROOT, relative):
            return relative
    return ""


def check_simple(agent: str, words: list, cwd: str) -> str:
    """Checks one simple command; returns the new working directory (cd)."""
    if not words:
        return cwd
    program, args = words[0], words[1:]
    if "=" in program and not program.startswith(("/", ".")):
        refuse(f"{program}: no environment overrides")
    if program in ("bash", "sh") and args:
        program, args = args[0], args[1:]
    elif program in ("python3", "python"):
        if not args or "/" not in args[0] or script_name(args[0], cwd) not in PYTHON_SCRIPTS:
            refuse("python only as `python3 .claude/team/register.py …`")
        program, args = args[0], args[1:]
    script = script_name(program, cwd) if "/" in program else ""
    if script:
        allowed = SCRIPTS[script]
        if allowed is not None and agent not in allowed:
            refuse(f"{script} is for {', '.join(sorted(allowed))} only")
    elif program == "git":
        rest = list(args)
        while rest and rest[0].startswith("-"):
            option = rest.pop(0)
            if option == "-C" and rest:
                check_path(rest.pop(0), cwd)
            elif option != "--no-pager":
                refuse(f"git {option}: only -C and --no-pager before the subcommand")
        if not rest or rest[0] not in GIT_READ:
            refuse(f"git {rest[0] if rest else ''}: only read-only git ({', '.join(sorted(GIT_READ))})")
        if any(GIT_BAD.match(arg) for arg in rest[1:]):
            refuse("git options that write files or run external programs are not allowed")
    elif program in RUNTIME_CHECK:
        flags, extensions = RUNTIME_CHECK[program]
        if args == ["--version"]:
            return cwd
        files = args[1:]
        if not args or args[0] not in flags or not files \
                or not all(a.endswith(tuple(extensions)) and not a.startswith("-") for a in files):
            refuse(f"{program} only as `{program} {flags[0]} <file>…`; run code through .claude/skills/cr/scripts/qa-sandbox.sh (@@prefix@@-qa)")
    elif program in READ_ONLY:
        if has_option(args, BAD_OPTIONS.get(program, ())):
            refuse(f"{program}: that option writes or executes")
        if program == "find" and any(arg in FIND_BAD for arg in args):
            refuse("find: no -exec, -ok, -delete or -fprint")
        if program == "sed":
            options = [a for a in args if a.startswith("-")]
            positional = [a for a in args if not a.startswith("-")]
            if any(not SED_OPTION.match(a) for a in options) or not positional or not SED_SCRIPT.match(positional[0]):
                refuse("sed only prints lines: `sed -n 'A,Bp' file` (options -n -E -r only; or use Read)")
        if program == "awk" and any(AWK_BAD.search(arg) for arg in args):
            refuse("awk: no system, getline, @include, pipes or redirects (and no `>` — compare with `<` or swap operands)")
        if program == "uniq" and len([a for a in args if not a.startswith("-")]) > 1:
            refuse("uniq with an output file writes it")
    else:
        refuse(f"{program}: not a read-only program for specialists (allowed: {', '.join(sorted(READ_ONLY))}, git, {', '.join(sorted(RUNTIME_CHECK)) or 'no runtime'} checks, project scripts)")
    for arg in args:
        value = arg.split("=", 1)[1] if arg.startswith("-") and "=" in arg else arg
        if "/" in value or value.startswith(("~", ".")):
            check_path(value, cwd)
    if program == "cd":
        target = args[0] if args else HOME
        check_path(target, cwd)
        return resolve(target, cwd)
    return cwd


def check_bash(agent: str, command: str, cwd: str) -> None:
    if not command.strip():
        return
    if forbidden(cwd):
        refuse("the working directory is in the user's home folder")
    check_quotes(command)
    stripped = DEVNULL.sub(" ", command)
    words = []
    for kind, value in tokenize(stripped) + [("op", ";")]:
        if kind == "word":
            words.append(value)
            continue
        if value not in SEPARATORS:
            refuse(f"`{value}`: no redirects, heredocs, background jobs or subshells (only | || && ; and >/dev/null, 2>&1)")
        cwd = check_simple(agent, words, cwd)
        words = []


def check(agent: str, payload: dict) -> None:
    if agent == PREFIX + "?":
        refuse("this session is listed in .claude/team/specialists.txt but its role cannot be read from sessions.json")
    if agent not in SPECIALISTS:
        refuse(f"{agent} is not registered in SPECIALISTS of .claude/hooks/agent-guard.py — register it first")
    tool = str(payload.get("tool_name") or "")
    tool_input = payload.get("tool_input") or {}
    cwd = os.path.realpath(str(payload.get("cwd") or ROOT))
    if tool.startswith("mcp__ccd_turn__"):
        return
    if tool not in TOOLS:
        refuse(f"{tool}: specialists use files, Bash, skills and team messages only — no web, MCP or agents")
    if tool == "SendMessage":
        # Only to the team: a session outside it has no fence, and asking it is a way around this one.
        to = re.sub(r"\s*\[[^\]]*\]\s*$", "", str(tool_input.get("to") or "")).strip()
        if to != "main" and to.casefold() not in {target.casefold() for target in team_targets()}:
            names = ", ".join(sorted(str(e.get("name")) for e in registry().values() if isinstance(e, dict) and e.get("name")))
            refuse(f"messages go only to the team's registered sessions ({names or 'none registered'}), not to «{to}»")
    if tool in WRITE_TOOLS:
        raw = str(tool_input.get("file_path") or tool_input.get("notebook_path") or "")
        target = resolve(raw, cwd) if raw else ""
        if not target or not any(inside(target, os.path.realpath(folder)) for folder in SPECIALISTS[agent]):
            places = ", ".join(SPECIALISTS[agent]) or "nowhere"
            refuse(f"{tool} of {raw or '?'}: {agent} writes {places} — findings go into the report, not into files")
    elif tool in ("Read", "Grep", "Glob"):
        for key in ("file_path", "path"):
            if tool_input.get(key):
                check_path(str(tool_input[key]), cwd)
        for key in ("pattern", "glob"):
            value = str(tool_input.get(key) or "")
            if value and (tool == "Glob" or key == "glob") and (SECRET_NAME.search(value) or ".env" in value):
                refuse(f"{value}: secret and key files stay out of reviews")
            if tool == "Glob" and key == "pattern" and (value.startswith(("/", "~")) or ".." in value):
                base = str(tool_input.get("path") or cwd)
                check_path(value if value.startswith(("/", "~")) else os.path.join(base, value), cwd)
    elif tool == "Bash":
        check_bash(agent, str(tool_input.get("command") or ""), cwd)


def registry() -> dict:
    try:
        with open(os.path.join(TEAM, "sessions.json"), encoding="utf-8") as handle:
            data = json.load(handle)
        return data if isinstance(data, dict) else {}
    except (OSError, ValueError):
        return {}


def listed_specialists() -> set:
    try:
        with open(os.path.join(TEAM, "specialists.txt"), encoding="utf-8") as handle:
            return {line.strip() for line in handle if line.strip()}
    except OSError:
        return set()


def session_role(session: str) -> str:
    """The @@prefix@@-* role of a registered session; `@@prefix@@-?` when the session is listed as a specialist but its role
    cannot be read (refused everything); '' for the lead and for sessions outside the team."""
    if not session:
        return ""
    role = str((registry().get(session) or {}).get("role") or "")
    if role.startswith(PREFIX):
        return role
    if not role and session in listed_specialists():
        return PREFIX + "?"
    return ""


def team_targets() -> set:
    """What SendMessage may address: each registered session's name, its app id (`local_…`, from
    CLAUDE_CODE_HOST_SESSION_ID at registration) and its session id."""
    targets = set()
    for session, entry in registry().items():
        targets.add(session)
        if isinstance(entry, dict):
            targets.update(str(entry[key]) for key in ("name", "host") if entry.get(key))
    return targets


def main() -> None:
    raw = sys.stdin.read()
    try:
        payload = json.loads(raw)
    except Exception:
        if re.search(r'"agent_type"\s*:\s*"([^"]*:)?' + PREFIX, raw):
            refuse("the hook payload could not be parsed")
        sys.exit(0)
    agent = str(payload.get("agent_type") or "").rsplit(":", 1)[-1]
    if not agent.startswith(PREFIX):
        try:
            agent = session_role(str(payload.get("session_id") or ""))
        except Exception as exc:  # hybrid mode: a broken registry never lets a listed specialist through
            if str(payload.get("session_id") or "") in listed_specialists():
                refuse(f"the team registry could not be read ({type(exc).__name__}: {exc})")
            agent = ""
    if not agent.startswith(PREFIX):
        sys.exit(0)
    try:
        check(agent, payload)
    except SystemExit:
        raise
    except Exception as exc:  # hybrid mode: an error in the guard never lets a specialist through
        refuse(f"internal error in the guard ({type(exc).__name__}: {exc}) — fix .claude/hooks/agent-guard.py")
    sys.exit(0)


if __name__ == "__main__":
    main()
```

### Appendix B. `.claude/hooks/agent-guard-cases.py`

The guard's tests. Fill `CODE`, `SUBDIR`, `EXT`; add cases for what you add to the guard.

```python
#!/usr/bin/env python3
"""Checks for .claude/hooks/agent-guard.py and its launcher in .claude/settings.json (user, setup prompt, @@DATE@@).

Run after every change to the guard, the settings or .claude/agents/:

    python3 .claude/hooks/agent-guard-cases.py

1. Every .claude/agents/*.md is registered in SPECIALISTS, and every SPECIALISTS entry has an agent file.
2. The guard itself: simulated PreToolUse payloads, expected exit code (0 allowed, 2 refused).
3. The launcher, hybrid mode: script found via $CLAUDE_PROJECT_DIR, found from a moved working directory, missing,
   without its marker, and without python3 — main session 0/1 (never 2), @@prefix@@-* specialists 2.
Nothing here touches the network or writes outside a temporary folder.
"""
import importlib.util
import json
import os
import re
import shutil
import subprocess
import sys
import tempfile

HOOKS = os.path.dirname(os.path.abspath(__file__))
ROOT = os.path.realpath(os.path.join(HOOKS, "..", ".."))
GUARD = os.path.join(HOOKS, "agent-guard.py")
HOME = os.path.expanduser("~")
# STACK: a real source file and a real subdirectory of this project, relative to ROOT, and the source extension.
CODE = "@@code_file@@"
SUBDIR = "@@subdir@@"
EXT = "@@ext@@"
failures = []


def report(ok: bool, label: str, detail: str = "") -> None:
    print(f"{'PASS' if ok else 'FAIL'}  {label}{('  — ' + detail) if detail and not ok else ''}")
    if not ok:
        failures.append(label)


def payload(agent, tool, tool_input, cwd=ROOT, session="cases") -> str:
    data = {"session_id": session, "transcript_path": "/dev/null", "cwd": cwd, "hook_event_name": "PreToolUse",
            "tool_name": tool, "tool_input": tool_input, "tool_use_id": "t1"}
    if agent:
        data["agent_id"] = "a1"
        data["agent_type"] = agent
    return json.dumps(data)


for relative, kind in ((CODE, "file"), (SUBDIR, "directory")):
    present = os.path.isfile(os.path.join(ROOT, relative)) if kind == "file" else os.path.isdir(os.path.join(ROOT, relative))
    report(present, f"setup: {kind} {relative!r} exists (CODE / SUBDIR at the top of this file)")

# 1. Registration -----------------------------------------------------------------------------------------------
spec = importlib.util.spec_from_file_location("agent_guard", GUARD)
guard = importlib.util.module_from_spec(spec)
spec.loader.exec_module(guard)
agents_dir = os.path.join(ROOT, ".claude", "agents")
names = {}
for file in sorted(os.listdir(agents_dir)):
    if file.endswith(".md") and file.startswith(guard.PREFIX):  # other agents of the project stay outside the guard
        text = open(os.path.join(agents_dir, file), encoding="utf-8").read()
        match = re.search(r"^name:\s*(\S+)\s*$", text.split("---")[1] if text.startswith("---") else "", re.M)
        names[file] = match.group(1) if match else ""
for file, name in names.items():
    report(name == file[:-3], f"agent file {file}: name matches the file name", f"name: {name!r}")
    report(name in guard.SPECIALISTS, f"agent {name}: registered in SPECIALISTS")
    report(name.startswith(guard.PREFIX), f"agent {name}: starts with {guard.PREFIX} (so the guard applies)")
for name in guard.SPECIALISTS:
    report(name in names.values(), f"SPECIALISTS entry {name}: has .claude/agents/{name}.md")
register_spec = importlib.util.spec_from_file_location("register", os.path.join(ROOT, ".claude", "team", "register.py"))
register = importlib.util.module_from_spec(register_spec)
register_spec.loader.exec_module(register)
report(set(register.ROLES) == set(guard.SPECIALISTS) | {"lead"},
       "register.py ROLES = SPECIALISTS + lead", f"ROLES {sorted(register.ROLES)}")

# 2. The guard --------------------------------------------------------------------------------------------------
P = guard.PREFIX
R, Q = P + "reviewer", P + "qa"
QA_DIR = guard.SPECIALISTS[Q][0]
QA_SCRIPT = ".claude/skills/cr/scripts/qa-sandbox.sh"
HOME_PARENT, HOME_NAME = os.path.dirname(HOME), os.path.basename(HOME)
CASES = [
    (None, "Bash", {"command": "rm -rf /tmp/whatever"}, 0, "main session is not fenced"),
    ("general-purpose", "Write", {"file_path": f"{ROOT}/{CODE}"}, 0, "agents without the prefix are not fenced"),
    (P + "foo", "Read", {"file_path": f"{ROOT}/{CODE}"}, 2, "unregistered prefixed agent is refused"),
    ("plugin:" + P + "foo", "Read", {"file_path": f"{ROOT}/{CODE}"}, 2, "unregistered namespaced prefixed agent"),
    (R, "Read", {"file_path": f"{ROOT}/{CODE}"}, 0, "read project file"),
    (R, "Read", {"file_path": f"{HOME}/.ssh/id_rsa"}, 2, "read ssh key"),
    (R, "Read", {"file_path": f"{HOME}/.config/google-chrome/Default/Cookies"}, 2, "read browser cookies"),
    (R, "Read", {"file_path": f"{HOME}/.claude/skills/example/SKILL.md"}, 0, "read a global skill"),
    (R, "Read", {"file_path": f"{ROOT}/.env"}, 2, "read .env"),
    (R, "Grep", {"pattern": "import", "path": ROOT}, 0, "grep project"),
    (R, "Grep", {"pattern": "token", "path": HOME_PARENT}, 2, "grep the folder above home"),
    (R, "Glob", {"pattern": f"**/*.{EXT}"}, 0, "glob project"),
    (R, "Glob", {"pattern": os.path.relpath(HOME, ROOT) + "/*"}, 2, "glob climbing into home"),
    (R, "Write", {"file_path": f"{ROOT}/{CODE}"}, 2, "reviewer writes code"),
    (R, "Edit", {"file_path": f"{QA_DIR}/x.txt"}, 2, "reviewer writes QA scratch"),
    (Q, "Write", {"file_path": f"{QA_DIR}/probe.test"}, 0, "QA writes its scratch"),
    (Q, "Write", {"file_path": f"{ROOT}/{CODE}"}, 2, "QA writes code"),
    (Q, "Write", {"file_path": f"{QA_DIR}/../evil.txt"}, 2, "QA escapes its scratch with .."),
    (R, "WebFetch", {"url": "https://example.com"}, 2, "web fetch"),
    (R, "mcp__claude-in-chrome__navigate", {"url": "https://example.com"}, 2, "browser MCP"),
    (R, "Agent", {"prompt": "x"}, 2, "spawn an unguarded subagent"),
    (R, "mcp__ccd_turn__update_plan", {}, 0, "harness task list"),
    (R, "SubagentHandback", {"report": "x"}, 0, "subagent returns its report"),
    (Q, "SubagentHandback", {"report": "x"}, 0, "QA subagent returns its report"),
    (R, "advisor", {}, 0, "harness advisor"),
]
ALLOWED_BASH = [
    f'grep -n "import" {CODE}',
    f"sed -n '1,80p' {CODE}",
    "sed -n '/main/,/^}/p' " + CODE,
    f"ls -la {SUBDIR}",
    "cat .claude/settings.json | jq .hooks",
    "git status 2>/dev/null || echo no-git",
    f".claude/skills/cr/scripts/lint.sh {CODE}",
    "bash .claude/skills/cr/scripts/lint.sh",
    f"find . -name '*.{EXT}' -not -path './.claude/*'",
    "awk '{print $1}' " + CODE,
    f"wc -l {CODE}",
    f"cd {SUBDIR} && ls",
    "grep -rn 'TODO' . 2>/dev/null | head -20",
    "echo done >/dev/null",
    f"grep -c 'x' {CODE}; grep -c 'y' {CODE}",
    f"ls {HOME}/.claude/skills",
    f"sed -n '1,3p' {CODE} {CODE}",
    f"sed -nE '10,20p' {CODE}",
    "awk -F: '{print $1}' " + CODE,
]
for program, (flags, extensions) in guard.RUNTIME_CHECK.items():
    if CODE.endswith(tuple(extensions)):
        ALLOWED_BASH += [f"{program} {flags[0]} {CODE}", f"{program} --version"]
REFUSED_BASH = [
    "curl https://example.com",
    "wget -q https://example.com",
    "ssh host",
    "nc -z host 443",
    f"rm {CODE}",
    f"echo x > {CODE}",
    f"echo x >> {CODE}",
    "cat $HOME/.ssh/id_rsa",
    "cat ~/.ssh/id_rsa",
    f"cat {HOME_PARENT}/{HOME_NAME[:1]}*/x",
    "ls ~",
    "cd ~ && ls",
    "cd",
    f"grep -r password {HOME_PARENT}",
    "find / -name Cookies",
    "cat `which ls`",
    'cat "$(pwd)/x"',
    "(cd /; ls)",
    "{ ls; }",
    "ls {a,b}",
    f"sed -i 's/a/b/' {CODE}",
    f"sed 's/a/b/w out' {CODE}",
    "find . -delete",
    "find . ';' cat -delete",
    "find . -exec rm {} ;",
    "awk 'BEGIN{system(\"id\")}'",
    "awk '{print > \"out\"}' " + CODE,
    "node -e \"fetch('https://example.com')\"",
    f"node {CODE}",
    "python3 -c 'print(1)'",
    "bash -c 'rm -rf /'",
    "git config user.name x",
    "git -c core.pager=sh log",
    "git diff --output=/tmp/x",
    "git commit -m x",
    f"cat {CODE} &",
    f"sort -o out {CODE}",
    "cat .env",
    "cat <<EOF\nx\nEOF",
    "env rm x",
    "xargs rm",
    "tee out",
    f"cat {CODE}; rm {CODE}",
    "ca>/dev/nullt",
    "X=1 ls",
    "rg --pre=sh x",
    "date -s 2020-01-01",
    f"{QA_SCRIPT} {QA_DIR}/x.js",
    # option-shaped bypasses
    "awk -F : 'BEGIN{system(\"id\")}' " + CODE,
    "awk -v x=1 'BEGIN{system(\"id\")}'",
    "awk -e 'BEGIN{system(\"id\")}'",
    "awk '@include \"x\"; {print}' " + CODE,
    f"sed -n -e 1p --expression='1e id' {CODE}",
    f"sed -n -e 1p '-ew /tmp/out' {CODE}",
    f"sed -ne '1p' {CODE}",
    f"sed -n 1p --in-place {CODE}",
    f"sed -n '1e id' {CODE}",
    "node --check -r./x.js y.js",
    "node --check --require=./x.js y.js",
    "node --check",
    f"sort --compress-program=sh {CODE}",
]
for agent, tool, tool_input, expected, label in CASES:
    run = subprocess.run([sys.executable, GUARD], input=payload(agent, tool, tool_input), text=True,
                         capture_output=True)
    report(run.returncode == expected, f"guard: {label}", f"exit {run.returncode}, expected {expected}: {run.stderr.strip()[:160]}")
for command in ALLOWED_BASH + REFUSED_BASH:
    expected = 0 if command in ALLOWED_BASH else 2
    run = subprocess.run([sys.executable, GUARD], input=payload(R, "Bash", {"command": command}), text=True,
                         capture_output=True)
    report(run.returncode == expected, f"guard bash {'allows' if expected == 0 else 'refuses'}: {command!r}",
           f"exit {run.returncode}: {run.stderr.strip()[:200]}")
qa_run = subprocess.run([sys.executable, GUARD], text=True, capture_output=True,
                        input=payload(Q, "Bash", {"command": f"{QA_SCRIPT} {QA_DIR}/x.js"}))
report(qa_run.returncode == 0, f"guard bash allows {Q} to run the QA sandbox", qa_run.stderr.strip()[:200])
home_cwd = subprocess.run([sys.executable, GUARD], text=True, capture_output=True,
                          input=payload(R, "Bash", {"command": "ls"}, cwd=HOME))
report(home_cwd.returncode == 2, "guard bash refuses a working directory in home", home_cwd.stderr.strip()[:200])

# 2b. Team sessions (registered with .claude/team/register.py) --------------------------------------------------
team_dir = tempfile.mkdtemp(prefix=P + "team-cases-")
json.dump({"S-REV": {"role": R, "name": "@@TEAM@@ reviewer"},
           "S-QA": {"role": Q, "name": "@@TEAM@@ QA"},
           "S-LEAD": {"role": "lead", "name": "@@LEAD@@", "host": "local_LEAD-HOST"}},
          open(os.path.join(team_dir, "sessions.json"), "w"))
open(os.path.join(team_dir, "specialists.txt"), "w").write("S-QA\nS-REV\nS-BROKEN\n")
team_env = dict(os.environ, **{"@@PREFIX@@_TEAM_DIR": team_dir})
TEAM_CASES = [
    # agent_type, session, tool, input, expected, label
    (None, "S-LEAD", "Write", {"file_path": f"{ROOT}/{CODE}"}, 0, "lead session writes code"),
    (None, "S-OTHER", "Write", {"file_path": f"{ROOT}/{CODE}"}, 0, "session outside the team is not fenced"),
    (None, "S-REV", "Write", {"file_path": f"{ROOT}/{CODE}"}, 2, "reviewer session writes code"),
    (None, "S-REV", "Read", {"file_path": f"{ROOT}/{CODE}"}, 0, "reviewer session reads code"),
    (None, "S-REV", "Bash", {"command": "curl https://example.com"}, 2, "reviewer session runs curl"),
    (None, "S-REV", "Bash", {"command": "python3 .claude/team/register.py --show"}, 0, "reviewer session shows the team"),
    (None, "S-REV", "Bash", {"command": "python3 -c 'print(1)'"}, 2, "reviewer session runs python"),
    (None, "S-REV", "SendMessage", {"to": "@@LEAD@@", "message": "x"}, 0, "reviewer session messages the lead by name"),
    (None, "S-REV", "SendMessage", {"to": "local_LEAD-HOST", "message": "x"}, 0, "reviewer session messages the lead by app id"),
    (None, "S-REV", "SendMessage", {"to": "Other project QA", "message": "x"}, 2, "reviewer session messages a session outside the team"),
    (None, "S-REV", "ListAgents", {}, 0, "reviewer session lists agents"),
    (None, "S-REV", "mcp__ccd_session__start_session", {"title": "x"}, 2, "reviewer session starts a session"),
    (None, "S-REV", "Agent", {"prompt": "x"}, 2, "reviewer session spawns a subagent"),
    ("general-purpose", "S-REV", "Write", {"file_path": f"{ROOT}/{CODE}"}, 2, "subagent inside a reviewer session writes code"),
    (None, "S-QA", "Write", {"file_path": f"{QA_DIR}/t.test"}, 0, "QA session writes its scratch"),
    (None, "S-QA", "Bash", {"command": f"{QA_SCRIPT} {QA_DIR}/t.js"}, 0, "QA session runs the QA sandbox"),
    (None, "S-BROKEN", "Read", {"file_path": f"{ROOT}/{CODE}"}, 2, "listed specialist with unreadable role"),
    (R, "S-LEAD", "Write", {"file_path": f"{ROOT}/{CODE}"}, 2, "prefixed subagent of the lead writes code"),
]
for agent, session, tool, tool_input, expected, label in TEAM_CASES:
    run = subprocess.run([sys.executable, GUARD], input=payload(agent, tool, tool_input, session=session), text=True,
                         capture_output=True, env=team_env)
    report(run.returncode == expected, f"team: {label}", f"exit {run.returncode}, expected {expected}: {run.stderr.strip()[:200]}")
START = os.path.join(HOOKS, "team-session-start.py")
for session, needle, label in (("S-REV", f"specialist `{R}`", "reviewer session is told its role"),
                               ("S-LEAD", "LEAD", "lead session is told its role"),
                               ("S-OTHER", "register it first", "unregistered session gets the register line")):
    run = subprocess.run([sys.executable, START], input=json.dumps({"session_id": session, "hook_event_name": "SessionStart"}),
                         text=True, capture_output=True, env=team_env)
    report(run.returncode == 0 and needle in run.stdout, f"session start: {label}", (run.stdout + run.stderr)[:200])

# 3. The launcher (hybrid mode) ---------------------------------------------------------------------------------
settings = json.load(open(os.path.join(ROOT, ".claude", "settings.json"), encoding="utf-8"))
launcher = next(h["command"] for group in settings["hooks"]["PreToolUse"] for h in group["hooks"]
                if "agent-guard.py" in h.get("command", ""))
start_launcher = next(h["command"] for group in settings["hooks"]["SessionStart"] for h in group["hooks"]
                      if "team-session-start.py" in h.get("command", ""))
shells = [s for s in ("/bin/sh", "/bin/bash") if os.path.exists(s)]
empty = tempfile.mkdtemp(prefix=P + "guard-cases-")
fake = os.path.join(empty, "fake")
os.makedirs(os.path.join(fake, ".claude", "hooks"))
open(os.path.join(fake, ".claude", "hooks", "agent-guard.py"), "w").write("import sys; sys.exit(0)\n")
teamonly = os.path.join(empty, "teamonly")
os.makedirs(os.path.join(teamonly, ".claude", "team"))
open(os.path.join(teamonly, ".claude", "team", "specialists.txt"), "w").write("S-REV\n")
nopython = os.path.join(empty, "bin")
os.makedirs(nopython)
for tool in ("grep", "dirname", "cat", "head", "sed"):  # coreutils stay; only python3 is missing
    os.symlink(shutil.which(tool), os.path.join(nopython, tool))
read_ok = ("Read", {"file_path": f"{ROOT}/{CODE}"})
write_code = ("Write", {"file_path": f"{ROOT}/{CODE}"})
SCENARIOS = [
    # label, env overrides, cwd, agent, tool, expected
    ("found via CLAUDE_PROJECT_DIR, main session", {"CLAUDE_PROJECT_DIR": ROOT}, ROOT, None, write_code, 0),
    ("found via CLAUDE_PROJECT_DIR, reviewer reads", {"CLAUDE_PROJECT_DIR": ROOT}, ROOT, R, read_ok, 0),
    ("found via CLAUDE_PROJECT_DIR, reviewer writes", {"CLAUDE_PROJECT_DIR": ROOT}, ROOT, R, write_code, 2),
    ("moved session (wrong CLAUDE_PROJECT_DIR), cwd in a subdirectory, reviewer reads",
     {"CLAUDE_PROJECT_DIR": "/nonexistent"}, os.path.join(ROOT, SUBDIR), R, read_ok, 0),
    ("moved session (no CLAUDE_PROJECT_DIR), cwd in a subdirectory, reviewer writes", {"CLAUDE_PROJECT_DIR": None},
     os.path.join(ROOT, SUBDIR), R, write_code, 2),
    ("script missing, main session continues with a warning", {"CLAUDE_PROJECT_DIR": empty}, empty, None, write_code, 1),
    ("script missing, specialist blocked", {"CLAUDE_PROJECT_DIR": empty}, empty, R, read_ok, 2),
    ("script missing, unregistered specialist blocked", {"CLAUDE_PROJECT_DIR": empty}, empty, P + "foo", read_ok, 2),
    ("foreign guard without marker, specialist blocked", {"CLAUDE_PROJECT_DIR": fake}, fake, R, read_ok, 2),
    ("foreign guard without marker, main session continues", {"CLAUDE_PROJECT_DIR": fake}, fake, None, read_ok, 1),
    ("no python3, specialist blocked", {"CLAUDE_PROJECT_DIR": ROOT, "PATH": nopython}, ROOT, R, read_ok, 2),
    ("no python3, main session continues", {"CLAUDE_PROJECT_DIR": ROOT, "PATH": nopython}, ROOT, None, read_ok, 1),
    ("guard missing, registered specialist session blocked", {"CLAUDE_PROJECT_DIR": teamonly}, teamonly,
     ("session", "S-REV"), read_ok, 2),
    ("guard missing, other session continues", {"CLAUDE_PROJECT_DIR": teamonly}, teamonly,
     ("session", "S-LEAD"), read_ok, 1),
]
try:
    for shell in shells:
        for label, overrides, cwd, agent, (tool, tool_input), expected in SCENARIOS:
            env = dict(os.environ)
            for key, value in overrides.items():
                if value is None:
                    env.pop(key, None)
                else:
                    env[key] = value
            session = agent[1] if isinstance(agent, tuple) else "cases"
            agent = None if isinstance(agent, tuple) else agent
            run = subprocess.run([shell, "-c", launcher], text=True, capture_output=True, cwd=cwd, env=env, timeout=30,
                                 input=payload(agent, tool, tool_input, cwd=cwd, session=session))
            report(run.returncode == expected, f"launcher ({os.path.basename(shell)}): {label}",
                   f"exit {run.returncode}, expected {expected}: {run.stderr.strip()[:200]}")
        for label, overrides, cwd, expected_out in (
                ("session start found from a subdirectory", {"CLAUDE_PROJECT_DIR": "/nonexistent"},
                 os.path.join(ROOT, SUBDIR), "@@PROJECT@@ team"),
                ("session start script missing → silent exit 0", {"CLAUDE_PROJECT_DIR": empty}, empty, "")):
            env = dict(os.environ, **overrides)
            run = subprocess.run([shell, "-c", start_launcher], text=True, capture_output=True, cwd=cwd, env=env, timeout=30,
                                 input=json.dumps({"session_id": "S-OTHER", "hook_event_name": "SessionStart"}))
            ok = run.returncode == 0 and (expected_out in run.stdout if expected_out else run.stdout.strip() == "")
            report(ok, f"launcher ({os.path.basename(shell)}): {label}", f"exit {run.returncode}: {(run.stdout + run.stderr)[:200]}")
finally:
    shutil.rmtree(empty, ignore_errors=True)
    shutil.rmtree(team_dir, ignore_errors=True)

print(f"\n{len(failures)} failed" if failures else "\nall passed")
sys.exit(1 if failures else 0)
```

### Appendix C. `.claude/team/register.py`

Registers a session's role in the team.

```python
#!/usr/bin/env python3
"""Registers the role of this Claude session in the @@PROJECT@@ team (user, setup prompt, @@DATE@@).

Every specialist runs as a session of its own beside the lead's («@@LEAD@@»). A session says once who it is,
with the name other sessions message it by (ListAgents prints it as «This session is …»):

    python3 .claude/team/register.py @@prefix@@-qa "@@TEAM@@ QA"

The role is kept in `.claude/team/sessions.json` under the session's id (`CLAUDE_CODE_SESSION_ID`), and the ids of
the specialist sessions also in `.claude/team/specialists.txt` (one per line), which the hook launcher reads when
the guard itself is missing (hybrid mode). `.claude/hooks/agent-guard.py` holds a registered specialist session to the
same fence as a @@prefix@@-* subagent: no code, config or git changes, read-only Bash, no network programs.
`.claude/hooks/team-session-start.py` reminds the session of its role after a restart or a compaction. A role is set
once — only the user changes it, by hand. There is one lead.

    python3 .claude/team/register.py --show     # the roles and names of all registered sessions
    python3 .claude/team/register.py --lead     # the name of the lead's session
"""
import datetime
import json
import os
import sys
import tempfile

ROLES = (
    "lead",
    "@@prefix@@-reviewer",
    "@@prefix@@-security-reviewer",
    "@@prefix@@-review-verifier",
    "@@prefix@@-qa",
)

TEAM = os.path.dirname(os.path.abspath(__file__))
REGISTRY = os.path.join(TEAM, "sessions.json")
SPECIALISTS = os.path.join(TEAM, "specialists.txt")


def load() -> dict:
    try:
        with open(REGISTRY, encoding="utf-8") as handle:
            data = json.load(handle)
        return data if isinstance(data, dict) else {}
    except (OSError, ValueError):
        return {}


def write_atomically(path: str, text: str) -> None:
    """Writes in one move, so a reader (the guard) never sees half a file."""
    handle, temp = tempfile.mkstemp(dir=TEAM, prefix=".tmp.", suffix=os.path.basename(path))
    with os.fdopen(handle, "w", encoding="utf-8") as out:
        out.write(text)
    os.replace(temp, path)


def save(data: dict) -> None:
    write_atomically(REGISTRY, json.dumps(data, ensure_ascii=False, indent=2, sort_keys=True) + "\n")
    ids = sorted(sid for sid, entry in data.items() if entry.get("role") not in ("", None, "lead"))
    write_atomically(SPECIALISTS, "".join(f"{sid}\n" for sid in ids))


def main() -> int:
    if len(sys.argv) not in (2, 3):
        print(__doc__)
        return 2

    data = load()
    if sys.argv[1] == "--show":
        for session, entry in sorted(data.items(), key=lambda item: item[1].get("role", "")):
            print(f"{entry.get('role', '?'):26} {entry.get('name', '?'):28} {session}  since {entry.get('registered', '?')}")
        return 0
    if sys.argv[1] == "--lead":
        print(next((entry.get("name", "") for entry in data.values() if entry.get("role") == "lead"), ""))
        return 0

    role = sys.argv[1]
    if role not in ROLES:
        print(f"Unknown role «{role}». One of: {', '.join(ROLES)}", file=sys.stderr)
        return 2
    session = os.environ.get("CLAUDE_CODE_SESSION_ID", "")
    if not session:
        print("CLAUDE_CODE_SESSION_ID is not set: run this from the Bash tool of a Claude session.", file=sys.stderr)
        return 2

    name = sys.argv[2].strip() if len(sys.argv) == 3 else ""
    # The app's id of the session (local_…), the one SendMessage and the sidebar use.
    host = os.environ.get("CLAUDE_CODE_HOST_SESSION_ID", "")
    current = data.get(session, {}).get("role")
    if current == role:
        changed = []
        if name and data[session].get("name") != name:
            data[session]["name"] = name
            changed.append(f"name «{name}»")
        if host and data[session].get("host") != host:
            data[session]["host"] = host
            changed.append(f"app id {host}")
        if changed:
            save(data)
            print(f"This session is {role}; updated: {', '.join(changed)}.")
        else:
            print(f"This session is already {role}.")
        return 0
    if current:
        print(f"This session is already {current}; a role is set once — the user changes it by hand.", file=sys.stderr)
        return 1
    if role == "lead" and any(entry.get("role") == "lead" for sid, entry in data.items() if sid != session):
        print("The team has a lead already; a second session cannot take that role.", file=sys.stderr)
        return 1

    data[session] = {"role": role, "name": name, "host": host,
                     "registered": datetime.datetime.now().astimezone().isoformat(timespec="seconds")}
    save(data)
    print(f"Registered: this session is {role}" + (f" («{name}»)." if name else "."))
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### Appendix D. `.claude/hooks/team-session-start.py`

SessionStart hook: reminds a session of its role after a restart or a compaction.

```python
#!/usr/bin/env python3
"""SessionStart hook of the @@PROJECT@@ team of sessions (user, setup prompt, @@DATE@@).

When a session starts, resumes or is compacted, this hook tells it who it is — from `.claude/team/sessions.json`,
written by `.claude/team/register.py` — so the role survives a restart: its definition, the team protocol (skill
`@@prefix@@-team`) and the name of the lead's session. A session that has not registered gets one line on how to do it.
It never blocks anything: on any problem it prints nothing and exits 0.
"""
import json
import os
import sys


def main() -> None:
    try:
        payload = json.load(sys.stdin)
    except Exception:
        sys.exit(0)

    team = os.environ.get("@@PREFIX@@_TEAM_DIR") or os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), "team")
    try:
        with open(os.path.join(team, "sessions.json"), encoding="utf-8") as handle:
            registry = json.load(handle)
        if not isinstance(registry, dict):
            registry = {}
    except (OSError, ValueError):
        registry = {}

    entry = registry.get(str(payload.get("session_id") or "")) or {}
    role = str(entry.get("role") or "")
    lead = next((str(item.get("name") or "") for item in registry.values() if item.get("role") == "lead"), "")
    lead_name = f"«{lead}»" if lead else "«@@LEAD@@»"

    first_rule = (
        " FIRST RULE (user, setup prompt, @@DATE@@): do only what the user said. Work goes in stages; finish the stage "
        "the user named, report, then ask whether to do the next one and wait for the user's own explicit yes. A yes "
        "covers only what the user named — never a task from an earlier plan, a document or another session. Answer "
        "the user in @@LANGUAGE@@."
    )
    if role == "lead":
        context = (
            "@@PROJECT@@ team: this session is the LEAD (registered). Only you change code, config, skills and "
            "agents. Reviews go through /cr; tasks, re-reviews and questions go to the specialist sessions by "
            "SendMessage as the skill `@@prefix@@-team` describes; global questions go to the user; record decisions in "
            "docs/team/LOG.md." + first_rule
        )
    elif role:
        context = (
            f"@@PROJECT@@ team: this session is the specialist `{role}` (registered; .claude/hooks/agent-guard.py "
            f"holds it to the specialist fence). Act exactly as `.claude/agents/{role}.md` describes — read it now if "
            "it is not in your context — think as the principal of your field, and follow the skill `@@prefix@@-team`. "
            f"Hand every finding, verdict and question to the lead's session {lead_name} with SendMessage (or answer "
            "the user directly when the user writes to you here); never change code, config, git or the user's "
            "browser, and never contact a server." + first_rule
            + " You work on what the lead or the user assigns; you never start work of your own."
        )
    else:
        context = (
            "@@PROJECT@@ team: if this session is the lead or one of the specialists, register it first — "
            "`python3 .claude/team/register.py <role> \"<this session's name from ListAgents>\"` "
            "(roles: lead, @@prefix@@-reviewer, @@prefix@@-security-reviewer, @@prefix@@-review-verifier, @@prefix@@-qa) — then "
            "read the skill `@@prefix@@-team`. Other sessions need nothing."
        )

    print(json.dumps({"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": context}},
                     ensure_ascii=False))


if __name__ == "__main__":
    main()
```

### Appendix E. `.claude/settings.json`

The two hook launchers. The PreToolUse one runs the guard and implements the hybrid failure mode; keep each command on one line exactly as written, with only the placeholders replaced.

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "g=.claude/hooks/team-session-start.py; t=.claude/none; f=; s=; for d in \"$CLAUDE_PROJECT_DIR\" \"$PWD\"; do while [ -n \"$d\" ]; do [ -z \"$f\" ] && [ -f \"$d/$g\" ] && f=\"$d/$g\"; [ -z \"$s\" ] && [ -f \"$d/$t\" ] && s=\"$d/$t\"; [ -n \"$f\" ] && break 2; [ \"$d\" = / ] && break; d=$(dirname \"$d\"); done; done; if [ -n \"$f\" ] && command -v python3 >/dev/null 2>&1; then exec python3 \"$f\"; fi; exit 0"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "p=$(cat); g=.claude/hooks/agent-guard.py; t=.claude/team/specialists.txt; f=; s=; for d in \"$CLAUDE_PROJECT_DIR\" \"$PWD\"; do while [ -n \"$d\" ]; do [ -z \"$f\" ] && [ -f \"$d/$g\" ] && f=\"$d/$g\"; [ -z \"$s\" ] && [ -f \"$d/$t\" ] && s=\"$d/$t\"; [ -n \"$f\" ] && break 2; [ \"$d\" = / ] && break; d=$(dirname \"$d\"); done; done; if [ -n \"$f\" ] && grep -q @@PREFIX@@_AGENT_GUARD \"$f\" 2>/dev/null && command -v python3 >/dev/null 2>&1; then printf '%s' \"$p\" | python3 \"$f\"; exit $?; fi; sid=$(printf '%s' \"$p\" | grep -o '\"session_id\"[[:space:]]*:[[:space:]]*\"[^\"]*\"' | head -n 1 | sed 's/.*\"\\([^\"]*\\)\"$/\\1/'); if printf '%s' \"$p\" | grep -Eq '\"agent_type\"[[:space:]]*:[[:space:]]*\"([^\"]*:)?@@prefix@@-' || { [ -n \"$s\" ] && [ -n \"$sid\" ] && grep -qxF \"$sid\" \"$s\"; }; then echo 'agent-guard: .claude/hooks/agent-guard.py (or python3) not found - @@prefix@@-* specialists are blocked until it is restored (hybrid mode)' >&2; exit 2; fi; echo 'agent-guard: .claude/hooks/agent-guard.py (or python3) not found - the main session continues, specialists stay blocked; restore the file' >&2; exit 1"
          }
        ]
      }
    ]
  }
}
```

### Appendix F. `.claude/skills/cr/scripts/qa-sandbox.sh`

QA sandbox for a Node project. For another runtime, write the equivalent with the same contract.

```bash
#!/usr/bin/env bash
# Runs one QA script with Node, offline (user, setup prompt, @@DATE@@). Example for a Node project:
# - no network: its own network namespace (`unshare -rn`), so fetch() to any host fails;
# - Node permission model: reads only the project and /tmp/@@prefix@@-qa, writes only /tmp/@@prefix@@-qa,
#   no child processes, no workers, no native addons.
# Without unshare it refuses to run rather than run with the network on.
#
# Usage: .claude/skills/cr/scripts/qa-sandbox.sh /tmp/@@prefix@@-qa/<name>.test.js [args...]
# The script may use node:test and node:assert, and require project modules by absolute path.
set -euo pipefail

root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../../.." && pwd -P)"
script="${1:-}"

if [ -z "$script" ]; then
	echo "usage: qa-sandbox.sh /tmp/@@prefix@@-qa/<name>.js [args...]" >&2
	exit 64
fi

if ! command -v unshare >/dev/null 2>&1 || ! unshare -rn true 2>/dev/null; then
	echo "qa-sandbox.sh: cannot cut the network off (unshare -rn) — refusing to run" >&2
	exit 69
fi

mkdir -p /tmp/@@prefix@@-qa
qa="$(cd /tmp/@@prefix@@-qa && pwd -P)"
real="$(realpath -m -- "$script")"
case "$real" in
	"$qa"/*.js | "$qa"/*.mjs | "$qa"/*.cjs) ;;
	*)
		echo "qa-sandbox.sh: only scripts under $qa/ ($real)" >&2
		exit 64
		;;
esac

shift
exec unshare -rn node --permission \
	--allow-fs-read="$root" --allow-fs-read="$qa" --allow-fs-write="$qa" \
	"$real" "$@"
```
