# Description of GitHub Actions in this repository

## Go Get (`go-get.yml`)

Go Get can be used to automate updating Go modules in this repository. It will run `make go-get` which is a helper script for running `go get -d $GOGET_MODULE@$GOGET_VERSION` in all needed places, commit and create a pull request.

If `Username of the source for this workflow run` is set, the username will be mentioned in the pull request and configured as assignee. This was added for automated workflows, where the user and URL can be used to link back to the source of the trigger.

If `URL of the source for this workflow run` is set, the URL will be mentioned in the pull request. This was added for automated workflows, where the user and URL can be used to link back to the source of the trigger.

## Replace environment variable in file (`replace-env-value.yml`)

Replace environment variable in file can be used to replace an environment variable in any file. It will run `./scripts/replace-env-value-in-file.sh`, commit and create a pull request with the changes. This is mostly used for bumping versions. This workflow should work for `KEY=VALUE` (equal sign separated) values and `KEY VALUE` (space separated) values.

# Rancher Image Build Workflow

This repository contains a GitHub Actions workflow (defined in `.github/workflows/push-release.yml`) for building, packaging, and pushing Docker images and a Helm chart for Rancher version 2.10.4. The workflow is tailored for a forked repository where images are pushed to a custom Google Cloud Artifact Registry (configured via environment variables like `REGISTRY`, `REPOSITORY_OWNER`, and secrets such as `GCP_ARTIFACT_REGISTRY_KEY`).

The workflow automates the process of creating Rancher server, agent, and installer images, along with publishing the associated Helm chart. It supports AMD64 architecture (x64) on Linux and uses multi-stage builds to handle dependencies, image creation, and manifest merging for multi-architecture support.

## Purpose
This workflow is designed to:
- Build and validate the Rancher Helm chart.
- Build Docker images for the Rancher server, agent, and installer.
- Push the images to your custom Artifact Registry.
- Create and push multi-architecture manifests for easier image pulling.

It triggers on pushes to the `2.10.4-branch` or `release/v*` branches, or when tags matching `v*` are created. The tag is hardcoded to `2.10.4` for this specific version.

**Note:** This is based on a forked Rancher repository. You've mentioned editing `push-release.yml` to point to your Artifact Registry—ensure that secrets (e.g., `GCP_ARTIFACT_REGISTRY_KEY`) and variables (e.g., `GCP_PROJECT_ID`) are set in your repository settings for authentication.

## Environment Variables
The workflow uses the following key environment variables:
- `COMMIT`: Set to the GitHub SHA of the commit.
- `REPOSITORY_OWNER`: Set to `${{ vars.GCP_PROJECT_ID }}/rancher-images` (customize via repo variables).
- `IMAGE`, `IMAGE_AGENT`, `IMAGE_INSTALLER`: All point to `${{ vars.GCP_PROJECT_ID }}/rancher-images/rancher-images`.
- `REGISTRY`: Set to `us-east1-docker.pkg.dev` (Google Artifact Registry endpoint).
- `TAG`: Hardcoded to `2.10.4`.

These ensure images are tagged like `us-east1-docker.pkg.dev/<project>/rancher-images/rancher-images:2.10.4-server-amd64`.

## Workflow Jobs
The workflow consists of several interdependent jobs, executed in sequence for efficiency (e.g., building the chart first, then images). All jobs run on `ubuntu-latest` runners.

### 1. `build-publish-chart`
   - **Purpose:** Builds, validates, packages, and pushes the Rancher Helm chart to Artifact Registry.
   - **Steps:**
     - Checkout code.
     - Install dependencies (jq, gawk).
     - Authenticate to Google Cloud using service account key.
     - Set up environment variables and install tools like `yq` and Helm.
     - Build the chart using `./scripts/chart/build`.
     - Validate the chart using `./scripts/chart/validate`.
     - Package the chart into a `.tgz` file using `./scripts/chart/package`.
     - Log in to Helm registry and push the chart to `oci://us-east1-docker.pkg.dev/<project>/rancher-images`.
   - **Dependencies:** None (runs first).
   - **Outputs:** Pushed Helm chart (e.g., `rancher-2.10.4.tgz`).

