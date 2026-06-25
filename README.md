# Multi-vCenter VM Snapshot Playbook (non-interactive, Vault-backed)

Creates snapshots for VMs listed in a CSV inventory across **any number** of
vCenter servers. The run is **fully non-interactive** — there are no prompts
during execution:

* vCenter **username and password** are read from **Ansible Vault**.
* The **snapshot name is auto-generated** in the format `yyyymmdd-security-update`
  (e.g. `20260625-security-update`).
* vCenter hostnames are auto-detected from the CSV `vcenter` column.

## File Structure

```
snapshot_vmware/
├── create_snapshots.yml          # main playbook (no prompts)
├── tasks/
│   └── snapshot_vcenter.yml       # per-vCenter snapshot logic
├── group_vars/
│   └── all/
│       ├── vars.yml               # maps vcenter_username/password -> vault vars
│       └── vault.yml.example      # template — copy to vault.yml and ENCRYPT
├── vm_inventory.csv               # VM inventory (edit with your real VMs)
├── inventory.ini                  # localhost inventory
├── ansible.cfg                    # points at inventory.ini
├── requirements.yml               # Ansible collection dependencies
└── README.md
```

## Prerequisites

```bash
pip install ansible pyVmomi
ansible-galaxy collection install -r requirements.yml
```

## One-time setup: store credentials in Ansible Vault

The playbook reads `vault_vcenter_username` and `vault_vcenter_password` from
`group_vars/all/vault.yml`. That file is **git-ignored** so plaintext secrets
are never committed.

```bash
# 1. Create the encrypted vault file
cp group_vars/all/vault.yml.example group_vars/all/vault.yml

# 2. Put your real credentials in it, then encrypt:
ansible-vault encrypt group_vars/all/vault.yml

# (or do both in one step)
ansible-vault create group_vars/all/vault.yml
```

`group_vars/all/vault.yml` must define:

```yaml
vault_vcenter_username: "administrator@vsphere.local"
vault_vcenter_password: "your-real-password"
```

`group_vars/all/vars.yml` (already provided, non-secret) wires those into the
names the playbook uses:

```yaml
vcenter_username: "{{ vault_vcenter_username }}"
vcenter_password: "{{ vault_vcenter_password }}"
```

## CSV Inventory Format

Edit `vm_inventory.csv` with your real VMs. Three columns are required:

| Column | Description |
|---|---|
| `vm_name` | VM display name exactly as it appears in vCenter |
| `datacenter` | Datacenter name the VM belongs to (folder is derived as `/<datacenter>/vm`) |
| `vcenter` | Hostname or IP of the vCenter that manages this VM |

```csv
vm_name,datacenter,vcenter
web-server-01,DatacenterA,vc1.example.com
db-server-01,DatacenterA,vc1.example.com
app-server-01,DatacenterB,vc2.example.com
```

## Usage

No prompts — just supply the Vault password:

```bash
# Prompt for the vault password interactively
ansible-playbook -i inventory.ini create_snapshots.yml --ask-vault-pass

# ...or use a password file (great for automation/cron)
ansible-playbook -i inventory.ini create_snapshots.yml \
  --vault-password-file ~/.vault_pass
```

Use a different CSV at runtime:

```bash
ansible-playbook -i inventory.ini create_snapshots.yml \
  --ask-vault-pass -e "csv_file=/path/to/my_inventory.csv"
```

## What it does

1. Asserts the Vault credentials are present (clear error if the vault wasn't supplied).
2. Validates the CSV file exists and has the required columns.
3. Auto-generates the snapshot name `yyyymmdd-security-update`.
4. Auto-detects all unique vCenters from the CSV `vcenter` column.
5. Fires snapshot tasks asynchronously per vCenter — one failed VM never stops the rest.
6. Prints per-vCenter and combined succeeded/failed counts.
7. Lists any failed VMs by name and exits non-zero if any failed.

## Snapshot name format

The name is built at runtime with Jinja's `now()`:

```yaml
snapshot_name: "{{ now(utc=true, fmt='%Y%m%d') }}-security-update"
```

Override it for a one-off run if ever needed:

```bash
ansible-playbook -i inventory.ini create_snapshots.yml \
  --ask-vault-pass -e "snapshot_name=manual-snap"
```

## Tuning

* **Per-VM timeout** (`async: 600` in `tasks/snapshot_vcenter.yml`): 10 minutes per VM.
* **Snapshot options:** `quiesce: true` freezes the guest filesystem (needs VMware Tools);
  `memory_dump: true` includes RAM state (slower, larger).
