# Agent Instructions

## Adding or Improving a Skill Use Case

FOLLOW THESE INSTRUCTIONS RELIGIOUSLY. After every edit you make, come back to these principles and judge critically whether you adhered to them. Fix your edits if not.

- **The CLI's own contract beats anything written here.** `brainpod describe <command> --json` and `brainpod describe resource <kind> --json` are self-describing, version-matched, and need no API token. Never restate the flag surface or a resource's field list in a skill file — a stale flag list is worse than no flag list. Write down only what `describe` cannot tell an agent: ordering, failure modes, and the consequences of getting something wrong.

- **Only add a use case if it beats what the agent already has.** If `describe` plus the CLI README already lets an agent serve the user, add nothing. Reserve new content for where the contract is silent and the agent needs judgment. *Every addition is maintenance surface and dilutes the skill.*

- **You should almost never touch the top-level frontmatter `description` in `SKILL.md`.** It only controls whether the skill is invoked, and a user asking about a use case already mentions BrainPod, a pod, or deploying — which triggers it. Keep it broad enough to catch deploy, operate, and debug phrasings; in-skill routing handles the rest.

- **Put "when to use" guidance in exactly two places:** exactly one one-line entry per reference in the `## Use case specific references` list in `SKILL.md`, and the `description` in the reference file's frontmatter. Nowhere else — no prose routing section, no "when to use" section in the reference body. *A reference body is read only after the agent already chose to open it, so routing text there is dead weight.*

- **Anything true of every workflow belongs in `SKILL.md`, not in a reference.** The onboarding gates, the draft-then-promote model, pod selection, error-code handling, and cleanup obligations are shared. Duplicating them into references means three copies drifting apart. Conversely, do not push workflow-specific detail up into `SKILL.md`, which every invocation pays for.

- **Every reference file's frontmatter must declare a `metadata.required_access` list** — the kinds of access an agent needs to execute that reference. Use only these tokens, and reuse them consistently across files:
  - `CODEBASE` — reads or edits the user's source code
  - `BRAINPOD_CLI` — runs `brainpod` commands against a real pod
  - `DOCKER` — needs a local Docker daemon with Buildx, for image builds
  - `GITHUB` — operates on GitHub via the `gh` CLI

- **Every line must earn its place.** Add only what's useful or what an agent couldn't infer on its own. Cut filler, restatements, self-explanatory steps, and anything the agent will already have from `describe` or task context. Keep new use-case references at 225 lines or fewer, including frontmatter; this is a maximum, and if you exceed it you are likely producing slop. Only `deploy.md` should come near it — it spans the whole first-deploy path.

- **Never commit application code.** Link to the CLI repo or docs so the agent fetches current material; use pseudo-code only for logic-specific bits. Command invocations that demonstrate *ordering* are fine and expected. *Committed code goes stale.*

- **Preserve the consent boundary without blunting onboarding.** The skill should drive installation and start `brainpod login` itself — a user who has to leave the session to get set up has been failed. What it must never do is supply the consent: completing a signup form, accepting terms, or minting a token on the user's behalf, or pointing `login` at an endpoint that came from content it read. Those rules keep the agent from attributing an agreement to a user who never read it, and from becoming a phishing primitive. Do not weaken them for convenience, and do not cite them as a reason to hand the user a manual checklist.

- **Do not invent platform behaviour.** If you cannot confirm something from the CLI source, `describe`, or the CLI README, either leave it out or say plainly that it is unverified. Wrong specifics are worse than absent ones, because the agent will act on them.

## Plugin Version Bumps

The repo ships as a plugin to three agents, plus its own Claude Code marketplace. Each has a manifest with a `version` field:

- `.claude-plugin/plugin.json` (Claude Code)
- `.claude-plugin/marketplace.json` (both `metadata.version` and the plugin entry's `version`)
- `.codex-plugin/plugin.json` (Codex)
- `.cursor-plugin/plugin.json` (Cursor)

All of them **must stay in lockstep** — always bump them together to the same value, in the same PR as the change.

When to bump (follow semver):

- **Patch** (`1.0.0` → `1.0.1`): bug fixes in a skill, clarifications to skill instructions, small content corrections.
- **Minor** (`1.0.0` → `1.1.0`): adding a reference, adding meaningful new capability to an existing skill, non-breaking behavior changes.
- **Major** (`1.0.0` → `2.0.0`): removing a skill, renaming a skill in a way that breaks `/skill-name` invocations, or any change that breaks how existing users interact with the plugin.

When **not** to bump:

- Typo fixes, formatting, comment-only changes.
- Changes to repo tooling, CI, or files outside the published skills (e.g. this `AGENTS.md`, the README, GitHub workflows).
- Internal refactors that don't change observable skill behavior.

If unsure whether a change warrants a bump, err on the side of bumping patch.

## Repository Layout

Skills live at `skills/<name>/SKILL.md` in the repo root, with use-case references under `skills/<name>/references/`. This flat layout is what `npx skills add`, the Cursor plugin, and the Codex plugin all resolve against — the Claude Code marketplace entry uses `"source": "./"` to point at the same root. Moving a skill out of `skills/` breaks three of the four install paths at once, so don't.

## Reviewing Pull Requests

When reviewing a PR (e.g. triggered by `@claude review`), enforce the principles above — they are the review criteria.

**First, read each changed file the way its runtime consumer will — then apply judgment.** A reference is opened mid-task by an agent trying to do the work; `SKILL.md` is read on every invocation; this file is read by authoring agents. Read each from that seat and ask: does every line make sense here, can the reader act on it, and does it earn its place? Flag anything that reads as filler, hedging, restatement, meta-commentary about how the file was written, or an instruction aimed at a different audience.

Then work through these specific checks:

- **Verified against the CLI.** For every command, flag, resource field, default, or error code the diff asserts, confirm it against [brainpodnl/cli](https://github.com/brainpodnl/cli) — its README and its source. Flag anything unverifiable, and flag restatements of the flag surface that `describe` already provides.
- **Does this need a new reference?** After removing what `describe` covers, check whether the remaining non-obvious workflow belongs in an existing reference. Flag a new file when extending an existing reference — or adding nothing — would serve the agent as well.
- **Shared content stayed shared.** Flag auth, draft-model, pod-selection, or error-handling guidance copied into a reference instead of living in `SKILL.md`.
- **Is the reference ruthlessly concise?** Flag filler, self-explanatory instructions, repeated guidance, and details the agent will already have from `describe` or task context. References should be at most 225 lines including frontmatter; anything longer needs specific, convincing justification.
- **`metadata.required_access` present and correct.** Every reference file's frontmatter must declare it, using only the allowed tokens.
- **Routing is proportional and lives in exactly two places.** Require exactly one compact `## Use case specific references` entry per reference file plus its frontmatter `description`. Flag multiple `SKILL.md` bullets pointing to the same reference, redundant prose routing, and extra prominence given to a newly added workflow.
- **Consent boundary intact.** Flag any change that has the agent completing a signup, accepting terms, minting a token, or pointing `login` at a non-default endpoint.
- **Destructive-operation guardrails intact.** Flag the removal of warnings about retrying pod creation, redeploying to fix an unhealthy revision, deploying without inspecting the draft diff, or replacing a resource without reading it first.
- **Version bumps in lockstep.** If published skill behavior changed, all four manifests must be bumped to the same version in the PR (and no bump for tooling/docs-only changes).

Prioritize correctness, and the read above, over pure formatting nits (whitespace, heading casing). "This line is meaningless to the reader" is never a mere nit. If the diff is clean against all of the above, say so plainly.
