Ansible role DSS
=========

An Ansible role automating Dataiku DSS deployment.

Requirements
------------
The role is compatible with **Debian 10**, **Centos 7**, and **Centos 8**. Debian 11 is not supported as it is not a DSS 11.x supported OS. 

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

### Optional variables for deploying JDBC drivers
DSS requires a third party JDBC connector JAR library provided by Oracle to be able to connect to MySQL databases.

The following ansible variables enable MySQL support in DSS :

| Variable | Sample value | Usage |
|----------|-------------|--------|
|configure_mysql| false | Triggers if MySQL JDBC driver has to be deployed or not. Default is `false` | 
|mysql_jdbc_connector_url| https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.31/mysql-connector-j-8.0.31.jar | URL of the MySQL JDBC connector JAR library | 


### Optional variables for tuning memory settings
As decribed in https://doc.dataiku.com/dss/latest/operations/memory.html, memory allocation of DSS components can be tuned :
 - The **backend** is a Java process that has a fixed memory allocation set by the `dss_backend_xmx` parameter. Backend memory requirement scales with the number of users, the number of projects, datasets, recipes, … For large production instances, allocating 12 to 20 GB of memory for the backend is recommended by Dataiku.
 - Each job in DSS runs in a separate process called a **JEK**. If you have 10 jobs running at a given time, there will be 10 running JEKs. The **default Xmx of the JEK is 2g**. This is enough for a large majority of jobs. However, some jobs with large number of partitions or large number of files to process may require more. This is configured by the `dss_jek_xmx` ini parameter.
- From time to time, the DSS backend will “delegate” part of its work to worker processes called the **FEKs**. This is done mostly for work that may consume huge amounts of memory. If a memory overrun happens, the FEK gets killed but the backend is unaffected. **The default Xmx of each FEK is 2g. This is enough for a large majority of tasks.** There may be some rare cases where you’ll need to allocate more memorry (**generally at the direction of Dataiku Support**). This is configured by the `dss_fek_xmx` ini parameter.

| Variable | Sample value |
|----------|-------------|
|dss_backend_xmx| 8g |
|dss_jek_xmx| 2g |
|dss_fek_xmx| 2g |

Default values are applied by DSS installer for memory parameter not configured as a variable in the ansible playbook using this role.

### Optional variables for enabling containerized execution on kubernetes
| Variable | Default value |
|----------|-------------|
|configure_k8s| false |
|k8s_executionconfigs| [] |
|download_dss_docker_images| false |
|download_dss_docker_images_url_tmp_dest | |
|download_dss_docker_images_url| `{{ dss_base_repository_url }}/{{ dss_version }}/container-images/dataiku-dss-ALL-base_dss-{{ dss_version }}-r-py3.6.tar.gz` |


The **k8s_executionconfigs** is an array which **can contain multiple containerized execution configurations** to match different business scenarios. Different kubernetes quotas can be allowed depending on user permissions, cuda ressources access can be limited to the data-scientist group, several base image can be offered to match business needs, ... 

DSS docker bases images can be automatically downloaded as an archive from a web URL by configuring `download_dss_docker_images: true`. Use `download_dss_docker_images_url_tmp_dest: /local/tmp` to configure a custom ansible tmp directory if the  `/tmp`partition of the server is too small to download the 4.5G docker images archive. The `download_dss_docker_images_url` download URL is configured to use the Dataiku public CDN by default, but can be changed if needed. **DSS version must be set in the docker archive file name to make this role able to check consistency between DSS version and DSS docker images version**

