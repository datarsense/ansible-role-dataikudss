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

### Optional variables for enabling containerized execution on kubernetes
| Variable | Default value |
|----------|-------------|
|configure_k8s| false |
|k8s_executionconfigs| [] |

The **k8s_executionconfigs** is an array which **can contain multiple containerized execution configurations** to match different business scenarios. Different kubernetes quotas can be allowed depending on user permissions, cuda ressources access can be limited to the data-scientist group, several base image can be offered to match business needs, ... 

A configuration example is provided below with **two kubernetes execution configs**. Please make sure to replace sample **repositoryURL**, **baseImage**, and set a valid kubernetes namespace when using this sample.
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

### Optional variables for enabling Spark support
| Variable | Default value |
|----------|-------------|
|configure_spark| true |
|dss_hadoop_package|"dataiku-dss-hadoop-standalone-libs-generic-hadoop3-10.0.7.tar.gz"|
|dss_spark_package| "dataiku-dss-spark-standalone-10.0.7-3.1.2-generic-hadoop3.tar.gz"|
|spark_executionconfigs| [] |

The **spark_executionconfigs** is an array which **can contain multiple spark execution configurations** to match different business scenario when managing a spark cluster on kubernetes from DSS. 

This variable is used to configure spark settings for **spark on kubernetes** when both **configure_spark** and **configure_k8s** are **true***. Other spark deployment scenarios are not supported in this ansible role.

A configuration example is provided below. Please make sure to replace sample **repositoryURL**, **baseImage**, and set a valid kubernetes namespace when using this sample. The **authenticationMode** can be either `BUILTIN` or `DYNAMIC_SERVICE_ACCOUNT` : set this variable according with your user isolation needs. Read more on [
Workload isolation on Kubernetes](https://doc.dataiku.com/dss/latest/user-isolation/capabilities/kubernetes.html) dataiku documentation.
```
spark_executionconfigs:
  - name": SparkOnKubernetes
    kubernetesSettings:
      managedKubernetes: true
      managedNamespace: testnamespace
      authenticationMode: BUILTIN
      ensureNamespaceCompliance: false
      createNamespace: false
      baseImageType: SPARK
      baseImage: dss_spark_base:latest
      repositoryURL: docker.io
      prePushMode: NONE
      dockerTLSVerify: false
```


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


### Optional variables for enabling User Isolation Framework (UIF)
| Variable | Default value |
|----------|-------------|
|configure_uif| false |
|uif_users| {} |
|uif_userrules| [] |
|uif_grouprules| [] |

The **uif_users** is a dict which **contains local unix users and groups for which UIF impersonation is allowed**. You must fill it with the **list of users which have to be created** by ansible and the **UNIX groups** to which they belong. Only users belonging to these groups will be allowed to use the local code impersonation mechanism (Python, R, visual ML, spark). More on https://knowledge.dataiku.com/latest/kb/governance/Which-activities-in-DSS-require-that-a-user-be-added-to-the.html

The **uif_userrules** and **uif_grouprules** are arrays which can contain **multiple uif user mapping** configuring mapping between a DSS user and an unix or hadoop user effectively running the DSS job when user isolation is enabled. Read more on https://doc.dataiku.com/dss/latest/user-isolation/initial-setup.html

UIF rules types can be :
 - **IDENTITY** : Make the rule map each DSS user to a UNIX user of the same name.
 - **SINGLE_MAPPING** :  Make the rule map a given DSS user/group configured in the **dssUser** or**dssGroup** variable to a given UNIX user defined in the **targetUnix** (or **targetHadoop**) variable.
 - **REGEXP_RULE** : Make the rule map a DSS users matching a given regular expression configured in the **ruleFrom** variable to a given UNIX user defined in the **targetUnix** (or**targetHadoop**) variable.

```
configure_uif: true
uif_users:
  userA:
    group: groupA
  userB:
    group: groupB
uif_userrules:
  - name: rule1
    scope: GLOBAL
    type: SINGLE_MAPPING
    dssUser: userA
    targetUnix: unix-userA
    targetHadoop: hadoop-userA
  - name: rule2
    scope: GLOBAL
    type: REGEXP_RULE
    ruleFrom: .*
    targetUnix: unix-userB
    targetHadoop: hadoop-userB
uif_grouprules:
  - name: ruleGroupA
    scope: GLOBAL
    type: SINGLE_MAPPING
    dssGroup: groupA
    targetUnix: unix-userA
    targetHadoop: hadoop-userA
  - name: ruleGroupB
    scope: GLOBAL
    type: REGEXP_RULE
    ruleFrom: .*
    targetUnix: unix-userB
    targetHadoop: hadoop-userB
```

Dependencies
------------
**Python3**, **ansible >= 5.8** and **jmespath** are required by this role. A requirements.txt file including those dependencies is provided with this role.

The following modules provided by Dataiku are required for DSS config automation :
  - https://github.com/dataiku/dataiku-ansible-modules
  - https://github.com/dataiku/dataiku-api-client-python

Create a `requirements.yml` file in your playbook. The `requirements.yml` has to include the following content to install the role and it's dependencies :
```
---
- src: git+https://github.com/dataiku/dataiku-api-client-python
  name: dataiku-api-client-python
  version: release/8.0

- src: git+https://github.com/dataiku/dataiku-ansible-modules
  name: dataiku.dataiku-ansible-modules
  version: master

- src: git+https://github.com/datarsense/ansible-role-dataikudss.git
  name: datarsense.dataikudss
  version: main
  ```

Then, install the role and it's dependencies with the following command :
```
ansible-galaxy install -r requirements.yml
```


Sample DSS deployment playbook
----------------

```
---

- hosts: all
  gather_facts: true
  tasks:
    - name: "Deploy DSS with containerized execution support"
      ansible.builtin.include_role:
        name: "datarsense.dataikudss"
      vars:
        dss_version: "11.1.1"
        dss_hadoop_package: "dataiku-dss-hadoop-standalone-libs-generic-hadoop3-11.1.1.tar.gz"
        dss_spark_package: "dataiku-dss-spark-standalone-11.1.1-3.2.1-generic-hadoop3.tar.gz"

        configure_ldap_settings: "true"
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
        
        configure_uif: true
        uif_users:
          userA:
            group: groupA
          userB:
            group: groupB
        uif_userrules:
          - name: rule1
            scope: GLOBAL
            type: SINGLE_MAPPING
            dssUser: userA
            targetUnix: unix-userA
            targetHadoop: hadoop-userA
          - name: rule2
            scope: GLOBAL
            type: REGEXP_RULE
            ruleFrom: .*
            targetUnix: unix-userB
            targetHadoop: hadoop-userB
        uif_grouprules:
          - name: ruleGroupA
            scope: GLOBAL
            type: SINGLE_MAPPING
            dssGroup: groupA
            targetUnix: unix-userA
            targetHadoop: hadoop-userA
          - name: ruleGroupB
            scope: GLOBAL
            type: REGEXP_RULE
            ruleFrom: .*
            targetUnix: unix-userB
            targetHadoop: hadoop-userB

        configure_spark: true
        spark_executionconfigs:
          - name": SparkOnKubernetes
            kubernetesSettings:
              managedKubernetes: true
              managedNamespace: testnamespace
              authenticationMode: BUILTIN
              ensureNamespaceCompliance: false
              createNamespace: false
              baseImageType: SPARK
              baseImage: dss_spark_base:latest
              repositoryURL: docker.io
              prePushMode: NONE
              dockerTLSVerify: false
        
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

