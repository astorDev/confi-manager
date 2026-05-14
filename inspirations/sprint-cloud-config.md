# Spring Cloud Config

Spring Cloud Config is a git-backed configuration server for distributed systems, primarily targeting the JVM/Spring ecosystem. It serves as a key inspiration and starting point for confi-manager.

## How It Works

A central config server clones a git repository and exposes its contents over HTTP. Services don't interact with git directly — they call the config server at startup:

```
GET http://config-server/{app-name}/{environment}
```

The server merges multiple files into a single flat response:
- `application.yml` — shared defaults for all services
- `{app}.yml` — service-specific config
- `{app}-{env}.yml` — environment override

The service receives one merged config object and knows nothing about git.

## What It Does Well

- **Git as storage** — versioning, diffs, and PR-based approvals come for free
- **Provider-independent** — works with GitHub, GitLab, Gitea, local folders; git auth (SSH/HTTPS) is the same everywhere
- **Centralized git credential** — services don't need git access; one token on the config server is enough
- **Encryption support** — stores `{cipher}...` ciphertext in git; the config server holds the decryption key and serves plaintext to services. Secrets stay safe even if the repo is compromised.
- **Environment merging** — base + service + environment files are merged automatically

## What It Does Poorly

### No Governance Layer
Access control is entirely delegated to the git host. There is no concept of "which team owns which config file", no approval workflows beyond PR reviews, and no audit log beyond git history. Non-engineers cannot safely interact with it.

### No Per-Service Read Isolation
The config server uses a single git credential and serves any config to any caller. Service A can request Service B's config with no restriction. There is no service identity or per-service access control at runtime.

### No UI for Non-Engineers
Editing config is a pure git activity — clone the repo, edit YAML, commit, push. A product manager or support engineer must either go through a developer or be given git access and know the file structure.

### Poor Developer Experience
The service and its config live in separate repositories with a hard runtime dependency between them. A developer adding a new config property must:
1. Update the config repo
2. Update the service code

If the config repo isn't updated first, the service fails to start. CI pipelines break silently. With multiple environments (dev/staging/prod), keeping the repos in sync becomes a persistent coordination burden.

### No Schema / Contract System
Services don't declare which config keys they depend on. There is no validation that all required config exists before deployment, and no alerting when a key a service depends on is removed from the config repo.

### Polling Only (No Push)
Services must poll the config server or use Spring Cloud Bus (requires a message broker) to receive updates. There is no native push model.

## Key Takeaway

Spring Cloud Config solves the "one git credential, centralized merge, decryption" problem well. It justifies the middleman through those three things. Everything above that layer — governance, access control, developer UX, schema contracts — is left to the operator to build or accept as absent.