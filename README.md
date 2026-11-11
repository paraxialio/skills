# Paraxial.io Security Skills

Security skills and agent instructions for AI coding agents, published by [Paraxial.io](https://paraxial.io).

They follow the open [Agent Skills](https://agentskills.io) format, so they work best with [Claude Code](https://claude.com/claude-code) or [OpenAI Codex](https://developers.openai.com/codex/skills) but can be used with any agent that reads files and runs shell commands.

| What | Type | Use it to |
|---|---|---|
| [`elixir-security-audit`](skills/elixir-security-audit/SKILL.md) | Skill | Run a security audit of one Elixir project, on demand |
| [`agents-md/elixir`](agents-md/elixir/AGENTS.md) | AGENTS.md rules | Keep your agent from writing the most serious Elixir security bugs, all the time |

The two are complementary: the **skill** finds problems in code that already exists; the **AGENTS.md rules** stop the agent from adding new ones.

## Skills

### elixir-security-audit

A guided security audit of the Elixir project in the current directory. Phoenix, LiveView, Absinthe (GraphQL), Ash, umbrella apps, Plug-only apps, and libraries are supported. It's written for people with no security background.

The skill was written by Michael Lubas of Paraxial.io, who worked on the [security audit of Hex](https://paraxial.io/blog/hex-pentest), the Elixir package manager. It's based on his experience doing security work with Claude and Codex.

Why use it? Asking Claude Code or Codex to "audit this app for security issues" without a skill has four common problems. This skill is designed to address each one:

1. **You burn unnecessary tokens.** A plain agent wanders the codebase and spends effort on low-impact issues. The skill follows a fixed plan. It asks all its questions once, up front, and hunts in order of impact: RCE, access control, SSRF, then path traversal. It offers a cheaper **quick check** mode and uses subagents only when they save work.
2. **The output is verbose and doesn't get to the point.** The skill's report has a fixed structure. Every finding opens with four things: severity, where the attacker starts, the attack, and what they gain. Technical detail comes after, and low-severity noise is kept out of the way.
3. **The results are confusing, even for security experts.** The skill cites `file:line` for every claim and explains each term the first time it's used. The report separates confirmed findings from "needs human review" and "checked and ruled out", and states what the audit did not cover.
4. **The false positive rate is high.** Before any finding reaches the report, a separate skeptical reviewer ("Eve") re-checks it against the code. Eve rejects anything she can't confirm herself. Scanner output from Sobelow and `mix hex.audit` is triaged by tracing whether an attacker can actually reach it, not reported as-is.

What it does:

1. Asks all its questions up front (what the app is, who can reach it, quick check or full audit, permissions), then works without interrupting.
2. Checks your security posture: Hex version, Sobelow usage, and whether CI runs dependency and security checks.
3. Runs `mix hex.audit`, `mix hex.outdated`, and [Sobelow](https://github.com/nccgroup/sobelow).
4. Maps the attack surface, triages scanner output, and does a manual hunt for high-severity bugs: remote code execution, access control, SSRF, and file inclusion / path traversal.
5. Sends every candidate finding to **Eve**, a skeptical second reviewer who must confirm it before it's reported. This cuts false alarms.
6. Writes a plain-language report, `security_audit_YYYY-MM-DD.md`, where each finding starts with severity, who the attacker is, what they do, and what they gain.
7. Offers fixes and CI improvements, and only makes changes you approve.

It reads code and runs tests locally. It never sends attack traffic to a deployed environment.

**Requirements:** an Elixir toolchain is recommended. Without one, the skill does a source-only review and gives you the scan commands to run yourself. Subagent support is optional but makes large audits faster.

#### Install

**Claude Code** (as a plugin):

```
/plugin marketplace add paraxialio/skills
/plugin install elixir-security-audit@paraxial
```

Or copy the skill folder by hand, for all your projects or for one:

```sh
git clone https://github.com/paraxialio/skills.git
cp -r skills/skills/elixir-security-audit ~/.claude/skills/          # all projects
cp -r skills/skills/elixir-security-audit your_app/.claude/skills/   # one project
```

**OpenAI Codex:**

```sh
git clone https://github.com/paraxialio/skills.git
cp -r skills/skills/elixir-security-audit ~/.codex/skills/           # all projects
cp -r skills/skills/elixir-security-audit your_app/.agents/skills/   # one project
```

**Other agents:** copy `skills/elixir-security-audit/` into the agent's skills directory. If it has no skills support, point it at the file: *"Follow the instructions in `path/to/elixir-security-audit/SKILL.md`."*

#### Use

From the root of your Elixir project, ask your agent:

> Run a security audit of this project.

Or invoke it directly: `/elixir-security-audit` in Claude Code, `$elixir-security-audit` in Codex.

A full audit of a large app can take a while and use a fair amount of your AI usage. Choose **quick check** when asked for a faster first look.

> [!WARNING]
> The report describes exploitable weaknesses in your app. Don't commit it. The skill offers to add `/security_audit_*.md` to your `.gitignore`.

## AGENTS.md rules

### Elixir

[`agents-md/elixir/AGENTS.md`](agents-md/elixir/AGENTS.md) is ten short rules that prevent the most serious bugs in Elixir and Phoenix code: mass assignment, remote code execution, unsafe deserialization, command injection, SSRF, path traversal, SQL injection, and missing authorization. It also tells the agent to run `mix hex.audit` and Sobelow before finishing, and never to silence a Sobelow warning on its own.

It's a concise companion to the audit skill and is meant to be **added to your project's existing `AGENTS.md`** (or `CLAUDE.md` for Claude Code), not used in place of it:

```sh
{ echo; curl -fsSL https://raw.githubusercontent.com/paraxialio/skills/main/agents-md/elixir/AGENTS.md; } >> AGENTS.md
```

Read what you append, and adjust it to your app if needed (for example, `current_scope` vs. `current_user`).

## Repository layout

```
skills/                    Agent Skills, one folder per skill (SKILL.md + references/)
agents-md/                 Rules to add to your project's AGENTS.md, one folder per language
.claude-plugin/            Claude Code plugin marketplace manifest
```

## Need help across many apps?

These skills are built for securing a single project. If your company relies on Elixir or Phoenix, [Paraxial.io](https://paraxial.io) provides application security across all your projects.

## License

[MIT](LICENSE)
