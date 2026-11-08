# claude-plugins

Claude Code plugin marketplace maintained by m-mizutani.

## Install

```
/plugin marketplace add m-mizutani/claude-plugins
/plugin install spec-driven-development@m-mizutani
```

On Claude Code 2.1.275 or later, a single command also works:

```
/plugin install spec-driven-development --marketplace m-mizutani/claude-plugins
```

## Plugins

### spec-driven-development

Skills for a spec-driven development workflow. Skills are written in Japanese.

| Skill | Purpose |
| --- | --- |
| `brainstorm` | Investigates the codebase and external references, then proposes implementation approaches before a spec is written. |
| `spec` | Writes or updates a spec (requirements, basic design, design decisions, test plan, detailed design, implementation plan) as an HTML Artifact and stops at user approval. |
| `implement` | Implements the approved spec, checks every requirement against the implementation, handles review findings, and creates the pull request with CI checked. |
| `check-pr` | Reads the current branch's pull request review threads and CI status, fixes failures and valid comments, and resolves threads. |

Dependencies outside this plugin:

- `spec` requires the `Artifact` tool and the `artifact-design` skill, and uses the `decision-brief` and `run` skills.
- `implement` runs the `codex-review` skill and falls back to `code-review` when it is unavailable.
- `brainstorm` uses the `decision-brief` skill when asking the user to choose between approaches.
- `implement` and `check-pr` use the `gh` CLI for pull request and CI operations.
