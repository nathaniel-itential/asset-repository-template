# GitLab CI/CD Pipelines

GitLab CI/CD pipelines for automatically versioning and deploying Itential Platform assets. These pipelines execute the shared scripts in `scripts/` to handle version bumping and asset deployment.

## Pipeline Jobs

### Automatic RC Tagging (`create-rc-tag`)

Triggers on any push to `master` (i.e., after a merge request is merged). It automatically:

1. Determines the version bump type by scanning commit messages using [Conventional Commits](https://www.conventionalcommits.org/)
   - `feat!:` or `BREAKING CHANGE:` &rarr; **major** bump
   - `feat:` &rarr; **minor** bump
   - All other commits &rarr; **patch** bump
2. Creates and pushes an annotated release candidate tag (e.g., `v1.1.0-rc.1`)

### Staging Deployment (`deploy-staging`)

Runs immediately after `create-rc-tag` on every push to `master`. Connects to the staging Itential Platform instance and imports all discovered assets.

### Production Deployment (`deploy-production`)

Triggers on any tag push matching `v*` without an `-rc` suffix (e.g., `v1.1.0`). Deploys to the production Itential Platform instance.

| Tag pattern | Example | Target environment |
| --- | --- | --- |
| Contains `-rc` | `v1.1.0-rc.1` | Staging (created automatically on push to master) |
| No `-rc` suffix | `v1.1.0` | Production |

> **Note:** RC tags are internal artifacts created by the pipeline and do **not** trigger a new pipeline run. Only pushes to `master` and final release tags trigger pipelines.

The job runs `scripts/deploy.py` which connects to the target Itential Platform instance and imports all discovered assets.

## Setup

### Prerequisites

- A GitLab repository using this template structure
- Two Itential Platform instances (staging and production) with service account credentials
- A GitLab Project Access Token with `write_repository` scope to push tags

### 1. Copy Pipeline and Script Files

The `.gitlab-ci.yml` at the root of this repository is the pipeline definition. Copy it along with the shared scripts into your repository:

```bash
mkdir -p <path-to-your-repo>/scripts
cp .gitlab-ci.yml <path-to-your-repo>/
cp scripts/* <path-to-your-repo>/scripts/
```

> **Note:** The pipeline references scripts at `scripts/` by default. If you place them elsewhere, update the script paths in `.gitlab-ci.yml` accordingly.

### 2. Create GitLab Environments

Create two environments in your GitLab project under **Operate** > **Environments** > **New environment**:

1. Create an environment named `staging`
2. Create an environment named `production`

Once both environments exist, add the following CI/CD variables under **Settings** > **CI/CD** > **Variables**. Expand the **Variables** section, click **Add variable**, and set the **Environment scope** for each variable as noted below.

`PLATFORM_HOST`, `PLATFORM_CLIENT_ID`, and `PLATFORM_CLIENT_SECRET` are environment-specific — add them twice, once scoped to `staging` and once scoped to `production`, each pointing to the respective platform instance.

| Type | Name | Flags | Environment scope | Description |
| --- | --- | --- | --- | --- |
| Variable | `PLATFORM_HOST` | Masked, Hidden | `staging` / `production` | Itential Platform instance hostname |
| Variable | `PLATFORM_CLIENT_ID` | Masked, Hidden | `staging` / `production` | Service account client ID |
| Variable | `PLATFORM_CLIENT_SECRET` | Masked, Hidden | `staging` / `production` | Service account client secret |
| Variable | `PROJECT_MEMBERS` | Expand variable | `*` (all) or per-environment | JSON array of project members — minified, no whitespace (see below) |

### 3. Create Project-Level Variable

Create a project-level CI/CD variable for the `create-rc-tag` job under **Settings** > **CI/CD** > **Variables**:

| Type | Name | Flags | Description |
| --- | --- | --- | --- |
| Variable | `CI_ACCESS_TOKEN` | Masked, Hidden, Protected | GitLab Project Access Token with `write_repository` scope |

### 4. Configure Project Members

> **Important:** When adding the `PROJECT_MEMBERS` variable, enable **Expand variable reference** so that the JSON value is correctly parsed at runtime.

The `PROJECT_MEMBERS` variable is a JSON array that controls who gets assigned to imported Studio projects. It supports two member types. It can be shared across all environments (scope `*`) or set per-environment if staging and production require different members.

The value must be minified JSON with no whitespace:

```
[{"type":"account","username":"user@example.com","role":"owner"},{"type":"group","name":"network-ops","role":"operator"}]
```

To minify an existing JSON file:

```bash
# using jq
jq -c . members.json

# using Python
cat members.json | python3 -m json.tool --compact
```

## Deploying

### To Staging (automatic)

1. Commit changes to `develop` using [Conventional Commits](https://www.conventionalcommits.org/) (e.g., `feat: add vlan provisioning use case`)
2. Open a merge request from `develop` to `master` and merge it
3. The `create-rc-tag` job creates an RC tag and `deploy-staging` deploys to staging in the same pipeline run

### To Staging (manual)

Trigger a pipeline manually on the `master` branch from **CI/CD** > **Pipelines** > **Run pipeline** in your GitLab repository, or push any commit to `master`.

### To Production

After validating in staging, create and push a clean release tag:

```bash
git tag v1.1.0
git push origin v1.1.0
```

Or create a release in GitLab:

1. Go to **Deploy** > **Releases** > **Create a new release** in your GitLab repository
2. Create a new tag using the version number (e.g., `v1.1.0`) — do not include the `-rc` suffix
3. Set the target branch to `master`
4. Add release notes describing the changes
5. Click **Create release**

Creating the release pushes the tag, which triggers the production deployment pipeline.

## Running the Deploy Script Locally

> **Note:** Run this script from the repository root directory, not from within `scripts/`.

```bash
export HOST="<platform-hostname>"
export CLIENT_ID="<client-id>"
export CLIENT_SECRET="<client-secret>"
export PROJECT_MEMBERS='[{"type":"account","username":"user@example.com","role":"admin"}]'

pip install git+https://github.com/Itential/asyncplatform.git
cd /path/to/repo/root  # navigate to the root of your repository
python scripts/deploy.py <Staging|Production>
```
