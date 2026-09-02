---
name: deploy-tencent-project
description: Deploy and test completed local projects on the Tencent Cloud host `tencent-dev` using Git, SSH, Docker Compose, health checks, SSH tunnels, and browser/API tests. Use when the user asks to publish, deploy, redeploy, update, restart, inspect, or test a current project on the Tencent server, including first-time setup and isolated multi-project deployments.
---

# Deploy Tencent Project

Deploy only the current Git project. Keep every project isolated by repository, deployment directory, Compose project name, ports, containers, networks, and volumes.

## Fixed Environment

- Use SSH alias `tencent-dev` as user `ubuntu`; never hard-code its IP.
- Target `linux/amd64`.
- Store the bare repository at `/home/ubuntu/git/<project>.git`.
- Store the deployment checkout at `/home/ubuntu/projects/<project>`.
- Treat the local repository as the source of truth. Never edit deployed source directly.
- Apply local RTK rules locally; do not assume `rtk` exists remotely.

## Isolate Concurrent Sessions

- Assume another Codex task may be using the visible checkout. Inspect `git worktree list` before switching branches.
- When branch switching could disturb another task, create a dedicated deployment worktree and run every local Git, test, and deploy command from it.
- Never switch, reset, clean, or remove another task's worktree. Keep the deployment worktree until deployment and verification finish.
- Remember that worktrees isolate working directories and checked-out HEADs, but still share repository configuration, remotes, refs, branch upstreams, and the default stash. Treat changes to those shared resources as cross-session mutations.
- Before pushing, record the selected branch's existing `branch.<name>.remote` and `branch.<name>.merge` values. Never use `git push -u` or `--set-upstream` on an existing or shared branch unless the user explicitly asks to change its upstream.
- Push with an explicit destination ref, such as `git push tencent HEAD:refs/heads/<branch>`. Afterward, verify the recorded upstream is unchanged and the exact `tencent` ref resolves to the intended SHA.

## Resolve Project Configuration

1. Find the Git root. Resolve `<project>` from, in order: explicit deployment configuration, the Compose top-level `name`, the Git remote repository basename, then the local root basename.
2. Require `<project>` to match `^[A-Za-z0-9._-]+$`. Do not reject a valid repository merely because its local parent or worktree path contains Chinese characters or spaces. If credible sources disagree, inspect their ownership and ask only when still ambiguous.
3. Read repository instructions, README, manifests, Dockerfile, Compose files, and deployment configuration.
4. Detect the real deployment branch instead of assuming `main`.
5. Read [references/project-config.md](references/project-config.md) when project-specific ports, commands, Compose files, health paths, registry prefixes, migration targets, or backup commands are missing.
6. Infer safe values when unambiguous. Ask before proceeding when a missing value affects secrets, data, public routing, or another project.

## Respect Authorization

- Treat an explicit request to deploy, publish, update, start, stop, or restart as authorization to modify only the current project.
- For inspection or testing requests, perform only read-only operations unless deployment is also requested.
- Never commit or push private keys or Codex credentials. Never print secret values.
- Keep `.env`, tokens, passwords, database snapshots, account state, and other private runtime files outside Git by default. Exception: when the user explicitly requests a self-contained deployment bundle and the destination is a user-authorized private repository whose visibility and exact remote URL have been verified, include the runtime files required for direct deployment. Never send that private-state commit to a public, unverified, or differently owned remote.
- A request to deploy to Tencent authorizes pushing the selected branch only to the verified `tencent` remote. Do not push `origin`, GitLab, GitHub, or any other remote unless the user explicitly requests that destination.
- A deployment may transfer an already-selected project environment file when its source and destination are unambiguous. On first deployment, ask if the secret source is unclear; copy it outside Git, set mode `0600`, and validate with `docker compose ... config --quiet`.
- Never change SSH, firewall, DNS, APT, global Docker settings, or another project without explicit approval.
- Never run destructive Git commands, Docker prune, delete volumes, delete databases, or run migrations without explicit approval.

## Build a Self-Contained Private Deployment Bundle

Use this workflow when the user explicitly wants operations staff or Jenkins to clone a private Git repository and deploy without separately transferring environment files, databases, contracts, or other runtime assets.

