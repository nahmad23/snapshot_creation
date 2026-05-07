# Multi-vCenter VM Snapshot Playbook

Creates snapshots for all VMs across two vCenter servers. Supports 100+ VMs
by running snapshot tasks asynchronously in configurable batches. Credentials
are entered interactively — nothing is stored in files.

## Prerequisites

```bash
pip install ansible pyVmomi
ansible-galaxy collection install -r requirements.yml
```

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

## What it does

1. Connects to both vCenters and retrieves all VMs.
2. Fires snapshot tasks asynchronously in batches of 10 per vCenter (configurable).
3. Waits for all jobs to finish, tolerating individual VM failures.
4. Prints a summary showing succeeded/failed counts per vCenter.
5. Lists any failed VMs by name and exits non-zero if any failed.

## Tuning batch size

The `batch_size` variable in `create_snapshots.yml` controls how many
concurrent snapshot operations are submitted per vCenter before a 1-second
pause. The default is **10**, which is safe for most vCenter deployments.
Raise it carefully — too many concurrent snapshots can overload storage I/O.

```yaml
vars:
  batch_size: 10   # increase to 20 if your storage can handle it
```

## Options

Edit the `vmware_guest_snapshot` task parameters to change snapshot behaviour:

- `quiesce: true` — freeze guest filesystem for a consistent snapshot (requires VMware Tools)
- `memory_dump: true` — capture RAM state in the snapshot (larger, slower)
- `async: 600` — per-VM timeout in seconds (default 10 min; increase for large VMs)

## Example output

```
=== vCenter 1 (vc1.example.com) — snapshot: 'pre-change-snapshot' ===
  Succeeded : 50
  Failed    : 0
  Failed VMs: none

=== vCenter 2 (vc2.example.com) — snapshot: 'pre-change-snapshot' ===
  Succeeded : 50
  Failed    : 0
  Failed VMs: none

=========================================
 SNAPSHOT RUN COMPLETE
=========================================
 Snapshot name : pre-change-snapshot
-----------------------------------------
 vCenter 1 (vc1.example.com)
   Succeeded : 50
   Failed    : 0
 vCenter 2 (vc2.example.com)
   Succeeded : 50
   Failed    : 0
-----------------------------------------
 Total succeeded : 100
 Total failed    : 0
=========================================
```
