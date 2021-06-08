# windows_vcenter_template
This repo contains an Ansible role that builds a Windows VM template from an ISO file on VMware vcenter.
You can run this role as a part of CI/CD pipeline for building Windows templates on VMware vcenter from an ISO file.

> **_Note:_** This role is provided as an example only. Do not use this in production. You can fork/clone and add/remove steps for your environment based on your organization's security and operational requirements.

Requirements
------------

You need to have the following packages installed on your control machine:

- mkisofs

Before you can use this role, you need to make sure you have Windows install media iso file uploaded to a datastore on your vcenter environment.

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

Dependencies
------------

A list of roles that this role utilizes:

- oatakan.windows_template_build

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - name: create a vmware windows template
      hosts: all
      gather_facts: False
      connection: local
      become: no
      vars:
        template_force: yes
        export_ovf: no
        distro_name: win2019 # this needs to be one of the standard values see 'os_short_names' var
        template_vm_name: win2019_template
        template_vm_root_disk_size: 30
        template_vm_guest_id: windows9Server64Guest
        template_vm_memory: 4096
        template_vm_efi: false
        template_vm_network_name: mgmt
        template_vm_ip_address: 192.168.10.99  # static ip is required
        template_vm_netmask: 255.255.255.0
        template_vm_gateway: 192.168.10.254
        template_vm_domain: example.com
        template_vm_dns_servers:
          - 8.8.4.4
          - 8.8.8.8
        iso_file_name: '' # name of the iso file, make sure it's uploaded a datastore
        datastore_iso_folder: iso # folder name on datastore where iso file resides
        iso_image_index: '' # put index number here from the order inside the iso, for example 1 - standard, 2 - core etc
        iso_product_key: ''
        vm_upgrade_powershell: false # only needed for 2008 R2
        install_updates: false # it will take longer to build with the updates, set to true if you want the updates
        vcenter_datacenter: cloud
        vcenter_cluster: mylab
        vcenter_resource_pool: pool1
        vcenter_folder: template
        vcenter_datastore: datastore1
    
      roles:
        - oatakan.windows_vcenter_template

For disconnected environments, you can overwrite this variable to point to a local copy of a script to enable winrm:

winrm_enable_script_url: https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1

License
-------

MIT

Author Information
------------------

Orcun Atakan
