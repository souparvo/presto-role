# Ansible Role: presto-role

Install and configure a [PrestoDB](https://prestodb.io/) or [PrestoSQL](https://prestosql.io/) cluster. Can be used to configure master and worker nodes.

Instalation will download a tar file and will be installed on the target directory, which will have the configuration files available throught symlinks on:

- Configuration files - `/etc/presto`
- Logs - `/var/log/presto`
- Data files - `/var/presto`

Presto coordinator and workers will be running as `systemd` services as `presto.service`. Install presto on `/apps` folder

## Requirements

No specific requirements are needed to install this role.

## Role Variables

Base Presto configuration variables, by default install PrestoDB:

```yaml
---

# Version to install
presto_version: 0.243
# Defauls to prestodb
presto_tarbal_url: "https://repo1.maven.org/maven2/com/facebook/presto/presto-server/{{ presto_version }}/presto-server-{{ presto_version }}.tar.gz"

# prestosql
# presto_version: 347
# presto_tarbal_url: "https://repo1.maven.org/maven2/io/prestosql/presto-server/{{ presto_version }}/presto-server-{{ presto_version }}.tar.gz"
# REQUIRES JAVA 11

# User to run presto
presto_user: presto

# Presto installation directories
presto_data_dir: /apps/presto/data
presto_install_dir: /apps/presto/src

presto_config_dir: /etc/presto

# Hostname and port of the coordinator host
presto_coordinator_host: localhost
presto_coordinator_port: 8080

# Presto environment
presto_environment: production

# jvm.options heap configuration for coordiantor and workers
coordinator_jvm_heap: 16
worker_jvm_heap: 16

presto_java_home: /usr/lib/jvm/java-1.8.0-openjdk
```

### Configuration files

Configuration files are created dynamicaly with each key-value pair corresponding to a setting.

#### Node Properties

Each node will have a `node.properties` generated from the `node_properties` variable

```yaml
# Config files
node_properties:
    node.environment: "{{ presto_environment }}"
    node.id: "{{ inventory_host }}-node-id"
    node.data: "{{ presto_data_dir }}"
```

#### Config Properties

For coordinator and worker the `config.properties` will be generated.

```yaml
# Coordinator configs
coordinator_config:
    coordinator: true
    node-scheduler.include-coordinator: false
    http-server.http.port: "{{ presto_coordinator_port }}"
    query.max-memory: 20GB
    query.max-memory-per-node: 1GB
    query.max-total-memory-per-node: 2GB
    discovery-server.enabled: true
    discovery.uri: "http://{{ presto_coordinator_host }}:{{ presto_coordinator_port }}"

# Worker configs
worker_config:
    coordinator: false
    http-server.http.port: "{{ presto_coordinator_port }}"
    query.max-memory: 50GB
    query.max-memory-per-node: 1GB
    query.max-total-memory-per-node: 2GB
    discovery.uri: "http://{{ presto_coordinator_host }}:{{ presto_coordinator_port }}"
```

#### Catalogs

Each item of the `presto_catalog` variable list will create a catalog file, e.g. the `hive.properties` and `jmx.properties` will be created with the bellow config.

```yaml
# Catalogs
## Hive
presto_catalogs:
    - name: hive
      configs:
        connector.name: hive-hadoop2
        hive.metastore.uri: thrift://example.com:9083
        hive.config.resources: /etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml
        hive.hdfs.authentication.type: NONE
        hive.hdfs.impersonation.enabled: true
    - name: jmx
      config:
        connector.name: jmx
```

#### Secure Configurations

Add configurations for authentication, as per Presto's documentation.

```yaml
secure_properties:
  http-server.authentication.type: PASSWORD
  http-server.https.enabled: "true"
  http-server.https.port: 8443
  http-server.https.keystore.path: "{{ presto_config_dir }}/keystore.jks"
  http-server.https.keystore.key: "{{ presto_keystore_password }}"

password_authenticator:
  password-authenticator.name: ldap
  ldap.url: ldaps://ldap-server:636
  ldap.user-bind-pattern: <Refer below for usage>
```

#### Resource Groups

Resource groups as per Presto's documentation. Deploy resource groups configuration file, directly translated to a JSON file.

```yaml
# Enable to deploy a reource groups file
presto_configure_resource_groups: false

resource_groups:
  resource-groups.configuration-manager: file
  resource-groups.config-file: etc/resource_groups.json

# will generate resource_groups.json
resource_groups_config:
  rootGroups:
    - name: global
      softMemoryLimit: 100%
      hardConcurrencyLimit: 100
      maxQueued: 1000
      schedulingPolicy: weighted
      jmxExport: "true"
    - name: jdbc
      softMemoryLimit: 50%
      hardConcurrencyLimit: 100
      maxQueued: 1000
      schedulingPolicy: weighted
      jmxExport: "true"
    selectors:
      - source: odbc
        group: global
      - source: pyhive
        group: global
```

## Dependencies

No other roles are required to install this role.

## Example Playbook

### inventory.yml

Single node deployment with default configs:

```yaml
presto:
  hosts:
    host1:
```

Multi node deployment, 1 master 2 workers with default configs:

```yaml
presto:
  hosts:
    host1:
    host2:
    host3:
  children:
    workers:
      hosts:
        host2:
        host3:
      vars:
        presto_coordinator: false
```

### playbook.yml

```yaml
- name: Install presto cluster
  hosts: presto
  become: yes

  roles:
    - souparvo.presto-role
```

## License

BSD
