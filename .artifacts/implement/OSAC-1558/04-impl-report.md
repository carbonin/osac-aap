# Implementation Report — OSAC-1558

## Changes Summary

| File | Action | Description |
|------|--------|-------------|
| `collections/ansible_collections/osac/templates/roles/bm_host_metal3_provisioning/tasks/create.yaml` | Created | Metal3 BMH provisioning: validates hostClass, parses templateParameters, creates networkData Secret, patches BMH spec.image, waits for provisioned state |
| `collections/ansible_collections/osac/templates/roles/bm_host_metal3_provisioning/tasks/delete.yaml` | Created | Metal3 BMH deprovisioning: removes spec.image and spec.networkData, waits for available/ready state, deletes networkData Secret |

## Commits

| Hash | Message |
|------|---------|
| 781800d | OSAC-1558: add bm_host_metal3_provisioning create.yaml |
| 930b0cd | OSAC-1558: add bm_host_metal3_provisioning delete.yaml |

## Deviations from Plan

No deviations from the implementation plan.

## Discoveries

- The `kubernetes.core.k8s` module with `definition` containing `null` values for merge-patching field removal (used in `delete.yaml` to clear `spec.image` and `spec.networkData`) may need E2E validation to confirm BMO correctly deregisters the image. If null patches don't work as expected with server-side apply, an alternative approach using `kubernetes.core.k8s_json_patch` with JSON Patch remove operations may be needed.

## Status

Complete — all 2 tasks implemented, lint passing, both commits on `feat/OSAC-1558` branch.
