Ansible role DSS
=========

An Ansible role automating Dataiku DSS deployment.

Requirements
------------
The role is compatible with **Debian 10**, **Centos 7**, and **Centos 8**. Debian 11 is not supported as it is not a DSS 10.x supported OS. 

**Ansible 5.8 or newer** is required on the host running the ansible playbook. The account used for running the playbook must have **sudo** privileges on the remote environment and must be allowed to become :
  - **root** for pre-install stage (installing packages, creating the dss servie user)
  - **dss service user** for DSS install as DSS is not run as root.

If ansible-playbook in executed with a non-root user on the remote environment, the following configuration is added by this role in `/etc/sudoers.d` to allow this non-root user to act on behalt of the dataiku service account.

```
non-root-user ALL = (dataiku) NOPASSWD: ALL
```

Role Variables
--------------

| Variable | Default value | Usage |
|----------|---------------|-------|
|dss_base_repository_url | https://cdn.downloads.dataiku.com/public/studio | Base URL of the Dataiku CDN. The base URL is variabilized to make the role compatible with offline deployment |
|dataiku_python_api_package | "git+https://github.com/dataiku/dataiku-api-client-python@release/5.1#egg=dataiku-api-client" | Source repository of the **dataiku-api-client-python** module|
|dss_version| "10.0.7" | The DSS version which has to be deployed |
|dss_api_version| "5.1" | Version of the DSS API |
|dss_service_user| dataiku | Name of the DSS service user which will be created by this playbook|
|dss_service_user_shell| "/bin/bash" | Shell of the `dss_service_user`. keep `/bin/bash`|
|dss_service_user_home_basedir| /home | Home directory of the DSS instance. In some rare deployment scenarios, it can be different than `/home`|
|dss_install_dir_location| /opt/dataiku | Directory in which Dataiku binaries are downloaded and installed |
|dss_node_poll_fqdn| true | If true, use ansible_fqdn else use ansible_host |
|dss_license_file| license.json | The DSS license file which has to be deployed on the DSS host. |
|type | design | DSS node type. The only supported value in this release is `design`|
|datadir | dss_data | Name of the DSS data directory|
|port | 10000 | DSS network port |

### Optional variables for enabling Spark support
| Variable | Default value |
|----------|-------------|
|configure_spark| "true" |
|dss_hadoop_package|"dataiku-dss-hadoop-standalone-libs-generic-hadoop3-10.0.7.tar.gz"|
|dss_spark_package| "dataiku-dss-spark-standalone-10.0.7-3.1.2-generic-hadoop3.tar.gz"|

### Optional variables for enabling LDAP authentication
| Variable | Default value |
|----------|-------------|
|configure_ldap_settings:| "false" |
|ldap_url| "ldap://ldap.internal.example.com/dc=example,dc=com"|
|ldap_binddn| "uid=readonly,ou=users,dc=example,dc=com"
|ldap_bindpassword: "" |
|ldap_usetls| true |
|ldap_autoimportusers| true |
|ldap_userfilter| "(&(objectClass=posixAccount)(uid={USERNAME}))" |
|ldap_defaultuserprofile|"READER"|
|ldap_displaynameattribute| "cn"|
|ldap_emailattribute| "mail"|
|ldap_enablegroups| true |
|ldap_groupfilter| "(&(objectClass=posixGroup)(memberUid={USERDN}))"|
|ldap_groupnameattribute| "cn" |
|ldap_groupprofiles| [] |
|ldap_authorizedgroups| "dss-users" |

Dependencies
------------
The following modules provided by dataiku are required for DSS config automation :
  - https://github.com/dataiku/dataiku-ansible-modules
  - https://github.com/dataiku/dataiku-api-client-python

Install the required modules with the following command :
```
ansible-galaxy install -r requirements.yml
```


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

