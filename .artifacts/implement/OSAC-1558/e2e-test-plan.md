# E2E Test Plan — OSAC-1558 Metal3 AAP Provisioning Role

## Context

The `bm_host_metal3_provisioning` Ansible role (OSAC-1558) is the last component needed for bare metal provisioning via the Metal3 backend. The role patches BareMetalHost CRs with image and networkData to provision hosts. Before publishing the PR, we need to validate it works end-to-end in the deployed environment.

The test environment (ostest) has:
- 6 available BareMetalHosts in `host-inventory` namespace (ostest-extraworker-0 through -5, all `available` state)
- Inventory and management configured for Metal3 (`osac-inventory-config` / `osac-management-config` secrets)
- AAP deployed and running in `osac-devel` namespace
- Bare-metal-fulfillment-operator deployed (`quay.io/carbonin/bare-metal-fulfillment-operator:latest`)
- The fulfillment service does not yet pass imageURL/checksum as templateParameters, so we test at the K8s CR level

## Setup

### 1. Get new role content into AAP

AAP syncs its project from git. Currently configured to pull from `https://github.com/osac-project/osac-aap` branch `main`. To test the new role:

**a) Push the branch to the fork:**
```bash
cd /home/root/sov/osac/osac-workspace/osac-aap
git push fork feat/OSAC-1558
```

**b) Update the AAP project git config** by patching the `config-as-code-ig` secret:
```bash
oc patch secret config-as-code-ig -n osac-devel \
  --type merge -p '{"stringData": {
    "AAP_PROJECT_GIT_URI": "https://github.com/carbonin/osac-aap",
    "AAP_PROJECT_GIT_BRANCH": "feat/OSAC-1558"
  }}'
```

**c) Trigger AAP project sync** — run the config-as-code job template from the AAP UI at `https://osac-aap-controller-osac-devel.apps.ostest.test.metalkube.org`, or wait for the scheduled 10-minute sync. Verify the project syncs successfully and the new role is available.

**d) Verify role is available** by checking the AAP project revision matches the branch HEAD.

### 2. Set up an image HTTP server

BMO needs to download the image via HTTP. Deploy a simple HTTP server pod in the cluster:

**a) Download a small test image** (Fedora Cloud or CirrOS — something lightweight that BMO can provision):
```bash
# Use CirrOS as a minimal test image (~15MB)
IMAGE_URL="https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
CHECKSUM_URL="https://download.cirros-cloud.net/0.6.2/MD5SUMS"
```

**b) Deploy an nginx pod serving the image** in the `host-inventory` namespace (same network as BMHs):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-server
  namespace: host-inventory
  labels:
    app: image-server
spec:
  containers:
  - name: nginx
    image: docker.io/library/nginx:alpine
    ports:
    - containerPort: 80
    command: ["/bin/sh", "-c"]
    args:
    - |
      mkdir -p /usr/share/nginx/html
      cd /usr/share/nginx/html
      wget -q https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
      md5sum cirros-0.6.2-x86_64-disk.img > cirros-0.6.2-x86_64-disk.img.md5sum
      nginx -g 'daemon off;'
---
apiVersion: v1
kind: Service
metadata:
  name: image-server
  namespace: host-inventory
spec:
  selector:
    app: image-server
  ports:
  - port: 80
    targetPort: 80
```

The image will be available at `http://image-server.host-inventory.svc.cluster.local/cirros-0.6.2-x86_64-disk.img`.

**c) Verify accessibility** from a debug pod or by curling the service.

**Note:** CirrOS is used as a small test image. If the BMHs require a specific format (qcow2 vs raw), adjust accordingly. CirrOS is qcow2 by default. An alternative is a Fedora Cloud image, but those are ~400MB.

### 3. Verify AAP job template exists

Confirm the `osac-create-bare-metal-instance` and `osac-delete-bare-metal-instance` job templates exist in AAP. These are pre-configured in the config-as-code and dynamically include the role based on `templateID`.

## Testing Process

### Test 1: Provision a BareMetalHost via Metal3

**a) Create a BareMetalInstance CR** that triggers the Metal3 provisioning role:

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: BareMetalInstance
metadata:
  name: bmi-metal3-test
  namespace: osac-devel
  annotations:
    osac.openshift.io/tenant: shared