1. Keep the local repository as the only delivery source of truth. Do not create the GitLab delivery commit from the Tencent deployment checkout, and do not configure Tencent-to-GitLab SSH credentials as part of this workflow.
2. Verify every destination before staging private state:
   - the Tencent bare repository belongs to the current project;
   - the GitLab repository URL and project identity are exact;
   - GitLab visibility is private and access is limited to authorized users or systems.
3. Resolve the latest database source using **Resolve the Database Data Source**. Export a fresh, consistent snapshot from that selected active database, verify it with `pg_restore --list` or the project's restore check, and replace superseded deployable snapshots rather than accumulating ambiguous alternatives.
4. Include only the runtime state required to reproduce the approved deployment, which may include the production env file, verified database snapshot, account data, migrations, contract/object assets, immutable image references, Compose files, Dockerfiles, and reverse-proxy template. Never include SSH private keys, Codex credentials, host keys, or unrelated server files.
5. Make the repository's default Compose path self-bootstrapping on a fresh volume: apply migrations, restore the approved snapshot or selected tables, initialize required file volumes, start services, and fail before Web startup when restore verification fails. Re-running the same command must be idempotent and must not duplicate or erase existing account data.
6. Parameterize the public address so operations changes only the documented IP value. Keep database credentials, image digests, service topology, migration commands, and bootstrap behavior in the private bundle; do not require manual transfer of another file after clone.
7. Test the exact bundle from a clean clone with a new isolated Compose project and fresh volumes. Run the documented one-command deployment, verify health, migration revision, account counts, representative business counts, file assets, and a second idempotent invocation.
8. Commit locally once. Push that exact local commit with explicit refspecs to the Tencent bare repository and the verified private GitLab branch. Use the same branch name at both destinations unless the user explicitly requests otherwise.
9. Verify equality by SHA and tree, not by branch labels alone: local `HEAD`, Tencent bare ref, Tencent checkout, GitLab ref, and the deployed OCI revision must match. A GitLab push from an identical local tree is valid even when the Tencent host cannot reach GitLab.
10. Remove or rewrite superseded database artifacts from reachable private Git history only when the user explicitly requests database-history cleanup. Re-verify and force-push each authorized private destination deliberately; never broaden the rewrite to unrelated branches.

The intended operator experience after clone is one documented Compose invocation, for example:

```bash
APP_REVISION="$(git rev-parse HEAD)" docker compose \
  -p <project> \
  --env-file <tracked-private-env-file> \
  up -d --build --remove-orphans
```

The operator may first copy the tracked private env file to a mode-`0600` host path when local policy requires it, but must not need another upload or an out-of-band database restore.

## Preflight

1. Check `git status`, branch, remotes, and HEAD SHA.
2. Deploy committed HEAD by default. If relevant changes are uncommitted, stop and ask whether to include and commit them.
3. Run documented formatting, type, unit, integration, and build checks that are practical locally.
4. Verify non-interactive SSH with `ssh -o BatchMode=yes tencent-dev true`. If the server accepts the public key but signing fails, check `ssh-add -l` and `ssh -vv`; ask the user to unlock the encrypted key locally (for example with `ssh-add --apple-use-keychain ...`). Never request or handle the passphrase in chat.
5. Check remote OS, architecture, disk, memory, Git, Docker, Compose, Docker access, existing paths, containers, and listening ports.
6. Check Docker Hub or the project's configured registry connectivity before a first image pull. Also inspect reverse-proxy ownership for ports that appear occupied or publicly routed.
7. Stop on authentication failure, insufficient resources, dirty remote checkout, port conflict, or conflicting Git remote. Do not weaken security or overwrite state.
8. For an existing deployment, record the current checkout SHA, database revision, running container image IDs/digests, and service creation times before any update. Preserve these as evidence and as an application rollback checkpoint; do not treat them as authorization to roll back data.

## Resolve the Database Data Source

