# DevOps Lab: Build and Publish Online Boutique Frontend to GHCR

## 1. Objective

Set up a CI workflow that builds a Docker image for the Online Boutique frontend and publishes it to GitHub Container Registry (GHCR).

**Expected outcome:**

- Fork the source repository into your personal GitHub account.
- Clone the repository and create a dedicated lab branch.
- Build and validate the frontend image locally.
- Test publishing the image to GHCR manually.
- Automate image publishing using GitHub Actions.

## 2. Prerequisites

Ensure the following tools and access are available:

- Git
- GitHub CLI (`gh`)
- Docker Desktop, with the Docker daemon running
- A GitHub account with permission to create repositories and publish packages
- A GitHub Personal Access Token (classic) with `write:packages` for local GHCR testing, subject to organization policies

Verify the local setup:

```
git --version
gh --version
docker --version
docker info
```

The `docker info` command should return Docker server details without a connection error.

## 3. Fork and Clone the Repository

Source repository:

https://github.com/GoogleCloudPlatform/microservices-demo

### Option A: Fork using the GitHub UI

1. Open the source repository.
2. Click **Fork**.
3. Select your personal GitHub account as the destination.
4. Create the fork.
5. Clone your fork using its URL.

```
mkdir -p ~/devops-labs
cd ~/devops-labs

git clone https://github.com/YOUR_GITHUB_USERNAME/online-boutique-application.git
cd online-boutique-application
```

Replace `YOUR_GITHUB_USERNAME` with your GitHub username.

### Option B: Fork using GitHub CLI

```
mkdir -p ~/devops-labs
cd ~/devops-labs

gh auth login

gh repo fork GoogleCloudPlatform/microservices-demo \
  --clone \
  --default-branch-only

cd online-boutique-application
```

If the repository is already cloned, skip the fork and clone steps.

## 4. Create a Dedicated Branch

Create a branch for the lab changes:

```
git switch -c feature/lab
```

Verify the current branch and repository:

```
git branch --show-current
git remote -v
```

Expected branch:

```
feature/lab
```

The `origin` remote should point to your personal fork.

## 5. Create the Frontend Dockerfile

Create a file named `Dockerfile.frontend` in the repository root.

Add the following content:

```
FROM us-central1-docker.pkg.dev/online-boutique-ci/microservices-demo/frontend:v0.10.7

LABEL org.opencontainers.image.title="Online Boutique Frontend"
LABEL org.opencontainers.image.description="DevOps lab frontend image"
```

This Dockerfile uses the published Online Boutique frontend image as its base and adds descriptive image metadata. It does not rebuild the frontend application from source.

## 6. Build and Validate the Image Locally

Run these commands from the repository root:

```
docker build \
  -f Dockerfile.frontend \
  -t online-boutique-frontend:dev .
```

Verify that the image exists:

```
docker image inspect online-boutique-frontend:dev
```

Check the image summary:

```
docker images online-boutique-frontend
```

**Expected result:** The image `online-boutique-frontend:dev` appears in the local image list.

If the build fails, verify that Docker Desktop is running and that the base image can be pulled from Google Artifact Registry.

## 7. Test GHCR Publishing Manually

This step verifies registry authentication and image publishing before introducing GitHub Actions.

### 7.1 Create a GitHub token

Create a Personal Access Token (classic) with the `write:packages` scope, subject to your organization's policies.

Use the token only for local testing. Do not commit it to the repository or place it in a workflow file.

### 7.2 Log in to GHCR

Run the following commands in your terminal:

```
export GH_USER="YOUR_GITHUB_USERNAME"
read -s -p "GitHub token: " GHCR_TOKEN
echo

echo "$GHCR_TOKEN" | docker login ghcr.io \
  -u "$GH_USER" \
  --password-stdin

unset GHCR_TOKEN
```

Expected result:

```
Login Succeeded
```

### 7.3 Tag and push the image

```
export IMAGE="ghcr.io/YOUR_GITHUB_USERNAME/online-boutique-frontend:dev"

docker tag online-boutique-frontend:dev "$IMAGE"
docker push "$IMAGE"

unset IMAGE
```

Replace `YOUR_GITHUB_USERNAME` with your GitHub username. Use your GitHub account name in lowercase for the image path.

### 7.4 Verify the package

Open your GitHub profile and navigate to **Packages**.

