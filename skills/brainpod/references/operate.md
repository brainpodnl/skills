---
description: Change what an existing pod runs — inspect current state, add or modify resources, scale, attach a database, install blueprints, and revert a bad deploy.
metadata:
  required_access:
    - BRAINPOD_CLI
---

# Operating an existing pod

Clear both onboarding gates in `SKILL.md` first. Everything here mutates the
pod's **draft**, which `deploy` then promotes wholesale — so the discipline is
always: inspect what is there, change one thing, diff, deploy.

## Inspect before changing

Never compose a change from assumption about what is running.

```bash
brainpod --pod <pod> pod get <pod> --json
brainpod --pod <pod> resource list --json
brainpod --pod <pod> resource get <kind> <name> --json
```

`pod get` reports `head` (the draft you are about to modify) and `deployed`
(what is actually serving), each with a `status`. If `head.id` already differs
from `deployed.id`, **someone else's uncommitted work is sitting in the
draft** — stop and report it before adding yours, because your deploy would
promote it too.

Both `resource list` and `resource get` accept `--revision <uuid>` or
`--at <timestamp>` to read a historical revision instead of the current one.
The two flags conflict; pass at most one. `revision list --json` enumerates
revisions (default limit 10, maximum 50, `--cursor` to page).

## Changing a resource

| Intent | Command | Dry-run |
|---|---|---|
| Add new resources | `resource create --file <path\|->` | Yes — `--dry-run` |
| Modify an existing resource | `resource replace <kind> <name> --file <path\|->` | **No** |
| Remove a resource | `resource delete <kind> <name>` | **No** |

`resource create` takes one JSON object or an array and is the only mutation
with a dry-run. Use it, always, before creating.

`resource replace` takes exactly one JSON object and **replaces the whole
resource** — it is not a patch. Read the current resource with `resource get`,
apply your change to that document, and send the result back. Composing a
replace body from scratch silently drops every field you forgot.

Because replace and delete cannot be dry-run, the safety net moves either side
of the call:

1. Read `brainpod describe resource <kind> --json` for the authoritative
   schema before composing the body.
2. Immediately after the mutation, capture `revisionId` and run
   `revision diff <revisionId> --json`. Account for every `entries[]` item
   (`{ kind, name, changeType, patch }`). If anything unintended appears,
   stop and report rather than deploying.

Common changes and where they live — confirm each against
`describe resource <kind> --json` rather than trusting this list:

- **Environment variables** — `App.spec.env`, a flat array of
  `{ name, value }`; every `value` must be a non-empty string.
- **Scale** — `App.spec.replicas`, from 1 through 10.
- **Instance size** — `App.spec.instance`. App sizes include `.25x`; database
  instance enums do not, so read each kind's own enum.
- **New image** — build and push first (see `deploy.md`), then replace
  `App.spec.image` with the digest-pinned `reference` from the build response.
- **Attaching a database** — run `brainpod describe resource` to see which
  database kinds this CLI version offers. Every one requires a `diskRef`, so
  create the `Disk` and the database in one `resource create` batch,
  referencing the disk as `urn:brain:disk:default:<name>`.

## Installing a blueprint

```bash
brainpod blueprint list --json
brainpod blueprint get <blueprint> --json
brainpod --pod <pod> blueprint install <blueprint> --file <input.json>
```

`blueprint get` returns documentation, defaults, and the input schema — read it
and fill inputs from that schema rather than from assumption. Omit `--file` to
install with defaults; `--file -` reads JSON from stdin. Install changes the
pod's draft and does **not** deploy, so the usual diff-then-deploy applies.

Blueprints often target automatically deployed applications and may define
`spec.artifactSelector.status: pending` on the generated App. For a manually
built and pushed image, install the blueprint first, then replace that
resource to remove `spec.artifactSelector` and set its static digest-pinned
`spec.image`. This is preferred to trying to rewrite the blueprint's App
manifest before installation.

## Deploying the change

```bash
brainpod --pod <pod> deploy --summary <summary> --wait --timeout 600 --json
```

Re-check `pod get` immediately before deploying: `head.id` must still be the
revision you diffed, and `head.status` must still be `draft`. Success requires
both `state: deployed` and every `resources[]` item `healthy: true`.

## Reverting a bad deploy

**There is no rollback command.** `redeploy` is not one — per the CLI's own
contract it redeploys the *currently deployed* revision, which is useful for
restarting workloads or retrying a transient infrastructure failure, but it
re-applies the same content and will not undo anything. It also has no
`--wait`, so capture its `revisionId` and follow with `revision wait`.

To actually revert, roll **forward** to the previous content:

1. Find the last good revision with `revision list --json`.
2. Read its resources with `resource list --revision <goodRevisionId> --json`.
3. Write those documents back onto the current draft with `resource replace`
   (one call per changed resource), deleting anything the good revision did
   not contain.
4. `revision diff` the draft and confirm it matches the good revision's
   content.
5. Deploy and wait as above.

Confirm with the user before reverting a resource whose content carries state
implications — a database or disk in particular, where "the previous manifest"
and "the previous data" are not the same thing.

## Reporting

Report the pod, the revision id and version, which resources changed and how
(`changeType` per entry), and the readiness evidence: `state: deployed` plus
every resource `healthy: true`. If you stopped after mutating the draft, say
so and name the pod — the next deploy on it would promote your partial work.
