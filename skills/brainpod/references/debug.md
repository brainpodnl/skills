---
description: Diagnose a BrainPod deployment that failed, timed out, is unhealthy, or reports as deployed but is not serving traffic.
metadata:
  required_access:
    - BRAINPOD_CLI
    - CODEBASE
---

# Diagnosing a BrainPod deployment

Clear both onboarding gates in `SKILL.md` first. Work top-down: establish
*what state the revision is in* before reading a single log line. Events are
drill-down, not a starting point.

## Step 1: Establish revision state

```bash
brainpod --pod <pod> pod get <pod> --json
brainpod --pod <pod> revision get <revisionId> --json
```

Mind the naming: revision payloads use `state`, while `pod get` reports
`head.status` and `deployed.status`. Report what you observed and do not
infer transitions you did not see.

| Observation | Meaning |
|---|---|
| `head.status: draft`, `deployed` older | Changes were never promoted — nothing is wrong yet; deploy them |
| `state: failed` or `canceled` | Promotion itself failed; read the revision's `error` field first |
| `state: deployed`, some resource `healthy: false` | Promotion succeeded, a workload did not come up — Step 2 |
| `state: deployed`, all `healthy: true`, app unreachable | Routing or the app's own behaviour — Step 4 |

A wait that timed out is not itself a diagnosis. `revision wait` returns as
soon as every resource is healthy and only aborts early on `failed` or
`canceled`, so a timeout means "still unhealthy after N seconds" — re-attach
with a longer timeout rather than redeploying:

```bash
brainpod --pod <pod> revision wait <revisionId> --timeout 600 --json
```

**Never redeploy to poke a timeout or an unhealthy revision.** It re-applies
the same content and destroys the evidence.

## Step 2: Identify the unhealthy resource

From the revision payload, list every `resources[]` item with
`healthy: false`, and report each one's URN plus its replica `name`, `phase`,
and `reason` where present.

`status` is kind-specific and may be null — a workload reports replica status,
a `Disk` reports `phase`/`bound`/`ready`, and others may expose only `ready`.
Do not require `status.phase: Ready` across kinds.

## Step 3: Read events for that resource

Events are addressed by the resource URN returned by `resource list`,
`resource get`, or a mutation response — for example
`urn:brain:app:default:api`.

```bash
brainpod --pod <pod> events --resource <urn> --range 1h --json
```

- **Omit `--kind` to get every stream available** for that resource. Reach for
  a specific stream only once you know which one matters.
- `--level` requires `--kind`, and levels apply to the `app` stream. Do not
  filter startup diagnostics by level — a crash-looping container often logs
  its real cause below `error`.
- The CLI value `http-access` maps to the API value `httpAccess`.
- `--range` accepts `5m`, `15m` (default), `30m`, `1h`, `24h`, `7d`. Widen it
  before concluding a stream is empty.
- Non-workload kinds such as `Route` and `Disk` have no workload event stream.
  **Zero events is not a health verdict.**

Prefer a bounded query for diagnosis. `--watch` reconnects until interrupted
and is for observing a fix land, not for investigating a past failure.

## Step 4: Match the symptom to a cause

Most BrainPod deployment failures are one of a small set, and the platform's
error text rarely names the real cause:

| Symptom | Likely cause |
|---|---|
| Container exits immediately, no app logs | Image runs as root — rejected at resource creation, so this usually surfaces earlier as a validation error |
| App starts, then dies binding its port | Listening below port 1024; the runtime uid cannot bind privileged ports |
| `exec format error` or instant crash loop | Architecture mismatch — compare the build's `platform` against `cluster list` |
| Permission denied writing a path | Path not writable by the runtime uid; prefer `App.spec.runtime.fsGroup` over a chown init step |
| Deployed and healthy, but requests 404 or hang | `Route.rules[].backendRef` points at the wrong App, or the Route port does not match the image's `exposedPorts` |
| Healthy app, database connection refused | `App.spec.env` holds a stale or invented connection value — the resource API publishes no exports or interpolation, so every value is literal |

Cross-check the image against the resource that references it:

```bash
brainpod --pod <pod> image inspect <repository> <tag> --json
brainpod cluster list --json
```

`image inspect` returns per-architecture digest references, UID/GID values,
and `exposedPorts` — enough to settle the architecture, runtime-user, and port
questions above in one call. It defaults to `pod` visibility.

## Step 5: Fix forward

Diagnosis ends in a resource change, not a redeploy. Apply the correction via
`operate.md` — replace the resource (or rebuild and repin the image), diff the
draft, deploy, and wait.

Do not unpack images, and do not create probe or debug resources on the user's
pod to investigate. Ask before any diagnostic deployment.

## Reporting

Report: the revision id and its `state`, the revision `error` when present,
each unhealthy resource's URN with replica `phase` and `reason`, the API
`code` and `requestId` for any API error, and the specific cause you matched —
or, honestly, that you could not narrow it further. Include events as
supporting evidence, not as the finding. Never present a partial or
unverified fix as success.
