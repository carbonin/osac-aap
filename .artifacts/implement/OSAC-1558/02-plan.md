# Implementation Plan — OSAC-1558

## Summary

Create a new `bm_host_metal3_provisioning` Ansible template role with `create.yaml` and `delete.yaml` that provisions and deprovisions bare metal hosts by patching BareMetalHost CRs via `kubernetes.core` modules. The role follows the structure of `bm_host_agent_provisioning` but targets Metal3/BMO instead of OpenStack Ironic, using the documented BareMetalHost provisioning workflow (patch `spec.image` + `spec.networkData`).

## Branch

- **Name:** feat/OSAC-1558
- **Base:** main

## Interface Definitions

### New Types

No new types required. This is an Ansible role — the "interface" is:
- **Input:** `bare_metal_instance` variable (BareMetalInstance CR from `ansible_eda.event.payload`)
- **Template parameters** (JSON in `bare_metal_instance.spec.templateParameters`):
  - `imageURL` (required) — URL of the image to deploy
  - `imageChecksum` (required) — checksum for the image (value or URL, per BMO docs)
  - `checksumType` (optional, default `"auto"`) — checksum algorithm (md5, sha256, sha512, auto)
  - `networkData` (optional) — cloud-init network configuration YAML content

### Modified Interfaces

No interface modifications required.

### New Functions

No new functions required (Ansible tasks, not Go code).

## Test Strategy

### Unit Tests

No Ansible unit test framework (Molecule) is configured in this project. The reference role (`bm_host_agent_provisioning`) has no tests either.

### Integration Tests

No integration tests required — this role interacts with BareMetalHost CRs which require a Metal3/BMO deployment that the existing integration test infrastructure (`tests/integration/`) does not provide.

### E2E Tests

Testing is covered by OSAC-1545 (separate task for E2E tests). This is consistent with the story's testing approach: "Tested via E2E (OSAC-1545)."

### Coverage Goals

Behavioral coverage via E2E:
- Provisioning a BMH with image and networkData end-to-end
- Idempotent re-provisioning (same image → skip)
- Deprovisioning: image removal, state wait, Secret cleanup
- Error cases: missing BMH, invalid hostClass, missing templateParameters

## Task Breakdown

### Task 1: Create `create.yaml` — provision BMH with image and networkData

- **Files:**
  - `collections/ansible_collections/osac/templates/roles/bm_host_metal3_provisioning/tasks/create.yaml` (new)
- **What:** Create the `create.yaml` task file with the following tasks:

  1. **Validate hostClass** — fail if `bare_metal_instance.spec.hostClass != "metal3"` (matches the pattern from `bm_host_agent_provisioning` which validates `hostClass == "openstack"`)

  2. **Parse template parameters** — `set_fact: template_params: "{{ bare_metal_instance.spec.templateParameters | from_json }}"`

  3. **Validate required params** — fail if `template_params.imageURL` is empty

  4. **Parse externalHostID** — split `bare_metal_instance.spec.externalHostID` on `/` to extract:
     - `bmh_namespace` (first part)
     - `bmh_name` (second part)

  5. **Fetch BMH** — `kubernetes.core.k8s_info` with apiVersion `metal3.io/v1alpha1`, kind `BareMetalHost`, namespace `bmh_namespace`, name `bmh_name`. Register as `bmh_result`.

  6. **Fail if BMH not found** — check `bmh_result.resources | length == 0`

  7. **Extract BMH info** — `set_fact: bmh_info: "{{ bmh_result.resources[0] }}"`

  8. **Check idempotency** — when `bmh_info.status.provisioning.state == "provisioned"` and `bmh_info.status.provisioning.image.url == template_params.imageURL`, skip remaining tasks with a debug message. Uses a block with `when: not (already_provisioned | default(false))` for the remaining provisioning steps.

  9. **Create networkData Secret** (conditional: only when `template_params.networkData` is defined and non-empty) — `kubernetes.core.k8s` with `state: present`:
     ```yaml
     apiVersion: v1
     kind: Secret
     metadata:
       name: "{{ bmh_name }}-network-data"
       namespace: "{{ bmh_namespace }}"
     stringData:
       networkData: "{{ template_params.networkData }}"
     ```
     Following the Metal3 convention: Secret key is `networkData`, naming is `<hostname>-network-data` (from BMaaS docs).

  10. **Patch BMH** — `kubernetes.core.k8s` merge patch on the BareMetalHost to set:
      - `spec.image.url`: `{{ template_params.imageURL }}`
      - `spec.image.checksum`: `{{ template_params.imageChecksum }}`
      - `spec.image.checksumType`: `{{ template_params.checksumType | default("auto") }}`
      - `spec.online`: `true`
      - `spec.networkData.name`: `{{ bmh_name }}-network-data` (only if networkData provided)
      - `spec.networkData.namespace`: `{{ bmh_namespace }}` (only if networkData provided)
      Per bmaas.txt: updating `spec.image` triggers provisioning immediately.

  11. **Wait for provisioned state** — `kubernetes.core.k8s_info` in an `until` loop checking `status.provisioning.state == "provisioned"`. Use `retries: 120`, `delay: 30` (60 min max wait, consistent with project patterns for long-running operations).

  12. **Debug message** — confirm provisioning complete.