### 2. `build-server`
   - **Purpose:** Builds the Rancher server Docker image for Linux/AMD64.
   - **Steps:**
     - Set architecture variables.
     - Checkout code and install `yq`.
     - Log in to Artifact Registry.
     - Set up environment variables (e.g., versions for dependencies like K3s, RKE).
     - Create a K3s images file and download metadata.
     - Build the image using `docker/build-push-action` with `package/Dockerfile`.
     - Export the image as a tarball and upload as an artifact.
   - **Dependencies:** `build-publish-chart`.
   - **Outputs:** Artifact `rancher-linux-amd64.tar` containing the server image.

### 3. `build-agent`
   - **Purpose:** Builds the Rancher agent Docker image for Linux/AMD64.
   - **Steps:** Similar to `build-server`, but uses `package/Dockerfile.agent` and tags the image with `-agent`.
   - **Dependencies:** `build-server`.
   - **Outputs:** Artifact `rancher-agent-linux-amd64.tar` containing the agent image.

### 4. `push-images`
   - **Purpose:** Downloads built image artifacts and pushes them to Artifact Registry.
   - **Steps:**
     - Download artifacts from previous jobs.
     - Load images from tarballs.
     - Tag and push server and agent images (e.g., `:2.10.4-server-amd64`).
   - **Dependencies:** `build-agent`.
   - **Outputs:** Pushed single-architecture images.

### 5. `merge-server-manifest`
   - **Purpose:** Creates and pushes a multi-architecture manifest for the server image.
   - **Steps:**
     - Log in to Artifact Registry.
     - Use `docker buildx imagetools` to create a manifest from the AMD64 image.
     - Optionally tags with head tag if on release branches.
   - **Dependencies:** `push-images`.
   - **Outputs:** Multi-arch image (e.g., `:2.10.4-server`).

### 6. `merge-agent-manifest`
   - **Purpose:** Similar to `merge-server-manifest`, but for the agent image.
   - **Dependencies:** `push-images`.
   - **Outputs:** Multi-arch image (e.g., `:2.10.4-agent`).

### 7. `build-installer`
   - **Purpose:** Builds the Rancher installer Docker image for Linux/AMD64.
   - **Steps:**
     - Download the Rancher Helm chart (hardcoded to `rancher-2.10.4.tgz` from official releases).
     - Build the image using `package/Dockerfile.installer`.
     - Push directly to Artifact Registry (e.g., `:2.10.4-installer-amd64`).
   - **Dependencies:** `merge-server-manifest`.
   - **Outputs:** Pushed single-architecture installer image.

### 8. `merge-installer-manifest`
   - **Purpose:** Creates and pushes a multi-architecture manifest for the installer image.
   - **Steps:** Similar to other merge jobs, using `docker buildx imagetools`.
   - **Dependencies:** `build-installer`.
   - **Outputs:** Multi-arch image (e.g., `:2.10.4-installer`).

## Usage
1. **Set Up Secrets and Variables:**
   - In your GitHub repo settings:
     - Add `GCP_ARTIFACT_REGISTRY_KEY` as a secret (JSON key for GCP service account with Artifact Registry access).
     - Add `GCP_PROJECT_ID` as a variable.
     - Optionally, add `DOCKER_USERNAME` and `DOCKER_PASSWORD` if needed for additional registries.

2. **Trigger the Workflow:**
   - Push to `2.10.4-branch` or a `release/v*` branch.
   - Or create a tag like `v2.10.4`.

3. **Monitor and Verify:**
   - Check GitHub Actions logs for build/push status.
   - Verify images in your Artifact Registry console (e.g., `us-east1-docker.pkg.dev/<project>/rancher-images/rancher-images:2.10.4-server`).
   - The Helm chart will be available as an OCI artifact.

## Customization
- **Version Changes:** Update the hardcoded `TAG: 2.10.4` in the workflow YAML if building for a different version.
- **Architecture Support:** Currently limited to AMD64; extend the matrix for ARM64 if needed.
- **Registry Changes:** If switching registries, update `REGISTRY` and login steps accordingly.
- **Dependencies:** Versions for components like K3s, RKE, and Fleet are pulled dynamically via `./.github/actions/setup-build-env`.

If you encounter issues, ensure Google Cloud authentication is correct and that the service account has `artifactregistry.writer` permissions.

For more details on Rancher, visit the official [Rancher documentation](https://rancher.com/docs/).