spec:
  hostType: default
  templateID: bm_host_metal3_provisioning
  templateParameters: |
    {"imageURL": "http://image-server.host-inventory.svc.cluster.local/cirros-0.6.2-x86_64-disk.img", "imageChecksum": "<md5sum>", "checksumType": "md5"}
```

Replace `<md5sum>` with the actual checksum from the image server, or use `checksumType: "auto"` if a checksum URL is available.

**b) Watch the BareMetalInstance progress:**
```bash
oc get bmi bmi-metal3-test -n osac-devel -w
```

Expected phase transitions: `Allocating` → `Progressing` → `Ready`

**c) Monitor the AAP job:**
- Check the AAP UI for the `osac-create-bare-metal-instance` job triggered by this BMI
- Verify the job completes successfully
- Review the job output for the "Metal3 provisioning completed" message

**d) Verify the BareMetalHost was patched:**
```bash
# Check which BMH was allocated (externalHostID will be set)
oc get bmi bmi-metal3-test -n osac-devel -o jsonpath='{.spec.externalHostID}'

# Check the BMH has the image set
oc get bmh <name> -n host-inventory -o jsonpath='{.spec.image}'

# Check provisioning state
oc get bmh <name> -n host-inventory -o jsonpath='{.status.provisioning.state}'
```

### Test 2: Idempotency

**a) Delete and recreate the BMI** with the same image URL, or trigger a re-reconciliation.

**b) Verify** the AAP job log shows "already provisioned with the desired image, skipping" and the BMH is not re-patched.

### Test 3: Deprovision

**a) Delete the BareMetalInstance:**
```bash
oc delete bmi bmi-metal3-test -n osac-devel
```

**b) Watch the BareMetalHost return to available state:**
```bash
oc get bmh -n host-inventory -w
```

**c) Verify the AAP delete job:**
- Check the AAP UI for the `osac-delete-bare-metal-instance` job
- Verify job completes successfully
- Verify the networkData Secret was cleaned up:
  ```bash
  oc get secret <bmh-name>-network-data -n host-inventory
  # Should return "not found"
  ```

**d) Verify BMH state:**
```bash
oc get bmh <name> -n host-inventory -o jsonpath='{.status.provisioning.state}'
# Should be "available" or "ready"

oc get bmh <name> -n host-inventory -o jsonpath='{.spec.image}'
# Should be empty/null
```

## Expected Results

| Test | Expected Outcome |
|------|-----------------|
| Provision | BMH `spec.image.url` set to CirrOS URL, `status.provisioning.state` reaches `provisioned`, BMI phase reaches `Ready` |
| Idempotency | AAP job skips provisioning with debug message, no BMH re-patch |
| Deprovision | BMH `spec.image` cleared, state returns to `available`/`ready`, networkData Secret deleted, BMI deleted |

## Rollback

After testing, clean up:
```bash
# Delete test BMI (if not already done)
oc delete bmi bmi-metal3-test -n osac-devel --ignore-not-found

# Delete image server
oc delete pod image-server -n host-inventory --ignore-not-found
oc delete service image-server -n host-inventory --ignore-not-found

# Restore AAP git config to upstream
oc patch secret config-as-code-ig -n osac-devel \
  --type merge -p '{"stringData": {
    "AAP_PROJECT_GIT_URI": "https://github.com/osac-project/osac-aap",
    "AAP_PROJECT_GIT_BRANCH": "main"
  }}'
# Trigger a project sync in AAP to revert
```

## Known Limitations

- **Full API chain not tested**: The fulfillment service does not yet pass `imageURL`/`imageChecksum`/`checksumType`/`networkData` as templateParameters. Testing at K8s CR level validates the operator→AAP→BMH chain.
- **Null patch for deprovisioning**: The `delete.yaml` uses `spec.image: null` in a merge patch to trigger deprovisioning. If `kubernetes.core.k8s` doesn't handle null correctly with server-side apply, an alternative JSON Patch approach may be needed.
- **Provisioning time**: BMH provisioning can take 10-30+ minutes depending on image size and virtual BMC performance. The `until` loop allows up to 60 minutes.