- **Why:** AC-1 (role with create.yaml), AC-2 (validates hostClass, parses params, patches BMH, waits for provisioned), AC-3 (idempotency check), AC-5 (parses externalHostID as namespace/name), AC-6 (kubernetes.core modules)
- **Commit message:** `OSAC-1558: add bm_host_metal3_provisioning create.yaml`
- **Status:** Done

### Task 2: Create `delete.yaml` — deprovision and clean up

- **Files:**
  - `collections/ansible_collections/osac/templates/roles/bm_host_metal3_provisioning/tasks/delete.yaml` (new)
- **What:** Create the `delete.yaml` task file with the following tasks:

  1. **Parse externalHostID** — split on `/` to extract `bmh_namespace` and `bmh_name`

  2. **Fetch BMH** — `kubernetes.core.k8s_info` to get the BareMetalHost. Register as `bmh_result`.

  3. **Skip if BMH not found** — if `bmh_result.resources | length == 0`, debug message and end (already cleaned up)

  4. **Extract BMH info** — `set_fact: bmh_info: "{{ bmh_result.resources[0] }}"`

  5. **Skip if no image set** — when `bmh_info.spec.image is not defined` or `bmh_info.spec.image.url is not defined`, debug and end

  6. **Patch BMH to remove image and networkData** — `kubernetes.core.k8s` merge patch setting `spec.image` to `null` and `spec.networkData` to `null`. This triggers deprovisioning in BMO.

  7. **Wait for available/ready state** — `kubernetes.core.k8s_info` in an `until` loop checking `status.provisioning.state` is in `["available", "ready"]`. Use `retries: 120`, `delay: 30` (60 min max wait).

  8. **Delete networkData Secret** — `kubernetes.core.k8s` with `state: absent` for Secret `{{ bmh_name }}-network-data` in `{{ bmh_namespace }}`. Idempotent — succeeds even if Secret doesn't exist.

  9. **Debug message** — confirm deprovisioning complete.

- **Why:** AC-1 (role with delete.yaml), AC-4 (removes spec.image, waits for available/ready, cleans up Secret), AC-5 (parses externalHostID), AC-6 (kubernetes.core modules)
- **Commit message:** `OSAC-1558: add bm_host_metal3_provisioning delete.yaml`
- **Status:** Done

## Acceptance Criteria Coverage

| AC | Description | Covered by |
|----|-------------|------------|
| AC-1 | New `bm_host_metal3_provisioning` role with `create.yaml` and `delete.yaml` | Task 1, Task 2 |
| AC-2 | `create.yaml` validates hostClass, parses templateParameters, patches BMH, waits for provisioned | Task 1 (steps 1-3, 10-11) |
| AC-3 | `create.yaml` is idempotent — skips if already provisioned with correct image | Task 1 (step 8) |
| AC-4 | `delete.yaml` removes spec.image, waits for available/ready, cleans up networkData Secret | Task 2 (steps 6-8) |
| AC-5 | Role parses `spec.externalHostID` as `namespace/name` | Task 1 (step 4), Task 2 (step 1) |
| AC-6 | Role uses `kubernetes.core` modules for all BMH interactions | Task 1, Task 2 (all k8s operations use kubernetes.core) |

All acceptance criteria are covered. No gaps.

## Risk Assessment

- **BMH provisioning timeout:** Metal3 provisioning can take 10-30+ minutes depending on image size and hardware. Mitigation: 120 retries × 30s delay = 60 minutes max wait, consistent with project patterns (hosted_cluster uses 720 retries × 10s = 120 min).
- **networkData parameter format:** The `networkData` template parameter must contain raw cloud-init network configuration YAML (per bmaas.txt). The caller (bare-metal-fulfillment-operator) is responsible for providing the correct format. Using `stringData` in the Secret ensures Kubernetes handles base64 encoding automatically.
- **Deprovisioning by nulling spec.image:** BMO behavior when `spec.image` is set to `null` should trigger deprovisioning. The bmaas.txt doc confirms this is the expected flow — removing the image field causes BMO to deprovision. If BMO doesn't recognize a null patch, the alternative is to remove the field entirely via a strategic merge patch. Mitigation: test during E2E (OSAC-1545).

## Open Questions

None — both open questions from ingest are resolved:
1. **meta/osac.yaml:** Not needed. The reference role (`bm_host_agent_provisioning`) does not have one, and host-level provisioning templates are not published as NetworkClass/template metadata.
2. **networkData Secret structure:** Per bmaas.txt, the Secret uses a `networkData` key containing cloud-init network configuration YAML, named `<hostname>-network-data` in the same namespace as the BMH.
