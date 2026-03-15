# .github

Reusable GitHub Actions workflows and composite actions for enyonee projects.

## Workflows

### nexus-push.yml

Build Docker image and push to private Nexus registry via SSH tunnel.

Full pipeline: SSH tunnel → docker login → build (with GHA cache) → push → verify manifest → cleanup. Telegram notification on failure.

```yaml
# Minimal — one image, amd64
jobs:
  push:
    uses: enyonee/.github/.github/workflows/nexus-push.yml@main
    with:
      image_name: my-project
    secrets: inherit
```

```yaml
# Multi-arch with custom Dockerfile
jobs:
  push:
    uses: enyonee/.github/.github/workflows/nexus-push.yml@main
    with:
      image_name: my-project
      dockerfile: docker/Dockerfile.prod
      context: .
      platforms: linux/amd64,linux/arm64
    secrets: inherit
```

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `image_name` (required) | — | Image name in Nexus |
| `dockerfile` | `Dockerfile` | Path to Dockerfile |
| `context` | `.` | Docker build context |
| `platforms` | `linux/amd64` | Target platforms |
| `tag` | git ref | Override image tag |

**Required org secrets:** `NEXUS_SSH_KEY`, `NEXUS_HOST_KEY`, `NEXUS_HOST`, `NEXUS_USERNAME`, `NEXUS_PASSWORD`

**Optional secrets:** `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` (for failure notifications)

**Image tags:**
- Tag push `v1.2.3` → `image:1.2.3` + `image:latest`
- Branch push `main` → `image:main`

**How it works:**
1. Opens SSH tunnel to VPS → port-forwards to `nexus.shared.svc.cluster.local:8082`
2. Builds with Docker Buildx (GHA layer cache for speed)
3. Pushes to `localhost:8082/{image_name}:{tag}` through tunnel
4. Verifies manifest exists in Nexus API
5. Cleans up SSH key and tunnel process
6. K3s pulls from `nexus.shared.svc.cluster.local:8082` internally (no tunnel needed)

---

### python-ci.yml

Lint (ruff) + test (pytest, cross-platform matrix) + Telegram notification.

```yaml
jobs:
  ci:
    uses: enyonee/.github/.github/workflows/python-ci.yml@v1
    with:
      project: MyProject
      test-command: "uv run pytest tests/ -x -q --tb=short"
    secrets: inherit
```

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `project` (required) | — | Project name for notifications |
| `python-version` | `3.12` | Python version |
| `uv-sync-args` | `--frozen --extra dev` | Args for `uv sync` |
| `uv-sync-env` | `""` | Env vars for sync/test (`KEY=VAL` per line) |
| `test-command` | `uv run pytest tests/ -x -q --tb=short` | Pytest command |
| `os-matrix` | `["ubuntu-latest"]` | JSON array of OS to test on |

**Required secrets:** `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`

---

## Actions

### nexus-tunnel

Composite action — SSH tunnel to Nexus only. For custom pipelines that need non-standard build flows (monorepo, multi-image, integration tests).

```yaml
steps:
  - name: Setup Nexus tunnel
    uses: enyonee/.github/actions/nexus-tunnel@main
    id: tunnel
    with:
      ssh_key: ${{ secrets.NEXUS_SSH_KEY }}
      host_key: ${{ secrets.NEXUS_HOST_KEY }}
      host: ${{ secrets.NEXUS_HOST }}

  # Now Nexus is available at localhost:8082
  - name: Login to Nexus
    uses: docker/login-action@v3
    with:
      registry: localhost:8082
      username: ${{ secrets.NEXUS_USERNAME }}
      password: ${{ secrets.NEXUS_PASSWORD }}

  # Custom build logic here...
  - run: |
      docker build -t localhost:8082/my-test-runner:latest .
      docker push localhost:8082/my-test-runner:latest

  # Cleanup tunnel when done
  - name: Cleanup tunnel
    if: always()
    run: /tmp/nexus-tunnel/cleanup.sh
```

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `ssh_key` (required) | — | SSH private key for ci-tunnel |
| `host_key` (required) | — | SSH host key for MITM protection |
| `host` (required) | — | VPS host address |
| `local_port` | `8082` | Local port for tunnel |

**Outputs:**

| Output | Description |
|--------|-------------|
| `tunnel_port` | Local port where Nexus is available |

---

## Architecture

```
GitHub Actions (cloud runner)
  │
  │ SSH tunnel (port-forward only, ci-tunnel user)
  │
  ▼
VPS:22 ──► nexus.shared.svc.cluster.local:8082 (K3s cluster)
                │
                │ K3s pulls (registries.yaml, imagePullSecret)
                ▼
          K3s pods (namespace prod, ArgoCD sync)
```

## Security

- SSH key restricted: `command="/bin/false"`, `no-pty`, `permitopen` only Nexus:8082
- Dedicated `ci-tunnel` user (isolated from deploy user with kubectl access)
- `StrictHostKeyChecking=yes` with pre-shared host key (MITM protection)
- Nexus CI user: push-only role, no admin
- SSH key cleaned up after every run
- Tunnel auto-killed on job completion