Confirm that `online-boutique-frontend` exists and that the `dev` tag is available.

If manual publishing fails, resolve the registry authentication or package permission issue before proceeding to CI.

## 8. Create the GitHub Actions Workflow

Create the workflow directory and file:

```
mkdir -p .github/workflows
touch .github/workflows/ci-ghcr.yml
```

Add the following content to `.github/workflows/ci-ghcr.yml`:

```
name: Build and Publish Frontend

on:
  push:
    branches:
      - feature/lab
    paths:
      - "Dockerfile.frontend"
      - ".github/workflows/ci-ghcr.yml"

  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set image name and tags
        id: meta
        shell: bash
        run: |
          echo "image=ghcr.io/${GITHUB_REPOSITORY_OWNER,,}/online-boutique-frontend" >> "$GITHUB_OUTPUT"
          echo "sha=${GITHUB_SHA}" >> "$GITHUB_OUTPUT"

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: Dockerfile.frontend
          push: true
          tags: |
            ${{ steps.meta.outputs.image }}:${{ steps.meta.outputs.sha }}
            ${{ steps.meta.outputs.image }}:latest
```

### How the workflow works

| Step | Purpose |
| --- | --- |
| Checkout repository | Downloads the source code |
| Set up Buildx | Prepares Docker's build tooling |
| Log in to GHCR | Authenticates using the workflow's `GITHUB_TOKEN` |
| Set image name and tags | Creates a lowercase registry path and unique commit tag |
| Build and push image | Builds the Dockerfile and publishes two tags |

The workflow publishes:

- A unique tag based on the commit SHA, for traceability.
- A `latest` tag, for convenient testing.

The workflow uses the repository owner's name for the image path, so it works with personal forks without hardcoding a username.

## 9. Commit and Push the Changes

Run these commands from the repository root:

```
git add Dockerfile.frontend .github/workflows/ci-ghcr.yml

git commit -m "Add frontend image build and publish workflow"

git push -u origin feature/lab
```

The push triggers the workflow when either of the configured files changes on the `feature/lab` branch.

## 10. Test the GitHub Actions Workflow

1. Open your fork on GitHub.
2. Select the **Actions** tab.
3. Open **Build and Publish Frontend**.
4. Select the latest run.
5. Confirm that every step completes successfully.

You can also trigger the workflow manually:

1. Open the workflow in the Actions tab.
2. Click **Run workflow**.
3. Select the `feature/lab` branch.
4. Click **Run workflow**.

### Validate the published image

After a successful run, check the Packages section of your GitHub profile.

Alternatively, pull the image using the commit SHA from the workflow run:

```
docker pull ghcr.io/YOUR_GITHUB_USERNAME/online-boutique-frontend:YOUR_COMMIT_SHA
```

Replace the placeholders with your lowercase GitHub username and the full commit SHA.

Inspect the downloaded image:

```
docker image inspect \
  ghcr.io/YOUR_GITHUB_USERNAME/online-boutique-frontend:YOUR_COMMIT_SHA
```

## 11. Troubleshooting

| Issue | Recommended action |
| --- | --- |
| Docker daemon unavailable | Start Docker Desktop and retry `docker info`. |
| Base image pull fails | Verify network access and the availability of the base image. |
| `docker login` fails | Verify the username, token, token permissions, and organization policies. |
| GHCR push returns `denied` | Confirm package write access and account or organization restrictions. |
| GitHub Actions fails during login or push | Verify that `packages: write` is present and repository Actions policies permit `GITHUB_TOKEN` package access. |
| Workflow does not trigger | Confirm the branch name and the changed file paths match the workflow filters. |
| Package is private | Review package visibility and access settings before attempting to pull it from another environment. |

## 12. Security and Team Standards

- Never commit access tokens, passwords, or other credentials.
- Use `GITHUB_TOKEN` for publishing from GitHub Actions instead of a personal access token.
- Keep `packages: write` scoped to the workflow that publishes the image.
- Prefer immutable commit-SHA tags for deployment and rollback.
- Treat `latest` as a convenience tag, not a reliable production deployment identifier.
- Verify package visibility and access permissions before using the image in another repository or deployment environment.

**Final outcome:** The frontend image can be built locally for testing and automatically published to GHCR whenever the configured lab files change on the `feature/lab` branch.
