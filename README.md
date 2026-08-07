# BrainPod Skills

[Agent Skills](https://github.com/anthropics/skills) that teach AI coding assistants (Claude Code, Cursor, Codex) how to deploy, operate, and debug applications on [BrainPod](https://brainpod.io) using the `brainpod` CLI.

## Skills

| Skill                             | Description                                                                                                                                                 |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [brainpod](./skills/brainpod)     | Main skill for working with BrainPod. Create pods, build and push container images, compose resources, promote draft revisions, change what a running pod runs, and diagnose unhealthy deployments. |

The skill routes to use-case references it loads on demand:

- [`references/deploy.md`](./skills/brainpod/references/deploy.md) — take an application from source to a running pod for the first time.
- [`references/operate.md`](./skills/brainpod/references/operate.md) — change what an existing pod runs: resources, env, scale, databases, blueprints, and reverting a bad deploy.
- [`references/debug.md`](./skills/brainpod/references/debug.md) — diagnose a deployment that failed, timed out, is unhealthy, or is not serving traffic.

## Installation

Ask your agent to install it, or run one of the following yourself.

### Cursor

Install as a [Cursor plugin](https://cursor.com/docs/plugins):

```bash
/add-plugin brainpod
```

### skills CLI

Install via the [skills CLI](https://github.com/anthropics/skills):

```bash
npx skills add brainpodnl/skills --skill "brainpod"
```

### Manual symlink

Clone this repo and symlink the skill into your agent's skills directory:

```bash
git clone https://github.com/brainpodnl/skills.git /path/to/brainpod-skills
```

```bash
ln -s /path/to/brainpod-skills/skills/brainpod /path/to/skills-directory/brainpod
```

## Prerequisites

You don't need to set anything up before you start. Ask the agent to deploy something to BrainPod and it handles onboarding for you — installing the CLI if it is missing, and signing you in.

For reference, what it sets up on your behalf:

**The `brainpod` CLI**, from [brainpodnl/cli](https://github.com/brainpodnl/cli). Linux and macOS on amd64 and arm64.

**A signed-in session.** The agent runs `brainpod login` and opens the sign-in page for you — in an embedded browser pane where its harness has one, otherwise in your default browser. You approve there and the CLI stores the token itself, so there is no token to copy and paste. Starting the flow is the agent's part; approving it is yours.

On a headless machine, in a container, or in CI, the browser flow isn't available, so create an API token in the [BrainPod dashboard](https://brainpod.io) and set it instead:

```bash
export BRAINPOD_API_TOKEN=brain_...
```

The CLI also accepts `--api-token` and `brainpod config set api-token <token>`. Configuration lives at `~/.config/brainpod/config.toml`.

**Docker with Buildx**, required only for image builds.

## Usage

Once installed, the agent uses the skill automatically when relevant — for example:

- Deploying a repository to BrainPod from scratch
- Adding a Postgres, MariaDB, or Valkey database to an existing pod
- Changing environment variables, replica counts, or instance sizes
- Installing a blueprint and pinning it to a manually built image
- Reverting a bad deploy
- Diagnosing a deployment that reports as deployed but is not responding

## Contributing

Authoring and review guidance lives in [AGENTS.md](./AGENTS.md). It is the source of truth for how skill content in this repo is written and reviewed.
