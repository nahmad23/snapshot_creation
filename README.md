# Multi-vCenter VM Snapshot Playbook

Creates snapshots for VMs listed in a CSV inventory file across two vCenter
servers. Supports 100+ VMs by running snapshot tasks asynchronously in
configurable batches. Credentials are entered interactively — nothing is
stored in files.

## File Structure

```
snapshot_creation/
├── create_snapshots.yml   # main playbook
├── vm_inventory.csv       # VM inventory (edit this with your real VMs)
├── inventory.ini          # localhost inventory
├── requirements.yml       # Ansible collection dependencies
└── README.md
```

## Prerequisites

```bash
pip install ansible pyVmomi
ansible-galaxy collection install -r requirements.yml
```

## CSV Inventory Format

Edit `vm_inventory.csv` with your real VM names. Three columns are required:

| Column | Description |
|---|---|
| `vm_name` | VM display name exactly as it appears in vCenter |
| `datacenter` | Datacenter name the VM belongs to |
| `folder` | VM folder path in vCenter, e.g. `/DatacenterName/vm` or `/DatacenterName/vm/subfolder` |
| `vcenter` | Hostname or IP of the vCenter that manages this VM — must match exactly what you type at the prompt |

**Example:**

```csv
vm_name,datacenter,folder,vcenter
web-server-01,DatacenterA,/DatacenterA/vm,us1sitvmw01v.company.com
db-server-01,DatacenterA,/DatacenterA/vm,us1sitvmw01v.company.com
app-server-01,DatacenterB,/DatacenterB/vm,us2sitvmw01v.company.com
db-server-11,DatacenterB,/DatacenterB/vm,us2sitvmw01v.company.com
```

For your environment, the folder path follows the pattern `/<DatacenterName>/vm`.
If VMs are in a subfolder (e.g. "Servers"), it would be `/<DatacenterName>/vm/Servers`.
You can verify the exact path in the vCenter UI under Inventory > VMs and Templates.

The sample `vm_inventory.csv` ships with 100 VMs — 50 per vCenter using
placeholder hostnames `vc1.example.com` and `vc2.example.com`.
Replace all three columns with your real VM names, datacenters, and vCenter hostnames.

## Usage

```bash
ansible-playbook -i inventory.ini create_snapshots.yml
```

You will be prompted for:

| Prompt | Description |
|---|---|
| vCenter 1 hostname or IP | FQDN or IP of the first vCenter |
| vCenter 1 username | e.g. `administrator@vsphere.local` |
| vCenter 1 password | Hidden input |
| vCenter 2 hostname or IP | FQDN or IP of the second vCenter |
| vCenter 2 username | e.g. `administrator@vsphere.local` |
| vCenter 2 password | Hidden input |
| Snapshot name | Default: `pre-change-snapshot` |
| Snapshot description | Default: `Automated snapshot before change` |
| Path to VM inventory CSV | Default: `vm_inventory.csv` |

## What it does

1. Validates the CSV file exists and has the required columns.
2. Splits VMs from the CSV into two groups — `vcenter1` and `vcenter2`.
3. Shows an inventory summary before starting (VM counts per vCenter).
4. Fires snapshot tasks asynchronously in batches of 10 per vCenter.
5. Waits for all jobs to finish — one failed VM never stops the rest.
6. Prints per-vCenter and combined totals with succeeded/failed counts.
7. Lists any failed VMs by name and exits non-zero if any failed.

## Playbook flow

```
PLAY 1 — Load CSV + gather credentials
  ├─ Validate CSV exists and has correct columns
  ├─ Split 100 VMs → vc1_vms (50) and vc2_vms (50)
  └─ Show inventory summary

PLAY 2 — vCenter 1 (50 VMs)
  ├─ Fire all 50 snapshot jobs asynchronously (batch of 10)
  ├─ Poll async_status until all 50 finish
  └─ Summary: succeeded / failed

PLAY 3 — vCenter 2 (50 VMs)
  ├─ Same as vCenter 1
  └─ Summary: succeeded / failed

PLAY 4 — Final report
  ├─ Combined totals across both vCenters
  ├─ List failed VMs by name
  └─ Exit non-zero if any failed
```

## Tuning

**Batch size** (`batch_size` in `create_snapshots.yml`, default `10`):
Controls concurrent snapshot submissions per vCenter before a 1-second pause.
Raise to `20` only if your storage I/O can handle it.

**Per-VM timeout** (`async: 600`): 10 minutes per VM. Increase for very large VMs.

**Snapshot options:**
- `quiesce: true` — freeze guest filesystem (requires VMware Tools)
- `memory_dump: true` — include RAM state (slower, larger snapshot)

## Use a different CSV at runtime

```bash
ansible-playbook -i inventory.ini create_snapshots.yml \
  -e "csv_file=/path/to/my_custom_inventory.csv"
```

This skips the interactive CSV prompt and uses the file you specify directly.

## Example output

```
CSV loaded: 100 total VMs
  vCenter 1 (vc1.example.com): 50 VMs
  vCenter 2 (vc2.example.com): 50 VMs

=== vCenter 1 (vc1.example.com) — snapshot: 'pre-change-snapshot' ===
  Total VMs : 50
  Succeeded : 50
  Failed    : 0
  Failed VMs: none

=== vCenter 2 (vc2.example.com) — snapshot: 'pre-change-snapshot' ===
  Total VMs : 50
  Succeeded : 50
  Failed    : 0
  Failed VMs: none

=========================================
 SNAPSHOT RUN COMPLETE
=========================================
 Snapshot name : pre-change-snapshot
 CSV inventory : 100 VMs total
-----------------------------------------
 vCenter 1 (vc1.example.com)
   Total VMs  : 50
   Succeeded  : 50
   Failed     : 0
 vCenter 2 (vc2.example.com)
   Total VMs  : 50
   Succeeded  : 50
   Failed     : 0
-----------------------------------------
 Total succeeded : 100
 Total failed    : 0
=========================================
```
