---
description: Change what an existing pod runs — ship a code change through a pull request, add or modify resources, set secrets, scale, attach a database, add a service, install blueprints, and roll back a bad deploy.
metadata:
  required_access:
    - CODEBASE
    - BRAINPOD_CLI
    - GITHUB
---

# Operating an existing pod

Clear both gates in `SKILL.md` first. A pod changes in two ways, and they
must not be confused:

- **Code** ships by merging into the base branch, which builds in GitHub
  Actions and deploys on its own. Your part ends at a pull request.
- **Pod** changes — resources, env, secrets, scale — go into the draft, and
  `deploy` promotes them.

## Inspect before changing

```bash
brainpod --pod <pod> pod get <pod> --json
brainpod --pod <pod> resource list --json
brainpod --pod <pod> resource get <kind> <name> --json
```

`pod get` reports `head` (the draft you would modify) and `deployed` (what
serves). If `head.id` differs from `deployed.id`, **someone's undeployed work
is in the draft** — stop and report it before adding yours, because your
deploy would promote it too. `resource list` and `resource get` take
`--revision` or `--at` to read history. Where `resource get` rejects a kind
that `describe resource` lists, read it with
`brainpod --pod <pod> api /v1/pods/{pod}/resources/<Kind>/default/<name>`.

**Check how the Apps get their images.** An App whose
`spec.artifactSelector.status` is `configured` deploys from GitHub. An App
without one was built on a machine, as every pod from before GitHub builds
was: a merge deploys nothing for it. Ship its code changes by
`local-build.md` Step 4 and a `resource replace` of its `spec.image`, and tell
the user that moving to GitHub means connecting the App to a repository in
the dashboard or onboarding the project as a new pod.

## Shipping a code change

1. Branch from the base branch, make the change, and verify it locally the way
   the project verifies itself: its tests, its build.
2. Push the branch and open a pull request with `gh pr create`. Hand the user
   the link with what changed and why, and say plainly that merging deploys
   to production.
3. Once they have merged, follow the build of the merge commit:

   ```bash
   brainpod --pod <pod> api /v1/pods/{pod}/builds -q rev=<merge-sha>
   ```

   A `ready` build deploys by itself; confirm with `revision list --json` that
   a new revision followed, then `revision wait <id> --json`. A `failed` one is
   `debug.md`'s.

**Merges that build nothing**, on a pod whose Apps are `configured`, mean the
base branch has no `deploy-to-brainpod-<pod>.yml`. Either the onboarding pull
request that adds it is still open — point the user at it — or the repository
predates the build action and its workflow uses `brainhivenl/steiger-action`
with a `BRAINPOD_API_KEY` secret. Propose migrating that in a pull request,
taking the workflow shape from the
[build action's README](https://github.com/brainpodnl/actions/tree/main/build)
at the time. The action's sign-in only works for the repository linked to the
pod: if the migrated build fails there, linking it is done in the dashboard,
not in the repository. Leave deleting the old secret to the user once a build
succeeds.

## Adding a service

A new deployable in the repository needs three things, in this order:

1. A pull request adding its entry to `steiger.yml`; the entry's key is the
   artifact name. Read steiger's documentation for the entry's shape.
2. After the merge, its first build. Nothing deploys it yet, because no App
   asks for that artifact; take the digest-pinned `uri` from the build's
   `artifacts[]`.
3. An App with that `uri` as `spec.image` and an `artifactSelector` that is
   `configured` with the artifact's `name`, `githubRepo` as `owner/repo`, and
   `refName` as `refs/heads/<base>`; builds match those strings exactly.
   Create it through the draft below and deploy; later merges then deploy it
   on their own.

## Changing resources

| Intent | Command | Dry-run |
|---|---|---|
| Add new resources | `resource create --file <path\|->` | Yes — `--dry-run` |
| Modify an existing resource | `resource replace <kind> <name> --file <path\|->` | **No** |
| Remove a resource | `resource delete <kind> <name>` | **No** |

`resource create` takes one object or an array: put related resources in one
batch, dry-run it with `--dry-run --json`, then send it once. `resource
replace` **replaces the whole resource**: read it with `resource get`, change
that document, and send it back. A body composed from scratch silently drops
every field you forgot. Read `describe resource <kind> --json` before
composing any body.

After every mutation, capture `revisionId` and run `revision diff
<revisionId> --json`. Account for every `entries[]` item; if anything is
unintended, stop and report rather than deploying.

### Contract details no schema states

- Cross-resource references are URNs with a **lowercase** kind,
  `urn:brain:disk:default:pg-disk`, while a manifest's own `kind` is
  PascalCase.
- **Omit `spec.hostname` on a `Route`**; the platform assigns it.
- `App.spec.env` is required even when empty, and every `value` is a
  non-empty string. Reach another resource's outputs by reference, such as
  `${postgres.uri}`, never by literal; `brainpod resource variables` lists
  what each resource exports. `--dry-run` does not check that a reference
  resolves: a wrong name fails at runtime.
- Instance size gates replicas: `.5x` allows only one, with the error on
  `replicas` rather than `limits.`. Read each kind's own instance and version
  enums.
- Every database kind needs a `Disk`; create both in one batch.
- For a mounted disk the runtime user cannot write, prefer
  `App.spec.runtime.fsGroup` over an init step.

## Secrets

An App reads a secret through `${secret.<name>}` in `spec.env`. A value only
the user knows, they enter in the dashboard. A value any random string
satisfies, BrainPod generates without it ever reaching you:

```bash
brainpod --pod <pod> api -X POST /v1/pods/{pod}/secrets --input secret.json
```

with `{ "name": "<name>", "generate": { "length": 48, "charset": "alphanumeric" } }`.
Secret values are write-only, and a new secret lands in the draft like any
other change. A referenced secret with no value fails the deploy.

## Installing a blueprint

```bash
brainpod blueprint list --json
brainpod blueprint get <blueprint> --json
brainpod --pod <pod> blueprint install <blueprint> --file <input.json>
```

Fill inputs from the schema `blueprint get` returns. Install changes the
draft and does not deploy. An App it creates with `artifactSelector.status:
pending` waits for a repository: wire it as in *Adding a service*, or, on a
pod built from this machine, replace the App without `artifactSelector` and
with a digest-pinned `spec.image`.

## Deploying the change

```bash
brainpod --pod <pod> deploy --summary <summary> --wait --timeout 600 --json
```

Immediately before, re-check `pod get`: `head.id` must still be the revision
you diffed with `head.status: draft`, and its diff must still match. A merged
build can land in between; it keeps your draft and refreshes its build
references, so the diff then shows only your edits. Then verify readiness as
`SKILL.md` defines it.

## Rolling back

Pick the target from `revision list --json` — any revision that is not a
draft — and show the user what would change before touching anything:

```bash
brainpod --pod <pod> revision diff <target> --base <deployed> --json
brainpod --pod <pod> rollback <target>
```

`rollback` copies the target into a new draft head, **replacing any
undeployed draft**, so check the head first and never discard someone's work
without asking. Deploy the restored draft as above, or pass `--deploy --wait`
to `rollback` once the user has seen the diff. `redeploy` is not a rollback:
it only retries a head whose deployment failed.

Two things a rollback does not undo. **Data**: a database or disk keeps what
was written since, so confirm with the user before rolling back across a
migration. **The next merge**: it deploys over the rollback, so revert the
offending commit in a pull request as well.

## Reporting

Report the pod, the revision id and version, which resources changed and how
(`changeType` per entry), the readiness evidence, and every pull request left
open for the user.
