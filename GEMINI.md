# NetBox Operator Gemini Context

This document provides context for the Gemini AI assistant regarding the `netbox-operator` project.

## Project Overview

This is a Kubernetes operator for managing resources in [NetBox](https://netbox.io/), a popular IP Address Management (IPAM) and Data Center Infrastructure Management (DCIM) tool. The operator extends the Kubernetes API with Custom Resource Definitions (CRDs) that represent NetBox objects. This allows users to manage NetBox resources like IP addresses, prefixes, and IP ranges declaratively using Kubernetes manifests and `kubectl`.

The project is built using the [Go programming language](https://go.dev/) and the [Kubebuilder framework](https://book.kubebuilder.io/).

### Key Features:

-   **Declarative Management**: Manage NetBox IPAM resources (IPAddress, Prefix, IPRange) as native Kubernetes objects.
-   **Claim Model**: Dynamically request IP addresses or prefixes from a parent prefix using `IpAddressClaim` and `PrefixClaim` resources, similar to PersistentVolumeClaims in Kubernetes.
-   **Reconciliation Loop**: The operator continuously monitors the state of its custom resources and ensures the corresponding objects in NetBox match the desired state defined in the Kubernetes API.
-   **Finalizers for Cleanup**: Uses Kubernetes finalizers to ensure that when a custom resource is deleted, the corresponding object in NetBox is also removed (unless configured otherwise).

### Architecture

The operator runs as a single controller manager process within a Kubernetes cluster. It contains several reconcilers (controllers), one for each CRD it manages:

-   `IpAddressReconciler`
-   `IpAddressClaimReconciler`
-   `PrefixReconciler`
-   `PrefixClaimReconciler`
-   `IpRangeReconciler`
-   `IpRangeClaimReconciler`

These controllers watch for changes to their respective resources and interact with the NetBox API to create, update, or delete objects.

## Building and Running

The project uses a `Makefile` to automate common development tasks.

### Prerequisites

-   Go 1.25+
-   Docker
-   `kubectl`
-   A running Kubernetes cluster (like `kind`)

### Key Commands

-   **Run unit tests**:
    ```sh
    make test
    ```

-   **Run integration tests**:
    ```sh
    make integration-test
    ```

-   **Run end-to-end (e2e) tests**: (Requires a `kind` cluster)
    ```sh
    # Example for a specific NetBox version
    make test-e2e-4.1.11
    ```

-   **Build the operator binary**:
    ```sh
    make build
    ```

-   **Build the Docker image**:
    ```sh
    make docker-build IMG=your-registry/netbox-operator:tag
    ```

-   **Run the operator locally** (outside the cluster, using local kubeconfig):
    ```sh
    make run
    ```

-   **Deploy to a Kubernetes cluster**: This uses `kustomize` to build the manifests.
    ```sh
    # First, set the image for the deployment
    kustomize edit set image controller=your-registry/netbox-operator:tag
    
    # Then, apply the manifests
    make deploy
    ```

-   **Deploy to a local `kind` cluster** (for development):
    ```sh
    # Create the kind cluster with NetBox pre-installed
    make create-kind

    # Build the local image and deploy the operator
    make deploy-kind
    ```

## Development Conventions

-   **Code Generation**: The project uses `controller-gen` (invoked via `make manifests` and `make generate`) to generate CRD manifests, RBAC roles, and deepcopy functions. Any changes to the `api/v1/*_types.go` files will require running these targets.
-   **Linting**: Code is linted using `golangci-lint`. Run `make lint` to check for style issues.
-   **Dependencies**: Go modules are used for dependency management. Tooling dependencies (like `kustomize`, `controller-gen`) are managed via the `Makefile` and installed into the `/bin` directory.
-   **CRD Schema**: The API types are defined in `api/v1/`. Each `*_types.go` file defines the `Spec` (desired state) and `Status` (observed state) for a Custom Resource.
-   **Controller Logic**: The core reconciliation logic resides in `internal/controller/`. Each `*_controller.go` file contains the `Reconcile` function for a specific CRD.