A configuration example is provided below with **two kubernetes execution configs**. Please make sure to replace sample **repositoryURL**, **baseImage**, and set a valid kubernetes namespace when using this sample.
```
 - name: "Deploy DSS with containerized execution support"
      ansible.builtin.include_role:
        name: "datarsense.dataikudss"
      vars:
        dss_version: "11.1.1"
        download_dss_docker_images: true
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
|spark_executionconfigs| _see below_ |

The **spark_executionconfigs** is an array which **can contain multiple spark execution configurations** to match different business scenario, including Spark on kubernetes. By default, this variable mirrors the default spark configuration of a DSS instance :

```
spark_executionconfigs:
  - name: default
    description: |- 
      This default configuration sets a few parameters that are suitable for a wide range of use cases.
      Importantly in order to work in all circumstances it does not set the spark master configuration.
      It will thus use the master defined by your default Spark configuration.
      This may lead Spark jobs to execute locally without using your cluster. You may need for example to add spark.master\u003dyarn-client
    conf:
      - key: spark.executor.memory
        value: 2400m
        isFinal: false
        secret: false
      - key: spark.sql.shuffle.partitions
        value: 40
        isFinal: false
        secret: false
      - key: spark.yarn.executor.memoryOverhead
        value: 600
        isFinal: false
        secret: false
      - key: spark.port.maxRetries
        value: 200
        isFinal: false
        secret: false
  - name: sample-yarn-config
    description: |- 
      This sample configuration shows a possible set of parameters for running DSS Spark jobs on YARN.
      These settings are suitable for a small cluster.
      You will need to tune spark.executor.instances spark.executor.cores and memory settings based on the size of your YARN cluster.
    conf :
      - key: spark.master
        value: yarn-client
        isFinal: false
        secret: false
      - key: spark.executor.memory
        value: 4g
        isFinal: false
        secret: false
      - key: spark.executor.instances
        value: 4
        isFinal: false
        secret: false
      - key: spark.executor.cores
        value: 2
        isFinal: false
        secret: false
      - key: spark.sql.shuffle.partitions
        value: 40
        isFinal: false
        secret: false
      - key: spark.yarn.executor.memoryOverhead
        value: 1200
        isFinal: false
        secret: false
      - key: spark.port.maxRetries
        value: 200
        isFinal: false
        secret: false      
  - name: sample-local-config
    description: |-
      This sample configuration shows a possible set of parameters for running DSS Spark jobs locally (non distributed).
      This can be useful for testing on small jobs as local Spark jobs start faster than YARN ones but is not suitable for production usage.
    conf:
      - key: spark.master
        value: local[4]
        isFinal: false
        secret: false  
      - key: spark.driver.memory
        value: 3g
        isFinal: false
        secret: false
      - key: spark.sql.shuffle.partitions
        value: 40
        isFinal: false
        secret: false
      - key: spark.port.maxRetries
        value: 200
        isFinal: false
        secret: false  

```

This variable can be used to configure spark settings for **spark on kubernetes** when both **configure_spark** and **configure_k8s** are **true***.

A configuration example is provided below. Please make sure to replace sample `repositoryURL: docker.io`, `baseImage: dss_spark_base:latest`, and to set a valid kubernetes namespace when using this sample. The **authenticationMode** can be either `BUILTIN` or `DYNAMIC_SERVICE_ACCOUNT` : set this variable according with your user isolation needs. Read more on [
Workload isolation on Kubernetes](https://doc.dataiku.com/dss/latest/user-isolation/capabilities/kubernetes.html) dataiku documentation.

Spark executors CPU limit is set by `spark.kubernetes.executor.limit.cores`. The CPU request is set by `spark.executor.cores`. The memory request and limit are set by summing the values of `spark.executor.memory` and `spark.executor.memoryOverhead`. 
```
spark_executionconfigs:
  - name: SparkOnKubernetes
    description: Execute Spark jobs in a Kuberntes cluster.
    conf:
      - key: spark.master
        value: k8s://https://IP_OF_YOUR_K8S_CLUSTER
        isFinal: false
        secret: false
      - key: spark.executor.memory
        value: 4g
        isFinal: false
        secret: false
      - key: spark.executor.memoryOverhead
        value: 8g
        isFinal: false
        secret: false
      - key: spark.executor.instances
        value: 4
        isFinal: false
        secret: false
      - key: spark.executor.cores
        value: 2
        isFinal: false
        secret: false
      - key: spark.kubernetes.executor.limit.cores
        value: 4
        isFinal: false
        secret: false
      - key: spark.sql.shuffle.partitions
        value: 40
        isFinal: false
        secret: false
      - key: spark.port.maxRetries
        value: 200
        isFinal: false
        secret: false
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
|configure_ldap_settings| false |

| Variable | Sample value |
|----------|-------------|
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

### Optional variables for enabling OpenID Connect (OIDC) SSO authentication
Configure `configure_oidc_sso: true` to enable OpenID Connect SSO.
| Variable | Default value |
|----------|-------------|
|configure_oidc_sso| false |

The following table shows an example of OIDC SSO configuration with Google IDP. Change the default values to match your IDP configuration.

Remapping rules are not supported by this role.

| Variable | Sample value |
|----------|-------------|
|oidc_clientid| test |
|oidc_clientsecret| test |
|oidc_scope| 'openid profile email' |
|oidc_issuer| https://accounts.google.com |
|oidc_authorizationendpoint| https://accounts.google.com/o/oauth2/v2/auth |
|oidc_tokenendpoint| https://oauth2.googleapis.com/token |
|oidc_jwksuri| https://www.googleapis.com/oauth2/v3/certs |
|oidc_claimkeyidentifier| email_verified |


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

        configure_ldap_settings: true
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
        
        configure_oidc_sso: true
        oidc_clientid: test
        oidc_clientsecret: test
        oidc_scope: 'openid profile email'
        oidc_issuer: https://accounts.google.com
        oidc_authorizationendpoint: https://accounts.google.com/o/oauth2/v2/auth
        oidc_tokenendpoint: https://oauth2.googleapis.com/token
        oidc_jwksuri: https://www.googleapis.com/oauth2/v3/certs
        oidc_claimkeyidentifier: email_verified

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
        download_dss_docker_images: true
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

