# Rancher GitHub Actions Workflows and Actions

This document provides a detailed overview of the GitHub Actions workflows and composite actions used in the Rancher repository to build, test, and release Rancher images (server, agent, and installer). It focuses on the key files involved in the release process, their paths under the `.github` directory, and their roles in the pipeline. This is particularly relevant for users who have forked the Rancher repository (e.g., `kalexiwells/rancher`) and aim to build a specific version (e.g., 2.10.3) and push images to a custom Docker Hub repository (e.g., `docker.io/kalexiwells/rancher-images:tagname`).

## Overview

The Rancher release pipeline automates the process of building Docker images for the Rancher server, agent, and system-agent-installer, running tests, pushing images to a Docker registry, and creating multi-architecture manifests. The pipeline is orchestrated by workflows in the `.github/workflows` directory, which call composite actions in the `.github/actions` directory. The primary workflow for tagged releases (e.g., `v2.10.3`) is `release.yml`, which triggers on tags matching `v*` (excluding hotfix tags).

Key components include:
- **Building images**: Actions to build server, agent, and installer images for Linux (amd64, arm64) and Windows (agent only).
- **Testing**: Unit and integration tests to validate the images.
- **Pushing images**: Pushing architecture-specific images to a registry (e.g., `docker.io`).
- **Merging manifests**: Creating multi-architecture manifests for cross-platform compatibility.
- **Environment setup**: Actions to configure tags and dependency versions.

## Workflows

The following workflows in `.github/workflows` are critical to the release process.

### release.yml
- **Path**: `.github/workflows/release.yml`
- **Purpose**: Orchestrates the release process for Rancher when a tag matching `v*` (e.g., `v2.10.3`) is pushed, excluding hotfix tags (`v*-hotfix*`). It builds, tests, and pushes Rancher server, agent, and installer images, creates multi-architecture manifests, publishes Helm charts, generates image lists and digests, and notifies the release.
- **Key Jobs**:
  - **unit-tests**: Calls `unit-test.yml` to run Go unit tests.
  - **build-server**: Builds server images for Linux (amd64, arm64) using `build-images/server/action.yml`.
  - **build-agent**: Builds agent images for Linux (amd64, arm64) using `build-images/agent/action.yml`.
  - **build-agent-windows**: Builds agent images for Windows (2019, 2022) using `build-images/agent-windows/action.yml`.
  - **integration-tests**: Runs integration tests using `integration-tests.yml`, testing server and agent images.
  - **push-images**: Pushes images to `docker.io/<owner>/rancher`, `rancher-agent`, and `system-agent-installer-rancher` using `push-images/action.yml`.
  - **merge-server-manifest**, **merge-agent-manifest**, **merge-installer-manifest**: Create multi-architecture manifests using `merge-manifests/server/action.yml`, `merge-agent-manifest/action.yml`, and `merge-installer-manifest/action.yml`.
  - **build-installer**: Builds installer images for Linux (amd64, arm64) using `build-images/installer/action.yml`.
  - **build-publish-chart**: Builds and publishes Helm charts.
  - **create-images-files**, **docker-image-digests**: Generate and publish image lists and digests.
  - **notify-release**: Sends a notification (e.g., to Slack) upon release completion.
- **Environment Variables**:
  - `REPOSITORY_OWNER`: `${{ github.repository_owner }}` (e.g., `rancher` or `kalexiwells` in a fork).
  - `IMAGE`: `${{ github.repository_owner }}/rancher` (server image).
  - `IMAGE_AGENT`: `${{ github.repository_owner }}/rancher-agent`.
  - `IMAGE_INSTALLER`: `${{ github.repository_owner }}/system-agent-installer-rancher`.
  - `REGISTRY`: `docker.io` (Docker Hub).
  - `COMMIT`: `${{ github.sha }}`.
- **Notes for Forked Repositories**:
  - To push to `docker.io/kalexiwells/rancher-images`, update `IMAGE` to `kalexiwells/rancher-images`, `IMAGE_AGENT` to `kalexiwells/rancher-agent`, and `IMAGE_INSTALLER` to `kalexiwells/system-agent-installer-rancher`.
  - Replace vault-based secrets with GitHub Actions secrets (`DOCKER_USERNAME`, `DOCKER_PASSWORD`) for Docker Hub authentication.
  - Ensure the `v2.10.3` tag exists and includes `package/Dockerfile` and `package/Dockerfile.installer`.

