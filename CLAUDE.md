# CLAUDE.md - AI Assistant Guide for containers

## Project Overview

This repository builds and publishes Docker container images to GitHub Container Registry (`ghcr.io/dylansdecoded/<app>`). It contains 17 application images with automated version detection, multi-architecture builds (linux/amd64, linux/arm64), and CI/CD via GitHub Actions.

Key principles: semantic versioning, rootless containers (`nobody:nogroup`), one process per container, Alpine/Ubuntu base images, SHA256 digest-based tag immutability.

## Repository Structure

```
containers/
├── apps/                          # All container image definitions (17 apps)
│   └── <app>/
│       ├── Dockerfile             # Image definition (required)
│       ├── metadata.yaml          # App config: channels, platforms, tests (required)
│       ├── entrypoint.sh          # Runtime initialization (optional)
│       ├── ci/
│       │   ├── latest.sh          # Upstream version detection script
│       │   └── goss.yaml          # Container test specs (Goss framework)
│       ├── scripts/               # App-specific helpers (optional)
│       ├── *.tmpl                 # Config templates for envsubst (optional)
│       └── README.md              # App-specific docs (optional)
├── .github/
│   ├── workflows/                 # GitHub Actions CI/CD (8 workflows)
│   │   ├── release-on-merge.yaml  # Main pipeline: push to main triggers build+push
│   │   ├── pr-validate.yaml       # PR pipeline: build without push
│   │   ├── build-images.yaml      # Core build logic (reusable workflow)
│   │   ├── get-changed-images.yaml# Detect which apps changed
│   │   ├── simple-checks.yaml     # CUE schema validation
│   │   ├── render-readme.yaml     # Auto-generate README.md
│   │   ├── renovate.yaml          # Automated dependency updates
│   │   └── label-sync.yaml        # GitHub label management
│   ├── scripts/
│   │   ├── prepare-matrices.py    # Build matrix generator (core build orchestration)
│   │   ├── render-readme.py       # README auto-generation from Jinja2 template
│   │   ├── json-to-yaml.py        # Utility converter
│   │   ├── requirements.txt       # Python deps: requests, pyyaml, packaging, jinja2
│   │   └── templates/
│   │       └── README.md.j2       # Jinja2 template for root README
│   ├── renovate.json5             # Renovate bot config (semantic commits)
│   ├── labels.yaml                # GitHub label definitions
│   └── CODEOWNERS                 # All files owned by @DylansDecoded
├── metadata.rules.cue             # CUE schema for validating app metadata
├── Taskfile.yml                   # go-task runner (label management)
├── README.md                      # AUTO-GENERATED - do not edit directly
├── .editorconfig                  # Editor settings
└── .gitignore                     # Ignores .goss, .private, *.pyc
```

## App Directory Layout

Every app under `apps/<app>/` must have at minimum:

| File | Required | Purpose |
|------|----------|---------|
| `Dockerfile` | Yes | Multi-stage Docker image build |
| `metadata.yaml` | Yes | App name, channels, platforms, test config |
| `ci/latest.sh` or `ci/latest.py` | Yes | Script to detect latest upstream version |
| `ci/goss.yaml` | If tests enabled | Goss test specifications |
| `entrypoint.sh` | No | Runtime config initialization |
| `*.tmpl` | No | Config templates processed by envsubst |

## metadata.yaml Schema

Validated by CUE schema in `metadata.rules.cue`:

```yaml
app: <app-name>          # Must match: ^[a-zA-Z0-9_-]+$
semver: true             # Optional: enables semantic version tag generation
channels:
  - name: main           # Channel name: ^[a-zA-Z0-9._-]+$
    platforms:            # "linux/amd64" and/or "linux/arm64"
      - "linux/amd64"
      - "linux/arm64"
    stable: true          # true = image tagged as <app>, false = <app>-<channel>
    tests:
      enabled: true
      type: web           # "web" (HTTP test) or "cli" (tail -f /dev/null)
```

## Dockerfile Conventions

- **Base images**: Alpine 3.19 or Ubuntu (pinned versions)
- **Required ARGs**: `TARGETPLATFORM`, `VERSION`, `CHANNEL`, `REVISION`
- **Init system**: `catatonit` for proper signal handling
- **User**: `nobody:nogroup` (UID 65534) - all containers run rootless
- **Working directory**: `/config` as a volume mount point
- **Entrypoint pattern**: `ENTRYPOINT ["/usr/bin/catatonit", "--"]` with `CMD ["/entrypoint.sh"]`
- **Build context**: Set to repo root (`.`), not the app directory. COPY paths must be relative to repo root (e.g., `COPY ./apps/sonarr/entrypoint.sh /entrypoint.sh`)
- **Multi-stage builds**: Used when build tools are needed (e.g., Go for envsubst)
- **Architecture handling**: Use `case "${TARGETPLATFORM}"` to select arch-specific binaries
- **Indentation**: 4 spaces (per `.editorconfig`)
- **Common packages**: bash, ca-certificates, catatonit, curl, jq, nano, tzdata

