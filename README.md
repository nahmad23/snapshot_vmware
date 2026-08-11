# Multi-vCenter VM Snapshot Playbooks (non-interactive, Vault-backed)

Two playbooks that manage the full snapshot lifecycle for VMs listed in a CSV
inventory, across **any number** of vCenter servers:

| Playbook | Purpose |
|---|---|
| `create_snapshots.yml` | Creates a `yyyymmdd-security-update` snapshot on every VM in the CSV |
| `delete_snapshots.yml` | **Retention** — deletes those snapshots once they are older than 10 days |

Both runs are **fully non-interactive** — there are no prompts during execution:

* vCenter **username and password** are read from **Ansible Vault**.
* The **snapshot name is auto-generated** in the format `yyyymmdd-security-update`
  (e.g. `20260625-security-update`).
* vCenter hostnames are auto-detected from the CSV `vcenter` column.

## File Structure

```
snapshot_vmware/
├── create_snapshots.yml          # create snapshots (no prompts)
├── delete_snapshots.yml          # delete snapshots older than N days (no prompts)
├── tasks/
│   ├── snapshot_vcenter.yml         # per-vCenter snapshot creation logic
│   └── delete_snapshots_vcenter.yml # per-vCenter retention/deletion logic
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
names the playbooks use:

```yaml
vcenter_username: "{{ vault_vcenter_username }}"
vcenter_password: "{{ vault_vcenter_password }}"
```

## vCenters with their own credentials

Not every vCenter shares one login. Any vCenter can be given its own username
and password through the `vcenter_credentials` map in
`group_vars/all/vars.yml`, keyed by the vCenter's **hostname exactly as it
appears in the CSV**:

```yaml
vcenter_credentials:
  # OVH Private Cloud vCenter (https://pcc-51-68-8-97.ovh.com)
  pcc-51-68-8-97.ovh.com:
    username: "{{ vault_ovh_vcenter_username | default('') }}"
    password: "{{ vault_ovh_vcenter_password | default('') }}"
```

with the matching secrets in the encrypted `group_vars/all/vault.yml`:

```yaml
# shared / default login
vault_vcenter_username: "administrator@vsphere.local"
vault_vcenter_password: "your-real-password"

# OVH vCenter's own login
vault_ovh_vcenter_username: "your-ovh-user"
vault_ovh_vcenter_password: "your-ovh-password"
```

Add them to an existing encrypted vault with:

```bash
ansible-vault edit group_vars/all/vault.yml
```

**Any vCenter not listed in `vcenter_credentials` keeps using the shared
`vcenter_username`/`vcenter_password`**, so existing vCenters need no changes.

Both playbooks print which credential set each vCenter resolved to at the start
of every run:

```
ggnsitvmw01v.unitedlex.global -> shared default credentials
pcc-51-68-8-97.ovh.com     -> dedicated credentials (vcenter_credentials)
```

If a vCenter ends up with an empty username or password — usually a misspelled
vault variable — the run stops before touching vCenter with a message naming
the vCenter in question.

To add a third vCenter with its own login, add one block to
`vcenter_credentials`, two variables to the vault, and its VM rows to the CSV.

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
ovh-vm-01,YourOvhDatacenter,pcc-51-68-8-97.ovh.com
```

The `vcenter` column is normalised to a bare lowercase hostname before use, so
pasting a full URL (`https://pcc-51-68-8-97.ovh.com/ui`) still resolves to
`pcc-51-68-8-97.ovh.com` and matches its `vcenter_credentials` entry.
Blank rows are ignored.

> Do **not** put `#` comment lines in the CSV — it is parsed as plain CSV and a
> comment line would be read as a VM row.

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

---

# Snapshot retention — `delete_snapshots.yml`

Deletes snapshots **older than 10 days** from every VM in the same CSV. It uses
the same Vault credentials, the same CSV and the same multi-vCenter handling as
`create_snapshots.yml`, so there is nothing extra to configure.

```bash
# See what WOULD be deleted — deletes nothing
ansible-playbook -i inventory.ini delete_snapshots.yml --ask-vault-pass \
  -e "delete_dry_run=true"

# Actually delete
ansible-playbook -i inventory.ini delete_snapshots.yml --ask-vault-pass

# Unattended (cron)
ansible-playbook -i inventory.ini delete_snapshots.yml \
  --vault-password-file ~/.vault_pass
```

> Run it once with `-e "delete_dry_run=true"` before the first real run.

## What it does

1. Runs the same preflight checks as the create playbook (Vault credentials, PyVmomi, CSV).
2. Computes a cutoff timestamp of `now (UTC) − snapshot_retention_days`.
3. Queries every VM in the CSV for its existing snapshots (asynchronously, per vCenter).
4. Selects snapshots that are **both** older than the cutoff **and** whose name
   matches `snapshot_name_pattern`.
5. Deletes them asynchronously — one failure never stops the rest.
6. Prints per-vCenter and combined found/deleted/failed counts, and exits
   non-zero if any deletion failed.

## Safety

The name filter is the important part. By default only snapshots whose name ends
in `-security-update` — i.e. the ones `create_snapshots.yml` created — are ever
eligible for deletion. A manual snapshot called `before-db-migration` is left
alone no matter how old it is.

Child snapshots are also protected: `snapshot_remove_children` is `false`, so
deleting an expired parent consolidates its delta into the disk (standard VMware
behaviour) rather than destroying newer snapshots taken on top of it.

## Options

All of these are overridable with `-e` at runtime:

| Variable | Default | Description |
|---|---|---|
| `snapshot_retention_days` | `10` | Delete snapshots older than this many days |
| `snapshot_name_pattern` | `-security-update$` | Regex a snapshot name must match to be eligible. Set to `""` to consider **every** snapshot (dangerous) |
| `delete_dry_run` | `false` | Report expired snapshots without deleting anything |
| `snapshot_remove_children` | `false` | Also delete child snapshots of an expired snapshot |
| `fail_on_unreadable` | `false` | Exit non-zero if a CSV VM could not be queried (renamed/decommissioned). Off by default so a stale CSV row does not break a cron run — such VMs are always listed in the report |
| `csv_file` | `vm_inventory.csv` | Path to the VM inventory CSV |

```bash
# Keep snapshots for 14 days instead of 10
ansible-playbook -i inventory.ini delete_snapshots.yml --ask-vault-pass \
  -e "snapshot_retention_days=14"
```

## Scheduling both playbooks

Snapshots created on day 0 are removed on day 10 by the retention run, so a
daily cron entry for each is all that is needed:

```cron
# Create the security-update snapshots at 01:00
0 1 * * * cd /opt/snapshot_vmware && ansible-playbook -i inventory.ini create_snapshots.yml --vault-password-file ~/.vault_pass >> /var/log/vm_snapshots.log 2>&1

# Purge snapshots older than 10 days at 03:00
0 3 * * * cd /opt/snapshot_vmware && ansible-playbook -i inventory.ini delete_snapshots.yml --vault-password-file ~/.vault_pass >> /var/log/vm_snapshot_purge.log 2>&1
```

## Retention tuning

* **Info timeout** (`async: 300`): 5 minutes per VM to read its snapshot list.
* **Delete timeout** (`async: 1800`, `retries: 120`, `delay: 15`): up to 30 minutes
  per snapshot removal. Consolidating a large delta disk can be slow — raise these
  if you snapshot very busy VMs.
* Age comparison is done in **UTC** against the `creation_time` vCenter reports,
  so controller timezone does not affect which snapshots expire.