### push.yml
- **Path**: `.github/workflows/push.yml`
- **Purpose**: Similar to `release.yml`, but triggers on pushes to `main` or `release/v*` branches (e.g., `release/v2.10`). It builds, tests, and pushes images, merges manifests, and generates image files, primarily for development or pre-release builds.
- **Key Differences from `release.yml`**:
  - Triggered by branch pushes, not tags.
  - `build-installer` depends on `merge-server-manifest`, unlike `release.yml` where it’s independent.
  - Publishes image files to AWS S3 instead of GitHub.
- **Notes for Forked Repositories**:
  - Less relevant for building `v2.10.3`, as `release.yml` is the correct workflow for tagged releases.
  - Useful for testing changes in a branch before tagging.

### unit-test.yml
- **Path**: `.github/workflows/unit-test.yml`
- **Purpose**: Runs Go unit tests (`go test -cover -tags=test ./pkg/...`) on the codebase. Called by `release.yml` and `push.yml` as a dependency for `push-images`.
- **Key Details**:
  - Runs on `ubuntu-latest` with a 60-minute timeout.
  - Simple workflow that checks out the code and runs tests.
- **Notes for Forked Repositories**:
  - Ensure tests pass for `v2.10.3` or skip by removing the `needs: [unit-tests]` dependency in `release.yml` if tests are not required.
  - Verify Go version compatibility (likely Go 1.23, as seen in `integration-tests.yml`).

### integration-tests.yml
- **Path**: `.github/workflows/integration-tests.yml`
- **Purpose**: Runs integration tests using server and agent images built by `build-server` and `build-agent`. It downloads artifacts (tar files), loads them into Docker, sets up a k3d cluster, and runs tests.
- **Key Details**:
  - Called by `release.yml` and `push.yml` with `parent_run_id` to download artifacts.
  - Tests only amd64 images (`ARCH: amd64`), loading `rancher-linux-amd64.tar` and `rancher-agent-linux-amd64.tar`.
  - Tags images as `<owner>/rancher:<TAG>` and `<owner>/rancher-agent:<TAG>` (e.g., `kalexiwells/rancher-images:v2.10.3`).
  - Requires Python 3.11, Go 1.23, k3d v5.7.1, and `data.json` from `https://releases.rancher.com/kontainer-driver-metadata/<CATTLE_KDM_BRANCH>/data.json`.
  - Uses custom actions (`setup-tag-env`, `setup-build-env`) for environment setup.
- **Notes for Forked Repositories**:
  - Ensure `CATTLE_KDM_BRANCH` (likely `release/v2.10` for 2.10.3) is valid and `data.json` is accessible.
  - Skip tests by removing `needs: [integration-tests]` in `release.yml` if not needed, as they only test amd64 and may be resource-intensive.
  - Update `IMAGE` and `IMAGE_AGENT` to `kalexiwells/rancher-images` and `kalexiwells/rancher-agent`.

### hotfix-release.yml
- **Path**: `.github/workflows/hotfix-release.yml`
- **Purpose**: Handles hotfix releases (tags like `v*-hotfix*`), building and pushing images to `stgregistry.suse.com` instead of `docker.io`. Similar to `release.yml` but with a different registry and credentials.
- **Key Details**:
  - Uses the same build actions (`build-images/server`, `build-images/agent`, `build-images/installer`) and push action (`push-images`).
  - Uses vault secrets for `stgregistry.suse.com` authentication.
- **Notes for Forked Repositories**:
  - Not relevant for building `v2.10.3`, as it’s not a hotfix. Use `release.yml` for `docker.io`.

## Actions

The following composite actions in `.github/actions` are used by the workflows to perform specific tasks.

### build-images/server/action.yml
- **Path**: `.github/actions/build-images/server/action.yml`
- **Purpose**: Builds the Rancher server image for Linux (amd64 or arm64), exports it as a tar file, and uploads it as an artifact.
- **Key Steps**:
  - Sets `ARCH` to `amd64` if `x64`.
  - Installs `yq` using `install-yq/action.yml`.
  - Sets up environment variables using `setup-tag-env/action.yml` and `setup-build-env/action.yml`.
  - Configures Docker metadata using `docker/metadata-action@v5`.
  - Sets up QEMU and Docker Buildx for multi-architecture builds.
  - Creates a k3s images file using `k3s-images/action.yml`.
  - Downloads `data.json` from `releases.rancher.com`.
  - Builds the image using `docker/build-push-action@v5` with `package/Dockerfile`, targeting the `server` stage.
  - Tags the image as `<owner>/rancher:<TAG>-<ARCH>` (e.g., `kalexiwells/rancher-images:v2.10.3-amd64`).
  - Exports the image to `/tmp/rancher-linux-<ARCH>.tar` and uploads it as `rancher-linux-<ARCH>`.
