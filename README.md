# claude-skills

My personal collection of Claude Code agents and skills.

> **Note:** These are tailored to my personal workflow and environment. They may reference tools, Bazel conventions, or internal systems that won't be present on every machine or in every project context. Use them as a reference or starting point, but expect to adapt them to your own setup.

## Installation

From within Claude Code, run these two commands:

```
/plugin marketplace add NaOHman/claude-skills
/plugin install claude-skills@naoh-tools
```

After installing, skills are invoked with the `claude-skills:` namespace:

```
/claude-skills:tdd-feature implement a login form with validation
```

## Structure

- `agents/` — Custom sub-agent definitions (architect, code-monkey, critic, qa)
- `skills/` — Skill definitions
- `.claude-plugin/` — Plugin and marketplace manifests

## Skills

### `/claude-skills:tdd-feature`

Coordinates architect, critic, qa, and code-monkey agents to implement a feature using TDD.

**Usage:** `/claude-skills:tdd-feature [--interactive] [--no-coverage] <task description>`

**Flags:**
- `--interactive` — Pause after the architect and critic agree on a plan and ask for your approval before implementation begins
- `--no-coverage` — Skip the QA coverage pass; the critic code review still runs
