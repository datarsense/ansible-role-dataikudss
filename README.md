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
|dss_node_type | design | DSS node type. The only supported value in this release is `design`|
|dss_datadir | dss_data | Name of the DSS data directory|
|dss_network_port | 10000 | DSS network port |

### Optional variables for enabling Spark support
| Variable | Default value |
|----------|-------------|
|configure_spark| true |
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

### Optional variables for enabling containerized execution on kubernetes
| Variable | Default value |
|----------|-------------|
|configure_k8s| false |
|k8s_executionconfigs| [] |

The **k8s_executionconfigs** is an array which **can contain multiple containerized execution configurations** to match different business scenarios. Different kubernetes quotas can be allowed depending on user permissions, cuda ressources access can be limited to the data-scientist group, several base image can be offered to match business needs, ... 

A configuration example is provided below with **two kubernetes execution configs** :
```
 - name: "Deploy DSS with containerized execution support"
      ansible.builtin.include_role:
        name: "datarsense.dataikudss"
      vars:
        dss_version: "11.1.1"
        [...]
        configure_k8s: true
        k8s_executionconfigs:
          - name: test1
            type: KUBERNETES
            properties: []
            usableBy: ALLOWED
            allowedGroups:
              - administrators
            dockerNetwork: host
            dockerResources: []
            kubernetesNamespace: testnamespace
            kubernetesResources:
              memRequestMB: 2048
              memLimitMB: 2048
              cpuRequest: 2.0
              cpuLimit: 2.0
              customLimits: []
              customRequests: []
            hostPathVolumes: []
            isFinal: false,
            ensureNamespaceCompliance: false
            createNamespace: false
            baseImageType: EXEC
            baseImage: dss_containerer_exec_base:latest
            repositoryURL: docker.io
            prePushMode: NONE
            dockerTLSVerify: false
          - name: test2
            type: KUBERNETES
            properties: []
            usableBy: ALLOWED
            allowedGroups:
              - data-scientists
            dockerNetwork: host
            dockerResources: []
            kubernetesNamespace: testnamespace
            kubernetesResources:
              memRequestMB: 8192
              memLimitMB: 8192
              cpuRequest: 16.0
              cpuLimit: 16.0
              customLimits: []
              customRequests: []
            hostPathVolumes: []
            isFinal: false,
            ensureNamespaceCompliance: false
            createNamespace: false
            baseImageType: EXEC
            baseImage: dss_containerer_exec_cuda_base:latest
            repositoryURL: docker.io
            prePushMode: NONE
            dockerTLSVerify: false
```

Dependencies
------------
The following modules provided by dataiku are required for DSS config automation :
  - https://github.com/dataiku/dataiku-ansible-modules
  - https://github.com/dataiku/dataiku-api-client-python

Install the required modules with the following command :
```
ansible-galaxy install -r requirements.yml
```


Sample DSS deployment playbook
----------------

```
---
- name: Deploy
  hosts: all
  tasks:
    - name: "Include datarsense.dataikudss"
      ansible.builtin.include_role:
        name: "datarsense.dataikudss"
      vars:
        dss_base_repository_url: https://cdn.downloads.dataiku.com/public/studio
        dataiku_python_api_package: "git+https://github.com/dataiku/dataiku-api-client-python@release/5.1#egg=dataiku-api-client"
        dss_version: "10.0.7"
        dss_api_version: "5.1"
        dss_service_user: dataiku
        dss_service_user_shell: "/bin/bash"
        dss_service_user_home_basedir: /home
        dss_install_dir_location: /opt/dataiku
        dss_node_poll_fqdn: true # If true, use ansible_fqdn else use ansible_host
        dss_license_file: license.json
        dss_node_type: design
        dss_datadir: dss_data
        dss_network_port: 10000

        # Optional : add dss spark support
        configure_spark: true
        dss_hadoop_package: "dataiku-dss-hadoop-standalone-libs-generic-hadoop3-10.0.7.tar.gz"
        dss_spark_package: "dataiku-dss-spark-standalone-10.0.7-3.1.2-generic-hadoop3.tar.gz"

        # Optional : add LDAP login capability
        configure_ldap_settings: "false"
        ldap_url: "ldap://ldap.internal.example.com/dc=example,dc=com"
        ldap_binddn: "uid=readonly,ou=users,dc=example,dc=com"
        ldap_bindpassword: ""
        ldap_usetls: true
        ldap_autoimportusers: true
        ldap_userfilter: "(&(objectClass=posixAccount)(uid={USERNAME}))"
        ldap_defaultuserprofile: "READER"
        ldap_displaynameattribute: "cn"
        ldap_emailattribute: "mail"
        ldap_enablegroups: true
        ldap_groupfilter: "(&(objectClass=posixGroup)(memberUid={USERDN}))"
        ldap_groupnameattribute: "cn"
        ldap_groupprofiles: []
        ldap_authorizedgroups: "dss-users"

        configure_k8s: true
        k8s_executionconfigs:
          - name: test1
            type: KUBERNETES
            properties: []
            usableBy: ALLOWED
            allowedGroups:
              - administrators
            dockerNetwork: host
            dockerResources: []
            kubernetesNamespace: testnamespace
            kubernetesResources:
              memRequestMB: 2048
              memLimitMB: 2048
              cpuRequest: 2.0
              cpuLimit: 2.0
              customLimits: []
              customRequests: []
            hostPathVolumes: []
            isFinal: false,
            ensureNamespaceCompliance: false
            createNamespace: false
            baseImageType: EXEC
            baseImage: dss_containerer_exec_base:latest
            repositoryURL: docker.io
            prePushMode: NONE
            dockerTLSVerify: false
          - name: test2
            type: KUBERNETES
            properties: []
            usableBy: ALLOWED
            allowedGroups:
              - data-scientists
            dockerNetwork: host
            dockerResources: []
            kubernetesNamespace: testnamespace
            kubernetesResources:
              memRequestMB: 8192
              memLimitMB: 8192
              cpuRequest: 16.0
              cpuLimit: 16.0
              customLimits: []
              customRequests: []
            hostPathVolumes: []
            isFinal: false,
            ensureNamespaceCompliance: false
            createNamespace: false
            baseImageType: EXEC
            baseImage: dss_containerer_exec_cuda_base:latest
            repositoryURL: docker.io
            prePushMode: NONE
            dockerTLSVerify: false
```

License
-------

BSD

Author Information
------------------