- **Notes for Forked Repositories**:
  - Update the `images` field in the `Docker meta` step to `kalexiwells/rancher-images`.
  - Ensure `package/Dockerfile` exists at the `v2.10.3` tag.
  - Verify `CATTLE_KDM_BRANCH` (e.g., `release/v2.10`) and dependency versions (e.g., `CATTLE_K3S_VERSION`) are compatible.

### build-images/agent/action.yml
- **Path**: `.github/actions/build-images/agent/action.yml`
- **Purpose**: Builds the Rancher agent image for Linux (amd64 or arm64), exports it as a tar file, and uploads it as an artifact.
- **Key Steps**:
  - Similar to the server action, sets `ARCH`, installs `yq`, and configures environment variables.
  - Builds the image using `package/Dockerfile`, targeting the `agent` stage.
  - Tags the image as `<owner>/rancher-agent:<TAG>-<ARCH>` (e.g., `kalexiwells/rancher-agent:v2.10.3-amd64`).
  - Exports to `/tmp/rancher-agent-linux-<ARCH>.tar` and uploads as `rancher-agent-linux-<ARCH>`.
- **Notes for Forked Repositories**:
  - Update the `images` field in the `Docker meta` step to `kalexiwells/rancher-agent`.
  - Ensure `package/Dockerfile` supports the `agent` target for 2.10.3.

### build-images/installer/action.yml
- **Path**: `.github/actions/build-images/installer/action.yml`
- **Purpose**: Builds the Rancher system-agent-installer image for Linux (amd64 or arm64) and pushes it directly to the registry.
- **Key Steps**:
  - Sets `ARCH` and environment variables using `setup-tag-env/action.yml`.
  - Configures Docker metadata for `<owner>/system-agent-installer-rancher`.
  - Logs in to the registry using `DOCKER_USERNAME` and `DOCKER_PASSWORD`.
  - Builds and pushes the image using `package/Dockerfile.installer`.
  - Tags the image as `<REGISTRY>/<owner>/system-agent-installer-rancher:<TAG>-<ARCH>` (e.g., `docker.io/kalexiwells/system-agent-installer-rancher:v2.10.3-amd64`).
- **Notes for Forked Repositories**:
  - Update the `images` field in the `Docker meta` step to `kalexiwells/system-agent-installer-rancher`.
  - Ensure `package/Dockerfile.installer` exists at the `v2.10.3` tag.
  - Set GitHub Actions secrets (`DOCKER_USERNAME`, `DOCKER_PASSWORD`) for Docker Hub.

### push-images/action.yml
- **Path**: `.github/actions/push-images/action.yml`
- **Purpose**: Downloads server and agent image artifacts, loads them into Docker, and pushes them to the registry.
- **Key Steps**:
  - Sets `ARCH` and environment variables.
  - Downloads artifacts matching `*-linux-<ARCH>` (e.g., `rancher-linux-amd64.tar`, `rancher-agent-linux-arm64.tar`).
  - Logs in to the registry (`docker.io`).
  - Loads and tags the server image as `<REGISTRY>/<owner>/rancher:<TAG>-<ARCH>`.
  - Loads and tags the agent image as `<REGISTRY>/<owner>/rancher-agent:<TAG>-<ARCH>`.
  - Pushes both images to the registry.
- **Notes for Forked Repositories**:
  - Ensure `REGISTRY` is `docker.io` and `REPOSITORY_OWNER` is `kalexiwells`.
  - Use GitHub Actions secrets for authentication.

### merge-manifests/server/action.yml
- **Path**: `.github/actions/merge-manifests/server/action.yml`
- **Purpose**: Creates a multi-architecture manifest for the Rancher server image, combining amd64 and arm64 images.
- **Key Steps**:
  - Sets up environment variables and Docker Buildx.
  - Logs in to the registry.
  - Creates a manifest for `<REGISTRY>/<owner>/rancher:<TAG>` using `docker buildx imagetools create`.
  - Optionally creates a `HEAD_TAG` manifest for `main` or `release/v*` branches.
  - Inspects the manifest for verification.
- **Notes for Forked Repositories**:
  - Update the manifest tag to `docker.io/kalexiwells/rancher-images:v2.10.3`.
  - Required for multi-arch images; skip if only amd64 is needed.

