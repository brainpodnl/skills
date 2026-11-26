---
description: Build an image on this machine and deploy it to a new or existing pod when the project cannot go through GitHub — not recommended, and nothing redeploys it automatically.
metadata:
  required_access:
    - CODEBASE
    - BRAINPOD_CLI
    - DOCKER
---

# Building and deploying from this machine

Clear both gates in `SKILL.md` first. Before starting, tell the user what
this path costs: the image is built here, every later change has to be
rebuilt and redeployed here by hand, nothing deploys on a merge, and local
toolchains are where builds go wrong, on Windows especially. On this path you
also decide the pod's shape yourself, which on GitHub is onboarding's job.

## Step 1: Confirm a container runtime

`image build` execs the literal `docker` binary, with no shell in between, so
an alias does not count, and drives `docker buildx`. Check both before
anything else; finding the gap after creating a pod orphans it.

```bash
docker buildx version   # a docker client on PATH, with the buildx plugin
docker info             # an engine answering behind it
```

`docker info` prints its client block before failing, so read the server
section. **An engine installed but stopped is the common case**: look for
Colima, OrbStack, Docker Desktop, or Rancher Desktop and start it. **Never
install a second engine beside one that exists.** Where nothing is
installed, name what is missing and what it costs, and ask whether the user
installs it or you do:

| Host | Route | Who runs it |
|---|---|---|
| macOS | `brew install docker docker-buildx colima && colima start` | You, where Homebrew exists; add the formula's `cliPluginsExtraDirs` to `~/.docker/config.json` as its caveat says |
| Linux | the distribution's Docker Engine and Buildx packages | The user: needs root |
| Windows | Docker Desktop | The user: admin rights, and the licence is not free for larger companies |

Railpack does not run on Windows, so a Windows build needs a Dockerfile or
WSL2.

## Step 2: Work out what the project needs

`image build` uses a `Dockerfile` at the context root when there is one and
otherwise Railpack, which detects the stack itself. Do not hand-write a
Dockerfile for a project Railpack handles. Steer Railpack with
`railpack.json` at the context root; the CLI passes no builder flags, so
`RAILPACK_*` variables do nothing. If Railpack cannot plan the build at all,
nothing is pushed, and the fix is a `railpack.json` or a Dockerfile, not a
retry. `brainpod detect --with-env .` reports the projects, frameworks, and
declared variables without running anything.

Settle the rest from the source and state it in one or two sentences before
building:

- **The port.** Read it from the app's configuration. Railpack's static, SPA,
  and PHP outputs default to 80, which the runtime uid cannot bind: set `PORT`
  ≥ 1024 in `App.spec.env` and give the Route that same number.
- **A database**, if ORM config, migrations, or connection-string variables
  say so. Every database kind needs a `Disk`.
- **Disk writes** at runtime — uploads, SQLite files, caches — each mean a
  `Disk` and a mount.
- **Boot configuration**: variables with no default become `App.spec.env`,
  secrets become Secret resources (`operate.md`), and config files a `Config`.

Every reachable app also needs a `Route`. Check `brainpod blueprint list
--json` first; a matching blueprint supplies the whole graph, and
`operate.md` covers pinning its App to a local image.

## Step 3: Create or select the pod

Create the pod as `SKILL.md` describes. Its registry namespace comes from the
returned pod name, so the pod must exist before the push. Open the pod page
by the rules in `SKILL.md`. For an existing pod, inspect its draft first as
`operate.md` describes.

## Step 4: Build and push the image

**Write a `.dockerignore` for this project first.** The working tree is
copied into the image *over* what the build installed, so a local
`node_modules` or `.venv` replaces the one built for the cluster and fails on
native binaries for the wrong platform, and `.env` files, keys, and dumps
ship too. `.gitignore` is not consulted.

```bash
brainpod --pod <pod> image build <image> . --json
```

The push needs `registry:push` on the pod; a `FORBIDDEN` here is the
read-only sign-in case in `SKILL.md`. Let the CLI choose the platform; it
resolves one from the live clusters. On an ARM host targeting amd64, the
engine must provide emulation. Progress goes to stderr and one JSON document
to stdout: parse stdout alone. Check what Railpack reports it detected
against the project; a wrong provider builds cleanly and fails at runtime.

**Containers must not run as root.** A Railpack build adds a final layer
running as uid 1000. A Dockerfile build keeps the Dockerfile's user, so it
must declare a non-root numeric `USER`. On either path, any path the app
writes must be writable by that uid: point runtime writes at a mounted
`Disk`.

Use the digest-pinned `reference` from the build response as `spec.image`.

## Step 5: Compose and deploy

Compose resources and deploy exactly as `operate.md` describes: dry-run one
batch, create it, diff the draft, re-check the head, and deploy with `--wait`.
Leave `spec.artifactSelector` out of the App: it ties the App to GitHub
builds.

Every later code change repeats Step 4 and a `resource replace` of the App's
`spec.image`, then a deploy.

## Step 6: Verify and report

Verify readiness as `SKILL.md` defines it, and lead with the live URL from
the Route's assigned `hostname`, confirmed to answer. Then the pod name, the
revision, the readiness evidence, and that changes do not deploy themselves.
Moving to GitHub later means connecting the App to a repository in the
dashboard, or onboarding the project as a new pod through `onboard.md`.
