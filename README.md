# AAP + OpenShift Virtualization VM Demo

Ansible content that creates/tears down a VM (and a sample container workload)
on a cluster running OpenShift Virtualization, driven from Ansible Automation
Platform (AAP) via the `kubernetes.core` collection.

| File | Purpose |
|---|---|
| `create_vm.yml` | Creates a `VirtualMachine` (instance type + preference + cloned golden image) and waits for it to boot |
| `create_container.yml` | Creates a Deployment/Service/Route — a quick non-VM comparison demo |
| `teardown.yml` | Deletes the demo workloads by label, optionally the namespace too |
| `group_vars/all.yml` | Default vars for `create_vm.yml` (VM name, size, image, SSH key, auth) |
| `templates/virtualmachine.yaml.j2` | The `VirtualMachine` manifest template rendered by `create_vm.yml` |
| `inventory/hosts.yml` | Single `localhost` host — everything runs against the Kubernetes API, not over SSH |

This README covers what you need to set up in AAP (you already have AAP and
OpenShift Virtualization running on the cluster).

## 1. Give AAP a cluster identity

AAP talks to the cluster as a service account, not as your personal `oc`
user. Create one scoped to what the demo needs, in the namespace(s) you want
the demo to use (example below uses `ansible-demo`, which matches the default
namespace baked into `create_container.yml`/`teardown.yml` — see the
**namespace mismatch** note in Known Gaps):

```bash
oc new-project ansible-demo   # or: oc create namespace ansible-demo
oc create serviceaccount aap-vm-demo -n ansible-demo
```

Grant it what it needs to manage VMs and the container demo. `admin` on the
namespace is the simplest option for a demo; for something tighter, bind to
a role covering `virtualmachines`, `datavolumes`, `deployments`, `services`,
`routes`, and `namespaces` (if you want it to create the namespace itself,
that needs a cluster-scoped binding):

```bash
oc adm policy add-role-to-user admin -z aap-vm-demo -n ansible-demo
```

If `create_namespace: true` (the default) should be allowed to create the
namespace itself, the SA also needs a cluster-scoped grant:

```bash
oc create clusterrole aap-vm-demo-ns-creator --verb=create,get,list --resource=namespaces
oc adm policy add-cluster-role-to-user aap-vm-demo-ns-creator -z aap-vm-demo -n ansible-demo
```

Get a token for the SA (OpenShift 4.11+ tokens expire by default — use
`--duration` for a longer-lived one for a lab/demo environment):

```bash
oc create token aap-vm-demo -n ansible-demo --duration=8760h
```

Note the OpenShift API URL too — you'll need it for the AAP credential:

```bash
oc whoami --show-server
```

## 2. Add an OpenShift/Kubernetes credential in AAP

In **Automation Execution → Infrastructure → Credentials**, create one with:

- **Credential type:** `OpenShift or Kubernetes API Bearer Token`
- **OpenShift or Kubernetes API Endpoint:** the URL from `oc whoami --show-server`
- **API Token:** the token from step 1
- **Verify SSL:** off for a demo cluster with a self-signed/internal CA, on otherwise

When this credential is attached to a job template, AAP injects
`K8S_AUTH_HOST`, `K8S_AUTH_API_KEY`, and `K8S_AUTH_VERIFY_SSL` as environment
variables — which is exactly what `group_vars/all.yml` reads via
`lookup('env', ...)`. No cluster details need to live in the playbooks or
survey.

## 3. Confirm the Execution Environment has what it needs

The playbooks need the `kubernetes.core` collection **and** its underlying
Python `kubernetes` client library. AAP's default execution environments
(e.g. `ee-supported-rhel9`) typically ship both already. To check the one
you use:

```bash
podman run --rm -it <your-ee-image> ansible-galaxy collection list kubernetes.core
podman run --rm -it <your-ee-image> python3 -c "import kubernetes; print(kubernetes.__version__)"
```

If either is missing, either switch the job templates to a supported EE
that includes them, or build a custom EE (via `ansible-builder`/Automation
Hub) with:

```
# requirements.yml
collections:
  - kubernetes.core

# requirements.txt
kubernetes
```

(AAP automatically runs `ansible-galaxy collection install -r
requirements.yml` from the project root on every project sync, so the
collection in this repo's `requirements.yml` is picked up for you — it's
the Python `kubernetes` package that depends on the EE.)

## 4. Create the Project

**Automation Execution → Projects → Add**:

- **Source control type:** Git
- **Source control URL:** `https://github.com/vegasboy99/aap-create-vm-demo.git`
- **Source control branch:** `main`
- **Options:** enable "Update Revision on Launch" so edits in GitHub are picked up automatically

## 5. Inventory

Everything targets `hosts: localhost` and reaches the cluster over the
Kubernetes API, so no SSH inventory is needed. Either point job templates at
AAP's built-in **Demo Inventory** (localhost is already in it), or import
`inventory/hosts.yml` from this repo as its own inventory/source.

## 6. Create the Job Templates

Create one job template per playbook, all using the Project, Inventory, and
Credential from above.

### Create VM

- **Playbook:** `create_vm.yml`
- **Credentials:** the OpenShift/Kubernetes credential from step 2
- **Survey** (recommended so this is a self-service demo button):

| Variable | Type | Default |
|---|---|---|
| `vm_name` | Text | `demo-vm-01` |
| `vm_namespace` | Text | `ansible-demo` |
| `ssh_authorized_key` | Text | *(your public key)* |
| `vm_instancetype` | Text | `u1.medium` |
| `vm_preference` | Text | `centos.stream9` |

### Create Container Demo (optional comparison)

- **Playbook:** `create_container.yml`
- Same credential; no survey required, or expose `app_replicas`/`app_name`.

### Teardown

- **Playbook:** `teardown.yml`
- **Survey:** boolean `delete_namespace` (default `false`)

## 7. Run it

Launch **Create VM** from the AAP UI (or `POST
/api/v2/job_templates/<id>/launch/`). Watch the job output — it waits for
the `VirtualMachineInstance` to reach `Running` and prints its IP. Verify
from the CLI if you like:

```bash
oc get vm,vmi -n ansible-demo
virtctl console demo-vm-01 -n ansible-demo
```

## Before you demo

- **Golden image name is cluster-specific.** Confirm `centos-stream9` exists
  before you demo: `oc get datasource -n openshift-virtualization-os-images`
  and `oc get virtualmachineclusterpreferences`. Swap `vm_datasource` /
  `vm_preference` in the survey if your cluster ships different names.
