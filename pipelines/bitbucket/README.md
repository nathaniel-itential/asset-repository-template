# Bitbucket Pipelines

Bitbucket Pipelines for automatically versioning and deploying Itential Platform assets. These pipelines execute the shared scripts in `scripts/` to handle version bumping and asset deployment.

## Pipeline Steps

### RC Tagging (`create-rc-tag`)

Triggers on any push to `master` (i.e., after a pull request is merged). It automatically:

1. Determines the version bump type by scanning commit messages using [Conventional Commits](https://www.conventionalcommits.org/)
   - `feat!:` or `BREAKING CHANGE:` &rarr; **major** bump
   - `feat:` &rarr; **minor** bump
   - All other commits &rarr; **patch** bump
2. Creates and pushes an annotated release candidate tag (e.g., `v1.1.0-rc.1`)

### Staging Deployment (`deploy-staging`)

Runs immediately after `create-rc-tag` on every push to `master`. Connects to the staging Itential Platform instance and imports all discovered assets.

### Production Deployment

Triggers on any tag push matching `v*` without an `-rc` suffix (e.g., `v1.1.0`). Deploys to the production Itential Platform instance.

| Tag pattern | Example | Target environment |
| --- | --- | --- |
| Contains `-rc` | `v1.1.0-rc.1` | Staging (created automatically on push to master) |
| No `-rc` suffix | `v1.1.0` | Production |

> **Note:** RC tags are internal artifacts created by the pipeline and do **not** trigger a new pipeline run. Only pushes to `master` and final release tags trigger pipelines.

The step runs `scripts/deploy.py` which connects to the target Itential Platform instance and imports all discovered assets.

## Setup

### Prerequisites

- A Bitbucket repository using this template structure
- Two Itential Platform instances (staging and production) with service account credentials
- A Bitbucket Repository Access Token with `write_repository` scope to push tags

### 1. Copy Pipeline and Script Files

The `bitbucket-pipelines.yml` at the root of this repository is the pipeline definition. Copy it along with the shared scripts into your repository:

```bash
mkdir -p <path-to-your-repo>/scripts
cp bitbucket-pipelines.yml <path-to-your-repo>/
cp scripts/* <path-to-your-repo>/scripts/
```

> **Note:** The pipeline references scripts at `scripts/` by default. If you place them elsewhere, update the script paths in `bitbucket-pipelines.yml` accordingly.

### 2. Create Bitbucket Deployments

Create two deployment environments in your Bitbucket repository under **Repository settings** > **Deployments**:

1. Create an environment named `staging`
2. Create an environment named `production`

Once both environments exist, add the following variables under each environment's **Variables** section. Add each variable twice — once scoped to `staging` and once scoped to `production` — each pointing to the respective platform instance.

| Name | Environment | Description |
| --- | --- | --- |
| `PLATFORM_HOST` | `staging` / `production` | Itential Platform instance hostname |
| `PLATFORM_CLIENT_ID` | `staging` / `production` | Service account client ID |
| `PLATFORM_CLIENT_SECRET` | `staging` / `production` | Service account client secret |
| `PROJECT_MEMBERS` | `staging` / `production` (or shared) | JSON array of project members — minified, no whitespace (see below) |

Mark `PLATFORM_CLIENT_ID` and `PLATFORM_CLIENT_SECRET` as **Secured** so they are masked in logs.

### 3. Create Repository Variable

Create a repository-level variable for the `create-rc-tag` step under **Repository settings** > **Repository variables**:

| Name | Description |
| --- | --- |
| `BB_AUTH_STRING` | Bitbucket Repository Access Token with `write_repository` scope for pushing tags |

Mark this variable as **Secured** so it is masked in logs.

### 4. Configure Project Members

The `PROJECT_MEMBERS` variable is a JSON array that controls who gets assigned to imported Studio and Agent projects. It supports two member types. It can be shared across all environments or set per-environment if staging and production require different members.

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
2. Open a pull request from `develop` to `master` and merge it
3. The `create-rc-tag` step creates an RC tag and `deploy-staging` deploys to staging in the same pipeline run

### To Staging (manual)

Trigger a pipeline manually on the `master` branch from **Pipelines** > **Run pipeline** in your Bitbucket repository.

### To Production

After validating in staging, create and push a clean release tag:

```bash
git tag v1.1.0
git push origin v1.1.0
```

Or create the tag from the Bitbucket UI:

1. Go to **Commits** in your Bitbucket repository
2. Select the commit on `master` you want to tag
3. Click **Create tag** on the right side and enter the version number (e.g., `v1.1.0`) — do not include the `-rc` suffix


Pushing the tag triggers the production deployment pipeline.

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
