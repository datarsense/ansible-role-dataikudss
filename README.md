Ansible role DSS
=========

An Ansible role automating Dataiku DSS deployment.

Requirements
------------

The following modules provided by dataiku are required for DSS config automation :
  - https://github.com/dataiku/dataiku-ansible-modules
  - https://github.com/dataiku/dataiku-api-client-python

Install the requirements with the following command :
```
ansible-galaxy install -r requirements.yml
```

Role Variables
--------------

| Variable | Default value |
|----------|-------------|
|dss_base_repository_url | https://cdn.downloads.dataiku.com/public/studio |
|dss_version| "10.0.7" |
|dss_api_version| "5.1" |
|dss_service_user| dataiku |
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
        dataiku_user_home: /home/dataiku
```

License
-------

BSD

Author Information
------------------