- Treat code deployment, schema migration, and data migration as three separate decisions. A successful `alembic upgrade`, migration-to-head, or healthy empty database proves only schema compatibility; it does not prove that accounts or business data were imported.
- Before first startup of a new Compose project, parallel branch deployment, or new database volume, inventory every credible source database and the target read-only. Record Compose/container ownership, PostgreSQL major version, database name, schema revision, database size, user count, and representative business-table counts. Do not infer that a database belongs to a branch merely from its container name; local branches may share one running Compose database.
- Resolve exactly one data strategy before creating or accepting the target database: deliberately fresh, restore the active local project database, copy an existing remote database, or restore a named backup. If the user's intent is not explicit, ask. Never default to a fresh database when the selected local project already has accounts or business rows.
- When the user requests the current or latest project data, treat the explicitly selected active database as the source of truth. Export it through a consistent database snapshot without stopping or mutating the source.
- Block deployment completion when a credible source has real users or business rows but the target contains only a bootstrap account or empty business tables, unless the user explicitly approved a fresh database. Report this as missing data, not as a successful database migration.
- Keep source and target evidence separate in progress updates and the final report. State both schema revision and data provenance so that "latest schema" cannot be mistaken for "latest data."

## Initialize the Project Once

When the remote project does not exist:

1. Create `/home/ubuntu/git/<project>.git` as a bare Git repository.
2. Add local remote `tencent` pointing to `tencent-dev:/home/ubuntu/git/<project>.git`. If `tencent` already points elsewhere, stop.
3. Push with an explicit refspec, for example `git push tencent HEAD:refs/heads/<branch>`, without changing the local branch upstream.
4. Clone that branch into `/home/ubuntu/projects/<project>`.
5. Verify local, bare-repository, and checkout SHAs match.

Do not reuse or replace an existing same-name path unless it clearly belongs to this repository.

## Update an Existing Project

1. Push the selected branch with `git push tencent HEAD:refs/heads/<branch>`; do not change its upstream.
2. Verify the remote checkout is clean and on the intended branch.
3. Update it with fast-forward-only Git operations.
4. Verify the local SHA, bare-repository ref, and checkout SHA are identical before building.
5. Determine which services are affected by the change and record which images must be rebuilt and which containers must be recreated.

## Isolate Docker Workloads

- Use `docker compose -p <project>` or the same unique `COMPOSE_PROJECT_NAME` for every command.
- Avoid fixed `container_name`; if required, include `<project>` in the name.
- Require unique remote host ports for concurrently running projects.
- Bind private development ports to remote `127.0.0.1`, not all interfaces.
- Never stop, rebuild, or remove containers, networks, or volumes belonging to another Compose project.
- Prefer server-side builds so images naturally target `linux/amd64`.
- If building locally for this server, use `docker buildx build --platform linux/amd64 ...`; never deploy an ARM64-only image.

### Freeze the Compose invocation

- Resolve one canonical Compose invocation before the first Compose command. It must include the explicit project name, every Compose file in order, and every `--env-file` in override order.
- Reuse that complete invocation verbatim for `config`, `pull`, `build`, `run`, `exec`, `up`, `ps`, and `logs`. Across separate SSH calls, reconstruct it from the recorded values; never fall back to a shorter plain `docker compose` command.
- Validate that every external Compose/env file exists and is owned by the expected deployment account. Keep secrets outside Git with mode `0600` unless the explicitly authorized self-contained private deployment workflow above applies; in that case, verify the private remote before every push and use a mode-`0600` host copy at runtime when supported. Run the canonical invocation with `config --quiet` before mutation and again after the remote checkout changes.
- Treat env-file ordering as configuration: later files override earlier ones. Record the ordered list without printing secret contents, and include the non-secret paths in the final report.

### Mainland registry fallback

- Keep immutable image digests. Never solve registry failure by dropping `@sha256`, switching to a floating tag, or using an unverified image.
- Prefer repository-supported registry-prefix variables or an environment-specific Compose override outside the Git checkout. Use the same locked digest through the reachable proxy.
- A local re-tag is insufficient for Compose or Dockerfile references written as `repository@sha256`; repository identity is part of resolution. Ensure Compose and every `FROM` resolve through the reachable prefix.
- If a locked package install stalls on a low-bandwidth host, test the exact artifact URL and mirror URL separately. Reduce download concurrency before changing versions or lock files.
- Some frozen lock workflows retain origin artifact URLs even when an alternate index is configured. Prefer the package manager's supported lock export with pinned versions and hashes, then sync that export from the mirror; never rewrite locked URLs or hashes ad hoc.
- After canceling a remote BuildKit build, verify its child processes exited. Terminate only confirmed orphan processes created by the current run before retrying.
- Do not change the global Docker daemon mirror, proxy, DNS, or registry settings without explicit approval. Avoid broad image or build-cache pruning.

Validate with `docker compose ... config`. Then use the project's documented deployment command. If absent:

