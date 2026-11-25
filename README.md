# claude-team-setup

One prompt that gives any project a guarded team of Claude Code agents.

Start a new Claude Code session in your project, hand it [`SETUP-PROMPT.md`](SETUP-PROMPT.md), and it builds:

- house rules read from your code (`CLAUDE.md`, `.claude/CLAUDE.md`, `AGENTS.md`) and a house skill with the
  project's facts;
- a team designed for the project: four core specialists (reviewer, security reviewer, review verifier, offline QA)
  plus project specialists chosen from the code — domain reviewers such as a SQL or 3D-scene reviewer, and advisers
  such as a product manager, designer or domain analyst who write only their own `docs/<area>/`;
- for a project that does not exist yet (no folder, an empty one, or only a template), a short interview first —
  what the project is, for whom, its stack, where mistakes hurt, which roles it needs — and a team built from the
  answers;
- `/cr`, a code-review skill that lints the change, runs the reviewers in parallel, has the verifier try to refute
  every blocking finding, and writes one report;
- a PreToolUse guard that fences the specialists (tool allowlist, read-only Bash, no network programs, no access to
  your home folder), with a hybrid failure mode and about 180 tests;
- helper skills copied verbatim from [ECC](https://github.com/affaan-m/ECC);
- one Claude session per agent in the Claude desktop app, registered and fenced by the same guard; a whole-project
  review or test runs in these sessions instead of `/cr` subagents.

## Use

1. Open your project folder in Claude Code and start a new session, or put the folder's path in the PROJECT field.
2. Copy `SETUP-PROMPT.md` into the project and fill in the fields at its top, or leave them blank.
3. Send `Read SETUP-PROMPT.md and follow it.` You can also paste the whole file as your message.

The prompt answers the design questions in advance, so the session builds everything in one go and stops only for
the interview (when there is no project yet), network access, git commits or anything the prompt does not list.

## Requirements

- Claude Code, bash, python3 3.8 or newer.
- Linux for the offline QA sandbox (`unshare -rn`). On other systems QA is limited to static checks.
- The Claude desktop app (Code tab) to start the team sessions automatically. Elsewhere the setup prints the session
  briefs for you to paste.
- Network: one `git clone` of ECC during setup, nothing else.

## Status

Appendices A to F (the guard, its tests, the team registry, the session-start hook, the hook launchers and the Node
QA sandbox) are tested on Linux. The templates and a full end-to-end run on a project are not tested yet.

## Credits

The helper skills come from [ECC](https://github.com/affaan-m/ECC) (MIT) and are cloned during setup; none are
included in this repository.

## License

[MIT](LICENSE)
