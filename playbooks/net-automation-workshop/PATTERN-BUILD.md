# Building the Network Automation Workshop Pattern

One-time workflow to deploy the lab, bootstrap it, verify student-facing
endpoints, and capture a golden pattern. Deployments from that pattern skip
router serial bootstrap — disks already contain day-0 config.

## Prerequisites

| Item | Notes |
|------|--------|
| Troshka host | Connected agent (`troshkad`), enough disk for 6 VMs |
| Library images | `aap-2.6-2-ceh-20251103`, `rhel-9.6`, `c8000v-rhdp-8g`, `veos-lab-4.32.0F`, `junos-vsrx3-23.2R2.21` on the host or central S4 |
| Troshka API | Dev: `http://localhost:8200` — prod: your cluster URL |
| Ansible | `ansible-playbook` + [troshka-ansible-collection](https://github.com/redhat-cop/troshka-ansible-collection) |
| Agent version | Serial router bootstrap needs current `troshkad` (multiline serial, Junos `fxp0`). Push with `./scripts/update-agent.sh <host-id>` |

**Required** for vscode bootstrap on unregistered RHEL images (package install
step). Without registration, `dnf` has no repos and bootstrap fails at
`Install vscode packages`.

This is different from the original CI
(`~/zt-ansiblebu-agnosticv/zt-ansiblebu/ansible-network-automation-workshop/`),
which wraps `zt-ans-bu-lab-developer-cnv` with `repo_method: none` and
`use_content_view: true` — demosat repos are applied at **provision** time, not
by a post-deploy bootstrap playbook.

For Troshka template deploys, register vscode against **demosat** (same secret
file as other RHDP labs). Vault-encrypted in
`~/zt-ansiblebu-agnosticv/includes/secrets/demosat-rhel-8-and-9-latest.yaml`
(password `~/secrets/vault_pw.txt`). Only individual fields are inline-vault —
`ansible-vault view` on the whole file fails. Extract key/value pairs with
ansible:

```bash
SATCREDS=~/zt-ansiblebu-agnosticv/includes/secrets/demosat-rhel-8-and-9-latest.yaml
VAULT=~/secrets/vault_pw.txt

vault_var() {
  ansible localhost -m debug -a "var=$1" -e @"$SATCREDS" --vault-password-file "$VAULT" 2>/dev/null \
    | python3 -c "import sys,re; m=re.search(r'\"$1\": \"([^\"]+)\"', sys.stdin.read()); print(m.group(1) if m else '')"
}

export SATELLITE_URL=$(vault_var satellite_url)
export REG_ORG=$(vault_var satellite_org)
export REG_ACTIVATION_KEY=$(vault_var satellite_activationkey)
export SATELLITE_HA=true   # set_repositories_satellite_ha in the secret file
```

`set_repositories_satellite_ha: true` means demosat runs behind a load balancer.
Plain `subscription-manager register --org … --activationkey …` talks to
`subscription.rhsm.redhat.com` and fails. The bootstrap role instead:

1. Installs the Katello CA RPM from `https://{{ satellite_url }}/pub/…`
2. Sets `server.hostname` and `server.prefix=/rhsm`
3. Registers with `--serverurl` and `--baseurl=…/pulp/repos`

Export the vars above (or pass `-e reg_org=… -e reg_activation_key=… -e
satellite_ha=true`) **before** prepare/bootstrap. Do **not** pass `reg_user` /
`reg_pass` for vscode RHSM — those are portal creds and a different registration
path.

Optional — `podman login registry.redhat.io` on the control VM still needs portal
creds from `~/agnosticv/includes/secrets/aap2-casc-registry-creds.yaml`
(`redhat_username` / `redhat_password` via the same `vault_var` helper). Pass
only when pulling the AAP EE image:

```bash
CREDS=~/agnosticv/includes/secrets/aap2-casc-registry-creds.yaml
export REG_USER=$(vault_var redhat_username)
export REG_PASS=$(vault_var redhat_password)

ansible-playbook -i "$INV" main.yml --skip-tags always --tags bootstrap \
  -e reg_user="$REG_USER" -e reg_pass="$REG_PASS"
```

Portal creds do not affect vscode registration when `reg_org` is set.

Confirm registration before re-running:

```bash
ansible -i "$INV" vscode -m shell -a "subscription-manager identity; dnf repolist"
```

## Dynamic inventory

The `troshka.cloud.troshka` plugin only loads inventory files named
`*.troshka.yml` or `*.troshka.yaml`. The role default
(`.generated/troshka_inventory.yml`) is ignored by the plugin — subsequent
`-i` runs see zero hosts and skip all plays.

Use a per-project path (recommended when bootstrapping multiple labs in
parallel):

```bash
INV=".generated/${TROSHKA_PROJECT_ID}/inventory.troshka.yml"

ansible-playbook main.yml --limit localhost -e "workshop_inventory_path=${INV}"
ansible-playbook -i "$INV" main.yml --skip-tags always --tags bootstrap
```

Verify before bootstrap:

```bash
ansible-inventory -i "$INV" --list
# groups: workshop_control, workshop_vscode, routers, aap, cisco_iosxe, ...
```

Install `troshka.cloud` **once** before parallel prepare/bootstrap runs.
Concurrent `ansible-galaxy collection install --force` calls race on
`~/.ansible/collections/ansible_collections/troshka/cloud`.

## Showroom (hybrid content build)

Showroom runs as a native **container pod** on the project **transit** network
(`172.30.{vni}.3`). Troshka auto-injects gateway port-forwards **80 and 443 →
transit:80** at deploy. AAP and code-server are reached through Showroom tabs
(`/aap/`, `/vscode/`) via the showroom nginx sidecar — they are **not**
exposed on separate external ports.

| Service | Student access | Internal target (mgmt) |
|---------|----------------|------------------------|
| Showroom UI | `http(s)://<gateway>/` | transit infra IP `:80` |
| AAP (control) | Showroom tab `/aap/` | `10.0.0.10:443` |
| code-server (vscode) | Showroom tab `/vscode/` | `10.0.0.20:8080` |

**First deploy from template** runs init containers (git clone, Troshka
`ui-config` overlay, Antora build) into the showroom content volume.

**Pattern capture** uploads that volume and sets `build_content: false` on the
saved topology so pattern redeploys skip init and serve pre-built content.

After pattern capture the showroom pod is stopped for volume capture — restart
it from the project page (container start) if the lab stays running.

Overlay files live in
`roles/troshka_workload_net_automation_workshop/files/configs/`.
After editing, sync into the Troshka template:

```bash
export TROSHKA_WORKSHOP_TEMPLATE=~/troshka/example_templates/net-automation-workshop.yaml
cd ~/demo_workloads/playbooks/net-automation-workshop
INV=".generated/${TROSHKA_PROJECT_ID}/inventory.troshka.yml"
ansible-playbook -i "$INV" main.yml --tags sync_overlays
```

## 1. Deploy from template

Import or deploy `troshka/example_templates/net-automation-workshop.yaml` as a
new project. Wait until all six VMs are **active** and powered on.

Gateway exposes only **`:80`** (Showroom). Students use Showroom tabs for AAP
and VS Code — **SSH to Linux VMs is not required** for the workshop.

## 2. Bootstrap vscode (first time only)

```bash
export TROSHKA_PROJECT_ID=<project-uuid>
export TROSHKA_API_URL=http://localhost:8200   # or prod URL

# RHSM — demosat activation key (see Prerequisites)
SATCREDS=~/zt-ansiblebu-agnosticv/includes/secrets/demosat-rhel-8-and-9-latest.yaml
VAULT=~/secrets/vault_pw.txt
vault_var() {
  ansible localhost -m debug -a "var=$1" -e @"$SATCREDS" --vault-password-file "$VAULT" 2>/dev/null \
    | python3 -c "import sys,re; m=re.search(r'\"$1\": \"([^\"]+)\"', sys.stdin.read()); print(m.group(1) if m else '')"
}
export SATELLITE_URL=$(vault_var satellite_url)
export REG_ORG=$(vault_var satellite_org)
export REG_ACTIVATION_KEY=$(vault_var satellite_activationkey)
export SATELLITE_HA=true

cd ~/demo_workloads/playbooks/net-automation-workshop
INV=".generated/${TROSHKA_PROJECT_ID}/inventory.troshka.yml"

ansible-playbook main.yml --limit localhost -e "workshop_inventory_path=${INV}"
ansible-playbook -i "$INV" main.yml --skip-tags always --tags bootstrap
```

See `roles/troshka_workload_net_automation_workshop/readme.adoc` for tags and options.

Skip this step if vscode `:8080` already works and exercise files are present
(e.g. re-capture after a small change).

Control uses the AAP CEH gold image — AAP on `:443` should be up without
bootstrap. The control play only pulls EEs and clones the upstream workshop
repo (optional for pattern capture).

## 3. Bootstrap routers (required before first capture)

```bash
cd ~/demo_workloads/playbooks/net-automation-workshop
INV=".generated/${TROSHKA_PROJECT_ID}/inventory.troshka.yml"
ansible-playbook -i "$INV" main.yml --tags configure_routers
```

Force re-push: `-e force_router_bootstrap=true`

Optional — wait for router SSH from control over lab L2:

```bash
ansible-playbook -i "$INV" main.yml --tags wait_routers
```

## 4. Verify before capture

```bash
curl -s  http://<gateway-host>/           # Showroom UI
curl -sk https://<gateway-host>/aap/      # AAP UI (Showroom proxy tab)
curl -s  http://<gateway-host>/vscode/   # code-server (Showroom proxy tab)
```

| Host | Lab IP | Credentials |
|------|--------|-------------|
| rtr1 | 172.20.20.10 | `admin` / `admin@123` |
| rtr2 | 172.20.20.20 | `admin` / `admin@123` |
| rtr3 | 172.20.20.30 | `admin` / `admin@123` |
| rtr4 | 172.20.20.40 | `admin` / `admin@123` |

rtr3 (vSRX) is slow to boot — allow **10–15 minutes** after power-on.

## 5. Capture the pattern

1. Open the project in the Troshka UI.
2. **Save as Pattern** (not “Export template”).
3. Name it (e.g. `net-automation-workshop-golden`).
4. Leave VMs running (default) so disks are frozen consistently.
5. Track progress on **Library → Patterns** until state is `available`.

## 6. Deploy from pattern

Create a new project from the pattern. After deploy:

- **:80** (Showroom) with **/aap/** and **/vscode/** tabs should work without bootstrap.
- Routers should answer SSH without `configure_routers`.
- rtr3 still needs boot time — no serial bootstrap.

## Quick reference

```bash
export TROSHKA_PROJECT_ID=<uuid>
cd ~/demo_workloads/playbooks/net-automation-workshop
INV=".generated/${TROSHKA_PROJECT_ID}/inventory.troshka.yml"
ansible-playbook main.yml --limit localhost -e "workshop_inventory_path=${INV}"
ansible-playbook -i "$INV" main.yml --skip-tags always --tags full
```

## Troubleshooting

### Bootstrap plays skip all hosts ("no hosts matched")

The inventory file must end in `.troshka.yml`. Check:

```bash
ansible-inventory -i "$INV" --list
```

If only `ungrouped` appears, re-run prepare with
`-e workshop_inventory_path=${INV}` and confirm the file extension.

### `Install vscode packages` — no packages available

The vscode VM is not registered with RHSM. Extract `satellite_org` and
`satellite_activationkey` from
`~/zt-ansiblebu-agnosticv/includes/secrets/demosat-rhel-8-and-9-latest.yaml`
using the `vault_var` helper in Prerequisites. Export as `REG_ORG` /
`REG_ACTIVATION_KEY` and set `SATELLITE_HA=true` (the secret has
`set_repositories_satellite_ha: true` — registration must use the Satellite HA
`subscription-manager` path, not portal `--auto-attach`).

```bash
ansible -i "$INV" vscode -m shell -a \
  "subscription-manager config --list | grep -E 'hostname|prefix'; subscription-manager identity"
```

### Parallel bootstrap corrupts Ansible collection

Install the collection once before starting background jobs:

```bash
ansible-galaxy collection install ~/troshka-ansible-collection \
  -p ~/.ansible/collections --force
```

### "Pattern disks are not available on any host"

In the Troshka repo:

```bash
cd ~/troshka/src/backend
./venv/bin/python3 -m app.scripts.backfill_pattern_locations
```