- With `compose.production.yaml`, use `docker compose -p <project> -f compose.yaml -f compose.production.yaml up -d --build --remove-orphans`.
- Otherwise use `docker compose -p <project> up -d --build --remove-orphans`.

On low-memory hosts, pull and build sequentially and recheck free disk and memory instead of launching parallel builds.

### Prove the running revision

- Build affected images only after the checkout reaches the intended committed SHA. Prefer an OCI `org.opencontainers.image.revision=<sha>` label when the project supports it; otherwise preserve the build ordering plus the resulting image IDs as provenance evidence.
- After any later commit, reassess affected services. Rebuild and recreate every service whose build context, Dockerfile, Compose runtime configuration, or mounted source changed.
- Do not report the final checkout SHA as the running revision when an affected container still uses an image built from an earlier SHA. If a later commit affects only documentation or unused deployment assets, state that classification and why no application image rebuild was required.
- After startup, compare each affected service's running image ID/digest and creation time with the pre-deployment snapshot, and verify the local SHA, `tencent` ref, bare ref, and remote checkout SHA again.

### Object storage browser access

- When the application returns presigned MinIO URLs to a browser, verify the public endpoint is reachable through the chosen public route or SSH tunnel; an internal Compose hostname is not a browser-reachable endpoint.
- Keep the bucket private and configure only the required browser origins. For tunnel-only testing, include the exact `http://localhost:<port>` origin in the deployment configuration.
- Detect the MinIO edition before choosing a CORS mechanism. Community MinIO releases do not support per-bucket `PutBucketCors`; configure their server-wide `MINIO_API_CORS_ALLOW_ORIGIN` value and recreate the MinIO service. AIStor supports per-bucket CORS, for which current `mc cors set` expects S3 CORS XML; its `--json` flag changes command output, not the input document format.
- Use a temporary `mc --config-dir` for bucket operations and verify bucket existence plus anonymous policy `none`. Verify CORS behavior with an `OPTIONS` request containing the real `Origin` and `Access-Control-Request-Method`; do not treat environment inspection as sufficient and do not persist root credentials in the default CLI config.

## Back Up and Migrate

- Require explicit migration approval. Resolve the current revision, target revision, exact upgrade operations, and downgrade behavior before execution.
- For an existing database, create the project's documented backup before migration and verify its checksum/archive plus `pg_restore --list` or the documented restore check.
- On a first deployment, state clearly that no previous database exists. If the user explicitly requests a pre-target backup, initialize the data service, migrate only to the target's predecessor, create and verify a dump, then run the approved target migration.
- For an approved full PostgreSQL data copy, use this order:
  1. Verify source and target PostgreSQL major versions and schema revisions; never use a data restore to perform an implicit schema downgrade.
  2. Back up the target first. Keep it outside the project volume, set mode `0600`, record its SHA-256, and verify the archive with `pg_restore --list`.
  3. Dump the source as a consistent custom-format archive with ownership and privileges excluded when environments use different database roles. Verify its SHA-256 and `pg_restore --list` locally and again after upload.
  4. Stop only target services that can write to the database. Preserve the source database and unrelated Compose projects. Replace or clean the target only after both archives pass validation.
  5. Restore with error-on-first-failure behavior, then verify the schema revision, all-table row-count digest, account credential/state digest, and role-assignment digest against the source without printing usernames, password hashes, or other secret values.
  6. Restart the target through the frozen Compose invocation, confirm the migration service exits successfully, check for active login blocks, and retest API/frontend health.
- Inspect object-storage references separately after a database copy. A PostgreSQL archive does not contain MinIO objects; compare active object keys with source/target object existence and obtain explicit scope before copying or replacing object data.
- Confirm the migration container exits successfully and verify the database revision after migration. Preserve the verified backup path in the final report.
- Never run an automatic database downgrade as part of application rollback. Require separate explicit approval, inspect downgrade data loss, and verify that the target application revision is schema-compatible before any downgrade.

## Remove Another Project Only When Explicitly Authorized

- Resolve exact containers, volumes, networks, bind mounts, images, and reverse-proxy files before deletion. Compose source files may already be missing; use `com.docker.compose.project` labels and container mounts when necessary.
- Distinguish project-owned resources from shared base images or infrastructure. Do not remove shared resources merely because the deleted project used them.
- Treat database and volume deletion as permanent and obtain explicit scope confirmation. Delete exact names, never broad globs or prune commands.
- For Nginx changes, remove only the confirmed project files, run `nginx -t`, reload only after validation, and verify no project references remain.

