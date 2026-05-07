# Multi-vCenter VM Snapshot Playbook

Creates snapshots for all VMs across two vCenter servers. Credentials are entered interactively at runtime — nothing is stored in files.

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

1. Connects to vCenter 1, retrieves all VMs, creates a snapshot on each.
2. Connects to vCenter 2, retrieves all VMs, creates a snapshot on each.
3. Reports which snapshots were created.

## Options

To skip quiescing the guest OS (default) or include a memory dump, edit the `vmware_guest_snapshot` task parameters in `create_snapshots.yml`:

- `quiesce: true` — freeze the guest filesystem for a consistent snapshot (requires VMware Tools)
- `memory_dump: true` — capture RAM state in the snapshot
