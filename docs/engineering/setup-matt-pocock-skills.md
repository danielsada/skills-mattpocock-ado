## What it does

`setup-matt-pocock-skills` answers three questions about one repo: where issues live, what the triage labels are called, and where the domain docs sit. It records the answers as markdown files under `docs/agents/`.

Those files are the only thing that varies between repos. The skills themselves are identical everywhere; they read `docs/agents/issue-tracker.md` at run time and do what it says. That is why the set is not tied to GitHub, and why no skill file ever needs editing to point it somewhere else. GitHub, GitLab, Azure DevOps, and local markdown ship as first-class templates. Invoking setup with "link the skills to a custom issue tracker" also works with anything you can connect to programmatically.

It is a prompt-driven skill, not a deterministic script. It reads your `git remote`, your existing `CLAUDE.md`, your existing `CONTEXT.md`, proposes what it found, and waits for you to confirm before writing anything.

## When to reach for it

You invoke this by typing `/setup-matt-pocock-skills`; the [agent](https://www.aihero.dev/ai-coding-dictionary/agent) won't reach for it on its own. It is deliberately marked non-invokable, so no other skill can fire it for you.

Reach for it once per repo, before the first use of any other engineering skill. If [triage](https://aihero.dev/skills-triage), [to-spec](https://aihero.dev/skills-to-spec), [to-tickets](https://aihero.dev/skills-to-tickets) or [wayfinder](https://aihero.dev/skills-wayfinder) start guessing where your issues go, or apply labels your tracker doesn't have, they have not been set up here yet. A repo already halfway through a project is a fine place to run it; the skill reads what is already there and no earlier work is wasted.

## Prerequisites

It writes into the repo you run it in:

| It writes | Where |
| --- | --- |
| `issue-tracker.md` | `docs/agents/` |
| `domain.md` | `docs/agents/` |
| `triage-labels.md` | `docs/agents/`, only when the `triage` skill is installed |
| An `## Agent skills` block | whichever of `CLAUDE.md` / `AGENTS.md` already exists |

All of it is committed markdown. There is no user-level or global mode: the config lives in the repo, so every repo gets its own copy.

## The three decisions

It leads each section with the recommended answer, and skips whatever exploration already settled. Most runs are two confirmations and done.

| Decision | What it proposes | When it actually asks |
| --- | --- | --- |
| **Issue tracker** | the one matching your `git remote` | always: this is the one real choice |
| **Triage labels** | keep the five canonical names (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`) | only if the `triage` skill is installed |
| **Domain docs** | single-context: one `CONTEXT.md` plus `docs/adr/` at the root | only if it spots monorepo signals, and then it offers a multi-context `CONTEXT-MAP.md` |

The tracker options:

| Option | Where issues live | Needs |
| --- | --- | --- |
| **GitHub** | the repo's GitHub Issues | the `gh` CLI |
| **GitLab** | the repo's GitLab Issues | the `glab` CLI |
| **Azure DevOps** | Azure Boards `Feature` parents with child `Task` tickets, plus Azure Repos PRs | Azure CLI with the `azure-devops` extension, plus an organization URL and project |
| **Local markdown** | files under `.scratch/<feature>/` in this repo | nothing: no remote at all |
| **Other** | wherever you say | one paragraph from you describing the workflow |

The first four ship as templates in the skill and work out of the box. Local markdown is a first-class option, not a fallback: a solo project with no remote is fully supported. One caveat is worth repeating: don't use local markdown if you're using a hosted tracker. They are alternatives, not layers.

For Azure DevOps, setup asks for the organization URL and project instead of relying only on git-remote detection. It defaults parent specs and maps to Azure Boards `Feature` work items and implementation and decision tickets to child `Task` work items. It verifies those types against the project first because the Basic process has `Issue` instead of `Feature`, and asks for an available replacement only when a default is missing. It maps triage roles to tags and records Azure Repos pull-request commands. Azure Boards process templates also disagree on whether a completed work item is named `Done`, `Closed`, or something custom, so the generated instructions resolve the state whose category is `Completed` instead of hard-coding a name.

"Other" is not a stub either. It is the reason Jira, Linear and Beads work: you describe the workflow, the skill records your prose in `docs/agents/issue-tracker.md`, and the downstream skills follow the prose. The community has already done this: a Jira-over-[MCP](https://www.aihero.dev/ai-coding-dictionary/mcp) variant, a Gitea CLI shaped like `gh`, a hand-built local dashboard.

## Common questions

**Do I have to use GitHub?**

No. GitHub, GitLab, Azure DevOps, and local markdown under `.scratch/` all ship as ready-made templates, and anything else works through the "other" path. This is the most-repeated question in the record, in roughly these words: *"hard locked to github"*, *"can I use GitLab / Jira"*, *"what about Azure DevOps"*. The answer every time is that the tracker is a setup answer, not a skill property.

**What does Azure DevOps setup ask me for?**

The Azure DevOps organization URL and project name or ID. Those values are written into `docs/agents/issue-tracker.md` and passed explicitly to Azure CLI commands, so Boards can remain the issue tracker even when the local git remote is hosted somewhere else. Parent artifacts default to `Feature` and their tickets to child `Task` work items. Setup asks about type names only when the project does not expose those defaults or cannot be inspected. Azure Repos PR triage is available but disabled until you flip its request-surface flag.

**Do I need to re-run it after updating the skills?**

Asked directly after v1.1, Matt said yes. The skill's own closing message is softer: it tells you re-running is only needed to switch trackers or start over. Both are defensible and the reason for the gap is real: the seed templates change between versions, so a `docs/agents/issue-tracker.md` written by an older release can go stale against the skills now reading it. If a downstream skill starts doing something the docs describe differently, re-running is the cheap fix.

**It wrote to `CLAUDE.md`, but I'm on Codex.**

Known gap, still open. The file-selection rule is "edit `CLAUDE.md` if it exists, else `AGENTS.md`": it checks which file exists, not which [harness](https://www.aihero.dev/ai-coding-dictionary/harness) is running. A repo with a `CLAUDE.md` left over from Claude Code will get its `## Agent skills` block somewhere Codex never reads. Two workarounds are in circulation: move the block to `AGENTS.md` by hand, or keep `AGENTS.md` canonical and make `CLAUDE.md` a one-line pointer at it. If neither file exists, the skill asks you which to create rather than picking, which has confused people who expected it to just decide.

**It didn't create my triage labels.**

It doesn't. `docs/agents/triage-labels.md` is a *mapping*: it tells `/triage` which strings in your tracker correspond to the five canonical roles. It does not run `gh label create`. On a fresh GitHub repo the labels genuinely do not exist yet, and this has been filed as a bug more than once. Two follow-ons:

- If your tracker already uses the canonical names, the mapping is an identity table and there is nothing to configure. That is the intended common case, not a missing step.
- [wayfinder](https://aihero.dev/skills-wayfinder)'s `wayfinder:map` and `wayfinder:<type>` labels are not created here either, and `gh issue create --label <missing>` fails outright rather than creating the label. Create them by hand before the first wayfinder run on a GitHub repo.

**Can I configure the other skills' behaviour here ([grilling](https://www.aihero.dev/ai-coding-dictionary/grilling) cadence, question format, tone)?**

No. It configures three things: tracker, labels, doc layout. There have been direct requests to make it the home for per-user preferences, and the standing answer is that skills stay opinionated: *"Config is death."* Preferences belong in your `CLAUDE.md` as plain instructions, which every skill already reads.

**Can I keep the config in `~/.claude` instead of committing it to every repo?**

Not today. There is an open request for exactly this from someone running the skills across many repos, and no user-level mode exists. Every repo carries its own `docs/agents/`.

**Isn't it strange to have a skill that configures the other skills?**

One long-standing complaint says yes, in these words: *"having a skill to set up the other skill does not feel right to me: that means the LLM is configuring its own skills."* The trade is real and acknowledged: the alternative to a setup step is duplicating tracker instructions into every skill that touches issues. The output is inspectable, editable markdown, which is the mitigation: you can read every file it wrote and change it by hand, and day-to-day tweaks are exactly that, not another run.

## It's working if

- `docs/agents/issue-tracker.md` and `docs/agents/domain.md` exist, plus `triage-labels.md` if `triage` is installed.
- An `## Agent skills` section appears in the instruction file your harness actually reads, with a one-line summary pointing at each of those files.
- The tracker it proposed matches the remote you really use, and the label strings match labels that really exist in your tracker.
- For Azure DevOps, the generated tracker file contains the intended organization and project, creates `Feature` parents with child `Task` tickets, and preserves unrelated tags when applying triage roles.
- Afterwards, `/to-tickets` publishes without asking you where issues live, and `/triage` applies labels rather than inventing them.
- Nothing in the skill files themselves changed. If setup edited a `SKILL.md`, something went wrong.

## Where it fits

`setup-matt-pocock-skills` is the **run-once setup** for the engineering flow, the precondition everything else assumes rather than a step in the chain. Its neighbours are its readers: [triage](https://aihero.dev/skills-triage), which applies the label vocabulary written here; [to-spec](https://aihero.dev/skills-to-spec) and [to-tickets](https://aihero.dev/skills-to-tickets), which publish into the tracker named here; and [wayfinder](https://aihero.dev/skills-wayfinder), which reads the "Wayfinding operations" section of the same tracker file to know how maps and child [tickets](https://www.aihero.dev/ai-coding-dictionary/ticket) are stored. The domain-doc layout it records is the one [domain-modeling](https://aihero.dev/skills-domain-modeling) fills in later: it creates `CONTEXT.md` and ADRs lazily, when a term or decision actually gets resolved, so an empty repo after setup is the expected state. For which skill to reach for next, [ask-matt](https://aihero.dev/skills-ask-matt) routes the whole set.
