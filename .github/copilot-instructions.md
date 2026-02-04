# Copilot instructions: `windows_vcenter_template`

## What this repo is
- An Ansible role that builds a Windows VM on VMware vCenter from an ISO, customizes it over WinRM, then converts it to a vCenter *template* (optionally exports an OVA).
- The role is designed to be CI/CD friendly and supports AWX/Tower-style uniqueness via `awx_job_id`.

## Execution flow (read these first)
- Entry point: `tasks/main.yml` → always runs `tasks/preflight_check.yml`, then runs `tasks/provision.yml` when `role_action: provision`.
- Provision pipeline: `tasks/provision.yml`
  - `make_iso.yml` (renders `Autounattend.xml` + downloads WinRM enable script + runs `mkisofs`)
  - `provision_vm.yml` (`vmware_guest` async deploy)
  - `add_host` as `template_host` and `include_role` of `windows_build_role` delegated to that WinRM host
  - `stop_vm.yml` (powers off and removes CD-ROMs), optional `create_snapshot.yml`, `convert_to_template.yml`, optional `export_ovf.yml`
  - `rescue`: converts back to VM and (optionally) deletes it
  - `always`: removes the autounattend ISO from datastore and deletes the temp dir

## Key variables and conventions
- User-overridable inputs are in `defaults/main.yml` (e.g., `template_vm_*`, `iso_file_name`, `distro_name`, `role_action`, `template_force`, `export_ovf`).
- Computed/structured vars live in `vars/main.yml`:
  - `temp_directory` and datastore `iso_file` include `awx_job_id | default('')` for uniqueness.
  - `template` dict is the source of truth for VM spec (disk/memory/cpu/networks/cdrom).
  - `remove_template` becomes true when `role_action: deprovision` or `template_force: true`.
- vCenter credentials are **not** defined in defaults/vars; tasks expect either:
  - environment variables: `VMWARE_HOST`, `VMWARE_USER`, `VMWARE_PASSWORD`, or
  - role vars: `providers.vcenter.hostname`, `providers.vcenter.username`, `providers.vcenter.password`.
- Networking is treated as static: the role waits for WinRM (`5986`) on `template_vm_ip_address`.

## External integration points
- VMware modules used throughout `tasks/*.yml`: `vmware_guest`, `vmware_vm_info`, `vmware_guest_info`, `vmware_export_ovf`, `vmware_guest_snapshot`, `vsphere_copy`, `vsphere_file`, `community.vmware.vmware_about_info`.
- Guest customization is delegated to an external role (`windows_build_role`, default `oatakan.windows_template_build`) from `tasks/provision.yml`.
- ISO generation requires `mkisofs` (see `README.md` and `tasks/make_iso.yml`).

## Repo-specific patterns to follow when changing tasks
- Preserve the `block`/`rescue`/`always` structure in `tasks/provision.yml` so failures don’t leave orphaned templates/VMs or datastore artifacts.
- Long-running vCenter operations are done via `async: 7200` + `async_status` polling; follow that pattern for new `vmware_*` tasks.
- CD-ROM handling matters: OS ISO is `template.cdrom[0]`, autounattend ISO is `autounattend_cdrom` appended at deploy time, and `tasks/stop_vm.yml` later removes both.
- `templates/windows_server/Autounattend.xml.j2` has BIOS vs EFI partitioning logic keyed off `template_vm_efi|bool` and optional Windows 11 bypass via `enable_lab_config`.

## Local/CI workflows
- Syntax check (mirrors `.travis.yml`):
  - `ansible-playbook tests/test.yml -i tests/inventory --syntax-check`
- Note: `export_ovf: true` keeps `temp_directory` (it becomes the `export_dir`); otherwise `tasks/provision.yml` deletes it in `always`.
- Example usage and required vars are in `README.md`.