## CI/CD Pipeline

### On push to `main` (release-on-merge.yaml):
1. **simple-checks** - Validates all metadata.yaml files against CUE schema
2. **get-changed-images** - Detects which apps had file changes
3. **build-images** - Builds, tests, and pushes changed images
4. **render-readme** - Regenerates README.md from Jinja2 template

### On pull request (pr-validate.yaml):
Same pipeline but `pushImages: false` and `sendNotifications: false`.

### Build process (build-images.yaml):
1. `prepare-matrices.py` generates a JSON build matrix from metadata
2. Per-platform images are built with Docker Buildx
3. Goss tests run on `linux/amd64` only
4. Platform digests are uploaded as artifacts
5. Multi-arch manifests are merged and tagged
6. Discord notifications sent on success/failure

### Image tagging strategy:
- `rolling` tag always points to latest build
- Full version tag (e.g., `3.0.8.1507`)
- If `semver: true`: additional partial version tags (`3.0.8`, `3.0`, `3`)
- Stable channels: `<app>:<tag>`
- Unstable channels: `<app>-<channel>:<tag>`

## Version Detection Scripts

Each app has `ci/latest.sh` (bash) or `ci/latest.py` (Python) that:
- Accepts a channel name as the first argument
- Queries upstream APIs (GitHub releases, app-specific APIs)
- Outputs the latest version string to stdout
- Must be executable (`chmod +x`)

## Testing

- **Framework**: Goss (container structure testing)
- **Config**: `apps/<app>/ci/goss.yaml`
- **Test types**:
  - `web`: Container starts normally, tests HTTP endpoints/ports
  - `cli`: Container runs with `tail -f /dev/null`, tests process/files
- **Only runs on**: `linux/amd64` platform
- **Goss options**: `--retry-timeout 60s --sleep 2s`

## Code Style and Formatting

| File Type | Indent | Size |
|-----------|--------|------|
| Default (`*`) | spaces | 2 |
| Bash/Shell (`*.sh`, `*.bash`) | spaces | 4 |
| Dockerfile | spaces | 4 |

All files: LF line endings, UTF-8 charset, trim trailing whitespace, insert final newline.

## Commit Message Conventions

This project uses **semantic commits** (enforced by Renovate config):
- `feat(<app>)!:` for major dependency updates
- `feat(<app>):` for minor updates / new features
- `fix(<app>):` for patch updates / bug fixes
- `chore(deps):` for CI/tooling dependency updates

## Adding a New App

1. Create `apps/<app-name>/` directory
2. Add `metadata.yaml` with app name, channels, platforms, and test config
3. Create `Dockerfile` following existing conventions (rootless, catatonit, proper ARGs)
4. Add `ci/latest.sh` or `ci/latest.py` for version detection
5. Add `ci/goss.yaml` if tests are enabled
6. Add `entrypoint.sh` if runtime configuration is needed
7. Run `task append-app-labels` to add the app label to `.github/labels.yaml`
8. Open a PR - the `pr-validate` workflow will build and test the image

## Key Dependencies

- **Python 3.x**: Build scripts (`requests`, `pyyaml`, `packaging`, `jinja2`)
- **go-task**: Local task runner (`brew install go-task`)
- **CUE**: Metadata schema validation
- **Goss**: Container testing framework
- **Docker Buildx**: Multi-platform builds
- **Renovate Bot**: Automated base image updates (runs hourly)

## Important Notes

- **README.md is auto-generated** from `.github/scripts/templates/README.md.j2` - never edit it directly
- **Metadata changes alone don't trigger builds** - the `release-on-merge.yaml` path filter excludes `metadata.json`, `metadata.yaml`, and `README.md`
- **Build context is the repo root**, not the app directory. All COPY paths in Dockerfiles must be relative to repo root
- **All code is owned by @DylansDecoded** (see `.github/CODEOWNERS`)
- **Secrets required**: `BOT_APP_ID`, `BOT_APP_PRIVATE_KEY` (GitHub App), `DISCORD_WEBHOOK` (notifications)

## Current Apps

actions-runner, bazarr, home-assistant, jbops, kubanetics, lidarr, par2cmdline-turbo, plex, postgres-init, prowlarr, qbittorrent, radarr, readarr, sabnzbd, sonarr, theme-park, volsync

Multi-channel apps: lidarr/prowlarr/radarr (master, develop, nightly), sonarr (main, develop), plex/qbittorrent (stable, beta), readarr (develop, nightly)
