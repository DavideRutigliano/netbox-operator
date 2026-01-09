This document outlines the tasks required to implement the `VLANClaim` CRD and its corresponding controller, as specified in `SPEC.md`. Use this checklist to track progress.

### `VLANClaim` CRD and Controller Implementation

-   [ ] **0. Prerequisites and Dependencies**
    -   [ ] Add `github.com/k8snetworkplumbingwg/network-attachment-definition-client` to `go.mod`.
    -   [ ] Run `go mod tidy`.

-   [ ] **1. `VLANClaim` API Definition (`api/v1/vlanclaim_types.go`)**
    -   [ ] Define the `VLANClaimSpec` struct with the following fields:
        -   [ ] `vlanId` (int, required)
        -   [ ] `name` (string, required)
        -   [ ] `site` (string, required)
        -   [ ] `sourceRef` (struct, required) containing:
            -   [ ] `apiVersion` (string)
            -   [ ] `kind` (string)
            -   [ ] `name` (string)
            -   [ ] `namespace` (string)
        -   [ ] `vlanGroup` (string, optional)
        -   [ ] `description` (string, optional)
        -   [ ] `comments` (string, optional)
        -   [ ] `customFields` (map[string]string, optional)
        -   [ ] `preserveInNetbox` (bool, optional, default: false)
    -   [ ] Define the `VLANClaimStatus` struct with the following fields:
        -   [ ] `vlanId` (int)
        -   [ ] `vlanUrl` (string)
        -   [ ] `conditions` ([]metav1.Condition)
    -   [ ] Add the `VLANClaim` struct combining `TypeMeta`, `ObjectMeta`, `VLANClaimSpec`, and `VLANClaimStatus`.
    -   [ ] Add the `VLANClaimList` struct.
    -   [ ] Run `make generate` to create the `zz_generated.deepcopy.go` updates.

-   [ ] **2. CRD Manifest Generation**
    -   [ ] Add the necessary `+kubebuilder` markers to the `vlanclaim_types.go` file.
    -   [ ] Run `make manifests` to generate the CRD YAML file in `config/crd/bases/netbox.dev_vlanclaims.yaml`.

-   [ ] **2.5. NetBox Client Extension (`pkg/netbox/api/vlan.go`)**
    -   [ ] Implement `Vlan` interface and methods in `pkg/netbox/api/vlan.go`.
    -   [ ] Implement `GetVlan`, `CreateVlan`, `UpdateVlan`, `DeleteVlan`.
    -   [ ] Add `Vlan()` accessor to the main NetBox client interface.
    -   [ ] Update mocks in `gen/mock_interfaces/netbox_mocks.go`.

-   [ ] **3. `VLANClaim` Controller Implementation (`internal/controller/vlanclaim_controller.go`)**
    -   [ ] Create the new controller file `vlanclaim_controller.go`.
    -   [ ] Implement the `VLANClaimReconciler` struct.
    -   [ ] Implement the `Reconcile` method:
        -   [ ] Fetch the `VLANClaim` resource.
        -   [ ] Add a finalizer to the `VLANClaim` resource for cleanup.
        -   [ ] Handle deletion: if `deletionTimestamp` is set, delete the VLAN in NetBox (if `preserveInNetbox` is false) and remove the finalizer.
        -   [ ] Fetch the `NetworkAttachmentDefinition` referenced in `spec.sourceRef`.
        -   [ ] Derive NetBox VLAN data:
            -   `name` from `{{cluster_name}}-{{source_resource_name}}`.
            -   VLAN `status` (`Active`/`Staged`) based on the `network.harvesterhci.io/ready` label on the NAD.
        -   [ ] Implement NetBox client logic to find an existing VLAN by `vlanId`, `site`, and `vlanGroup`.
        -   [ ] **Conflict Resolution**:
            -   If VLAN exists and is not managed by the operator (missing `managed_by` custom field), take ownership by updating it.
            -   If VLAN exists and is managed, update it with any changes.
            -   If VLAN does not exist, create it.
        -   [ ] Update the `VLANClaim.status` with:
            -   `vlanId` (NetBox object ID).
            -   `vlanUrl`.
            -   `conditions` reflecting the reconciliation state (e.g., `Ready`=True/False).

-   [ ] **4. Controller Setup and RBAC**
    -   [ ] Update `cmd/main.go` to set up and start the `VLANClaimReconciler`.
    -   [ ] Generate RBAC permissions for `VLANClaim` resources (`config/rbac/vlanclaim_editor_role.yaml`, `vlanclaim_viewer_role.yaml`).
    -   [ ] Update the main operator role in `config/rbac/role.yaml` to include permissions for:
        -   [ ] `vlanclaims` (get, list, watch, update, patch, create, delete).
        -   [ ] `vlanclaims/status` (get, update, patch).
        -   [ ] `vlanclaims/finalizers` (update).
        -   [ ] `networkattachmentdefinitions` (get, list, watch) from the `k8s.cni.cncf.io` API group.

-   [ ] **5. Configuration**
    -   [ ] Ensure operator configuration (e.g., via environment variables in `config/manager/manager.yaml`) supports the static `cluster_name` and NetBox `site` required by the controller.

-   [ ] **6. Testing**
    -   [ ] Create `internal/controller/vlanclaim_controller_test.go`.
    -   [ ] Write unit tests for the `Reconcile` function, covering:
        -   [ ] VLAN creation.
        -   [ ] VLAN update.
        -   [ ] VLAN deletion (respecting `preserveInNetbox`).
        -   [ ] Taking ownership of an unmanaged VLAN.
        -   [ ] Handling of the `network.harvesterhci.io/ready` label (`true`, `false`, and absent).
    -   [ ] (Optional) Add integration or e2e tests.

-   [ ] **7. Documentation and Samples**
    -   [ ] Create a sample `VLANClaim` manifest in `config/samples/netbox_v1_vlanclaim.yaml`.
    -   [ ] Update project documentation (e.g., `README.md`) to include information about the new `VLANClaim` functionality.
