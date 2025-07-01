# Custom Rancher Build and Release Workflow

This repository contains a GitHub Actions workflow (`push-release.yml`) designed to build and release custom Docker images and Helm charts for Rancher version 2.10.6. The official Rancher images and Helm charts for versions 2.10.4, 2.10.5, and 2.10.6 are currently behind the Rancher Prime Paywall, making them inaccessible publicly without a Rancher Prime subscription. This workflow provides an alternative by building and hosting these components in a custom Google Artifact Registry, potentially for internal use or to bypass the paywall restrictions.

## Workflow Overview

The workflow, named `build-docker-images`, automates the process of building and releasing the following components:
- Rancher server Docker image
- Rancher agent Docker image
- Rancher installer Docker image
- Rancher Helm chart

It is triggered on pushes to specific branches and tags:
- **Branches**:
  - `2.10.6-branch-custom-automation`
  - Any branch matching the pattern `release/v*` (e.g., `release/v2.10.6`)
- **Tags**:
  - Any tag matching the pattern `v*` (e.g., `v2.10.6`)

The workflow builds these components for the `amd64` architecture (mapped from `x64`) on a Linux operating system and pushes them to a custom Google Artifact Registry.

## Environment Variables

The workflow defines the following environment variables at the top level:
- `COMMIT`: The SHA of the commit that triggered the workflow (`${{ github.sha }}`).
- `REPOSITORY_OWNER`: Set to `rock-atlas-424720-e5/rancher-images`, indicating the custom repository owner.
- `IMAGE`, `IMAGE_AGENT`, `IMAGE_INSTALLER`: All set to `rock-atlas-424720-e5/rancher-images/rancher-images`, defining the base image name for server, agent, and installer images.
- `REGISTRY`: Set to `us-east1-docker.pkg.dev`, specifying the Google Artifact Registry location.
- `TAG`: Hardcoded to `2.10.6`, indicating that this workflow is tailored specifically for Rancher version 2.10.6.

## Jobs

The workflow consists of eight jobs, each with a specific role in the build and release process. All jobs run on `ubuntu-latest` runners and require authentication to Google Artifact Registry using the `GCP_ARTIFACT_REGISTRY_KEY` secret.

### 1. `build-server`
- **Purpose**: Builds the Rancher server Docker image.
- **Key Steps**:
  - Maps `x64` architecture to `amd64`.
  - Checks out the repository code.
  - Installs `yq` (a YAML processor).
  - Authenticates to Google Artifact Registry.
  - Sets up environment variables and Docker metadata.
  - Creates a k3s images file and downloads metadata from Rancher's Kontainer Driver Metadata.
  - Builds the server image using `docker/build-push-action`, exports it to a tar file (e.g., `/tmp/rancher-linux-amd64.tar`), and uploads it as an artifact.
- **Output**: A tar file artifact containing the server image.

### 2. `build-agent`
- **Purpose**: Builds the Rancher agent Docker image.
- **Depends on**: `build-server`
- **Key Steps**:
  - Similar to `build-server`, but targets the "agent" stage in the Dockerfile.
  - Exports the agent image to a tar file (e.g., `/tmp/rancher-agent-linux-amd64.tar`) and uploads it as an artifact.
- **Output**: A tar file artifact containing the agent image.

### 3. `push-images`
- **Purpose**: Pushes the server and agent images to the Google Artifact Registry.
- **Depends on**: `build-agent`
- **Key Steps**:
  - Downloads the server and agent image tar files from artifacts.
  - Loads the images into Docker.
  - Tags them (e.g., `us-east1-docker.pkg.dev/rock-atlas-424720-e5/rancher-images/rancher-images:2.10.6-server-amd64`).
  - Pushes the tagged images to the registry.

### 4. `build-publish-chart`
- **Purpose**: Builds and publishes the Rancher Helm chart.
- **Depends on**: `push-images`
- **Key Steps**:
  - Installs dependencies (`jq`, `gawk`).
  - Authenticates to Google Cloud and sets up the Google Cloud SDK.
  - Installs Helm and the `helm-unittest` plugin.
  - Builds, validates, and packages the Helm chart using custom scripts.
  - Pushes the packaged chart to `oci://us-east1-docker.pkg.dev/rock-atlas-424720-e5/rancher-images`.

### 5. `merge-server-manifest`
- **Purpose**: Creates a multi-architecture manifest for the server image.
- **Depends on**: `push-images`
- **Key Steps**:
  - Sets up Docker Buildx.
  - Creates a manifest list (e.g., `us-east1-docker.pkg.dev/rock-atlas-424720-e5/rancher-images/rancher-images:2.10.6-server`) combining architecture-specific images (currently only `amd64`).
  - Inspects the manifest to verify.

### 6. `merge-agent-manifest`
- **Purpose**: Creates a multi-architecture manifest for the agent image.
- **Depends on**: `push-images`
- **Key Steps**:
  - Similar to `merge-server-manifest`, but for the agent image (e.g., `us-east1-docker.pkg.dev/rock-atlas-424720-e5/rancher-images/rancher-images:2.10.6-agent`).

### 7. `build-installer`
- **Purpose**: Builds and pushes the Rancher installer Docker image.
- **Depends on**: `merge-server-manifest`
- **Key Steps**:
  - Downloads the official Rancher Helm chart (version 2.10.6) from `https://releases.rancher.com`.
  - Builds the installer image using `Dockerfile.installer` and pushes it directly to the registry (e.g., `us-east1-docker.pkg.dev/rock-atlas-424720-e5/rancher-images/rancher-images:2.10.6-installer-amd64`).

### 8. `merge-installer-manifest`
- **Purpose**: Creates a multi-architecture manifest for the installer image.
- **Depends on**: `build-installer`
- **Key Steps**:
  - Creates a manifest list (e.g., `us-east1-docker.pkg.dev/rock-atlas-424720-e5/rancher-images/rancher-images:2.10.6-installer`) for the installer image.

## Key Notes
- **Version Specificity**: The workflow is hardcoded to version `2.10.6` via the `TAG` variable. To support other versions (e.g., 2.10.4 or 2.10.5), you would need to modify this variable and related settings.
- **Custom Repository**: All images and the Helm chart are stored in `us-east1-docker.pkg.dev/rock-atlas-424720-e5/rancher-images`, a custom Google Artifact Registry repository.
- **Authentication**: The workflow requires the `GCP_ARTIFACT_REGISTRY_KEY` secret for Google Cloud authentication.
- **Purpose**: By building and hosting these components separately, this workflow enables access to Rancher 2.10.6 components outside the Rancher Prime Paywall, likely for custom or internal deployments.
- **Architecture**: Currently, only `amd64` (x64) is supported in the matrix strategy, but the manifest jobs allow for future multi-architecture support.

This workflow provides a comprehensive solution for building and distributing Rancher components independently of the official paywalled releases.
