---
description: Take a project from source to a live pod — get it onto GitHub, create and link a pod, start BrainPod onboarding, answer its questions with evidence from the code, fix failing builds through pull requests, and hand over the live URL and the pull request that turns on automatic deploys.
metadata:
  required_access:
    - CODEBASE
    - BRAINPOD_CLI
    - GITHUB
---

# Onboarding a project onto BrainPod

Clear both gates in `SKILL.md` first. BrainPod's onboarding agent works out
the pod — projects, Dockerfiles, `steiger.yml`, resources, variables — and
builds it in the repository's GitHub Actions. It works from a server on the
commit it pinned: it cannot see uncommitted work, cannot ask the person at the
keyboard, and commits only build files to its own branch, never a fix to the
application. Those are yours.

## Step 1: Find out where the project stands

Onboarding runs only on a pod that was never deployed; the API refuses any
other with `PRECONDITION_FAILED`. Do not work around that refusal by
redeploying, recreating, or emptying a pod. Before creating one, look for this
project on BrainPod: a `.github/workflows/deploy-to-brainpod-*.yml` on the base
branch, or a pod in `brainpod pod list --json` whose `repo get` names this
repository. Only a matching pod whose `deployed` is set is `operate.md`'s; a
link alone does not mean it was deployed.

A matching pod, or one the user already created for this project, that was
never deployed is the pod to onboard. It may have an onboarding session:
`sessions list --label purpose=repo-onboarding`. Follow an `active` one from
Step 6 rather than starting another.

## Step 2: Get the code onto GitHub

Onboarding reads only what is on GitHub. Establish, with evidence, which
repository and branch hold this project: `git remote -v`, the current branch,
and `gh repo view --json nameWithOwner,defaultBranchRef,visibility`.

- **Uncommitted or unpushed work** is not seen. Show the user what is missing
  and commit or push it only on their yes.
- **No GitHub remote** means a repository has to be created. Offer
  `gh repo create` (private unless the user says otherwise) and run it on
  their yes. Name what it costs: Actions minutes on a private repository are
  billed to their GitHub account.
- **Committed secrets** (`.env` files, keys, dumps) are worth flagging before
  the push, not after.

Onboarding builds from the default branch unless given `--ref`. Note the full
commit SHA it should see.

## Step 3: Give BrainPod access to the repository

```bash
brainpod github repos <owner/name> --json
```

The repository is reachable when it is listed. When it is not, read
`brainpod github auth --json`: an installation limited to selected
repositories has a `_links.settings` page where the repository is granted,
and `_links.connect` connects GitHub or another account or organization.
Open the one that applies by the rules in `SKILL.md`, tell the user which
repository to grant, and list again once they say it is done. Installing the
BrainPod GitHub App, choosing its repositories, and accepting the permissions
it asks for are the user's. A permission still waiting for approval is
reported there too; hand over its link the same way.

## Step 4: Create and link the pod

Reuse the project's never-deployed pod from Step 1; create one as `SKILL.md`
describes only when there is none. Open the pod page.

Linking is the user's consent to onboarding, so before linking tell them what
it will do and get an explicit yes: BrainPod commits the pod's build
configuration to the `brainpod/onboarding-<pod>` branch of their repository,
builds it in their GitHub Actions (billed minutes on a private repository),
deploys the pod on its own unless it needs their input, and opens a pull
request whose merge turns on deploying every push to the base branch. Without
that yes, stop here.

Link only when `brainpod --pod <pod> repo get` shows no repository:

```bash
brainpod --pod <pod> repo link <owner/name>
```

A pod keeps its repository; linking another is refused.

## Step 5: Start onboarding

```bash
brainpod --pod <pod> onboard --json
```

Note the session it returns. Starting again for the same ref returns the same
run, so an ambiguous failure is safe to repeat.

## Step 6: Follow the session

```bash
brainpod --pod <pod> sessions get <session> --json
```

On the first read, including a resumed session, compare `subject.sha` with
the commit noted in Step 2. If they differ, onboarding is building other code
than the user expects: tell them before going on. Then poll every 15 to 30
seconds, narrating changes rather than repeating them.
`runs` holds one run per step — `plan`, `image:<service>`, `wiring`,
`assemble` — each with its state and error; `pullRequest` the pull request
and its status.

