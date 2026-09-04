# Project Deployment Configuration

Add a section like this to the project's nearest `AGENTS.md` when deployment values cannot be inferred safely:

```markdown
## Remote deployment

- Project name: `<repository-basename>`
- SSH alias: `<your-ssh-alias>`
- Deployment branch: `<branch>`
- Existing branch upstream: `<remote-and-merge-ref-or-unset>`
- Compose project name: `<project>`
- Compose files: `compose.yaml[, compose.production.yaml]`
- Compose env files in override order: `<project-env>[, <registry-or-site-env>]`
- Docker library prefix: `<optional-registry/library-prefix-ending-in-slash>`
- Docker Hub prefix: `<optional-registry-prefix-ending-in-slash>`
- Remote frontend port: `<unique-port>`
- Remote backend port: `<unique-port>`
- Frontend container port: `<port>`
- Backend container port: `<port>`
- Health path: `/health`
- Local checks: `<commands>`
- Remote checks: `<commands>`
- Affected-service rebuild mapping: `<paths-or-change-types-to-services>`
- Image revision label/provenance check: `<oci-label-or-image-id-procedure>`
- Migration target and predecessor: `<revision-or-unconfigured>`
- Application/schema rollback compatibility: `<documented-rule-or-unconfigured>`
- Backup and verification commands: `<commands-or-unconfigured>`
- Object storage public endpoint: `<browser-reachable-host:port-or-unconfigured>`
- Object storage edition and CORS mechanism/origins: `<community-global-env-or-aistor-bucket-xml-and-required-origins>`
- Public URL: `<url-or-unconfigured>`
```

Use unique remote host ports for every running project. Local tunnel ports may be reused for sequential tests:

```text
project-a: localhost:5173 -> server:15173; localhost:8000 -> server:18000
project-b: localhost:5174 -> server:25173; localhost:8001 -> server:28000
```

Do not store secrets in this file.
