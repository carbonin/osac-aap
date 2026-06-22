# Story Context — OSAC-1558

## Story Summary

- **Title:** [DEV] Metal3 AAP provisioning role
- **Type:** Task
- **Jira:** OSAC-1558
- **Epic:** OSAC-1544 — Metal3 Backend Implementation
- **Feature:** OSAC-1543 — Metal3 (BareMetalHost) Backend Integration for BMaaS

### User Story

As a Cloud Infrastructure Admin,
I want AAP to provision and deprovision hosts by patching BareMetalHost image and network data,
So that BaremetalInstance provisioning works end-to-end with the Metal3 backend.

### Acceptance Criteria

1. A new `bm_host_metal3_provisioning` template role exists with `create.yaml` and `delete.yaml` task files
2. `create.yaml` validates `hostClass == "metal3"`, parses `templateParameters` for `imageURL`, `imageChecksum`, `checksumType`, and `networkData`, patches BMH `spec.image` and `spec.networkData`, and waits for `status.provisioning.state` to reach `provisioned`
3. `create.yaml` is idempotent — skips provisioning if the host is already provisioned with the correct image
4. `delete.yaml` removes `spec.image` from the BMH and waits for `status.provisioning.state` to return to `available` or `ready`, then cleans up the `networkData` Secret
5. The role parses `spec.externalHostID` as `namespace/name` to locate the BareMetalHost
6. The role uses `kubernetes.core` modules for all BMH interactions

### Implementation Guidance

Create `collections/ansible_collections/osac/templates/roles/bm_host_metal3_provisioning/` in the osac-aap repo with `tasks/create.yaml` and `tasks/delete.yaml`.

Follow the structure of `bm_host_agent_provisioning`. Receive the BMI CR as `bare_metal_instance` from `ansible_eda.event.payload`, validate `hostClass`, parse `templateParameters | from_json`.

For `create.yaml`: split `spec.externalHostID` on `/` to get namespace and name, use `kubernetes.core.k8s_info` to get the BMH, check provisioning state for idempotency, create `networkData` Secret, patch BMH `spec.image` and `spec.networkData`, use `until` loop to wait for `status.provisioning.state == "provisioned"`.

For `delete.yaml`: remove `spec.image`, wait for `status.provisioning.state` to reach `available` or `ready`, delete `networkData` Secret.

### Testing Approach

Tested via E2E (OSAC-1545). If Molecule is used in the project, add Molecule tests with mock BMH resources.

Note: No Molecule testing infrastructure exists in the project. Testing is via integration tests in `tests/integration/` and E2E tests in a separate repo.

### Dependencies

No story dependencies. Can be developed in parallel with OSAC-1556 and OSAC-1557 (different repos).

## Design Context

### Relevant Design Sections

The bare-metal-fulfillment EP (`enhancement-proposals/enhancements/bare-metal-fulfillment/README.md`) defines the overall architecture:
- **Host Templates** are Ansible roles that detail setup/teardown for individual hosts [EP: §Proposal/CRDs]
- **Host Operator** triggers provisioning templates specified by the Host Lease (now BareMetalInstance) CR [EP: §Workflow]
- **Multiple backends** are supported via a management interface — Metal3 uses `BareMetalHost` CRs instead of OpenStack Ironic [EP: §Implementation Details]

The `baremetal-instance-api` EP (`enhancement-proposals/enhancements/baremetal-instance-api/README.md`) describes how the provisioning chain works:
- Fulfillment service → BareMetalInstance CR → bare-metal-fulfillment-operator → osac-aap template role
- The operator sets `hostClass: "metal3"` and `externalHostID: "namespace/name"` on the BareMetalInstance
- Template parameters (imageURL, imageChecksum, etc.) are JSON-encoded in `spec.templateParameters`

The Metal3 management backend (`bare-metal-fulfillment-operator/internal/management/metal3.go`) confirms:
- `externalHostID` uses `namespace/name` format, parsed by `inventory.ParseHostID()`
- `hostClass: "metal3"` is the discriminator for the Metal3 backend

### PRD Requirements Covered

FR-5 (AAP template role for image provisioning via BareMetalHost resources)

## Codebase Context

### Affected Components

#### bm_host_metal3_provisioning (NEW)
- **Location:** `collections/ansible_collections/osac/templates/roles/bm_host_metal3_provisioning/`
- **Purpose:** Provision and deprovision hosts by patching BareMetalHost `spec.image` and `spec.networkData`
- **Current patterns:** Follow `bm_host_agent_provisioning` role structure
- **What changes:** New role — `tasks/create.yaml` and `tasks/delete.yaml`
- **Existing tests:** None (new role)

### Relevant Types and Interfaces