### merge-manifests/agent/action.yml
- **Path**: `.github/actions/merge-manifests/agent/action.yml`
- **Purpose**: Creates a multi-architecture manifest for the Rancher agent image, combining Linux (amd64, arm64) and Windows (2019, 2022) images.
- **Key Steps**:
  - Uses `docker manifest create` for Windows images to preserve `os.version`.
  - Uses `docker buildx imagetools create` for Linux images.
  - Creates manifests for `<REGISTRY>/<owner>/rancher-agent:<TAG>` and optionally `<HEAD_TAG>`.
- **Notes for Forked Repositories**:
  - Update to `docker.io/kalexiwells/rancher-agent:v2.10.3`.
  - Skip Windows images (`build-agent-windows`) if not needed.

### merge-manifests/installer/action.yml
- **Path**: `.github/actions/merge-manifests/installer/action.yml`
- **Purpose**: Creates a multi-architecture manifest for the Rancher installer image (amd64, arm64).
- **Key Steps**:
  - Similar to the server manifest action, creates a manifest for `<REGISTRY>/<owner>/system-agent-installer-rancher:<TAG>`.
  - Optionally creates a `HEAD_TAG` manifest for `release/v*` branches.
- **Notes for Forked Repositories**:
  - Update to `docker.io/kalexiwells/system-agent-installer-rancher:v2.10.3`.
  - Required for multi-arch installer images.

### setup-tag-env/action.yml
- **Path**: `.github/actions/setup-tag-env/action.yml`
- **Purpose**: Sets `TAG` and `HEAD_TAG` environment variables based on the Git reference.
- **Key Logic**:
  - For tags (`refs/tags/*`), sets `TAG` to the tag name (e.g., `v2.10.3`).
  - For `main`, sets `TAG` to `v2.12-<sha>-head` and `HEAD_TAG` to `head`.
  - For `release/v*` branches, sets `TAG` to `<branch>-<sha>-head` and `HEAD_TAG` to `<branch>-head`.
  - Sets `GIT_TAG` to match `TAG`.
- **Notes for Forked Repositories**:
  - For `v2.10.3`, `TAG` will be `v2.10.3`, ensuring correct image tagging.
  - Verify the logic works in your fork; no changes needed for tagged releases.

### setup-build-env/action.yml
- **Path**: `.github/actions/setup-build-env/action.yml`
- **Purpose**: Sets environment variables for dependency versions used in building and testing Rancher images. It extracts versions from various files and scripts, ensuring consistency across the pipeline.
- **Key Steps**:
  - Sources `scripts/export-config` to load variables like `CATTLE_RANCHER_WEBHOOK_VERSION`.
  - Extracts versions from files:
    - `CATTLE_KDM_BRANCH`: From `package/Dockerfile` (e.g., `release/v2.10` for 2.10.3).
    - `CATTLE_K3S_VERSION`: From `package/Dockerfile` (e.g., a specific k3s version).
    - `HELM_VERSION`: From `package/Dockerfile.installer` (Helm version for the installer).
    - `GO_VERSION`: From `package/Dockerfile` (Go version for the build).
    - `HELM_UNITTEST_VERSION`: From `Dockerfile.dapper` (for Helm unit tests).
    - `RKE_VERSION`: From `go.mod` (Rancher Kubernetes Engine version).
    - `CATTLE_RANCHER_WEBHOOK_VERSION`, `CATTLE_REMOTEDIALER_PROXY_VERSION`, `CATTLE_RANCHER_PROVISIONING_CAPI_VERSION`, `CATTLE_CSP_ADAPTER_MIN_VERSION`, `CATTLE_FLEET_VERSION`: From `scripts/export-config`.
  - Outputs these variables to `$GITHUB_OUTPUT` for use in other actions (e.g., `build-images/server`, `integration-tests`).
  - Fails if any variable is not found, ensuring all dependencies are defined.
- **Notes for Forked Repositories**:
  - Verify that `package/Dockerfile`, `package/Dockerfile.installer`, `Dockerfile.dapper`, `go.mod`, and `scripts/export-config` exist at the `v2.10.3` tag and contain valid version information.
  - Ensure `CATTLE_KDM_BRANCH` (e.g., `release/v2.10`) is accessible at `https://releases.rancher.com/kontainer-driver-metadata/<CATTLE_KDM_BRANCH>/data.json`.
  - Check compatibility of versions (e.g., `CATTLE_K3S_VERSION`, `GO_VERSION`) with 2.10.3 to avoid build or test failures.

## Considerations for Building Rancher 2.10.3 in a Forked Repository

To build Rancher 2.10.3 and push to `docker.io/kalexiwells/rancher-images:tagname`, adapt the `release.yml` workflow as follows:

