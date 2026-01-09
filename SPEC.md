# SPEC.md: VLAN and VirtualMachine Synchronization from Harvester to NetBox

This document outlines the specification for extending the NetBox Operator to synchronize VLAN and VirtualMachine resources from a Harvester Kubernetes cluster to a NetBox instance.

## Guiding Principles

- **Source of Truth**: Harvester is the definitive source of truth. Changes should flow one-way from Harvester to NetBox.
- **Declarative Configuration**: The operator will reconcile the state of Harvester resources with objects in NetBox.
- **Traceability**: Objects created in NetBox by the operator should be clearly identifiable and linked back to the source Harvester resource.

---

## VLAN Synchronization via `VLANClaim` CRD

To align with the existing "request-based" pattern of the NetBox Operator, synchronization of Harvester VLANs will be managed through a new Custom Resource Definition (CRD) named `VLANClaim` in the `netbox.dev` API group.

A `VLANClaim` object will explicitly state the desired state of a specific VLAN in NetBox, mirroring an existing Harvester VLAN. The operator will reconcile this `VLANClaim` by ensuring the corresponding VLAN exists in NetBox with the specified attributes.

### 1. `VLANClaim` Resource Definition (`spec`)

The `VLANClaim` `spec` will define the properties of the NetBox VLAN and link back to the source Harvester resource.

-   **`vlanId` (required)**: The unique VLAN ID (VID) for the NetBox VLAN. This value is expected to correspond to the `network.harvesterhci.io/vlan-id` label on the referenced `NetworkAttachmentDefinition`.

-   **`name` (required)**: The desired name for the VLAN in NetBox. This will be constructed using a template, typically `{{cluster_name}}-{{source_resource_name}}`. The `cluster_name` will be a static configuration for the operator instance, and `source_resource_name` will be the name of the `NetworkAttachmentDefinition` referenced by `sourceRef`.

-   **`vlanGroup` (optional)**: The NetBox `VLANGroup` where this VLAN should be organized. If provided, the controller will ensure the VLAN is associated with this group in NetBox.

-   **`site` (required)**: The NetBox `Site` where this VLAN should exist. This will be a static configuration for the operator instance (e.g., via an environment variable).

-   **`sourceRef` (required)**: A reference to the original Harvester `NetworkAttachmentDefinition` that this `VLANClaim` represents. The controller will use this reference to fetch the NAD for status and data consistency checks. It will include:
    -   `apiVersion`: `k8s.cni.cncf.io/v1`
    -   `kind`: `NetworkAttachmentDefinition`
    -   `name` (of the `NetworkAttachmentDefinition`)
    -   `namespace` (of the `NetworkAttachmentDefinition`)

-   **`description` (optional)**: A free-form text description for the NetBox VLAN. This can be directly provided in the `VLANClaim` and may optionally derive from annotations or fields on the source `NetworkAttachmentDefinition`.

-   **`comments` (optional)**: Additional comments for the NetBox VLAN.

-   **`customFields` (optional)**: A map of key-value pairs for NetBox Custom Fields. This will explicitly include a `managed_by: harvester-netbox-operator` custom field. Other custom fields can be defined here as desired.

-   **`preserveInNetbox` (optional, boolean, default: `false`)**: If set to `true`, the NetBox VLAN will not be deleted when the `VLANClaim` Kubernetes resource is deleted. This allows for persistent NetBox records even if the Kubernetes resource is transiently removed.

---

### 2. `VLANClaim` Status Definition (`status`)

The `VLANClaim`'s `status` field will report the observed state of the synchronization process and the resulting NetBox VLAN.

-   **`vlanId`**: The NetBox internal database ID of the created/managed VLAN.
-   **`vlanUrl`**: The URL to the VLAN object in the NetBox UI.
-   **`conditions`**: Standard Kubernetes conditions (`metav1.Condition`) to reflect the reconciliation state and health (e.g., `Ready`, `Failed`).
    -   The `Ready` condition's status will be directly influenced by the `network.harvesterhci.io/ready` label on the `NetworkAttachmentDefinition` referenced by `sourceRef`. If the label is `"true"`, `Ready` will be `True` and the NetBox VLAN `status` will be set to `Active`. If the label is `"false"` or absent, `Ready` will be `False` (with an appropriate reason indicating the source is not ready) and the NetBox VLAN `status` will be set to `Staged`.

### 3. Controller Behavior (`VLANClaimController`)

The `VLANClaimController` will manage the lifecycle and state synchronization of NetBox VLANs based on `VLANClaim` resources.

-   **Watch**: The `VLANClaimController` will watch for `VLANClaim` resources in all namespaces.
-   **Reconciliation Logic**: Upon a `VLANClaim` event (creation, update, deletion):
    1.  **Fetch Source NAD**: The controller will attempt to fetch the `NetworkAttachmentDefinition` referenced by `spec.sourceRef`.
    2.  **Derive NetBox Data**: It will extract the `vlanId` from `spec.vlanId` and determine the `name` based on the configured cluster name and the NAD's name. The NetBox `status` will be derived from the `network.harvesterhci.io/ready` label on the fetched NAD.
    3.  **NetBox Operation**: It will ensure the corresponding VLAN exists in NetBox (creating if new, updating if changed) with the attributes defined in the `VLANClaim` `spec` and derived from the NAD.
    4.  **Conflict Resolution**: If a VLAN with the same `vlanId`, `site`, and `vlanGroup` (if specified) already exists in NetBox but is missing the `managed_by: harvester-netbox-operator` custom field, the controller will **take ownership**. This means it will overwrite the NetBox VLAN's details with data from the `VLANClaim` and set the `managed_by` custom field. From that point forward, this operator instance will manage its lifecycle.
    5.  **Status Update**: It will update the `VLANClaim`'s `status` with the NetBox ID, URL, and conditions, reflecting the current state of synchronization.
    6.  **Finalizer**: A finalizer will be managed on the `VLANClaim` to ensure proper cleanup of the NetBox VLAN upon `VLANClaim` deletion (unless `preserveInNetbox` is `true`).

---

## VirtualMachine Synchronization

Now that the `VLANClaim` design is finalized, we will proceed with the `VirtualMachineClaim` to synchronize VirtualMachine resources from Harvester to NetBox, following a similar pattern.