Ansible role DSS
=========

An Ansible role automating Dataiku DSS deployment.

Requirements
------------
**Python 3.8 or newer** is required to make this role work.

The following modules provided by dataiku are required for DSS config automation :
  - https://github.com/dataiku/dataiku-ansible-modules
  - https://github.com/dataiku/dataiku-api-client-python

Install the requirements with the following command :
```
ansible-galaxy install -r requirements.yml
```

The account used for running playbook must have **sudo** privileges on the remote environment and must be allowed to become :
  - **root** for pre-install stage (installing packages, creating the dss servie user)
  - **dss service user** for DSS install as DSS is not run as root.

If ansible-playbook in executed with a non-root user on the remote environment, the following configuration is added by this role in `/etc/sudoers.d` to allow this non-root user to act on behalt of the dataiku service account.

```
non-root-user ALL = (dataiku) NOPASSWD: ALL
```

Role Variables
--------------

| Variable | Default value |
|----------|-------------|
|dss_base_repository_url | https://cdn.downloads.dataiku.com/public/studio |
|dss_version| "10.0.7" |
|dss_api_version| "5.1" |
|dss_service_user| dataiku |
|dss_service_user_shell| "/bin/bash" |
|dss_service_user_home_basedir| /home |
|dss_install_dir_location| /opt/dataiku |
|datadir| dss_data |
|dss_node_poll_fqdn| true # If true, use ansible_fqdn else |use ansible_host |
|dss_license_file| license.json |

Dependencies
------------


Example Playbook
----------------

```
---
- name: Converge
  hosts: all
  tasks:
    - name: "Include datarsense.dataikudss"
      include_role:
        name: {{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') | basename }}
      vars:
        dss_version: "10.0.7"
```

License
-------

BSD

Author Information
------------------