1. **Update Repository Names**:
   - Modify `release.yml` environment variables:
     ```yaml
     env:
       REPOSITORY_OWNER: kalexiwells
       IMAGE: kalexiwells/rancher-images
       IMAGE_AGENT: kalexiwells/rancher-agent
       IMAGE_INSTALLER: kalexiwells/system-agent-installer-rancher
       REGISTRY: docker.io
     ```
   - Update actions (`build-images/server`, `build-images/agent`, `merge-manifests/*`) to use `kalexiwells/rancher-images`, `kalexiwells/rancher-agent`, and `kalexiwells/system-agent-installer-rancher`.

2. **Configure Docker Hub Authentication**:
   - Replace vault-based secrets in `push-images`, `build-installer`, `merge-manifests/*` with:
     ```yaml
     - name: Log in to Docker Hub
       uses: docker/login-action@v3
       with:
         registry: ${{ env.REGISTRY }}
         username: ${{ secrets.DOCKER_USERNAME }}
         password: ${{ secrets.DOCKER_PASSWORD }}
     ```
   - Add GitHub Actions secrets:
     - `DOCKER_USERNAME`: `kalexiwells`
     - `DOCKER_PASSWORD`: Your Docker Hub access token or password

3. **Simplify the Pipeline**:
   - Skip `unit-tests` and `integration-tests` by removing `needs: [unit-tests, integration-tests]` from `push-images` if tests are not required.
   - Skip `build-agent-windows` and Windows-specific steps in `merge-agent-manifest` if only Linux images are needed.
   - Skip `build-publish-chart`, `create-images-files`, `docker-image-digests`, and `notify-release` unless you need Helm charts or additional artifacts.

4. **Handle Multi-Architecture Images**:
   - Keep `merge-manifests/*` jobs for multi-arch images (`amd64`, `arm64`).
   - For single-architecture (e.g., `amd64`), remove `arm64` from the matrix and skip manifest jobs.

5. **Verify Dependency Versions**:
   - The `setup-build-env/action.yml` extracts critical versions:
     - `CATTLE_KDM_BRANCH`: Ensure it’s set to `release/v2.10` for 2.10.3 and that `https://releases.rancher.com/kontainer-driver-metadata/release/v2.10/data.json` is accessible.
     - `CATTLE_K3S_VERSION`: Verify the k3s version is compatible with 2.10.3 (check `package/Dockerfile` at the `v2.10.3` tag).
     - `GO_VERSION`: Ensure the Go version (e.g., 1.23 or earlier) is supported in your build environment.
     - `RKE_VERSION`, `CATTLE_RANCHER_WEBHOOK_VERSION`, etc.: Confirm these versions are valid in `go.mod` and `scripts/export-config` at the `v2.10.3` tag.
   - Check that `package/Dockerfile`, `package/Dockerfile.installer`, `Dockerfile.dapper`, `go.mod`, and `scripts/export-config` exist and contain the expected version information.

6. **Ensure Compatibility**:
   - Verify the `v2.10.3` tag exists in your fork (`git tag -l | grep v2.10.3`).
   - Confirm `package/Dockerfile` and `package/Dockerfile.installer` exist at `v2.10.3`.
   - Ensure `scripts/export-config` is present and defines variables like `CATTLE_RANCHER_WEBHOOK_VERSION`.

7. **Trigger the Build**:
   - Push the `v2.10.3` tag to your fork:
     ```bash
     git checkout v2.10.3
     git push origin v2.10.3
     ```
   - Monitor the GitHub Actions run to ensure images are pushed to `docker.io/kalexiwells/rancher-images:v2.10.3`, `kalexiwells/rancher-agent:v2.10.3`, and `kalexiwells/system-agent-installer-rancher:v2.10.3`.

## Additional Notes

- **Runners**: Custom runners (`runs-on,runner=4cpu-<os>-<arch>`) may not be available in your fork. Update `runs-on` to `ubuntu-latest` or a compatible runner.
- **Tagname**: The `tagname` in `kalexiwells/rancher-images:tagname` will be `v2.10.3` unless customized in `setup-tag-env/action.yml`.
- **Testing**: If tests are required, ensure `unit-test.yml` and `integration-tests.yml` pass for 2.10.3, particularly the k3d-based integration tests. The `setup-build-env/action.yml` confirms that `GO_VERSION` and `CATTLE_K3S_VERSION` must align with test requirements.

This README provides a foundation for understanding and adapting the Rancher release pipeline. For further assistance, provide details about specific requirements (e.g., custom tag names, skipping multi-arch manifests) or any issues encountered during the build process.