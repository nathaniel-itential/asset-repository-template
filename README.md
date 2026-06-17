# Asset Repository Template

A template repository that demonstrates how to manage Itential Platform assets using Git and automatic promotion through CI/CD pipelines. Use this as a starting point to version-control your asset bundles and automate deployments across environments.

> **Currently supported:** GitHub Actions, GitLab CI/CD
>
> **Coming soon:** Bitbucket Pipelines, Jenkins

## How It Works

This repository uses a tag-based promotion model. CI/CD pipelines execute the shared scripts in `pipelines/scripts/` to automatically version, tag, and deploy assets across environments.

> **Note:** Throughout this documentation, `main` is used as the default branch name. If your repository uses `master`, substitute `master` wherever `main` appears.

```text
 develop branch       main/master branch           Staging                Production
 ──────────────       ──────────────────           ───────                ──────────
       |                       |                      |                       |
   commit work                 |                      |                       |
       |                       |                      |                       |
   open PR ──── merge ────►    |                      |                       |
       |                   auto-rc-tag                |                       |
       |                   creates v1.1.0-rc.1        |                       |
       |                       |                      |                       |
       |                  tag push triggers           |                       |
       |                  asset-promotion ────────► deploy to staging         |
       |                       |                      |                       |
       |                       |                  validate & test             |
       |                       |                      |                       |
       |  ◄───── fix in develop ◄──────────── issues found                   |
       |                       |                      |                       |
   open PR ──── merge ────►    |                      |                       |
       |                   auto-rc-tag                |                       |
       |                   creates v1.1.0-rc.2        |                       |
       |                       |                      |                       |
       |                  tag push triggers           |                       |
       |                  asset-promotion ────────► re-deploy to staging      |
       |                       |                      |                       |
       |                       |                  validate & test             |
       |                       |                      |                       |
       |                  manual tag push              |                       |
       |                     v1.1.0                    |                       |
       |                       |                      |                       |
       |                  asset-promotion ─────────────────────────► deploy to prod
```

### Automatic Versioning

The `auto-rc-tag` step in the diagram above uses [Conventional Commits](https://www.conventionalcommits.org/) to determine the next semantic version. The script scans all commit messages since the last production tag and picks the highest-priority bump:

| Bump Type | Commit Prefix | Example | Version Change |
| --- | --- | --- | --- |
| **Major** | `feat!:` or `feat(scope)!:` or `BREAKING CHANGE:` | `feat!: redesign asset schema` | `v1.2.3` → `v2.0.0` |
| **Minor** | `feat:` or `feat(scope):` | `feat: add lifecycle manager support` | `v1.2.3` → `v1.3.0` |
| **Patch** | Anything else (`fix:`, `chore:`, `docs:`, etc.) | `fix: correct automation import` | `v1.2.3` → `v1.2.4` |

If any commit in the range is a breaking change, the bump is **major** regardless of other commits. If no `feat` or breaking change commits are found, the bump defaults to **patch**.

Once the version is determined, the script creates a release candidate tag (e.g., `v1.3.0-rc.1`). If an RC tag for that version already exists then the RC number is incremented (e.g., `v1.3.0-rc.2`).

## Repository Structure

```text
.
├── Asset Bundle/              # Example asset bundle
│   ├── studio/
│   ├── operations_manager/
│   ├── lifecycle_manager/
│   ├── configuration_manager/
│   └── integration_models/
│
├── pipelines/
│   ├── github/                # GitHub Actions pipeline definitions
│   ├── gitlab/                # GitLab CI/CD pipeline definitions
│   ├── scripts/               # Shared deployment scripts
│   │   ├── deploy.py
│   │   ├── deploy_integrations.sh
│   │   └── bump-version.sh
│   └── ...                    # Future Git platform directories
│
└── README.md
```

### `Asset Bundle/`

An example asset bundle that shows the expected directory layout. It includes sample assets for each supported Itential Platform application:

- **`studio/`** — Studio project files (`.project.json`)
- **`operations_manager/`** — Operations Manager automation files (`.automation.json`)
- **`lifecycle_manager/`** — Lifecycle Manager resource model files (`.model.json`)
- **`configuration_manager/`** — Configuration Manager golden config files (`.gctree.json`)
- **`integration_models/`** — OpenAPI spec files for Itential integration models (`.json`)

You can add multiple bundles at the repo root and the deploy script will auto-discover them.

### `pipelines/`

Contains CI/CD pipeline definitions organized by platform, along with shared scripts that are called by each pipeline.

**`pipelines/scripts/`** — Shared scripts used across all platforms:

- **`deploy.py`** — Connects to an Itential Platform instance and imports all discovered assets from the repository. Supports Studio projects, Operations Manager automations, Lifecycle Manager resources, and Configuration Manager golden configs.
- **`deploy_integrations.sh`** — Imports OpenAPI specs as integration models using `ipctl`. Reads a JSON array of spec paths from the `CHANGED_SPECS` environment variable.
- **`bump-version.sh`** — Calculates the next semantic version based on commit messages and creates release candidate tags

**Platform-specific pipeline directories:**

Each platform directory contains workflow/pipeline definitions and a README with setup and usage instructions for that CI/CD system.

| Directory | Platform | Status |
| --- | --- | --- |
| [`pipelines/github/`](pipelines/github) | GitHub Actions | Available |
| [`pipelines/gitlab/`](pipelines/gitlab) | GitLab CI/CD | Available |
| `pipelines/bitbucket/` | Bitbucket Pipelines | Coming soon |
| `pipelines/jenkins/` | Jenkins | Coming soon |

See the README in each platform directory for setup and deployment instructions.

## Adding a New Asset Bundle

1. Create a new directory at the repo root — not nested inside other directories — (e.g., `My Use Case Bundle/`)
2. Add subdirectories for each asset type and place your exported JSON files inside
3. Commit and push to your develop branch

```text
My Use Case Bundle/
├── studio/
│   └── My Project.project.json
├── operations_manager/              
│   └── My Automation.automation.json
├── lifecycle_manager/               
│   └── My Resource.model.json
├── configuration_manager/           
│   └── My Config.gctree.json
└── integration_models/
    └── My Integration.json
```

The deploy script auto-discovers any bundles matching this structure. No additional configuration is needed.

## License

Copyright 2026 Itential, LLC

This project is licensed under the GNU General Public License v3.0 — see the [LICENSE](LICENSE) file for details.
