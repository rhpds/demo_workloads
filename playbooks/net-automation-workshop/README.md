# Network Automation Workshop bootstrap

Playbook for first-time lab setup and pattern capture. Full workflow:
**PATTERN-BUILD.md**. Role reference:
**../../roles/troshka_workload_net_automation_workshop/readme.adoc**.

## Quick start (single project)

```bash
export TROSHKA_PROJECT_ID=<uuid>
export TROSHKA_API_URL=http://localhost:8200   # optional

# RHSM — demosat activation key (required before bootstrap)
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

# Prepare dynamic inventory (always run from playbook dir)
ansible-playbook main.yml --limit localhost -e "workshop_inventory_path=${INV}"

# Bootstrap vscode + control
ansible-playbook -i "$INV" main.yml --skip-tags always --tags bootstrap
```

RHSM credentials are **required** on first bootstrap for vscode package installs
(git, podman, sshpass, python3-pip). Use the demosat secret above (vault
password `~/secrets/vault_pw.txt`) — see PATTERN-BUILD.md § Prerequisites for
Satellite HA registration details. This differs from the original zt-ansiblebu
CI, which applies demosat content views at provision time.

## Inventory file naming

The `troshka.cloud.troshka` inventory plugin only accepts files ending in
`.troshka.yml` or `.troshka.yaml`. The role default path
(`.generated/troshka_inventory.yml`) does not match — use a per-project path as
shown above, or any `*.troshka.yml` path via `-e workshop_inventory_path=...`.

Verify inventory before bootstrap:

```bash
ansible-inventory -i "$INV" --list | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print([k for k in d if not k.startswith('_')])"
# expect: workshop_control, workshop_vscode, routers, ...
```

## Parallel bootstrap (multiple projects)

Use a separate inventory file per project. Install the Ansible collection once
before fanning out — parallel `ansible-galaxy collection install --force` calls
race and can corrupt `~/.ansible/collections`.

```bash
ansible-galaxy collection install ~/troshka-ansible-collection \
  -p ~/.ansible/collections --force

# Export demosat vars once (see Quick start)
SATCREDS=~/zt-ansiblebu-agnosticv/includes/secrets/demosat-rhel-8-and-9-latest.yaml
VAULT=~/secrets/vault_pw.txt
vault_var() {
  ansible localhost -m debug -a "var=$1" -e @"$SATCREDS" --vault-password-file "$VAULT" 2>/dev/null \
    | python3 -c "import sys,re; m=re.search(r'\"$1\": \"([^\"]+)\"', sys.stdin.read()); print(m.group(1) if m else '')"
}
export REG_ORG=$(vault_var satellite_org)
export REG_ACTIVATION_KEY=$(vault_var satellite_activationkey)
export SATELLITE_HA=true

bootstrap_one() {
  local pid="$1"
  local inv=".generated/${pid}/inventory.troshka.yml"
  export TROSHKA_PROJECT_ID="$pid"
  ansible-playbook main.yml --limit localhost -e "workshop_inventory_path=${inv}"
  ansible-playbook -i "$inv" main.yml --skip-tags always --tags bootstrap
}

bootstrap_one <uuid-1> > /tmp/bootstrap-1.log 2>&1 &
bootstrap_one <uuid-2> > /tmp/bootstrap-2.log 2>&1 &
wait
```
