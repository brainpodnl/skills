---
description: Take an application from source to a running pod for the first time — get a container runtime in place, create the pod, build and push the image, compose resources, promote the draft, and verify it is actually serving.
metadata:
  required_access:
    - CODEBASE
    - BRAINPOD_CLI
    - DOCKER
---

# Deploying to BrainPod

Clear both onboarding gates in `SKILL.md` first — a working binary and an
authenticated session.

## Step 1: Confirm a container runtime

Do this before anything else. Every build needs one — Railpack replaces the
Dockerfile, not the runtime — so an install, where one proves necessary, runs
while you work through Step 2. Finding the gap after Step 3 orphans a pod.

`image build` execs the literal `docker` binary and drives `docker buildx`.
Two independent things must therefore hold, and they fail differently:

```bash
docker buildx version   # a `docker` client on PATH, with the buildx plugin
docker info             # an engine actually answering behind it
```

No shell is involved, so `docker` itself must resolve on `PATH` — an alias
will not do — and `docker info` prints its client block before failing, so
read the server section rather than the opening lines. If both succeed,
**install nothing over it**: any Docker-compatible engine serves.

**An engine installed but stopped is the common case.** Look for `colima`,
OrbStack, Docker Desktop or Rancher Desktop and start what you find
(`colima start`, or launch the app). **Never install a second engine beside
one already present**; two daemons contending for the same socket is worse
than the problem you set out to fix.

### Installing what is genuinely missing

Name what is missing and what it costs, then **ask whether they install it or
you do**. Neither a persistent Linux VM holding real disk and memory nor a
licensed desktop app is something a user should meet after the fact. Call out
an admin password up front — you cannot supply one, so those routes are
theirs by definition.

| Host | Route | Who runs it |
|---|---|---|
| macOS | `brew install docker docker-buildx colima && colima start` | You, where Homebrew is present — no admin password |
| Linux | the distro's Docker Engine and Buildx packages | The user — needs root |
| Windows (WSL2) | Docker Desktop on the WSL2 backend, integration enabled for the distro running `brainpod` | The user — admin rights, and the licence is not free for larger companies |

The macOS route is three pieces deliberately: `colima` is the engine, `docker`
the client the CLI execs, `docker-buildx` the plugin — and Homebrew does not
wire that plugin in, so follow the formula's caveat to add its
`cliPluginsExtraDirs` path to `~/.docker/config.json`, then confirm with
`docker buildx version`. `brainpod` has no Windows build, so a Windows user is
already working inside WSL2 and the only question is whether that distro
reaches an engine.

If they would rather install nothing, be straight: a local runtime is the only
route today. A hosted build pipeline is planned and not yet available — promise
no timing, and do not go hunting for a flag that exposes it.

## Step 2: Work out what the project needs

The user may not know, and should not have to. Decide from the source, then
state the shape you settled on in one or two plain sentences before building —
a wrong assumption is cheap to correct now and expensive after a failed deploy.

Determine from the source itself:

- **How it builds.** A `Dockerfile` at the context root means the Dockerfile
  path; otherwise Railpack detects the stack. Only the former obliges you to
  add a non-root `USER` (Step 4).
- **What port it listens on.** Read it from the app's own configuration rather
  than assuming, and check whether the framework expects to be told via a
  `PORT` environment variable. It must be ≥ 1024.
- **Whether it needs a database.** ORM configuration, migration directories,
  and connection-string environment variables are the signal. Every database
  kind also requires a `Disk`.
- **Whether it writes to disk at runtime.** Uploads, SQLite files, caches, and
  scratch directories each mean a `Disk` and a mount.
- **What it needs configured at boot.** Environment variables with no default,
  and any config file the app reads, become `App.spec.env` and `Config`.

Every reachable app also needs a `Route`. A stateless web service is therefore
`App` + `Route`; add `Disk` plus a database kind when it stores data, and
`Config` when it reads config files.

Check `brainpod blueprint list --json` before hand-composing this set — a
blueprint matching the stack supplies the whole resource graph already wired
together. `operate.md` covers installing one.

## Step 3: Create or select the pod

Pod creation is available to the agent — no dashboard step is needed.

**Never retry a pod creation.** There is no idempotency key on this
operation and pod names are server-generated, so a retry after a timeout
or ambiguous failure silently creates a second pod. If a create fails
ambiguously, list pods and reconcile before doing anything else. Report the
ambiguity to the user rather than guessing.

Capture the returned pod name from the response. It is required for every
subsequent pod-scoped command, and the pod's private registry namespace is
derived from it — which means **pod creation must precede the image push**. If
the user pointed you at an existing pod, use it, and treat Step 5's draft
check as mandatory rather than a formality.

## Step 4: Build and push the image

`brainpod image build` detects the stack and pushes to the pod's namespace
under `registry.brainpod.io`. Docker login is not required; the CLI
authenticates with the API token, which must allow `registry:push` for the
pod. Let it choose the platform too unless the user has a specific
requirement — it resolves one from the active clusters and validates an
explicit `--platform` against them. Do not duplicate that selection logic or
infer support from the build result. On ARM hosts, the runtime must provide
emulation when targeting amd64.

**Containers must not run as root, and the image must match the cluster's
architecture.** Both are enforced when the resource is created, so a bad image
fails at dry-run with a validation error rather than deploying and
crash-looping:

- **Railpack builds** (no Dockerfile): the builder adds a final layer running
  as `railpack` with uid/gid 1000. Nothing to do.
