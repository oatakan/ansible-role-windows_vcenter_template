# windows_vcenter_template
This repo contains an Ansible role that builds a Windows VM template from an ISO file on VMware vcenter.
You can run this role as a part of CI/CD pipeline for building Windows templates on VMware vcenter from an ISO file.

> **_Note:_** This role is provided as an example only. Do not use this in production. You can fork/clone and add/remove steps for your environment based on your organization's security and operational requirements.

Requirements
------------

You need to have the following packages installed on your control machine:

- mkisofs

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
        providers:
          vcenter:
            cluster: mylab
            datacenter: cloud
            datastore: datastore2
            folder: template
            resource_pool: manageto
    
      roles:
        - oatakan.windows_vcenter_template

License
-------

MIT

Author Information
------------------

Orcun Atakan
