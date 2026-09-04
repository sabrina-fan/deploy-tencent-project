# deploy-tencent-project

Deploy and test completed local projects on a remote cloud host using Git push, SSH, Docker Compose, health checks, SSH tunnels, and browser/API tests. Each project is fully isolated — its own deployment directory, Compose project name, ports, containers, networks, and volumes.

## Why

Deploying multiple projects to one server is error-prone without strict isolation. This skill enforces per-project isolation, a clean-build gate, provenance verification (local SHA = bare ref = checkout SHA = running image), and a three-layer test strategy (local deep → tunnel medium → public light) so deployments are repeatable and verifiable.

## Install

### Option A — let your agent install it

Give your agent this repo URL and ask it to add the skill:

```
https://github.com/sabrina-fan/deploy-tencent-project
```

### Option B — manual

Copy the `deploy-tencent-project/` directory into your agent's skills folder.

## Configuration

- **SSH alias**: set up an SSH alias (e.g. `tencent-dev`) in `~/.ssh/config` pointing to your server. Never hard-code the IP.
- **SSH user home**: bare repos stored at `~/git/<project>.git`, deployments at `~/projects/<project>` under the SSH user's home.
- **Target platform**: `linux/amd64` by default.
- **Project config**: add a deployment section to the project's `AGENTS.md` for project-specific ports, compose files, health paths, etc. See [references/project-config.md](references/project-config.md) for the template.
- **No secrets in Git**: `.env` files, tokens, passwords, and database snapshots stay outside Git by default (mode `0600`), unless an explicit self-contained private deployment is authorized.

## Usage

Trigger it when you need to deploy, redeploy, inspect, or test a project on the server. The skill will:

1. **Preflight** — check Git status, SSH connectivity, remote resources, port conflicts.
2. **Resolve config** — detect project name, branch, compose files, ports from `package.json` and `AGENTS.md`.
3. **Initialize or update** — create bare repo and checkout on first deploy; fast-forward update on subsequent deploys.
4. **Build and start** — server-side Docker Compose build with isolated project name and ports.
5. **Prove the revision** — verify SHA chain: local HEAD → bare ref → checkout → running image.
6. **Test** — three-layer validation: local deep, tunnel medium, public light.
7. **Report** — deployed SHA, database strategy, port mappings, test results, tunnel commands.

## Compatibility

- **macOS** — primary development platform (uses SSH, Docker, local browser testing).
- **Linux** — fully supported as both development and deployment platform.
- **Windows** — works with WSL2 or native Docker Desktop; adapt SSH tunnel commands if needed.

## Security & Boundary

This skill deploys and tests; it does not write source code. It never prints secrets, tokens, passwords, or `.env` contents. Database migrations require explicit approval. Destructive operations (volume deletion, database drops, Git force-push) require separate explicit authorization. SSH credentials are never requested or handled in chat.