- **Dockerfile builds**: the image is used as-is and the Dockerfile's runtime
  user is preserved, so it **must** declare a non-root `USER`. Prefer a
  numeric uid over a username, since a name only resolves against
  `/etc/passwd` inside the image. If a Dockerfile has no `USER` directive,
  add one before building.

Design around the runtime uid either way. Binding a port below 1024 fails at
runtime with an error that looks nothing like a permissions problem, and any
path the app writes must be writable by that uid — if the build creates it as
root, `chown` it in the build, to 1000 on the Railpack path or to whatever uid
the Dockerfile declares. Common offenders: framework build caches, SQLite
files, upload and scratch directories.

**Pin the image by digest.** The build response returns a digest-pinned
`reference` along with `platform` and the resolved `user` — use that
reference directly as `spec.image` rather than re-deriving it. A follow-up
`image inspect` is worth one call to cross-check `exposedPorts` against the
Route's port, but is not needed to obtain the digest.

**Parse stdout normally.** Build progress goes to stderr while `--json` emits
one final JSON document on stdout. Parse stdout as a single value; do not
merge the two streams or scrape the last brace from combined output.

## Step 5: Compose resources — validate before mutating

The draft is **shared mutable state**. A human editing the same pod in the
dashboard, or a previous failed run, can leave content there that you did not
put in — and `deploy` promotes all of it.

Work in this order:

1. **Build one JSON batch.** `resource create --file` accepts either one JSON
   resource or an array; the CLI normalizes one object to an array. Put every
   related new resource — for example the App and Route — in one array file.
   `--file -` reads the JSON from stdin.
2. **Dry-run the whole batch first.** Run `resource create` on that file with
   `--dry-run --json`. A valid batch returns `{ "valid": true }` and creates
   nothing. A `VALIDATION_ERROR` carries `details[]` entries with a `path` and
   `message`, so correct those paths and re-validate rather than guessing.
3. **Then mutate once** by sending the same file without `--dry-run`. Capture
   `revisionId` from the mutation response.
4. **Inspect the draft with the CLI's diff command.** Run `revision diff`
   with the captured revision id and `--json`; with no `--base`, it compares
   against the revision's parent. Account for every `entries[]` item:
   `{ kind, name, changeType, patch }`, where `changeType` is `create`,
   `update`, or `delete`. If the diff contains anything unintended, **stop
   and report**.
5. **Re-check the current head and diff immediately before deploy.** Run
   `pod get <pod> --json`; it must still show the expected `head.id` and
   `head.status: draft`. Pod revision state is named `status` here. Re-run
   `revision diff <head.id> --json` and abort if it no longer matches. Do not
   use `RevisionDetail.checksum` as the sole guard: it is nullable. If you
   surface the checksum at all, handle `null` explicitly.

### Resource kinds

Discover the vocabulary and read each kind's schema before composing the
batch. Namespace is always `default`. Contract details worth pinning:

- **Omit `spec.hostname` on `Route`.** `hostname` is required on the returned
  `Resource` but not on `ResourceInput`: the platform assigns it during
  creation. Report that returned value rather than inventing one.
- Cross-resource references are URNs — an App's `mounts[].disk`, a database's
  `diskRef`, and a Route's `rules[].backendRef` take
  `urn:brain:<kind>:default:<name>`. Because names are in the batch, compose
  those references before sending it.
- `App.spec.env` is required even when empty.
- Instance-size enums differ per kind, so read each kind's own rather than
  reusing the App value, and take a database's `version` enum from its schema
  rather than assuming a supported release.

### Runtime values and disk permissions

`App.spec.env` is a flat array of `{ name, value }`, and every `value` must be
a non-empty string. The resource API exposes no exports or interpolation
contract, so do not generate `${resource.field}` references or claim that
created resources publish credentials. Only use substitution syntax when a
selected blueprint's own documentation explicitly defines it; otherwise use
concrete values supplied through a documented input.

For a mounted disk that is not writable by the runtime user, prefer
`App.spec.runtime.fsGroup`; `runtime` also provides `uid` and `gid`. A chown
init step is only the workaround when `fsGroup` cannot solve it. Its actual
field is `App.spec.lifecycle.init`, and it accepts a command string,
`{ cmd: string }`, or `{ image: string, cmd: string[] }`.

## Step 6: Deploy and wait

Use the CLI's health-gated wait instead of polling by hand:

```bash
brainpod --pod <pod> deploy --summary <summary> --wait --timeout 600 --json
```

The default timeout is 90 seconds, and `--timeout` requires `--wait`. On
success this returns the complete revision after every resource reports
`healthy: true`; `failed` and `canceled` stop the wait immediately. To
reattach without deploying again, run `revision wait <revisionId> --timeout
600 --json`. Never redeploy to poke a timeout or an unhealthy revision — see
`operate.md` for what `redeploy` actually does.

## Step 7: Verify and report

Report success only when the returned revision has `state: deployed` **and**
every `resources[]` item has `healthy: true`. The wait returns as soon as
health is satisfied and only aborts on `failed` or `canceled`, so it does not
by itself confirm the final revision state. Verify both.

Do not require `status.phase: Ready` for every kind. `status` is
kind-specific and may be null: a workload reports replica status, a `Disk`
reports `phase`/`bound`/`ready`, and others may expose only `ready`.

**Lead the success report with the live URL**, built from the `hostname` the
platform assigned to the Route — it is the one thing the user actually asked
for. Follow it with the pod name, the revision id and version, and both
readiness facts above, stating that this was the readiness definition used.

If a resource remains unhealthy, go to `debug.md` rather than redeploying, and
never hand over a URL you have not confirmed is serving — a partial deployment
reported as success is worse than an honest failure.