The session completes itself at the pod's first healthy deploy. There is no
finish step.

## Step 7: Answer BrainPod's questions

A run awaiting input lists the questions onboarding could not settle on its
own, each with its options, a recommendation, and the evidence it read. For
each one:

1. Read the cited lines in the local checkout and anything else in the code
   that bears on it.
2. Put it to the user with the recommendation and what the code says,
   including where you disagree with the recommendation and why.
3. Take the user's answer. These are decisions about their production pod:
   never answer on assumption.

Answer every open question of the run in one `runs answer` call; the run
resumes only then. An answer is the chosen option's label exactly, unless the
question allows free text; a question asking for a value takes the value
itself, never a sentence about it.

Secret values only the user has are not questions: onboarding names them as
required values, which the user enters on the pod page in the console, never
in chat. Onboarding then does not deploy; the user deploys from the console
once they are in. Values any random string satisfies, such as session keys,
BrainPod generates itself.

## Step 8: Follow the builds

Each `image:<service>` run waits for its build in GitHub Actions. For detail
and logs:

```bash
brainpod --pod <pod> api /v1/pods/{pod}/builds/<build-id>
brainpod --pod <pod> api /v1/pods/{pod}/builds/<build-id>/log -q offset=<n>
```

A build is `waiting`, `ready`, or `failed`, and a failed one carries
`failureReason` and `failureLogTail`. The log is paged in bytes: the next
`offset` is the previous one plus the UTF-8 byte length of the `text`
returned, and the log ends at `length`. Append each page to the console as
`SKILL.md` describes.

An image run failing with `actions_limit` is the user's GitHub Actions
minutes or spending limit: give them its message. With `actions_failed`, or
no build after a few minutes, check `gh run list --branch
brainpod/onboarding-<pod>` and whether Actions is enabled for the repository;
enabling it is the user's.

## Step 9: When a step fails

A run that is `failed`, `timed_out`, or `canceled` ends with an error; report
it in plain words. `runs retry` reruns that step once its cause is dealt with,
such as an Actions limit the user raised. While onboarding watches its own
deploy it may revise its plan on the same draft: do not touch that pod's
draft, and leave `debug.md` until the session completes — a fix from outside
is undone by its next revision.

When a build fails, read the failure, then sort it:

- **Build configuration onboarding wrote** — its Dockerfile or `steiger.yml`
  — is onboarding's next move: it fixes the build or revises the plan. Wait
  for that, and never edit `brainpod/onboarding-<pod>`.
- **The application's own code** — a compile error, a missing lockfile, a
  dependency that only resolves locally — is yours. Reproduce it locally where
  you can, fix it on a new branch from the base branch, and open a pull
  request against the base branch with `gh pr create`. Give the user the link
  and a two-line account of the failure and the fix. Merging is theirs.
- **A private dependency or a resource limit** is reported as unsupported.
  Explain it and stop; working around it is a decision for the user.

A merged fix does not reach this onboarding: it stays pinned to its commit,
and a retry builds that same commit again. Tell the user before they spend
Actions minutes on it. If other services went live, the fixed one joins the
pod later through `operate.md`'s *Adding a service*.

## Step 10: Review and going live

When the last run, a succeeded `assemble`, waits for review, put what
onboarding decided in front of the user. Changes they ask for go back with
`runs reply`; never reply on assumption. Do not deploy an onboarding draft
yourself: the review and, where values were required, the deploy are the
user's.

When the pod has a deployed revision, verify readiness as `SKILL.md` defines
it with `revision get`, read the Route's assigned `hostname` from `resource
list`, and confirm it answers before handing it over.

## Step 11: The pull request

`pullRequest` is a draft until the pod is healthy, then ready for review.
Hand the user its URL and summarize what it adds (`gh pr view <number> --json
files`). Merging it is theirs: from then on every push to the base branch
deploys the pod, and until then merges build nothing. Closed unmerged, it
leaves deploy on push off.

## Reporting

Lead with the live URL. Then: the pod name, the deployed revision, the
readiness evidence, every answer the user gave, the onboarding pull request
and every other pull request still open for them, and what merging it turns
on. Point to the pod page, and to BrainPod Desktop for the pod at a glance and
database tunnels.
