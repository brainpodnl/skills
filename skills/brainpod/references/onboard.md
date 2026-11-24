---
description: Take a project from source to a live pod — get it onto GitHub, start BrainPod onboarding, answer its questions with evidence from the code, fix failing builds through pull requests, and report the live URL.
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

The console steps for this workflow, in order: *Checking your setup*,
*Getting your code on GitHub*, *Connecting the repository*, *Answering
BrainPod's questions*, *Building your app*, *Going live*, *Turning on
automatic deploys*.

## Step 1: Find out where the project stands

**Every onboarding creates a new pod.** Before starting one, check that this
project is not already on BrainPod: a `.github/workflows/deploy-to-brainpod-*.yml`
on the base branch, or a matching pod in `brainpod pod list --json`. If it is,
the work is `operate.md`'s.

```bash
brainpod api /v1/onboarding
```

`step` is where the user stands: `connect`, `repository`, `analysis`,
`review`, `deploy`, or `done`. A user has one live onboarding at a time, so
when this returns a session for this repository, follow it from its `step`
rather than starting another. When it is for a different repository and not
`done`, tell the user and let them finish or leave it in the dashboard. A
`NOT_FOUND` saying repository onboarding is not available means the account
does not have it: tell the user, and offer `local-build.md`.

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

Onboarding builds from the branch the user picks, normally the default
branch. Note the full commit SHA it should see.

## Step 3: Connect the repository

Open `<dashboard>/new-pod/github` by the rules in `SKILL.md`; a user who has
not finished onboarding is sent on to `/onboarding`, which is the same flow.
Tell the user, before the page appears, which repository and branch to pick.
Connecting GitHub, installing the BrainPod GitHub App, and choosing its
repositories are theirs; connecting a repository is also their consent to
BrainPod committing to it. Keep this tab open: it is the onboarding canvas,
where review, required values, and the final pull request appear.

Follow progress with `brainpod api /v1/onboarding` every 15 to 30 seconds,
narrating changes rather than repeating them. Read `pod.name` as soon as it
appears; every later command needs it with `--pod`. Open the pod page in its
own tab then, as the deliverable that outlives the session.

Check `repository.sha` against the commit you noted. If it differs, the
onboarding is building something other than what the user expects; say so
before going further.

## Step 4: Answer BrainPod's questions

When `run.state` is `awaiting_input`, `run.questions[]` holds what onboarding
could not settle on its own. Each carries `options` (each with a `label`), a
`recommended` index into `options`, `allowOther`, and `evidence` (file, lines,
excerpt). For each one:

1. Read the cited lines in the local checkout and anything else in the code
   that bears on it.
2. Put it to the user with the recommendation and what the code says,
   including where you disagree with the recommendation and why.
3. Take the user's answer. These are decisions about their production pod:
   never answer on assumption.

An `answer` is the chosen option's `label`, exactly, unless `allowOther`
permits free text; a question asking for a value takes the value itself,
never a sentence about it. Submit every answer at once — the run resumes only
when all are answered — in the canvas or with:

```bash
brainpod --pod <pod> api -X POST /v1/pods/{pod}/agent-runs/<run-id>/resume --input answers.json
```

where `answers.json` is `{ "answers": [ { "id": "<question id>", "answer": "<text>" } ] }`
and each answer is at most 4000 characters.

Secret values the app needs, the user enters in the canvas. Values any random
string satisfies, such as session keys, BrainPod generates itself.

## Step 5: Follow the builds

Once the plan settles, BrainPod commits the workflow and build configuration
to `brainpod/onboarding-<pod>` and builds there in GitHub Actions.
`builds[]` in the onboarding response lists them; for detail and logs:

```bash
brainpod --pod <pod> api /v1/pods/{pod}/builds/<build-id>
brainpod --pod <pod> api /v1/pods/{pod}/builds/<build-id>/log -q offset=<n>
```

A build is `waiting`, `ready`, or `failed`, and a failed one carries
`failureReason` and `failureLogTail`. The log is paged in bytes: the next
`offset` is the previous one plus the UTF-8 byte length of the `text`
returned, and the log ends at `length`. Append each page to the console as
`SKILL.md` describes.

**No build after a few minutes** means GitHub did not start one. Ask BrainPod
why, with the onboarding branch's head commit (`git ls-remote origin
brainpod/onboarding-<pod>`):

```bash
brainpod --pod <pod> api /v1/pods/{pod}/onboarding/ci -q sha=<full-sha>
```

A `blocker` of kind `actions_limit` is the user's GitHub Actions minutes or
spending limit: give them its `message` and `billingUrl`. A permission the
GitHub App still needs is named in the canvas with a link to accept it on
GitHub. With no run and no blocker, check `gh run list --branch
brainpod/onboarding-<pod>` and whether Actions is enabled for the repository;
enabling it is the user's.

## Step 6: When onboarding stops

`run.state` of `failed`, `timed_out`, or `canceled` ends the run; report its
`errorCode` and `errorMessage` in plain words. Retrying is the user's, from
the canvas. While `run.health.state` is `checking` or `failing`, onboarding is
watching its own deploy and may revise its plan on the same draft: report the
`service` and `reason` and wait. Do not touch that pod's draft, and leave
`debug.md` until onboarding has ended — a fix from outside is undone by its
next revision.

When a build fails, read the failure, then sort it:

- **Build configuration onboarding wrote** — its Dockerfile or `steiger.yml`
  — is onboarding's next move: it retries the build or reports the service as
  unsupported in its review. Wait for that, and never edit
  `brainpod/onboarding-<pod>`.
- **The application's own code** — a compile error, a missing lockfile, a
  dependency that only resolves locally — is yours. Reproduce it locally where
  you can, fix it on a new branch from the base branch, and open a pull
  request against the base branch with `gh pr create`. Give the user the link
  and a two-line account of the failure and the fix. Merging is theirs.
- **A private dependency or a resource limit** is reported as unsupported.
  Explain it and stop; working around it is a decision for the user.

A merged fix does not reach this onboarding: it stays pinned to its commit,
and a retry from the canvas builds that same commit again. Tell the user
before they spend Actions minutes on it. If other services went live, the
fixed one joins the pod later through `operate.md`'s *Adding a service*. If
nothing was deployed, starting over in the canvas reads the branch again;
that is the user's call.

## Step 7: Going live

Onboarding deploys the pod itself unless the user still has required values
to enter; then it waits in `review`, and the user deploys from the canvas
once they are in. Do not deploy an onboarding draft yourself: the deploy is
refused while an onboarding run is active or a build is not ready, and the
review is the user's.

When `pod.deployedRevisionId` is set, verify readiness as `SKILL.md` defines
it:

```bash
brainpod --pod <pod> revision get <deployedRevisionId> --json
```

Read the Route's assigned `hostname` from `resource list` and confirm it
answers before handing it over.

## Step 8: Turning on automatic deploys

When the pod is healthy, BrainPod makes the pull request from
`brainpod/onboarding-<pod>` ready for review. Ask the user to open it from the
canvas's last step: that is also what finishes onboarding, which their next
onboarding needs. Summarize what it adds (`gh pr view <number> --json files`).
Merging it, which is theirs, puts the workflow on the base branch; from then
on every merge into that branch deploys, and until then merges build nothing.

## Reporting

Lead with the live URL. Then: the pod name, the deployed revision, the
readiness evidence, every answer the user gave, every pull request still open
for them, and that merging into the base branch now deploys. Point to the pod
page, and to BrainPod Desktop for the pod at a glance and database tunnels.