**BareMetalInstance CR** (input to the playbook via `ansible_eda.event.payload`):
```go
type BareMetalInstanceSpec struct {
    HostType           string            `json:"hostType"`
    ExternalHostID     string            `json:"externalHostID"`     // "namespace/name" format
    ExternalHostName   string            `json:"externalHostName,omitempty"`
    HostClass          string            `json:"hostClass,omitempty"`  // "metal3" for this role
    NetworkClass       string            `json:"networkClass,omitempty"`
    TemplateID         string            `json:"templateID"`
    TemplateParameters string            `json:"templateParameters,omitempty"` // JSON-encoded
    RunStrategy        string            `json:"runStrategy,omitempty"`
}
```

**BareMetalHost CR** (Metal3 — target resource to patch):
- `spec.image.url` — URL of the image to provision
- `spec.image.checksum` — Checksum of the image
- `spec.image.checksumType` — Type of checksum (md5, sha256, sha512)
- `spec.networkData.name` — Reference to a Secret containing network data
- `spec.networkData.namespace` — Namespace of the network data Secret
- `spec.online` — Whether the host should be powered on
- `status.provisioning.state` — Current provisioning state (registering, inspecting, available, provisioning, provisioned, deprovisioning, etc.)

**Template Parameters** (JSON in `spec.templateParameters`):
- `imageURL` — URL of the image
- `imageChecksum` — Checksum value
- `checksumType` — Checksum algorithm
- `networkData` — Network configuration data (JSON/YAML for the networkData Secret)

### Relevant APIs

**Playbook entry points** (already exist, dynamically include role by `templateID`):
- `playbook_osac_create_bare_metal_instance.yml` — Includes `osac.templates.{{ bare_metal_instance.spec.templateID }}` with `tasks_from: create`
- `playbook_osac_delete_bare_metal_instance.yml` — Includes `osac.templates.{{ bare_metal_instance.spec.templateID }}` with `tasks_from: delete`

The role will be invoked when `bare_metal_instance.spec.templateID == "bm_host_metal3_provisioning"`.

### Reference Role: bm_host_agent_provisioning

The existing `bm_host_agent_provisioning` role follows this pattern:
- **create.yaml**: Validates `hostClass == "openstack"`, parses `templateParameters | from_json`, uses `openstack.cloud.baremetal_node_info` to find node, checks idempotency via provision state, deploys via `openstack.cloud.baremetal_node_action`
- **delete.yaml**: No-op (agent provisioning doesn't handle deletion)

Key differences for the Metal3 role:
- Uses `kubernetes.core.k8s_info` / `kubernetes.core.k8s` instead of OpenStack modules
- Validates `hostClass == "metal3"` instead of `"openstack"`
- Parses `externalHostID` as `namespace/name` instead of using it as an Ironic node name
- Patches BMH `spec.image` and creates a networkData Secret instead of calling Ironic deploy
- `delete.yaml` is active — removes `spec.image` and cleans up the networkData Secret

## Repository Topology

- **Origin:** osac-project/osac-aap
- **Type:** Direct (not a fork)
- **Fork remote:** `fork` → `carbonin/osac-aap` (push target for PRs)

## Validation Profile

### Commit Format
- **Pattern:** `OSAC-NNNN: description` or `NO-ISSUE: description`
- **Discovered from:** git log

### Pre-PR Checks (ordered)
1. `uv run ansible-lint` — Lint all playbooks and roles
2. `ansible-playbook --syntax-check playbook_osac_create_bare_metal_instance.yml` — Syntax check
3. `uv run make test` — Integration tests (requires kind cluster)

### PR Conventions
- **Title format:** `OSAC-1558: description`
- **PR template:** None — use default template
- **Description guidance:** From CLAUDE.md PR checklist: ansible-lint passes, `meta/osac.yaml` updated for template role changes, cross-repo dependencies documented, playbook tested
- **Fork-based workflow:** Push to `fork` remote, PR from `fork/<branch>` to `origin/main`

### Coverage Tooling
- **Command:** N/A (Ansible — no unit test coverage tooling configured)
- **Report location:** N/A
- **View command:** N/A
- **Minimum new-code coverage:** N/A (Ansible roles — coverage via E2E and integration tests)

### Discovered from
- `CLAUDE.md`
- `.claude/rules/playbook-patterns.md`
- `.claude/rules/networking-cudn.md`
- `.github/workflows/tests.yml`
- `.github/workflows/pre-commit.yaml`
- `.pre-commit-config.yaml`
- `Makefile`
- `git log --oneline -10`

## Open Questions

1. Should `bm_host_metal3_provisioning` have a `meta/osac.yaml` file? The reference role (`bm_host_agent_provisioning`) does not have one, but the CLAUDE.md says template roles should. Since this is a host-level provisioning template (not a network/compute template published as a NetworkClass), it may not need one — `/plan` should determine this.
2. What exact structure should the `networkData` Secret content use? The task says "networkData" is a template parameter — `/plan` should define the Secret manifest shape based on Metal3/BMO documentation.

## Warnings

- **`.artifacts/` not in `.gitignore`**: The `.artifacts/` directory is not listed in osac-aap's `.gitignore`. Implementation artifacts could be accidentally committed. Consider adding `.artifacts/` to `.gitignore`.
