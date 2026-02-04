# oatakan.windows_vcenter_template

Provision a **Windows VM template on VMware vCenter** from a Windows installer ISO that already exists on a vCenter datastore.

This role focuses on the **vCenter lifecycle** (create VM → unattended install via Autounattend → in-guest customization over WinRM → convert to template → optional OVF/OVA export).
For OS “golden image” configuration (updates, drivers/tools, cleanup, sysprep), it delegates to the companion role
[`oatakan.windows_template_build`](https://github.com/oatakan/ansible-role-windows_template_build) over WinRM during the build.

> Note: VMware module calls in `tasks/*.yml` use `validate_certs: no`.

## What this role does

High-level flow (see [tasks/main.yml](tasks/main.yml) and [tasks/provision.yml](tasks/provision.yml)):

1. **Preflight** ([tasks/preflight_check.yml](tasks/preflight_check.yml))
   - Validates the installer ISO exists in the datastore at `{{ datastore_iso_folder }}/{{ iso_file_name }}`.
   - Detects whether a template already exists with `template_vm_name`.
2. **Autounattend ISO** ([tasks/make_iso.yml](tasks/make_iso.yml))
   - Renders `Autounattend.xml` from [templates/windows_server/Autounattend.xml.j2](templates/windows_server/Autounattend.xml.j2).
   - Downloads `ConfigureRemotingForAnsible.ps1` (URL overridable via `winrm_enable_script_url`).
   - Builds a small ISO via `mkisofs` and uploads it to the datastore.
3. **Provision VM** ([tasks/provision_vm.yml](tasks/provision_vm.yml))
   - Creates/powers on the VM with two CDROMs: Windows installer ISO + the Autounattend ISO.
   - Waits for WinRM `5986` to be reachable on `template_vm_ip_address`.
4. **In-guest template build over WinRM** ([tasks/provision.yml](tasks/provision.yml))
   - Adds a runtime host `template_host` and delegates `windows_build_role` (defaults to `oatakan.windows_template_build`).
5. **Finalize**
   - Powers off and removes CDROM drives ([tasks/stop_vm.yml](tasks/stop_vm.yml)).
   - Optionally snapshots ([tasks/create_snapshot.yml](tasks/create_snapshot.yml)).
   - Converts the VM into a vCenter template ([tasks/convert_to_template.yml](tasks/convert_to_template.yml)).
   - Optionally exports OVF/OVA and uploads the OVA back to the datastore ([tasks/export_ovf.yml](tasks/export_ovf.yml)).

Failures are handled with `block`/`rescue`/`always` in [tasks/provision.yml](tasks/provision.yml) to avoid leaving orphaned VMs and datastore artifacts.

## Supported systems

This repo includes Windows unattended templates under [templates/](templates/) (default: `windows_server`).

The `distro_name` mapping in [defaults/main.yml](defaults/main.yml) includes `win2008`, `win2012`, `win2016`, `win2019`, `win2022`, `win10`, `win11`.
Windows Server 2008 R2 has special-case logic in the unattended template (network profile + WinRM bootstrap).

## Requirements

Control node requirements:

- Ansible (this role declares `min_ansible_version: 2.9` in [meta/main.yml](meta/main.yml))
- Ansible collections:
  - `community.vmware` (VMware modules used throughout `tasks/*.yml`)
- A working VMware Python stack available to Ansible (commonly `pyvmomi`)
- `mkisofs` available on the control node (often provided by `genisoimage`)

vCenter requirements:

- Windows installer ISO already uploaded to the datastore (path: `{{ datastore_iso_folder }}/{{ iso_file_name }}`)
- Access to the target `datacenter`, `cluster`, and optional `resource_pool`/`folder`
- Network reachability from the control node to the guest static IP on WinRM TLS (`5986`)

## VMware credentials

The VMware tasks default to environment variables:

- `VMWARE_HOST`
- `VMWARE_USER`
- `VMWARE_PASSWORD`

Alternatively, you can provide auth under `providers.vcenter.hostname`, `providers.vcenter.username`, `providers.vcenter.password`.

## Integration with oatakan.windows_template_build

This role runs OS “golden image” configuration by delegating to `template_host` over WinRM during provisioning.

- Configure it with `windows_build_role` (default: `oatakan.windows_template_build`).
- Ensure WinRM is reachable using the credentials created by Autounattend:
  - `local_account_username`
  - `local_account_password`
  - `template_vm_ip_address` (static)

The delegated execution happens in [tasks/provision.yml](tasks/provision.yml).

## Common variables

See [defaults/main.yml](defaults/main.yml) for the full list. The most commonly set variables are:

| Variable | Default | Description |
| --- | ---: | --- |
| `role_action` | `provision` | `provision` or `deprovision` |
| `distro_name` | `win2019` | Selects unattended settings and naming conventions |
| `iso_file_name` | (example ISO) | Installer ISO name in datastore |
| `datastore_iso_folder` | `iso` | Folder on datastore that contains installer ISO |
| `template_vm_name` | `windows-2022-standard-core-auto` | VM/template name in vCenter |
| `template_force` | `false` | Overwrite existing template (convert to VM + delete) |
| `vcenter_datacenter` / `vcenter_cluster` | | vCenter placement |
| `vcenter_resource_pool` / `vcenter_folder` | | Optional placement controls |
| `vcenter_datastore` | | Datastore for the VM and Autounattend ISO |
| `template_vm_network_name` | `mgmt` | Port group name |
| `template_vm_ip_address` | `192.168.10.99` | Static IP used for WinRM wait + delegation |
| `template_vm_netmask` / `template_vm_gateway` | | Static network config |
| `template_vm_dns_servers` | `[8.8.4.4, 8.8.8.8]` | DNS servers |
| `iso_image_index` | `1` | Image index inside the Windows ISO |
| `iso_product_key` | `""` | Optional product key used by Autounattend |
| `template_vm_guest_id` | `windows9Server64Guest` | vSphere guest ID |
| `template_vm_efi` | `false` | BIOS vs EFI partitioning in Autounattend |
| `enable_lab_config` | `false` | Optional Windows 11 setup bypasses (TPM/SecureBoot/RAM) |
| `local_administrator_password` | `""` | Sets built-in Administrator password (optional) |
| `local_account_username` / `local_account_password` | `ansible` / `""` | Account used for WinRM delegation |
| `create_snapshot` | `yes` | Create snapshot before converting to template |
| `export_ovf` | `no` | Export OVF/OVA after converting to template |
| `windows_build_role` | `oatakan.windows_template_build` | Role delegated into the guest |

## Example playbook

Provision a Windows template and run `oatakan.windows_template_build` afterwards:

```yaml
---
- name: Build a vCenter Windows template from ISO
  hosts: localhost
  gather_facts: false
  connection: local
  vars:
    # vCenter placement
    vcenter_datacenter: cloud
    vcenter_cluster: mylab
    vcenter_resource_pool: pool1
    vcenter_folder: template
    vcenter_datastore: datastore1

    # Installer ISO already uploaded to datastore
    datastore_iso_folder: iso
    iso_file_name: "17763.253.190108-0006.rs5_release_svc_refresh_SERVER_EVAL_x64FRE_en-us.iso"
    iso_image_index: 1

    # VM/template identity + hardware
    distro_name: win2019
    template_vm_name: win2019-template-v1
    template_vm_guest_id: windows9Server64Guest
    template_vm_memory: 4096
    template_vm_cpu: 2
    template_vm_root_disk_size: 30
    template_vm_efi: false

    # Network (static IP is required for WinRM wait/delegation)
    template_vm_network_name: mgmt
    template_vm_ip_address: 192.168.10.99
    template_vm_netmask: 255.255.255.0
    template_vm_gateway: 192.168.10.254
    template_vm_domain: example.com
    template_vm_dns_servers: [8.8.4.4, 8.8.8.8]

    # Autounattend-created account for WinRM delegation
    local_account_username: ansible
    local_account_password: "CHANGE_ME"

    # Overwrite existing template if needed
    template_force: false

    # Optional outputs
    create_snapshot: true
    export_ovf: false

  roles:
    - role: oatakan.windows_vcenter_template
```

Deprovision/remove an existing template:

```yaml
---
- name: Remove a vCenter template
  hosts: localhost
  gather_facts: false
  connection: local
  roles:
    - role: oatakan.windows_vcenter_template
      vars:
        role_action: deprovision
        template_vm_name: win2019-template-v1
```

## Testing (local)

This repo’s CI runs syntax checks (see [.travis.yml](.travis.yml)):

```bash
ansible-playbook tests/test.yml -i tests/inventory --syntax-check
```

## Troubleshooting

- **Installer ISO not found**: ensure the datastore path `{{ datastore_iso_folder }}/{{ iso_file_name }}` exists (see [tasks/preflight_check.yml](tasks/preflight_check.yml)).
- **Template already exists**: set `template_force: true` (or run `role_action: deprovision` first).
- **WinRM delegation fails**: confirm `template_vm_ip_address` is reachable on `5986` and the unattended-created `local_account_username`/`local_account_password` are correct.
- **Windows 11 install checks fail**: set `enable_lab_config: true` to apply LabConfig bypass registry entries in `windowsPE` (see [templates/windows_server/Autounattend.xml.j2](templates/windows_server/Autounattend.xml.j2)).

## License

MIT

## Author

Orcun Atakan
