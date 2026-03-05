# Windows VMware VM — Terraform Template

Terraform template for deploying Windows virtual machines on vSphere. Wraps the [`Jeff8247/module-vmware-virtual-machine`](https://github.com/Jeff8247/module-vmware-virtual-machine) module with Windows-specific defaults, input validation, and sensible out-of-the-box configuration.

## Requirements

| Tool | Version |
|------|---------|
| Terraform | `>= 1.3, < 2.0` |
| vSphere provider | `~> 2.6` |
| vCenter | 7.0+ recommended |

A Windows VM template with VMware Tools installed must already exist in vCenter and have been prepared with Sysprep support (i.e. not yet Sysprepped — the provider runs Sysprep on clone).

## Quick Start

```bash
# 1. Copy the example vars file and fill in your values
cp terraform.tfvars.example terraform.tfvars
$EDITOR terraform.tfvars

# 2. Set credentials via environment variables (recommended — avoids storing them in files)
export TF_VAR_vsphere_password="..."
export TF_VAR_windows_admin_password="..."

# 3. Initialize and deploy
terraform init
terraform plan
terraform apply
```

## Credentials

Passwords should **not** be stored in `terraform.tfvars`. Use environment variables instead:

```bash
export TF_VAR_vsphere_password="your-vcenter-password"
export TF_VAR_windows_admin_password="your-local-admin-password"
export TF_VAR_windows_domain_password="your-domain-join-password"  # if joining AD
```

The `.gitignore` in this repo excludes `terraform.tfvars` and `*.auto.tfvars` to prevent accidental commits of credentials.

## Examples

### Minimal — DHCP, workgroup, single NIC

```hcl
vsphere_server         = "vcenter.example.com"
vsphere_user           = "administrator@vsphere.local"
datacenter             = "dc01"
cluster                = "cluster01"
datastore              = "datastore01"
vm_name                = "win-test-01"
template_name          = "WIN2022-TEMPLATE"
guest_id               = "windows2022srvNext_64Guest"

network_interfaces = [{ network_name = "VM Network" }]
```

### Static IP

```hcl
network_interfaces = [{ network_name = "VM Network" }]

ip_settings = [
  {
    ipv4_address = "192.168.1.100"
    ipv4_netmask = 24
  }
]

ipv4_gateway    = "192.168.1.1"
dns_servers     = ["192.168.1.10", "192.168.1.11"]
dns_suffix_list = ["corp.example.com"]
```

### Active Directory domain join

```hcl
windows_domain          = "corp.example.com"
windows_domain_user     = "svc-domain-join"
# windows_domain_password via TF_VAR_windows_domain_password
windows_workgroup       = null
```

### Multiple disks

```hcl
disks = [
  {
    label            = "disk0"
    size             = 100
    unit_number      = 0
    thin_provisioned = true
  },
  {
    label            = "disk1"
    size             = 500
    unit_number      = 1
    thin_provisioned = true
  }
]
```

### Run a script on first boot

```hcl
windows_run_once = [
  "powershell.exe -ExecutionPolicy Bypass -File C:\\setup.ps1"
]
```

## Variable Reference

### vCenter Connection

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `vsphere_server` | `string` | required | vCenter server hostname or IP |
| `vsphere_user` | `string` | required | vCenter username |
| `vsphere_password` | `string` | required | vCenter password (sensitive) |
| `vsphere_allow_unverified_ssl` | `bool` | `false` | Skip TLS certificate verification |

**`vsphere_server`** — hostname or IP address only, no protocol or port (e.g. `vcenter.example.com`, not `https://vcenter.example.com`).

**`vsphere_user`** — the vSphere provider requires UPN format: `user@domain` (e.g. `administrator@vsphere.local`). The `DOMAIN\user` format is not supported.

### Infrastructure Placement

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `datacenter` | `string` | required | vSphere datacenter name |
| `cluster` | `string` | `null` | vSphere cluster name (mutually exclusive with `host`) |
| `host` | `string` | `null` | vSphere host name (mutually exclusive with `cluster`) |
| `resource_pool` | `string` | `null` | Resource pool name; `null` uses the cluster/host root pool |
| `datastore` | `string` | `null` | Datastore name (mutually exclusive with `datastore_cluster`) |
| `datastore_cluster` | `string` | `null` | Datastore cluster name (mutually exclusive with `datastore`) |

All inventory names (`datacenter`, `cluster`, `host`, `datastore`, `datastore_cluster`, `resource_pool`) must match **exactly** as they appear in the vCenter inventory — they are case-sensitive. Find them in the vSphere Client under the Hosts & Clusters and Storage views.

Set exactly one of `cluster` or `host`, and exactly one of `datastore` or `datastore_cluster`. The template enforces this with `check` blocks that fail at plan time if both are set.

### VM Identity

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `vm_name` | `string` | required | VM name in vSphere inventory (max 80 chars) |
| `vm_folder` | `string` | `null` | vSphere folder path, e.g. `"VMs/Windows"` |
| `annotation` | `string` | `null` | VM notes / annotation |
| `tags` | `map(string)` | `{}` | vSphere tags as `{ category = "tag-name" }` |

### Template

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `template_name` | `string` | required | vSphere template to clone |
| `template_datacenter` | `string` | `null` | Datacenter where the template lives (if different from target) |
| `linked_clone` | `bool` | `false` | Create a linked clone instead of a full clone |

### CPU

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `num_cpus` | `number` | `2` | Total vCPU count |
| `num_cores_per_socket` | `number` | `1` | Cores per socket |
| `cpu_hot_add_enabled` | `bool` | `false` | Allow CPU hot-add |
| `cpu_reservation` | `number` | `0` | CPU reservation in MHz |
| `cpu_limit` | `number` | `-1` | CPU limit in MHz (`-1` = unlimited) |
| `cpu_share_level` | `string` | `"normal"` | Share level: `low`, `normal`, `high`, or `custom` |

### Memory

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `memory` | `number` | `4096` | Memory in MB — must be a multiple of 4 |
| `memory_hot_add_enabled` | `bool` | `false` | Allow memory hot-add |
| `memory_reservation` | `number` | `0` | Memory reservation in MB |
| `memory_limit` | `number` | `-1` | Memory limit in MB (`-1` = unlimited) |
| `memory_share_level` | `string` | `"normal"` | Share level: `low`, `normal`, `high`, or `custom` |

### Storage

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `disks` | `list(object)` | 100 GB thin disk | List of disk configs — see [Disk Object](#disk-object) |
| `scsi_type` | `string` | `"pvscsi"` | SCSI controller type: `pvscsi` or `lsilogicsas` |
| `scsi_controller_count` | `number` | `1` | Number of SCSI controllers |

#### Disk Object

```hcl
{
  label            = "disk0"          # required
  size             = 100              # required, in GB
  unit_number      = 0                # optional, SCSI unit number
  thin_provisioned = true             # optional, default true
  eagerly_scrub    = false            # optional, default false
  datastore        = null             # optional, override per-disk datastore
}
```

### Networking

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `network_interfaces` | `list(object)` | required | At least one NIC — see [Network Interface Object](#network-interface-object) |
| `ip_settings` | `list(object)` | `[]` | Static IP per NIC — leave empty for DHCP |
| `ipv4_gateway` | `string` | `null` | Default IPv4 gateway |
| `dns_servers` | `list(string)` | `[]` | DNS server addresses |
| `dns_suffix_list` | `list(string)` | `[]` | DNS search suffixes |

#### Network Interface Object

```hcl
{
  network_name = "VM Network"   # required — port group or DVS port group name
  adapter_type = "vmxnet3"      # optional — vmxnet3 (default), e1000e, or e1000
}
```

#### IP Settings Object

```hcl
{
  ipv4_address = "192.168.1.100"   # required
  ipv4_netmask = 24                # required — prefix length, 1–32
}
```

One entry per NIC, in the same order as `network_interfaces`.

### Guest OS — Windows

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `guest_id` | `string` | required | vSphere guest OS ID, e.g. `windows2022srvNext_64Guest` |
| `windows_admin_password` | `string` | required | Local Administrator password (sensitive) |
| `computer_name` | `string` | `null` | NetBIOS name, max 15 chars. Defaults to first 15 chars of `vm_name` |
| `domain` | `string` | `null` | DNS domain suffix for guest customization |
| `time_zone` | `number` | `85` | Windows timezone index (0–235). `85` = Eastern Standard Time |
| `windows_domain` | `string` | `null` | AD domain to join — `null` skips domain join |
| `windows_domain_user` | `string` | `null` | AD user with join permissions |
| `windows_domain_password` | `string` | `null` | Domain join password (sensitive) |
| `windows_workgroup` | `string` | `"WORKGROUP"` | Workgroup name when not joining a domain |
| `windows_auto_logon` | `bool` | `false` | Auto-logon Administrator after Sysprep |
| `windows_auto_logon_count` | `number` | `1` | Number of auto-logon sessions |
| `windows_run_once` | `list(string)` | `[]` | Commands to run once at first boot (RunOnce) |

Common `guest_id` values:

| OS | `guest_id` |
|----|-----------|
| Windows Server 2025 | `windows2025srv_64Guest` |
| Windows Server 2022 | `windows2022srvNext_64Guest` |
| Windows Server 2019 | `windows2019srv_64Guest` |
| Windows 11 | `windows11_64Guest` |
| Windows 10 | `windows9_64Guest` |

Common `time_zone` values:

| Index | Timezone |
|-------|----------|
| `85` | Eastern Standard Time |
| `90` | Central Standard Time |
| `96` | Mountain Standard Time |
| `105` | Pacific Standard Time |
| `0` | Dateline Standard Time |
| `110` | GMT Standard Time |

Full list: [Microsoft Windows Time Zone Index Values](https://learn.microsoft.com/en-us/previous-versions/windows/embedded/ms912391(v=winembedded.11))

### Hardware

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `firmware` | `string` | `"efi"` | Firmware type: `efi` or `bios` |
| `hardware_version` | `number` | `null` | VMware hardware version; `null` keeps the template version |
| `enable_disk_uuid` | `bool` | `true` | Expose disk UUIDs to the guest OS |
| `vbs_enabled` | `bool` | `false` | Enable Virtualization-Based Security (requires EFI) |
| `wait_for_guest_net_timeout` | `number` | `5` | Minutes to wait for guest networking (`0` disables) |
| `wait_for_guest_net_routable` | `bool` | `true` | Require a routable IP before marking VM ready |
| `customize_timeout` | `number` | `60` | Minutes to wait for Sysprep to complete |
| `extra_config` | `map(string)` | `{}` | Additional VMX key/value pairs |

## Outputs

| Output | Description |
|--------|-------------|
| `vm_name` | Name of the deployed virtual machine |
| `vm_id` | Managed object ID (MOID) of the VM |
| `vm_uuid` | BIOS UUID — useful for CMDB and monitoring integration |
| `power_state` | Current power state of the VM |
| `default_ip_address` | Primary IP address as reported by VMware Tools |
| `ip_addresses` | All IP addresses reported by VMware Tools |

## File Structure

```
.
├── main.tf                    # Module call, locals, mutual-exclusivity checks
├── variables.tf               # All input variables with validation
├── outputs.tf                 # Outputs exposed after deployment
├── versions.tf                # Terraform and provider version constraints
├── terraform.tfvars.example   # Annotated example — copy to terraform.tfvars
└── .gitignore                 # Excludes state, .terraform/, and tfvars files
```

## Security Notes

- `vsphere_password`, `windows_admin_password`, and `windows_domain_password` are marked `sensitive = true` and will not appear in plan/apply output.
- `terraform.tfvars` is excluded by `.gitignore`. Never commit credentials.
- `vsphere_allow_unverified_ssl` defaults to `false`. Only set to `true` in non-production lab environments.
- Terraform state (`terraform.tfstate`) contains all resource attributes including sensitive values. Store state in a secured remote backend (e.g. S3 with encryption, Terraform Cloud) for any shared or production use. See `versions.tf` for where to add a backend block.
