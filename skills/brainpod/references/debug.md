---
description: Diagnose a BrainPod build or deployment that failed, timed out, is unhealthy, or reports as deployed but is not serving traffic.
metadata:
  required_access:
    - BRAINPOD_CLI
    - CODEBASE
    - GITHUB
---

# Diagnosing a BrainPod pod

Clear both gates in `SKILL.md` first. While an onboarding is still running on
the pod, follow `onboard.md` instead: onboarding fixes its own deploy, and a
change from outside is undone by its next revision.

Work top-down: establish which stage failed — build, promotion, or runtime —
before reading a single log line. Investigation is read-only; every fix ends
in a pull request or a draft change, never a redeploy.

## Step 1: Locate the failure

```bash
brainpod --pod <pod> pod get <pod> --json
brainpod --pod <pod> revision get <revisionId> --json
```

Revision payloads use `state`; `pod get` reports `head.status` and
`deployed.status`. Report what you observed and do not infer transitions you
did not see.

| Observation | Meaning |
|---|---|
| A merge deployed nothing, `deployed` unchanged | The build failed or never ran — Step 2 |
| `head.status: draft`, `deployed` older | Changes were never promoted; nothing is broken yet |
| `state: failed` or `canceled` | Promotion failed; read the revision's `error` first. A `${secret.<name>}` whose secret has no value fails here |
| `state: deployed`, a resource `healthy: false` | A workload did not come up — Step 3 |
| `state: deployed`, all healthy, app unreachable | Routing or the app itself — Step 4 |

A wait that timed out is not a diagnosis: it means "still unhealthy after N
seconds". Re-attach with `revision wait <revisionId> --timeout 600 --json`.
**Never redeploy to poke a timeout or an unhealthy revision**: `redeploy`
only retries a failed head, and it destroys the evidence.

## Step 2: A build that failed or never ran

```bash
brainpod --pod <pod> api /v1/pods/{pod}/builds -q rev=<commit-sha>
brainpod --pod <pod> api /v1/pods/{pod}/builds/<build-id>/log -q offset=<n>
```

A `failed` build carries `failureReason` and `failureLogTail`; read the log
from the start when the tail does not explain it. The log is paged in bytes,
as `onboard.md` describes. A build that fails on the application's code is
fixed by a pull request (`operate.md`); compare the failing command with what
the project runs locally before guessing.

No build for the commit means GitHub never got as far as BrainPod. Look at
the run with `gh run list --commit <sha>` and `gh run view <run-id>
--log-failed`:

- **No run at all**: the base branch has no `deploy-to-brainpod-<pod>.yml`
  (see `operate.md`), Actions is disabled, or the account's Actions minutes or
  spending limit stopped it. The last two are the user's to fix in GitHub.
- **Failed before the build**: the GitHub OIDC sign-in is refused unless the
  job has `permissions: id-token: write` and runs for the repository linked to
  the pod, on its base branch, its onboarding branch, or a `v*` tag.

## Step 3: An unhealthy resource

List every `resources[]` item with `healthy: false`, with its URN and each
replica's `name`, `phase`, and `reason`. `status` is kind-specific and may be
null: a workload reports replicas, a `Disk` reports `phase`/`bound`/`ready`,
others only `ready`.

Then read that resource's events, addressed by URN:

```bash
brainpod --pod <pod> events --resource <urn> --range 1h --json
```

- Omit `--kind` to get every stream; narrow only once you know which one
  matters.
- Do not filter startup diagnostics by `--level`: a crash-looping container
  often logs its real cause below `error`.
- Widen `--range` before concluding a stream is empty. `Route` and `Disk` have
  no workload stream, so **zero events is not a health verdict.**
- `--watch` is for watching a fix land, not for investigating the past.

## Step 4: Match the symptom to a cause

| Symptom | Likely cause |
|---|---|
| Starts, then dies binding its port | Listening below 1024, which the runtime uid cannot bind. Set `PORT` ≥ 1024 in `App.spec.env` and use the same port in the Route |
| Permission denied writing a path | The path is not writable by the runtime uid; mount a `Disk` there and prefer `App.spec.runtime.fsGroup` |
| `exec format error` or an instant crash loop | The image's architecture differs from the cluster's; compare with `brainpod cluster list --json` |
| Healthy, but requests 404 or hang | `Route.rules[].backendRef` names the wrong App, or the Route port differs from the image's exposed ports |
| Healthy, database connection refused | A literal connection string, or a `${<name>.<field>}` naming a resource that does not exist; check names against `resource variables` |
| Crashes reading a variable at boot | A variable nothing documented and nothing set |

Read the image before guessing at any of these. A GitHub build's
`artifacts[].config` reports its `user`, `workingDir`, and `exposedPorts`; for
an image built on a machine, `brainpod --pod <pod> image inspect <repository>
<tag> --json` reports the same per architecture. When a GitHub-built image
would run as root, the build action rebuilds it to run as uid 1000 with its
working directory owned by that uid, so anything the app writes outside that
directory fails. An image whose working directory is `/` is not rebuilt and
stays root, which BrainPod does not run.

To look at a database's actual contents, open a tunnel:

```bash
brainpod --pod <pod> tunnel <database> --json
```

It prints `listening` and then `credentials` events (with a password and a
DSN) and runs until interrupted. Use that password, never repeat it in chat,
and run read-only queries only: a change to data is the user's decision. The
user can do the same from BrainPod Desktop, which also shows the pod's logs
live.

## Step 5: Fix forward

Diagnosis ends in a change: a pull request for code (`operate.md`), or a
draft change for resources followed by diff and deploy. Do not create probe
or debug resources on the user's pod, and ask before any diagnostic deploy.

## Reporting

Report the failing stage, the revision id and its `state`, the revision
`error` or the build's `failureReason`, each unhealthy resource's URN with
replica `phase` and `reason`, the API `code` and `requestId` for any API
error, and the cause you matched — or, honestly, that you could not narrow it
further. Events and log lines are supporting evidence, not the finding.
Never present an unverified fix as success.