## Test the Deployment

Validate in three layers. Use the deepest coverage at the cheapest, safest layer, and do not repeat an already-passed full regression at every later layer.

### 1. Local deep validation

- Treat local validation as the primary functional acceptance gate before deployment.
- Run the project's practical formatting, lint, type, unit, integration, migration, and build checks against the exact commit to deploy.
- Use local Playwright, ego-browser, or another real browser to exercise the complete changed feature set, critical user journeys, relevant roles, validation errors, responsive layouts, console errors, and failed network requests.
- Fix failures locally and rerun the affected local suites before pushing. Do not use the production public endpoint as the main development test environment.

### 2. Tencent loopback or tunnel medium validation

1. Inspect `docker compose -p <project> ps` and relevant recent logs.
2. Run health and API checks on the server against `127.0.0.1:<remote-port>`.
3. Verify deployment-specific state that local tests cannot prove: running revision, migration revision, data provenance and representative row counts, account counts and role states, object references, mounted volumes, environment-dependent integrations, and the core login/authorization paths.
4. Exercise representative data and account flows through the remote loopback service or an SSH tunnel. Keep mutations minimal and reversible; do not repeat every local browser scenario when the same committed image already passed locally.
5. Treat healthy containers as necessary but not sufficient. Inspect scheduled-worker and external-integration logs after at least one polling cycle; report degraded integrations separately from core service health.
6. When an integration uses a one-time rotating refresh credential, never consume it merely to diagnose deployment. Respect the project's persistence callback or dedicated refresh script, and require explicit authorization before rotating credentials or disabling an enabled integration.
7. Establish an SSH tunnel when browser access to the loopback service is useful. Prefer local frontend/backend ports `5173` and `8000` when free:

   ```bash
   ssh -N \
     -L 5173:127.0.0.1:<remote-frontend-port> \
     -L 8000:127.0.0.1:<remote-backend-port> \
     tencent-dev
   ```

8. Test the tunneled application at `http://localhost:5173` and API at `http://localhost:8000`.
9. When testing projects concurrently, allocate different free local ports, such as `5174` and `8001` for the second project. Sequential tests may reuse `5173` and `8000`.
10. Keep the tunnel alive during tests and close only the tunnel created for this run.

Remember that `localhost` is the local tunnel endpoint, not the server's public address.

### 3. Public route light validation

- Treat the public IP or domain as a production smoke-test surface, not a second full acceptance environment.
- Verify only the externally exposed boundary by default: trusted HTTPS when configured, HTTP-to-HTTPS redirect, readiness, login or landing-page access, static assets, one representative protected page, critical navigation, and absence of fatal console/network errors.
- Prefer read-only checks. Avoid creating production business data, replaying every role matrix, exhaustively clicking maps or dashboards, or repeating all form and responsive cases already proven locally and through the tunnel.
- Expand public testing only when the user explicitly requests exhaustive production acceptance, a public-only setting changed, or a smoke check exposes a failure. Diagnose the failing boundary narrowly instead of restarting the entire test suite.
- Stop when the public route opens correctly and its critical smoke checks pass. Record concise evidence and deliver the URL; do not continue clicking without a specific unresolved risk.

## Finish

Report:

- Project, branch, and deployed SHA
- For a self-contained private delivery, the GitLab project/branch SHA and proof that its tree equals the Tencent bare ref, Tencent checkout, local commit, and running OCI revision
- Chosen database data strategy and the exact source/target provenance
- Previous and current checkout SHAs, database revisions, and running-image provenance for affected services
- Source/target account counts and representative or all-table row-count verification; never report schema revision alone as proof that data was migrated
- Bare repository and deployment paths
- Compose project name and remote ports
- Canonical Compose files and ordered env-file paths, without secret contents
- Whether object-storage data was copied, verified unnecessary, or remains pending
- Commands and test results
- Validation performed at each layer: local deep, Tencent loopback/tunnel medium, and public light; state any risk-based expansion explicitly
- Container health and relevant errors
- Local tunnel command and test URLs
- Public URL or reverse-proxy status, if configured

On failure, preserve logs and existing data, stop further mutations, and state the exact failing step. Do not silently roll back, reset, or delete anything.
