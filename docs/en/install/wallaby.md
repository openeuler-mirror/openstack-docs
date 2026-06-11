# OpenStack-Wallaby Deployment Guide

[TOC]

## Introduction to OpenStack

OpenStack is a community and also a project. It provides an operational platform or toolset for deploying clouds, enabling organizations to deliver scalable, flexible cloud computing.

As an open source cloud computing management platform, OpenStack is composed of several major components including nova, cinder, neutron, glance, keystone, horizon, and others that work together to accomplish specific tasks. OpenStack supports almost all types of cloud environments. The project's goal is to provide a cloud computing management platform that is simple to implement, massively scalable, rich in features, and uniformly standardized. OpenStack provides Infrastructure as a Service (IaaS) solutions through various complementary services, with each service providing APIs for integration.

The official repository of openEuler 24.03-LTS-SP4 version already supports OpenStack-Wallaby. Users can configure the yum repository and deploy OpenStack according to this document.

## Conventions

OpenStack supports multiple deployment forms. This document supports both `ALL in One` and `Distributed` deployment modes, using the following conventions:

`ALL in One` mode:

```text
Ignore all possible suffixes
```

`Distributed` mode:

```text
(CTL) suffix indicates the configuration or command applies only to the `Control Node`
(CPT) suffix indicates the configuration or command applies only to the `Compute Node`
(STG) suffix indicates the configuration or command applies only to the `Storage Node`
Without any suffix, the configuration or command applies to both `Control Node` and `Compute Node`
```

***Note***

The services related to the above conventions are as follows:

- Cinder
- Nova
- Neutron

## Environment Preparation

### Environment Configuration

1. Configure the 24.03 LTS SP4 official yum repository, and enable the EPOL software repository to support OpenStack

    ```shell
    yum update
    yum install openstack-release-wallaby
    yum clean all && yum makecache
    ```

    **Note**: If the EPOL repository is not enabled in your environment, you need to configure EPOL as well. Make sure EPOL is configured as shown below.

    ```shell
    vi /etc/yum.repos.d/openEuler.repo

    [EPOL]
    name=EPOL
    baseurl=http://repo.openeuler.org/openEuler-24.03-LTS-SP4/EPOL/main/$basearch/
    enabled=1
    gpgcheck=1
    gpgkey=http://repo.openeuler.org/openEuler-24.03-LTS-SP4/OS/$basearch/RPM-GPG-KEY-openEuler
    EOF
    ```

2. Modify the hostname and mappings

    Set the hostname for each node

    ```shell
    hostnamectl set-hostname controller                                                            (CTL)
    hostnamectl set-hostname compute                                                               (CPT)
    ```

    Assuming the controller node IP is `10.0.0.11` and the compute node IP is `10.0.0.12` (if it exists), add the following to `/etc/hosts`:

    ```shell
    10.0.0.11   controller
    10.0.0.12   compute
    ```

### Install SQL DataBase

1. Run the following command to install the software packages.

    ```shell
    yum install mariadb mariadb-server python3-PyMySQL
    ```

2. Run the following command to create and edit the `/etc/my.cnf.d/openstack.cnf` file.

    ```shell
    vim /etc/my.cnf.d/openstack.cnf

    [mysqld]
    bind-address = 10.0.0.11
    default-storage-engine = innodb
    innodb_file_per_table = on
    max_connections = 4096
    collation-server = utf8_general_ci
    character-set-server = utf8
    ```

    ***Note***

    **The `bind-address` is set to the management IP address of the control node.**

3. Start the DataBase service and enable it to start on boot:

    ```shell
    systemctl enable mariadb.service
    systemctl start mariadb.service
    ```

4. Configure the default password for DataBase (optional)

    ```shell
    mysql_secure_installation
    ```

    ***Note***

    **Follow the prompts as needed**

### Install RabbitMQ

1. Run the following command to install the software packages.

    ```shell
    yum install rabbitmq-server
    ```

2. Start the RabbitMQ service and enable it to start on boot.

    ```shell
    systemctl enable rabbitmq-server.service
    systemctl start rabbitmq-server.service
    ```

3. Add the OpenStack user.

    ```shell
    rabbitmqctl add_user openstack RABBIT_PASS
    ```

    ***Note***

    **Replace `RABBIT_PASS` with the password you want to set for the OpenStack user**

4. Set permissions for the openstack user to allow configuration, writing, and reading:

    ```shell
    rabbitmqctl set_permissions openstack ".*" ".*" ".*"
    ```

### Install Memcached

1. Run the following command to install the dependent software packages.

    ```shell
    yum install memcached python3-memcached
    ```

2. Edit the `/etc/sysconfig/memcached` file.

    ```shell
    vim /etc/sysconfig/memcached

    OPTIONS="-l 127.0.0.1,::1,controller"
    ```

3. Run the following command to start the Memcached service and enable it to start on boot.

    ```shell
    systemctl enable memcached.service
    systemctl start memcached.service
    ```

    ***Note***

    **After the service starts, you can verify that it is running normally and available using the command `memcached-tool controller stats`. You can replace `controller` with the management IP address of the control node.**

## Install OpenStack

### Keystone Installation

1. Create the keystone database and grant privileges.

    ``` sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE keystone;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
    IDENTIFIED BY 'KEYSTONE_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
    IDENTIFIED BY 'KEYSTONE_DBPASS';
    MariaDB [(none)]> exit
    ```

    ***Note***

    **Replace `KEYSTONE_DBPASS` with the password you want to set for the Keystone database**

2. Install software packages.

    ```shell
    yum install openstack-keystone httpd mod_wsgi
    ```

3. Configure keystone-related settings

    ```shell
    vim /etc/keystone/keystone.conf

    [database]
    connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

    [token]
    provider = fernet
    ```

    ***Explanation***

    In the [database] section, configure the database connection

    In the [token] section, configure the token provider

    ***Note:***

    **Replace `KEYSTONE_DBPASS` with the password for the Keystone database**

4. Synchronize the database.

    ```shell
    su -s /bin/sh -c "keystone-manage db_sync" keystone
    ```

5. Initialize the Fernet key repository.

    ```shell
    keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
    keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
    ```

6. Start the service.

    ```shell
    keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
    --bootstrap-admin-url http://controller:5000/v3/ \
    --bootstrap-internal-url http://controller:5000/v3/ \
    --bootstrap-public-url http://controller:5000/v3/ \
    --bootstrap-region-id RegionOne
    ```

    ***Note***

    **Replace `ADMIN_PASS` with the password you want to set for the admin user**

7. Configure the Apache HTTP server

    ```shell
    vim /etc/httpd/conf/httpd.conf

    ServerName controller
    ```

    ```shell
    ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
    ```

    ***Explanation***

    Configure the `ServerName` option to reference the control node

    ***Note***
    **If the `ServerName` option does not exist, you need to create it**

8. Start the Apache HTTP service.

    ```shell
    systemctl enable httpd.service
    systemctl start httpd.service
    ```

9. Create environment variable configuration.

    ```shell
    cat << EOF >> ~/.admin-openrc
    export OS_PROJECT_DOMAIN_NAME=Default
    export OS_USER_DOMAIN_NAME=Default
    export OS_PROJECT_NAME=admin
    export OS_USERNAME=admin
    export OS_PASSWORD=ADMIN_PASS
    export OS_AUTH_URL=http://controller:5000/v3
    export OS_IDENTITY_API_VERSION=3
    export OS_IMAGE_API_VERSION=2
    EOF
    ```

    ***Note***

    **Replace `ADMIN_PASS` with the password for the admin user**

10. Create domains, projects, users, and roles in order. You need to install python3-openstackclient first:

    ```shell
    yum install python3-openstackclient
    ```

    Source the environment variables

    ```shell
    source ~/.admin-openrc
    ```

    Create the `service` project, where the `default` domain was already created during keystone-manage bootstrap

    ```shell
    openstack domain create --description "An Example Domain" example
    ```

    ```shell
    openstack project create --domain default --description "Service Project" service
    ```

    Create a (non-admin) project `myproject`, user `myuser`, and role `myrole`. Add the role `myrole` to `myproject` and `myuser`

    ```shell
    openstack project create --domain default --description "Demo Project" myproject
    openstack user create --domain default --password-prompt myuser
    openstack role create myrole
    openstack role add --project myproject --user myuser myrole
    ```

11. Verification

    Unset the temporary environment variables OS_AUTH_URL and OS_PASSWORD:

    ```shell
    source ~/.admin-openrc
    unset OS_AUTH_URL OS_PASSWORD
    ```

    Request a token for the admin user:

    ```shell
    openstack --os-auth-url http://controller:5000/v3 \
    --os-project-domain-name Default --os-user-domain-name Default \
    --os-project-name admin --os-username admin token issue
    ```

    Request a token for the myuser user:

    ```shell
    openstack --os-auth-url http://controller:5000/v3 \
    --os-project-domain-name Default --os-user-domain-name Default \
    --os-project-name myproject --os-username myuser token issue
    ```

### Glance Installation

1. Create the database, service credentials, and API endpoints

    Create the database:

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE glance;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
    IDENTIFIED BY 'GLANCE_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
    IDENTIFIED BY 'GLANCE_DBPASS';
    MariaDB [(none)]> exit
    ```

    ***Note:***

    **Replace `GLANCE_DBPASS` with the password you want to set for the glance database**

    Create service credentials

    ```shell
    source ~/.admin-openrc

    openstack user create --domain default --password-prompt glance
    openstack role add --project service --user glance admin
    openstack service create --name glance --description "OpenStack Image" image
    ```

    Create the Image service API endpoints:

    ```shell
    openstack endpoint create --region RegionOne image public http://controller:9292
    openstack endpoint create --region RegionOne image internal http://controller:9292
    openstack endpoint create --region RegionOne image admin http://controller:9292
    ```

2. Install software packages

    ```shell
    yum install openstack-glance
    ```

3. Configure glance-related settings:

    ```shell
    vim /etc/glance/glance-api.conf

    [database]
    connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

    [keystone_authtoken]
    www_authenticate_uri  = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = glance
    password = GLANCE_PASS

    [paste_deploy]
    flavor = keystone

    [glance_store]
    stores = file,http
    default_store = file
    filesystem_store_datadir = /var/lib/glance/images/
    ```

    ***Explanation:***

    In the [database] section, configure the database connection

    In the [keystone_authtoken] [paste_deploy] sections, configure the authentication service connection

    In the [glance_store] section, configure the local filesystem storage and the location of image files

    ***Note***

    **Replace `GLANCE_DBPASS` with the password for the glance database**

    **Replace `GLANCE_PASS` with the password for the glance user**

4. Synchronize the database:

    ```shell
    su -s /bin/sh -c "glance-manage db_sync" glance
    ```

5. Start the service:

    ```shell
    systemctl enable openstack-glance-api.service
    systemctl start openstack-glance-api.service
    ```

6. Verification

    Download an image

    ```shell
    source ~/.admin-openrc
    
    wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
    ```

    ***Note***

    **If your environment uses the Kunpeng architecture, please download the aarch64 version of the image; the image cirros-0.5.2-aarch64-disk.img has been tested.**

    Upload an image to the Image service:

    ```shell
    openstack image create --disk-format qcow2 --container-format bare \
                           --file cirros-0.4.0-x86_64-disk.img --public cirros
    ```

    Confirm the image is uploaded and verify its properties:

    ```shell
    openstack image list
    ```

### Placement Installation

1. Create the database, service credentials, and API endpoints

    Create the database:

    Access the database as the root user, create the placement database and grant privileges.

    ```shell
    mysql -u root -p
    MariaDB [(none)]> CREATE DATABASE placement;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
    IDENTIFIED BY 'PLACEMENT_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
    IDENTIFIED BY 'PLACEMENT_DBPASS';
    MariaDB [(none)]> exit
    ```

    ***Note***

    **Replace `PLACEMENT_DBPASS` with the password you want to set for the placement database**

    ```shell
    source ~/.admin-openrc
    ```

    Run the following commands to create the placement service credentials, create the placement user, and add the 'admin' role to the 'placement' user.

    Create the Placement API service

    ```shell
    openstack user create --domain default --password-prompt placement
    openstack role add --project service --user placement admin
    openstack service create --name placement --description "Placement API" placement
    ```

    Create the placement service API endpoints:

    ```shell
    openstack endpoint create --region RegionOne placement public http://controller:8778
    openstack endpoint create --region RegionOne placement internal http://controller:8778
    openstack endpoint create --region RegionOne placement admin http://controller:8778
    ```

2. Installation and Configuration

    Install software packages:

    ```shell
    yum install openstack-placement-api
    ```

    Configure placement:

    Edit the /etc/placement/placement.conf file:

    In the [placement_database] section, configure the database connection

    In the [api] [keystone_authtoken] sections, configure the authentication service connection

    ```shell
    # vim /etc/placement/placement.conf
    [placement_database]
    # ...
    connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
    [api]
    # ...
    auth_strategy = keystone
    [keystone_authtoken]
    # ...
    auth_url = http://controller:5000/v3
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = placement
    password = PLACEMENT_PASS
    ```

    Among these, replace PLACEMENT_DBPASS with the password for the placement database, and replace PLACEMENT_PASS with the password for the placement user.

    Synchronize the database:

    ```shell
    su -s /bin/sh -c "placement-manage db sync" placement
    ```

    Start the httpd service:

    ```shell
    systemctl restart httpd
    ```

3. Verification

    Run the following command to perform a status check:

    ```shell
    source ~/.admin-openrc
    placement-status upgrade check
    ```

    Install osc-placement and list available resource classes and traits:

    ```shell
    yum install python3-osc-placement
    openstack --os-placement-api-version 1.2 resource class list --sort-column name
    openstack --os-placement-api-version 1.6 trait list --sort-column name
    ```

### Nova Installation

1. Create the database, service credentials, and API endpoints

    Create the database:

    ```sql
    mysql -u root -p                                                                               (CTL)

    MariaDB [(none)]> CREATE DATABASE nova_api;
    MariaDB [(none)]> CREATE DATABASE nova;
    MariaDB [(none)]> CREATE DATABASE nova_cell0;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
    IDENTIFIED BY 'NOVA_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
    IDENTIFIED BY 'NOVA_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
    IDENTIFIED BY 'NOVA_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
    IDENTIFIED BY 'NOVA_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
    IDENTIFIED BY 'NOVA_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
    IDENTIFIED BY 'NOVA_DBPASS';
    MariaDB [(none)]> exit
    ```

    ***Note***

    **Replace NOVA_DBPASS with the password for the nova database**

    ```shell
    source ~/.admin-openrc                                                                         (CTL)
    ```

    Create nova service credentials:

    ```shell
    openstack user create --domain default --password-prompt nova                                  (CTL)
    openstack role add --project service --user nova admin                                         (CTL)
    openstack service create --name nova --description "OpenStack Compute" compute                 (CTL)
    ```

    Create nova API endpoints:

    ```shell
    openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1        (CTL)
    openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1      (CTL)
    openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1         (CTL)
    ```

2. Install software packages

    ```shell
    yum install openstack-nova-api openstack-nova-conductor \                                      (CTL)
    openstack-nova-novncproxy openstack-nova-scheduler 

    yum install openstack-nova-compute                                                             (CPT)
    ```

    ***Note***

    **If using arm64 architecture, you also need to run the following command**

    ```shell
    yum install edk2-aarch64                                                                       (CPT)
    ```

3. Configure nova-related settings

    ```shell
    vim /etc/nova/nova.conf

    [DEFAULT]
    enabled_apis = osapi_compute,metadata
    transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
    my_ip = 10.0.0.1
    use_neutron = true
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
    compute_driver=libvirt.LibvirtDriver                                                           (CPT)
    instances_path = /var/lib/nova/instances/                                                      (CPT)
    lock_path = /var/lib/nova/tmp                                                                  (CPT)

    [api_database]
    connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api                              (CTL)

    [database]
    connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova                                  (CTL)

    [api]
    auth_strategy = keystone

    [keystone_authtoken]
    www_authenticate_uri = http://controller:5000/
    auth_url = http://controller:5000/
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = nova
    password = NOVA_PASS

    [vnc]
    enabled = true
    server_listen = $my_ip
    server_proxyclient_address = $my_ip
    novncproxy_base_url = http://controller:6080/vnc_auto.html                                     (CPT)

    [libvirt]
    virt_type = qemu                                                                               (CPT)
    cpu_mode = custom                                                                              (CPT)
    cpu_model = cortex-a72                                                                         (CPT)

    [glance]
    api_servers = http://controller:9292

    [oslo_concurrency]
    lock_path = /var/lib/nova/tmp                                                                  (CTL)

    [placement]
    region_name = RegionOne
    project_domain_name = Default
    project_name = service
    auth_type = password
    user_domain_name = Default
    auth_url = http://controller:5000/v3
    username = placement
    password = PLACEMENT_PASS

    [neutron]
    auth_url = http://controller:5000
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    region_name = RegionOne
    project_name = service
    username = neutron
    password = NEUTRON_PASS
    service_metadata_proxy = true                                                                  (CTL)
    metadata_proxy_shared_secret = METADATA_SECRET                                                 (CTL)
    ```

    ***Explanation***

    In the [default] section, enable the Compute and Metadata APIs, configure the RabbitMQ message queue connection, configure my_ip, and enable the Neutron networking service;

    In the [api_database] [database] sections, configure the database connections;

    In the [api] [keystone_authtoken] sections, configure the authentication service connection;

    In the [vnc] section, enable and configure the remote console access;

    In the [glance] section, configure the Image service API address;

    In the [oslo_concurrency] section, configure the lock path;

    In the [placement] section, configure the placement service connection.

    ***Note***

    **Replace `RABBIT_PASS` with the password for the openstack account in RabbitMQ;**

    **Configure `my_ip` with the management IP address of the control node;**

    **Replace `NOVA_DBPASS` with the password for the nova database;**

    **Replace `NOVA_PASS` with the password for the nova user;**

    **Replace `PLACEMENT_PASS` with the password for the placement user;**

    **Replace `NEUTRON_PASS` with the password for the neutron user;**

    **Replace `METADATA_SECRET` with an appropriate metadata proxy secret.**

    **Additional**

    Determine if hardware acceleration for virtual machines is supported (x86 architecture):

    ```shell
    egrep -c '(vmx|svm)' /proc/cpuinfo                                                             (CPT)
    ```

    If the return value is 0, hardware acceleration is not supported, and you need to configure libvirt to use QEMU instead of KVM:

    ```shell
    vim /etc/nova/nova.conf                                                                        (CPT)

    [libvirt]
    virt_type = qemu
    ```

    If the return value is 1 or greater, hardware acceleration is supported and no additional configuration is needed

    ***Note***

    **If using arm64 architecture, you also need to run the following commands**

    ```shell
    vim /etc/libvirt/qemu.conf

    nvram = ["/usr/share/AAVMF/AAVMF_CODE.fd: \
             /usr/share/AAVMF/AAVMF_VARS.fd", \
             "/usr/share/edk2/aarch64/QEMU_EFI-pflash.raw: \
             /usr/share/edk2/aarch64/vars-template-pflash.raw"]

    vim /etc/qemu/firmware/edk2-aarch64.json

    {
        "description": "UEFI firmware for ARM64 virtual machines",
        "interface-types": [
            "uefi"
        ],
        "mapping": {
            "device": "flash",
            "executable": {
                "filename": "/usr/share/edk2/aarch64/QEMU_EFI-pflash.raw",
                "format": "raw"
            },
            "nvram-template": {
                "filename": "/usr/share/edk2/aarch64/vars-template-pflash.raw",
                "format": "raw"
            }
        },
        "targets": [
            {
                "architecture": "aarch64",
                "machines": [
                    "virt-*"
                ]
            }
        ],
        "features": [

        ],
        "tags": [

        ]
    }

    (CPT)
    ```

4. Synchronize the database

    Synchronize the nova-api database:

    ```shell
    su -s /bin/sh -c "nova-manage api_db sync" nova                                                (CTL)
    ```

    Register the cell0 database:

    ```shell
    su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova                                          (CTL)
    ```

    Create cell1 cell:

    ```shell
    su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova                 (CTL)
    ```

    Synchronize the nova database:

    ```shell
    su -s /bin/sh -c "nova-manage db sync" nova                                                    (CTL)
    ```

    Verify that cell0 and cell1 are registered correctly:

    ```shell
    su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova                                         (CTL)
    ```

    Add compute nodes to the openstack cluster

    ```shell
    su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova                           (CPT)
    ```

5. Start services

    ```shell
    systemctl enable \                                                                             (CTL)
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service

    systemctl start \                                                                              (CTL)
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
    ```

    ```shell
    systemctl enable libvirtd.service openstack-nova-compute.service                               (CPT)
    systemctl start libvirtd.service openstack-nova-compute.service                                (CPT)
    ```

6. Verification

    ```shell
    source ~/.admin-openrc                                                                         (CTL)
    ```

    List the service components to verify that each process started and registered successfully:

    ```shell
    openstack compute service list                                                                 (CTL)
    ```

    List the API endpoints in the Identity service to verify connectivity to the Identity service:

    ```shell
    openstack catalog list                                                                         (CTL)
    ```

    List the images in the Image service to verify connectivity to the Image service:

    ```shell
    openstack image list                                                                           (CTL)
    ```

    Check if cells are functioning correctly and if other necessary conditions are met.

    ```shell
    nova-status upgrade check                                                                      (CTL)
    ```

### Neutron Installation

1. Create the database, service credentials, and API endpoints

    Create the database:

    ```sql
    mysql -u root -p                                                                               (CTL)

    MariaDB [(none)]> CREATE DATABASE neutron;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
    IDENTIFIED BY 'NEUTRON_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
    IDENTIFIED BY 'NEUTRON_DBPASS';
    MariaDB [(none)]> exit
    ```

    ***Note***

    **Replace `NEUTRON_DBPASS` with the password for the neutron database.**

    ```shell
    source ~/.admin-openrc                                                                         (CTL)
    ```

    Create neutron service credentials

    ```shell
    openstack user create --domain default --password-prompt neutron                               (CTL)
    openstack role add --project service --user neutron admin                                      (CTL)
    openstack service create --name neutron --description "OpenStack Networking" network           (CTL)
    ```

    Create Neutron service API endpoints:

    ```shell
    openstack endpoint create --region RegionOne network public http://controller:9696             (CTL)
    openstack endpoint create --region RegionOne network internal http://controller:9696           (CTL)
    openstack endpoint create --region RegionOne network admin http://controller:9696              (CTL)
    ```

2. Install software packages:

    ```shell
    yum install openstack-neutron openstack-neutron-linuxbridge ebtables ipset \                   (CTL)
    openstack-neutron-ml2
    ```

    ```shell
    yum install openstack-neutron-linuxbridge ebtables ipset                                       (CPT)
    ```

3. Configure neutron-related settings:

    Configure the main settings

    ```shell
    vim /etc/neutron/neutron.conf

    [database]
    connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron                         (CTL)

    [DEFAULT]
    core_plugin = ml2                                                                              (CTL)
    service_plugins = router                                                                       (CTL)
    allow_overlapping_ips = true                                                                   (CTL)
    transport_url = rabbit://openstack:RABBIT_PASS@controller
    auth_strategy = keystone
    notify_nova_on_port_status_changes = true                                                      (CTL)
    notify_nova_on_port_data_changes = true                                                        (CTL)
    api_workers = 3                                                                                (CTL)

    [keystone_authtoken]
    www_authenticate_uri = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = neutron
    password = NEUTRON_PASS

    [nova]
    auth_url = http://controller:5000                                                              (CTL)
    auth_type = password                                                                           (CTL)
    project_domain_name = Default                                                                  (CTL)
    user_domain_name = Default                                                                     (CTL)
    region_name = RegionOne                                                                        (CTL)
    project_name = service                                                                         (CTL)
    username = nova                                                                                (CTL)
    password = NOVA_PASS                                                                           (CTL)

    [oslo_concurrency]
    lock_path = /var/lib/neutron/tmp
    ```

    ***Explanation***

    In the [database] section, configure the database connection;

    In the [default] section, enable the ml2 and router plugins, allow overlapping IP addresses, and configure the RabbitMQ message queue connection;

    In the [default] [keystone] sections, configure the authentication service connection;

    In the [default] [nova] section, configure networking to notify Compute of network topology changes;

    In the [oslo_concurrency] section, configure the lock path.

    ***Note***

    **Replace `NEUTRON_DBPASS` with the password for the neutron database;**

    **Replace `RABBIT_PASS` with the password for the openstack account in RabbitMQ;**

    **Replace `NEUTRON_PASS` with the password for the neutron user;**

    **Replace `NOVA_PASS` with the password for the nova user.**

    Configure the ML2 plugin:

    ```shell
    vim /etc/neutron/plugins/ml2/ml2_conf.ini

    [ml2]
    type_drivers = flat,vlan,vxlan
    tenant_network_types = vxlan
    mechanism_drivers = linuxbridge,l2population
    extension_drivers = port_security

    [ml2_type_flat]
    flat_networks = provider

    [ml2_type_vxlan]
    vni_ranges = 1:1000

    [securitygroup]
    enable_ipset = true
    ```

    Create a symbolic link for /etc/neutron/plugin.ini

    ```shell
    ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
    ```

    **Note**

    **In the [ml2] section, enable flat, vlan, and vxlan networks, enable linuxbridge and l2population mechanisms, and enable the port security extension driver;**

    **In the [ml2_type_flat] section, configure the flat network as the provider virtual network;**

    **In the [ml2_type_vxlan] section, configure the VXLAN network identifier range;**

    **In the [securitygroup] section, configure to allow ipset.**

    **Supplementary**

    **The specific L2 configuration can be modified according to user requirements. This document uses provider network + linuxbridge**

    Configure the Linux bridge agent:

    ```shell
    vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini

    [linux_bridge]
    physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

    [vxlan]
    enable_vxlan = true
    local_ip = OVERLAY_INTERFACE_IP_ADDRESS
    l2_population = true

    [securitygroup]
    enable_security_group = true
    firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
    ```

    ***Explanation***

    In the [linux_bridge] section, map the provider virtual network to the physical network interface;

    In the [vxlan] section, enable VXLAN overlay networks, configure the physical network interface IP address for handling overlay networks, and enable layer-2 population;

    In the [securitygroup] section, enable security groups and configure the linux bridge iptables firewall driver.

    ***Note***

    **Replace `PROVIDER_INTERFACE_NAME` with the physical network interface;**

    **Replace `OVERLAY_INTERFACE_IP_ADDRESS` with the management IP address of the control node.**

    Configure the Layer-3 agent:

    ```shell
    vim /etc/neutron/l3_agent.ini                                                                  (CTL)

    [DEFAULT]
    interface_driver = linuxbridge
    ```

    ***Explanation***

    In the [default] section, configure the interface driver as linuxbridge

    Configure the DHCP agent:

    ```shell
    vim /etc/neutron/dhcp_agent.ini                                                                (CTL)

    [DEFAULT]
    interface_driver = linuxbridge
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    enable_isolated_metadata = true
    ```

    ***Explanation***

    In the [default] section, configure the linuxbridge interface driver, Dnsmasq DHCP driver, and enable isolated metadata.

    Configure the metadata agent:

    ```shell
    vim /etc/neutron/metadata_agent.ini                                                            (CTL)

    [DEFAULT]
    nova_metadata_host = controller
    metadata_proxy_shared_secret = METADATA_SECRET
    ```

    ***Explanation***

    In the [default] section, configure the metadata host and shared secret.

    ***Note***

    **Replace `METADATA_SECRET` with an appropriate metadata proxy secret.**

4. Configure nova-related settings

    ```shell
    vim /etc/nova/nova.conf

    [neutron]
    auth_url = http://controller:5000
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    region_name = RegionOne
    project_name = service
    username = neutron
    password = NEUTRON_PASS
    service_metadata_proxy = true                                                                  (CTL)
    metadata_proxy_shared_secret = METADATA_SECRET                                                 (CTL)
    ```

    ***Explanation***

    In the [neutron] section, configure access parameters, enable the metadata proxy, and configure the secret.

    ***Note***

    **Replace `NEUTRON_PASS` with the password for the neutron user;**

    **Replace `METADATA_SECRET` with an appropriate metadata proxy secret.**

5. Synchronize the database:

    ```shell
    su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
    --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
    ```

6. Restart the Compute API service:

    ```shell
    systemctl restart openstack-nova-api.service
    ```

7. Start networking services

    ```shell
    systemctl enable neutron-server.service neutron-linuxbridge-agent.service \                    (CTL)
    neutron-dhcp-agent.service neutron-metadata-agent.service 
    systemctl enable neutron-l3-agent.service
    systemctl restart openstack-nova-api.service neutron-server.service                            (CTL)
    neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
    neutron-metadata-agent.service neutron-l3-agent.service

    systemctl enable neutron-linuxbridge-agent.service                                             (CPT)
    systemctl restart neutron-linuxbridge-agent.service openstack-nova-compute.service             (CPT)
    ```

8. Verification

    Verify that the neutron agents started successfully:

    ```shell
    openstack network agent list
    ```

### Cinder Installation

1.Create the database, service credentials, and API endpoints

Create the database:

```sql
mysql -u root -p

MariaDB [(none)]> CREATE DATABASE cinder;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
IDENTIFIED BY 'CINDER_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
IDENTIFIED BY 'CINDER_DBPASS';
MariaDB [(none)]> exit
```

***Note***

**Replace `CINDER_DBPASS` with the password for the cinder database.**

```shell
source ~/.admin-openrc
```

Create cinder service credentials:

```shell
openstack user create --domain default --password-prompt cinder
openstack role add --project service --user cinder admin
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
```

Create Block Storage service API endpoints:

```shell
openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
```

2.Install software packages:

```shell
yum install openstack-cinder-api openstack-cinder-scheduler                                    (CTL)
```

```shell
yum install lvm2 device-mapper-persistent-data scsi-target-utils rpcbind nfs-utils \           (STG)
            openstack-cinder-volume openstack-cinder-backup
```

3.Prepare storage devices, the following is only an example:

```shell
pvcreate /dev/vdb
vgcreate cinder-volumes /dev/vdb

vim /etc/lvm/lvm.conf


devices {
...
filter = [ "a/vdb/", "r/.*/"]
```

***Explanation***

In the devices section, add filters to accept the /dev/vdb device and reject all other devices.

4.Prepare NFS

```shell
mkdir -p /root/cinder/backup

cat << EOF >> /etc/export
/root/cinder/backup 192.168.1.0/24(rw,sync,no_root_squash,no_all_squash)
EOF

```

5.Configure cinder-related settings:

```shell
vim /etc/cinder/cinder.conf

[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = 10.0.0.11
enabled_backends = lvm                                                                         (STG)
backup_driver=cinder.backup.drivers.nfs.NFSBackupDriver                                        (STG)
backup_share=HOST:PATH                                                                         (STG)

[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password = CINDER_PASS

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver                                      (STG)
volume_group = cinder-volumes                                                                  (STG)
iscsi_protocol = iscsi                                                                         (STG)
iscsi_helper = tgtadm                                                                          (STG)
```

***Explanation***

In the [database] section, configure the database connection;

In the [DEFAULT] section, configure the RabbitMQ message queue connection and my_ip;

In the [DEFAULT] [keystone_authtoken] sections, configure the authentication service connection;

In the [oslo_concurrency] section, configure the lock path.

***Note***

**Replace `CINDER_DBPASS` with the password for the cinder database;**

**Replace `RABBIT_PASS` with the password for the openstack account in RabbitMQ;**

**Configure `my_ip` with the management IP address of the control node;**

**Replace `CINDER_PASS` with the password for the cinder user;**

**Replace `HOST:PATH` with the NFS host IP and share path;**

6.Synchronize the database:

```shell
su -s /bin/sh -c "cinder-manage db sync" cinder                                                (CTL)
```

7.Configure nova:

```shell
vim /etc/nova/nova.conf                                                                        (CTL)

[cinder]
os_region_name = RegionOne
```

8.Restart the Compute API service

```shell
systemctl restart openstack-nova-api.service
```

9.Start cinder services

```shell
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service               (CTL)
systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service                (CTL)
```

```shell
systemctl enable rpcbind.service nfs-server.service tgtd.service iscsid.service \              (STG)
                    openstack-cinder-volume.service \
                    openstack-cinder-backup.service
systemctl start rpcbind.service nfs-server.service tgtd.service iscsid.service \               (STG)
                openstack-cinder-volume.service \
                openstack-cinder-backup.service
```

***Note***

When cinder uses `tgtadm` to attach volumes, you need to modify /etc/tgt/tgtd.conf with the following content to ensure that tgtd can discover cinder-volume's iscsi target.

```shell
include /var/lib/cinder/volumes/*
```

10.Verification

```shell
source ~/.admin-openrc
openstack volume service list
```

### Horizon Installation

1. Install software packages

    ```shell
    yum install openstack-dashboard
    ```

2. Modify the file

    Modify variables

    ```text
    vim /etc/openstack-dashboard/local_settings

    OPENSTACK_HOST = "controller"
    ALLOWED_HOSTS = ['*', ]

    SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

    CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
        }
    }

    OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
    OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
    OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
    OPENSTACK_KEYSTONE_DEFAULT_ROLE = "member"
    WEBROOT = '/dashboard'
    POLICY_FILES_PATH = "/etc/openstack-dashboard"

    OPENSTACK_API_VERSIONS = {
        "identity": 3,
        "image": 2,
        "volume": 3,
    }
    ```

3. Restart the httpd service

    ```shell
    systemctl restart httpd.service memcached.service
    ```

4. Verification
    Open a browser and navigate to `http://HOSTIP/dashboard` to log in to Horizon.

    ***Note***

    **Replace HOSTIP with the management plane IP address of the control node**

### Tempest Installation

Tempest is the integration testing service for OpenStack. If users need comprehensive automated testing of the installed OpenStack environment's functionality, it is recommended to use this component. Otherwise, installation is optional.

1. Install Tempest

    ```shell
    yum install openstack-tempest
    ```

2. Initialize the directory

    ```shell
    tempest init mytest
    ```

3. Modify the configuration file.

    ```shell
    cd mytest
    vi etc/tempest.conf
    ```

    The tempest.conf file needs to contain information about the current OpenStack environment. For specific content, please refer to the [official example](https://docs.openstack.org/tempest/latest/sampleconf.html)

4. Execute tests

    ```shell
    tempest run
    ```

5. Install Tempest extensions (optional)
   The various OpenStack services also provide their own tempest test packages. Users can install these packages to enrich tempest's test coverage. In Wallaby, we provide extension tests for Cinder, Glance, Keystone, Ironic, and Trove. Users can run the following commands to install and use them:

   ```shell
   yum install python3-cinder-tempest-plugin python3-glance-tempest-plugin python3-ironic-tempest-plugin python3-keystone-tempest-plugin python3-trove-tempest-plugin
   ```

### Ironic Installation

Ironic is the bare metal service for OpenStack. If users need to perform bare metal deployment, it is recommended to use this component. Otherwise, installation is optional.

1. Set up the database

   The bare metal service stores information in the database. Create an **ironic** database accessible by the **ironic** user. Replace **IRONIC_DBPASSWORD** with an appropriate password

   ```sql
   mysql -u root -p
   
   MariaDB [(none)]> CREATE DATABASE ironic CHARACTER SET utf8;
   MariaDB [(none)]> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'localhost' \
   IDENTIFIED BY 'IRONIC_DBPASSWORD';
   MariaDB [(none)]> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'%' \
   IDENTIFIED BY 'IRONIC_DBPASSWORD';
   ```

2. Create service user authentication

    1. Create the Bare Metal service user

        ```shell
        openstack user create --password IRONIC_PASSWORD \
                            --email ironic@example.com ironic
        openstack role add --project service --user ironic admin
        openstack service create --name ironic
                                --description "Ironic baremetal provisioning service" baremetal

        openstack service create --name ironic-inspector --description     "Ironic inspector baremetal provisioning service" baremetal-introspection
        openstack user create --password IRONIC_INSPECTOR_PASSWORD --email ironic_inspector@example.com ironic_inspector
        openstack role add --project service --user ironic-inspector admin
        ```

    2. Create Bare Metal service endpoints

        ```shell
        openstack endpoint create --region RegionOne baremetal admin http://$IRONIC_NODE:6385
        openstack endpoint create --region RegionOne baremetal public http://$IRONIC_NODE:6385
        openstack endpoint create --region RegionOne baremetal internal http://$IRONIC_NODE:6385
        openstack endpoint create --region RegionOne baremetal-introspection internal http://172.20.19.13:5050/v1
        openstack endpoint create --region RegionOne baremetal-introspection public http://172.20.19.13:5050/v1
        openstack endpoint create --region RegionOne baremetal-introspection admin http://172.20.19.13:5050/v1
        ```

3. Configure the ironic-api service

   Configuration file path: /etc/ironic/ironic.conf

    1. Configure the database location through the **connection** option as shown below. Replace **IRONIC_DBPASSWORD** with the password for the **ironic** user, and replace **DB_IP** with the IP address of the DB server:

        ```shell
        [database]
        
        # The SQLAlchemy connection string used to connect to the
        # database (string value)
        
        connection = mysql+pymysql://ironic:IRONIC_DBPASSWORD@DB_IP/ironic
        ```

    2. Configure the ironic-api service to use RabbitMQ message broker through the following options. Replace **RPC_*** with the detailed RabbitMQ address and credentials

        ```shell
        [DEFAULT]
        
        # A URL representing the messaging driver to use and its full
        # configuration. (string value)
        
        transport_url = rabbit://RPC_USER:RPC_PASSWORD@RPC_HOST:RPC_PORT/
        ```

        Users can also choose to use json-rpc instead of rabbitmq on their own

    3. Configure the ironic-api service to use credentials from the authentication service. Replace **PUBLIC_IDENTITY_IP** with the public IP of the authentication server, replace **PRIVATE_IDENTITY_IP** with the private IP of the authentication server, and replace **IRONIC_PASSWORD** with the password for the **ironic** user in the authentication service:

        ```shell
        [DEFAULT]
        
        # Authentication strategy used by ironic-api: one of
        # "keystone" or "noauth". "noauth" should not be used in a
        # production environment because all authentication will be
        # disabled. (string value)
        
        auth_strategy=keystone
        host = controller
        memcache_servers = controller:11211
        enabled_network_interfaces = flat,noop,neutron
        default_network_interface = noop
        transport_url = rabbit://openstack:RABBITPASSWD@controller:5672/
        enabled_hardware_types = ipmi
        enabled_boot_interfaces = pxe
        enabled_deploy_interfaces = direct
        default_deploy_interface = direct
        enabled_inspect_interfaces = inspector
        enabled_management_interfaces = ipmitool
        enabled_power_interfaces = ipmitool
        enabled_rescue_interfaces = no-rescue,agent
        isolinux_bin = /usr/share/syslinux/isolinux.bin
        logging_context_format_string = %(asctime)s.%(msecs)03d %(process)d %(levelname)s %(name)s [%(global_request_id)s %(request_id)s %(user_identity)s] %(instance)s%(message)s
        
        [keystone_authtoken]
        # Authentication type to load (string value)
        auth_type=password
        # Complete public Identity API endpoint (string value)
        www_authenticate_uri=http://PUBLIC_IDENTITY_IP:5000
        # Complete admin Identity API endpoint. (string value)
        auth_url=http://PRIVATE_IDENTITY_IP:5000
        # Service username. (string value)
        username=ironic
        # Service account password. (string value)
        password=IRONIC_PASSWORD
        # Service tenant name. (string value)
        project_name=service
        # Domain name containing project (string value)
        project_domain_name=Default
        # User's domain name (string value)
        user_domain_name=Default
        
        [agent]
        deploy_logs_collect = always
        deploy_logs_local_path = /var/log/ironic/deploy
        deploy_logs_storage_backend = local
        image_download_source = http
        stream_raw_images = false
        force_raw_images = false
        verify_ca = False
        
        [oslo_concurrency]
        
        [oslo_messaging_notifications]
        transport_url = rabbit://openstack:123456@172.20.19.25:5672/
        topics = notifications
        driver = messagingv2
        
        [oslo_messaging_rabbit]
        amqp_durable_queues = True
        rabbit_ha_queues = True
        
        [pxe]
        ipxe_enabled = false
        pxe_append_params = nofb nomodeset vga=normal coreos.autologin ipa-insecure=1
        image_cache_size = 204800
        tftp_root=/var/lib/tftpboot/cephfs/
        tftp_master_path=/var/lib/tftpboot/cephfs/master_images
        
        [dhcp]
        dhcp_provider = none
        ```

    4. Create the bare metal service database tables

        ```shell
        ironic-dbsync --config-file /etc/ironic/ironic.conf create_schema
        ```

    5. Restart the ironic-api service

        ```shell
        sudo systemctl restart openstack-ironic-api
        ```

4. Configure the ironic-conductor service

    1. Replace **HOST_IP** with the IP of the conductor host

        ```shell
        [DEFAULT]
        
        # IP address of this host. If unset, will determine the IP
        # programmatically. If unable to do so, will use "127.0.0.1".
        # (string value)
        
        my_ip=HOST_IP
        ```

    2. Configure the database location. ironic-conductor should use the same configuration as ironic-api. Replace **IRONIC_DBPASSWORD** with the password for the **ironic** user, and replace **DB_IP** with the IP address of the DB server:

        ```shell
        [database]
        
        # The SQLAlchemy connection string to use to connect to the
        # database. (string value)
        
        connection = mysql+pymysql://ironic:IRONIC_DBPASSWORD@DB_IP/ironic
        ```

    3. Configure the ironic-api service to use RabbitMQ message broker through the following options. ironic-conductor should use the same configuration as ironic-api. Replace **RPC_*** with the detailed RabbitMQ address and credentials

        ```shell
        [DEFAULT]
        
        # A URL representing the messaging driver to use and its full
        # configuration. (string value)
        
        transport_url = rabbit://RPC_USER:RPC_PASSWORD@RPC_HOST:RPC_PORT/
        ```

        Users can also choose to use json-rpc instead of rabbitmq on their own

    4. Configure credentials to access other OpenStack services

        To communicate with other OpenStack services, the bare metal service needs to authenticate with OpenStack Identity service using a service user when requesting other services. These user credentials must be configured in each configuration file related to the corresponding service.

        ```shell
        [neutron] - Access OpenStack Networking service
        [glance] - Access OpenStack Image service
        [swift] - Access OpenStack Object Storage service
        [cinder] - Access OpenStack Block Storage service
        [inspector] - Access OpenStack Bare Metal Introspection service
        [service_catalog] - A special entry for storing the credentials used by the bare metal service to discover its own API URL endpoints registered in the OpenStack Identity service catalog
        ```

        For simplicity, you can use the same service user for all services. For backward compatibility, this user should be the same as the one configured in the [keystone_authtoken] section of the ironic-api service. However, this is not mandatory. You can also create and configure different service users for each service.

        In the following example, the authentication information for accessing OpenStack Networking service is configured as:

        ```shell
        The networking service is deployed in the authentication service domain named RegionOne, with only the public endpoint interface registered in the service catalog
        
        HTTPS connections are used for requests with a specific CA SSL certificate
        
        The same service user as the ironic-api service configuration
        
        The dynamic password authentication plugin discovers the appropriate authentication service API version based on other options
        ```

        ```shell
        [neutron]
        
        # Authentication type to load (string value)
        auth_type = password
        # Authentication URL (string value)
        auth_url=https://IDENTITY_IP:5000/
        # Username (string value)
        username=ironic
        # User's password (string value)
        password=IRONIC_PASSWORD
        # Project name to scope to (string value)
        project_name=service
        # Domain ID containing project (string value)
        project_domain_id=default
        # User's domain id (string value)
        user_domain_id=default
        # PEM encoded Certificate Authority to use when verifying
        # HTTPs connections. (string value)
        cafile=/opt/stack/data/ca-bundle.pem
        # The default region_name for endpoint URL discovery. (string
        # value)
        region_name = RegionOne
        # List of interfaces, in order of preference, for endpoint
        # URL. (list value)
        valid_interfaces=public
        ```

        By default, to communicate with other services, the bare metal service will try to discover the appropriate endpoint for that service through the service catalog of the authentication service. If you want to use a different endpoint for a specific service, you can specify it through the endpoint_override option in the bare metal service configuration file:

        ```shell
        [neutron] ... endpoint_override = <NEUTRON_API_ADDRESS>
        ```

    5. Configure allowed drivers and hardware types

        Set the hardware types allowed by the ironic-conductor service through enabled_hardware_types:

        ```shell
        [DEFAULT] enabled_hardware_types = ipmi
        ```

        Configure hardware interfaces:

        ```shell
        enabled_boot_interfaces = pxe enabled_deploy_interfaces = direct,iscsi enabled_inspect_interfaces = inspector enabled_management_interfaces = ipmitool enabled_power_interfaces = ipmitool
        ```

        Configure default interface values:

        ```shell
        [DEFAULT] default_deploy_interface = direct default_network_interface = neutron
        ```

        If any drivers using Direct deploy are enabled, you must install and configure the Swift backend for the Image service. Ceph Object Gateway (RADOS Gateway) is also supported as a backend for the Image service.

    6. Restart the ironic-conductor service

        ```shell
        sudo systemctl restart openstack-ironic-conductor
        ```

5. Configure the ironic-inspector service

    Configuration file path: /etc/ironic-inspector/inspector.conf

    1. Create the database

        ```shell
        # mysql -u root -p
        
        MariaDB [(none)]> CREATE DATABASE ironic_inspector CHARACTER SET utf8;
        
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON ironic_inspector.* TO 'ironic_inspector'@'localhost' \     IDENTIFIED BY 'IRONIC_INSPECTOR_DBPASSWORD';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON ironic_inspector.* TO 'ironic_inspector'@'%' \
        IDENTIFIED BY 'IRONIC_INSPECTOR_DBPASSWORD';
        ```

    2. Configure the database location through the **connection** option as shown below. Replace **IRONIC_INSPECTOR_DBPASSWORD** with the password for the **ironic_inspector** user, and replace **DB_IP** with the IP address of the DB server:

        ```shell
        [database]
        backend = sqlalchemy
        connection = mysql+pymysql://ironic_inspector:IRONIC_INSPECTOR_DBPASSWORD@DB_IP/ironic_inspector
        min_pool_size = 100
        max_pool_size = 500
        pool_timeout = 30
        max_retries = 5
        max_overflow = 200
        db_retry_interval = 2
        db_inc_retry_interval = True
        db_max_retry_interval = 2
        db_max_retries = 5
        ```

    3. Configure the message queue communication address

        ```shell
        [DEFAULT] 
        transport_url = rabbit://RPC_USER:RPC_PASSWORD@RPC_HOST:RPC_PORT/
        
        ```

    4. Configure Keystone authentication

        ```shell
        [DEFAULT]
        
        auth_strategy = keystone
        timeout = 900
        rootwrap_config = /etc/ironic-inspector/rootwrap.conf
        logging_context_format_string = %(asctime)s.%(msecs)03d %(process)d %(levelname)s %(name)s [%(global_request_id)s %(request_id)s %(user_identity)s] %(instance)s%(message)s
        log_dir = /var/log/ironic-inspector
        state_path = /var/lib/ironic-inspector
        use_stderr = False
        
        [ironic]
        api_endpoint = http://IRONIC_API_HOST_ADDRRESS:6385
        auth_type = password
        auth_url = http://PUBLIC_IDENTITY_IP:5000
        auth_strategy = keystone
        ironic_url = http://IRONIC_API_HOST_ADDRRESS:6385
        os_region = RegionOne
        project_name = service
        project_domain_name = Default
        user_domain_name = Default
        username = IRONIC_SERVICE_USER_NAME
        password = IRONIC_SERVICE_USER_PASSWORD
        
        [keystone_authtoken]
        auth_type = password
        auth_url = http://control:5000
        www_authenticate_uri = http://control:5000
        project_domain_name = default
        user_domain_name = default
        project_name = service
        username = ironic_inspector
        password = IRONICPASSWD
        region_name = RegionOne
        memcache_servers = control:11211
        token_cache_time = 300
        
        [processing]
        add_ports = active
        processing_hooks = $default_processing_hooks,local_link_connection,lldp_basic
        ramdisk_logs_dir = /var/log/ironic-inspector/ramdisk
        always_store_ramdisk_logs = true
        store_data =none
        power_off = false
        
        [pxe_filter]
        driver = iptables
        
        [capabilities]
        boot_mode=True
        ```

    5. Configure the ironic inspector dnsmasq service

        ```shell
        # Configuration file path: /etc/ironic-inspector/dnsmasq.conf
        port=0
        interface=enp3s0                         #Replace with the actual listening network interface
        dhcp-range=172.20.19.100,172.20.19.110   #Replace with the actual DHCP address range
        bind-interfaces
        enable-tftp
        
        dhcp-match=set:efi,option:client-arch,7
        dhcp-match=set:efi,option:client-arch,9
        dhcp-match=aarch64, option:client-arch,11
        dhcp-boot=tag:aarch64,grubaa64.efi
        dhcp-boot=tag:!aarch64,tag:efi,grubx64.efi
        dhcp-boot=tag:!aarch64,tag:!efi,pxelinux.0
        
        tftp-root=/tftpboot                       #Replace with the actual tftpboot directory
        log-facility=/var/log/dnsmasq.log
        ```

    6. Disable DHCP on the ironic provision network subnet

        ```shell
        openstack subnet set --no-dhcp 72426e89-f552-4dc4-9ac7-c4e131ce7f3c
        ```

    7. Initialize the ironic-inspector service database

        Execute on the control node:

        ```shell
        ironic-inspector-dbsync --config-file /etc/ironic-inspector/inspector.conf upgrade
        ```

    8. Start services

        ```shell
        systemctl enable --now openstack-ironic-inspector.service
        systemctl enable --now openstack-ironic-inspector-dnsmasq.service
        ```

6. Configure the httpd service

    1. Create the httpd root directory to be used by ironic and set the owner and group. The directory path must match the path specified by the http_root configuration item in the [deploy] group in /etc/ironic/ironic.conf.

        ```shell
        mkdir -p /var/lib/ironic/httproot ``chown ironic.ironic /var/lib/ironic/httproot
        ```

    2. Install and configure the httpd service

        1. Install the httpd service. Skip if already installed

            ```shell
            yum install httpd -y
            ```

        2. Create the /etc/httpd/conf.d/openstack-ironic-httpd.conf file with the following content:

            ```shell
            Listen 8080
            
            <VirtualHost *:8080>
                ServerName ironic.openeuler.com
            
                ErrorLog "/var/log/httpd/openstack-ironic-httpd-error_log"
                CustomLog "/var/log/httpd/openstack-ironic-httpd-access_log" "%h %l %u %t \"%r\" %>s %b"
            
                DocumentRoot "/var/lib/ironic/httproot"
                <Directory "/var/lib/ironic/httproot">
                    Options Indexes FollowSymLinks
                    Require all granted
                </Directory>
                LogLevel warn
                AddDefaultCharset UTF-8
                EnableSendfile on
            </VirtualHost>
            
            ```

            Note that the listening port must match the port specified by the http_url configuration item in the [deploy] section of /etc/ironic/ironic.conf.

        3. Restart the httpd service.

            ```shell
            systemctl restart httpd
            ```

7. Deploy ramdisk image creation

    The W release ramdisk image supports creation through the ironic-python-agent service or the disk-image-builder tool. You can also use the community's latest ironic-python-agent-builder. Users can also choose other tools to create images.
    If using the W release native tools, you need to install the corresponding software packages.

    ```shell
    yum install openstack-ironic-python-agent
    or
    yum install diskimage-builder
    ```

    For specific usage, please refer to the [official documentation](https://docs.openstack.org/ironic/queens/install/deploy-ramdisk.html)

    Here is the complete process for building the ironic deploy image using ironic-python-agent-builder.

    1. Install ironic-python-agent-builder

        1. Install the tool:

            ```shell
            pip install ironic-python-agent-builder
            ```

        2. Modify the Python interpreter in the following files:

            ```shell
            /usr/bin/yum /usr/libexec/urlgrabber-ext-down
            ```

        3. Install other required tools:

            ```shell
            yum install git
            ```

            Since `DIB` depends on the `semanage` command, make sure this command is available before creating the image: `semanage --help`. If it reports that the command does not exist, install it:

            ```shell
            # First, query which package needs to be installed
            [root@localhost ~]# yum provides /usr/sbin/semanage
            Loaded plugins: fastestmirror
            Loading mirror speeds from cached hostfile
            * base: mirror.vcu.edu
            * extras: mirror.vcu.edu
            * updates: mirror.math.princeton.edu
            policycoreutils-python-2.5-34.el7.aarch64 : SELinux policy core python utilities
            Source    : base
            Match from:
            File name    : /usr/sbin/semanage
            # Install
            [root@localhost ~]# yum install policycoreutils-python
            ```

    2. Build the image

        If using `arm` architecture, you need to add:

        ```shell
        export ARCH=aarch64
        ```

        Basic usage:

        ```shell
        usage: ironic-python-agent-builder [-h] [-r RELEASE] [-o OUTPUT] [-e ELEMENT]
                                            [-b BRANCH] [-v] [--extra-args EXTRA_ARGS]
                                            distribution

        positional arguments:
            distribution          Distribution to use

        optional arguments:
            -h, --help            show this help message and exit
            -r RELEASE, --release RELEASE
                                Distribution release to use
            -o OUTPUT, --output OUTPUT
                                Output base file name
            -e ELEMENT, --element ELEMENT
                                Additional DIB element to use
            -b BRANCH, --branch BRANCH
                                If set, override the branch that is used for ironic-
                                python-agent and requirements
            -v, --verbose         Enable verbose logging in diskimage-builder
            --extra-args EXTRA_ARGS
                                Extra arguments to pass to diskimage-builder
        ```

        Example:

        ```shell
        ironic-python-agent-builder centos -o /mnt/ironic-agent-ssh -b origin/stable/rocky
        ```

    3. Enable SSH login

        Initialize environment variables, then build the image:

        ```shell
        export DIB_DEV_USER_USERNAME=ipa \
        export DIB_DEV_USER_PWDLESS_SUDO=yes \
        export DIB_DEV_USER_PASSWORD='123'
        ironic-python-agent-builder centos -o /mnt/ironic-agent-ssh -b origin/stable/rocky -e selinux-permissive -e devuser
        ```

    4. Specify the code repository

        Initialize the corresponding environment variables, then build the image:

        ```shell
        # Specify the repository address and version
        DIB_REPOLOCATION_ironic_python_agent=git@172.20.2.149:liuzz/ironic-python-agent.git
        DIB_REPOREF_ironic_python_agent=origin/develop

        # Clone code directly from gerrit
        DIB_REPOLOCATION_ironic_python_agent=https://opendev.org/openstack/ironic-python-agent.git
        DIB_REPOREF_ironic_python_agent=unmaintained/wallaby
        ```

        Reference: [source-repositories](https://docs.openstack.org/diskimage-builder/latest/elements/source-repositories/README.html).

        Specifying the repository address and version has been verified to work.

    5. Note

        The native openstack PXE configuration file template does not support arm64 architecture. Users need to modify the native openstack code themselves:

        In the W release, the community's ironic still does not support arm64 UEFI PXE boot. The generated grub.cfg file (usually located under /tftpboot/) has an incorrect format, causing PXE boot to fail, as shown below:

        The incorrectly generated configuration file:

        ![ironic-err](../img/install/ironic-err.png)

        As shown above, in the arm architecture, the commands to find vmlinux and ramdisk images are linux and initrd respectively. The commands highlighted in red in the image above are for x86 architecture UEFI PXE boot.

        Users need to modify the code logic that generates grub.cfg themselves.

        TLS error when ironic sends query requests to ipa for command execution status:

        In the W release, both ipa and ironic enable TLS authentication by default to send requests to each other. You can disable this according to the official documentation.

        1.Modify the ironic configuration file (/etc/ironic/ironic.conf) and add ipa-insecure=1 to the following configuration:

        ```shell
        [agent]
        verify_ca = False

        [pxe]
        pxe_append_params = nofb nomodeset vga=normal coreos.autologin ipa-insecure=1
        ```

        2.Add the ipa configuration file /etc/ironic_python_agent/ironic_python_agent.conf in the ramdisk image and configure TLS settings as follows:

        /etc/ironic_python_agent/ironic_python_agent.conf (you need to create the /etc/ironic_python_agent directory in advance)

        ```shell
        [DEFAULT]
        enable_auto_tls = False
        ```

        Set permissions:

        ```shell
        chown -R ipa.ipa /etc/ironic_python_agent/
        ```

        3.Modify the ipa service startup file to add the configuration file option

        vim usr/lib/systemd/system/ironic-python-agent.service

        ```shell
        [Unit]
        Description=Ironic Python Agent
        After=network-online.target

        [Service]
        ExecStartPre=/sbin/modprobe vfat
        ExecStart=/usr/local/bin/ironic-python-agent --config-file /etc/ironic_python_agent/ironic_python_agent.conf
        Restart=always
        RestartSec=30s

        [Install]
        WantedBy=multi-user.target
        ```

### Kolla Installation

Kolla provides production-ready containerized deployment for OpenStack services. Kolla and Kolla-ansible services have been introduced in openEuler 24.03 LTS SP4.

Kolla installation is very simple. You just need to install the corresponding RPM package

```shell
yum install openstack-kolla openstack-kolla-ansible
```

After installation, you can use commands like `kolla-ansible`, `kolla-build`, `kolla-genpwd`, `kolla-mergepwd`, etc.

### Trove Installation

Trove is the database service for OpenStack. If users use the database service provided by OpenStack, it is recommended to use this component. Otherwise, installation is optional.

1.Set up the database

   The database service stores information in the database. Create a **trove** database accessible by the **trove** user. Replace **TROVE_DBPASSWORD** with an appropriate password

   ```sql
   mysql -u root -p
   
   MariaDB [(none)]> CREATE DATABASE trove CHARACTER SET utf8;
   MariaDB [(none)]> GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'localhost' \
   IDENTIFIED BY 'TROVE_DBPASSWORD';
   MariaDB [(none)]> GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'%' \
   IDENTIFIED BY 'TROVE_DBPASSWORD';
   ```

2.Create service user authentication

   1.Create the **Trove** service user

   ```shell
   openstack user create --password TROVE_PASSWORD \
                         --email trove@example.com trove
   openstack role add --project service --user trove admin
   openstack service create --name trove
                            --description "Database service" database
   ```

   **Explanation:** Replace `TROVE_PASSWORD` with the password for the `trove` user

   2.Create the **Database** service endpoints

   ```shell
   openstack endpoint create --region RegionOne database public http://controller:8779/v1.0/%\(tenant_id\)s
   openstack endpoint create --region RegionOne database internal http://controller:8779/v1.0/%\(tenant_id\)s
   openstack endpoint create --region RegionOne database admin http://controller:8779/v1.0/%\(tenant_id\)s
   ```

3.Install and configure **Trove** components

   1.Install **Trove** packages

   ```shell
   yum install openstack-trove python-troveclient
   ```

   2.Configure `trove.conf`

   ```shell
   vim /etc/trove/trove.conf
   
   [DEFAULT]
   bind_host=TROVE_NODE_IP
   log_dir = /var/log/trove
   network_driver = trove.network.neutron.NeutronDriver
   management_security_groups = <manage security group>
   nova_keypair = trove-mgmt
   default_datastore = mysql
   taskmanager_manager = trove.taskmanager.manager.Manager
   trove_api_workers = 5
   transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
   reboot_time_out = 300
   usage_timeout = 900
   agent_call_high_timeout = 1200
   use_syslog = False
   debug = True
   
   # Set these if using Neutron Networking
   network_driver=trove.network.neutron.NeutronDriver
   network_label_regex=.*
   
   
   transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
   
   [database]
   connection = mysql+pymysql://trove:TROVE_DBPASS@controller/trove
   
   [keystone_authtoken]
   project_domain_name = Default
   project_name = service
   user_domain_name = Default
   password = trove
   username = trove
   auth_url = http://controller:5000/v3/
   auth_type = password
   
   [service_credentials]
   auth_url = http://controller:5000/v3/
   region_name = RegionOne
   project_name = service
   password = trove
   project_domain_name = Default
   user_domain_name = Default
   username = trove
   
   [mariadb]
   tcp_ports = 3306,4444,4567,4568
   
   [mysql]
   tcp_ports = 3306
   
   [postgresql]
   tcp_ports = 5432
   ```

   **Explanation:**

- In the `[Default]` section, `bind_host` is configured as the IP of the node where Trove is deployed
- `nova_compute_url` and `cinder_url` are the endpoints created for Nova and Cinder in Keystone
- `nova_proxy_XXX` is the user information that can access the Nova service. The example above uses the `admin` user
- `transport_url` is the `RabbitMQ` connection information. Replace `RABBIT_PASS` with the RabbitMQ password
- The `connection` in the `[database]` section is the database information created for Trove in MySQL
- In Trove user information, replace `TROVE_PASS` with the actual password for the trove user  

   3.Configure `trove-guestagent.conf`

   ```shell
   vim /etc/trove/trove-guestagent.conf
   
   [DEFAULT]
   log_file = trove-guestagent.log
   log_dir = /var/log/trove/
   ignore_users = os_admin
   control_exchange = trove
   transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
   rpc_backend = rabbit
   command_process_timeout = 60
   use_syslog = False
   debug = True
   
   [service_credentials]
   auth_url = http://controller:5000/v3/
   region_name = RegionOne
   project_name = service
   password = TROVE_PASS
   project_domain_name = Default
   user_domain_name = Default
   username = trove
   
   [mysql]
   docker_image = your-registry/your-repo/mysql
   backup_docker_image = your-registry/your-repo/db-backup-mysql:1.1.0
   ```

   **Explanation:** `guestagent` is a separate component in Trove. It needs to be pre-loaded into the virtual machine image created by Trove through Nova. After the database instance is created, the guestagent process will start and report heartbeats to Trove through the message queue (RabbitMQ). Therefore, RabbitMQ user and password information needs to be configured.
   **Starting from the Victoria release, Trove uses a unified image to run different types of databases. Database services run in Docker containers inside the Guest VM.**

- `transport_url` is the `RabbitMQ` connection information. Replace `RABBIT_PASS` with the RabbitMQ password
- In Trove user information, replace `TROVE_PASS` with the actual password for the trove user  

   4.Generate Trove database tables

   ```shell
   su -s /bin/sh -c "trove-manage db_sync" trove
   ```

4.Complete the installation and configuration

   1.Configure **Trove** services to start on boot

   ```shell
   systemctl enable openstack-trove-api.service \
   openstack-trove-taskmanager.service \
   openstack-trove-conductor.service 
   ```

   2.Start services

   ```shell
   systemctl start openstack-trove-api.service \
   openstack-trove-taskmanager.service \
   openstack-trove-conductor.service
   ```

### Swift Installation

Swift provides elastic, scalable, and highly available distributed object storage services, suitable for storing large-scale unstructured data.

1.Create service credentials and API endpoints.

Create service credentials

``` shell
#Create swift user:
openstack user create --domain default --password-prompt swift                 
#Add admin role to swift user:
openstack role add --project service --user swift admin                        
#Create swift service entity:
openstack service create --name swift --description "OpenStack Object Storage" object-store                         
```

Create swift API endpoints:

```shell
openstack endpoint create --region RegionOne object-store public http://controller:8080/v1/AUTH_%\(project_id\)s
openstack endpoint create --region RegionOne object-store internal http://controller:8080/v1/AUTH_%\(project_id\)s
openstack endpoint create --region RegionOne object-store admin http://controller:8080/v1
```

2.Install software packages:

```shell
yum install openstack-swift-proxy python3-swiftclient python3-keystoneclient python3-keystonemiddleware memcached （CTL）
```

3.Configure proxy-server related settings

The Swift RPM package already includes a basically usable proxy-server.conf. You only need to manually modify the IP and swift password in it.

***Note***

**Note to replace the password with the password you selected for the swift user in the identity service**

4.Install and configure storage nodes （STG）

Install supported packages:

```shell
yum install xfsprogs rsync
```

Format /dev/vdb and /dev/vdc devices as XFS

```shell
mkfs.xfs /dev/vdb
mkfs.xfs /dev/vdc
```

Create the mount point directory structure:

```shell
mkdir -p /srv/node/vdb
mkdir -p /srv/node/vdc
```

Find the UUID of the new partition:

```shell
blkid
```

Edit the /etc/fstab file and add the following content to it:

```shell
UUID="<UUID-from-output-above>" /srv/node/vdb xfs noatime 0 2
UUID="<UUID-from-output-above>" /srv/node/vdc xfs noatime 0 2
```

Mount the devices:

```shell
mount /srv/node/vdb
mount /srv/node/vdc
```

***Note***

**If users do not need disaster recovery functionality, the above steps only require creating one device. You can also skip the rsync configuration below**

(Optional) Create or edit the /etc/rsyncd.conf file to include the following:

```shell
[DEFAULT]
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = MANAGEMENT_INTERFACE_IP_ADDRESS

[account]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/account.lock

[container]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/container.lock

[object]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/object.lock
```

**Replace MANAGEMENT_INTERFACE_IP_ADDRESS with the IP address of the management network on the storage node**

Start the rsyncd service and configure it to start on system boot:

```shell
systemctl enable rsyncd.service
systemctl start rsyncd.service
```

5.Install and configure components on storage nodes （STG）

Install software packages:

```shell
yum install openstack-swift-account openstack-swift-container openstack-swift-object
```

Edit the account-server.conf, container-server.conf, and object-server.conf files in the /etc/swift directory. Replace bind_ip with the IP address of the management network on the storage node.

Ensure correct ownership of the mount point directory structure:

```shell
chown -R swift:swift /srv/node
```

Create the recon directory and ensure it has correct ownership:

```shell
mkdir -p /var/cache/swift
chown -R root:swift /var/cache/swift
chmod -R 775 /var/cache/swift
```

6.Create account rings （CTL）

Switch to the /etc/swift directory.

```shell
cd /etc/swift
```

Create the base account.builder file:

```shell
swift-ring-builder account.builder create 10 1 1
```

Add each storage node to the ring:

```shell
swift-ring-builder account.builder add --region 1 --zone 1 --ip STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS --port 6202  --device DEVICE_NAME --weight DEVICE_WEIGHT
```

**Replace STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS with the IP address of the management network on the storage node. Replace DEVICE_NAME with the storage device name on the same storage node**

***Note***

**Repeat this command for each storage device on each storage node**

Verify the ring contents:

```shell
swift-ring-builder account.builder
```

Rebalance the ring:

```shell
swift-ring-builder account.builder rebalance
```

7.Create container rings （CTL）

Switch to the `/etc/swift` directory.

Create the base `container.builder` file:

```shell
    swift-ring-builder container.builder create 10 1 1
```

Add each storage node to the ring:

```shell
swift-ring-builder container.builder \
    add --region 1 --zone 1 --ip STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS --port 6201 \
    --device DEVICE_NAME --weight 100

```

**Replace STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS with the IP address of the management network on the storage node. Replace DEVICE_NAME with the storage device name on the same storage node**

***Note***
**Repeat this command for each storage device on each storage node**

Verify the ring contents:

```shell
swift-ring-builder container.builder
```

Rebalance the ring:

```shell
swift-ring-builder container.builder rebalance
```

8.Create object rings （CTL）

Switch to the `/etc/swift` directory.

Create the base `object.builder` file:

```shell
swift-ring-builder object.builder create 10 1 1
```

Add each storage node to the ring

```shell
    swift-ring-builder object.builder \
    add --region 1 --zone 1 --ip STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS --port 6200 \
    --device DEVICE_NAME --weight 100
```

**Replace STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS with the IP address of the management network on the storage node. Replace DEVICE_NAME with the storage device name on the same storage node**

***Note***

**Repeat this command for each storage device on each storage node**

Verify the ring contents:

```shell
swift-ring-builder object.builder
```

Rebalance the ring:

```shell
swift-ring-builder object.builder rebalance
```

Distribute ring configuration files:

Copy the `account.ring.gz`, `container.ring.gz`, and `object.ring.gz` files to the `/etc/swift` directory on each storage node and any other node running the proxy service.

9.Complete the installation

Edit the `/etc/swift/swift.conf` file

``` shell
[swift-hash]
swift_hash_path_suffix = test-hash
swift_hash_path_prefix = test-hash

[storage-policy:0]
name = Policy-0
default = yes
```

**Replace test-hash with unique values**

Copy the swift.conf file to the /etc/swift directory on each storage node and any other node running the proxy service.

On all nodes, ensure correct ownership of the configuration directory:

```shell
chown -R root:swift /etc/swift
```

On the controller node and any other node running the proxy service, start the object storage proxy service and its dependencies, and configure them to start on system boot:

```shell
systemctl enable openstack-swift-proxy.service memcached.service
systemctl start openstack-swift-proxy.service memcached.service
```

On storage nodes, start the object storage services and configure them to start on system boot:

```shell
systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service openstack-swift-account-reaper.service openstack-swift-account-replicator.service

systemctl start openstack-swift-account.service openstack-swift-account-auditor.service openstack-swift-account-reaper.service openstack-swift-account-replicator.service

systemctl enable openstack-swift-container.service openstack-swift-container-auditor.service openstack-swift-container-replicator.service openstack-swift-container-updater.service

systemctl start openstack-swift-container.service openstack-swift-container-auditor.service openstack-swift-container-replicator.service openstack-swift-container-updater.service

systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service openstack-swift-object-replicator.service openstack-swift-object-updater.service

systemctl start openstack-swift-object.service openstack-swift-object-auditor.service openstack-swift-object-replicator.service openstack-swift-object-updater.service
```

### Cyborg Installation

Cyborg provides accelerator device support for OpenStack, including GPU, FPGA, ASIC, NP, SoCs, NVMe/NOF SSDs, ODP, DPDK/SPDK, and more.

1.Initialize the corresponding database

```shell
CREATE DATABASE cyborg;
GRANT ALL PRIVILEGES ON cyborg.* TO 'cyborg'@'localhost' IDENTIFIED BY 'CYBORG_DBPASS';
GRANT ALL PRIVILEGES ON cyborg.* TO 'cyborg'@'%' IDENTIFIED BY 'CYBORG_DBPASS';
```

2.Create the corresponding Keystone resource objects

```shell
$ openstack user create --domain default --password-prompt cyborg
$ openstack role add --project service --user cyborg admin
$ openstack service create --name cyborg --description "Acceleration Service" accelerator

$ openstack endpoint create --region RegionOne \
  accelerator public http://<cyborg-ip>:6666/v1
$ openstack endpoint create --region RegionOne \
  accelerator internal http://<cyborg-ip>:6666/v1
$ openstack endpoint create --region RegionOne \
  accelerator admin http://<cyborg-ip>:6666/v1
```

3.Install Cyborg

```shell
yum install openstack-cyborg
```

4.Configure Cyborg

Modify `/etc/cyborg/cyborg.conf`

```shell
[DEFAULT]
transport_url = rabbit://%RABBITMQ_USER%:%RABBITMQ_PASSWORD%@%OPENSTACK_HOST_IP%:5672/
use_syslog = False
state_path = /var/lib/cyborg
debug = True

[database]
connection = mysql+pymysql://%DATABASE_USER%:%DATABASE_PASSWORD%@%OPENSTACK_HOST_IP%/cyborg

[service_catalog]
project_domain_id = default
user_domain_id = default
project_name = service
password = PASSWORD
username = cyborg
auth_url = http://%OPENSTACK_HOST_IP%/identity
auth_type = password

[placement]
project_domain_name = Default
project_name = service
user_domain_name = Default
password = PASSWORD
username = placement
auth_url = http://%OPENSTACK_HOST_IP%/identity
auth_type = password

[keystone_authtoken]
memcached_servers = localhost:11211
project_domain_name = Default
project_name = service
user_domain_name = Default
password = PASSWORD
username = cyborg
auth_url = http://%OPENSTACK_HOST_IP%/identity
auth_type = password
```

Modify the corresponding username, password, IP, and other information as needed

5.Synchronize database tables

```shell
cyborg-dbsync --config-file /etc/cyborg/cyborg.conf upgrade
```

6.Start Cyborg services

```shell
systemctl enable openstack-cyborg-api openstack-cyborg-conductor openstack-cyborg-agent
systemctl start openstack-cyborg-api openstack-cyborg-conductor openstack-cyborg-agent
```

### Aodh Installation

1.Create the database

```shell
CREATE DATABASE aodh;

GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'localhost' IDENTIFIED BY 'AODH_DBPASS';

GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'%' IDENTIFIED BY 'AODH_DBPASS';
```

2.Create the corresponding Keystone resource objects

```shell
openstack user create --domain default --password-prompt aodh

openstack role add --project service --user aodh admin

openstack service create --name aodh --description "Telemetry" alarming

openstack endpoint create --region RegionOne alarming public http://controller:8042

openstack endpoint create --region RegionOne alarming internal http://controller:8042

openstack endpoint create --region RegionOne alarming admin http://controller:8042
```

3.Install Aodh

```shell
yum install openstack-aodh-api openstack-aodh-evaluator openstack-aodh-notifier openstack-aodh-listener openstack-aodh-expirer python3-aodhclient
```

***Note***

`aodh` depends on the software package `python3-pyparsing` which is not adapted in openEuler's OS repository. You need to overwrite install the OpenStack corresponding version. You can use `yum list |grep pyparsing |grep OpenStack | awk '{print $2}'` to get the corresponding version

VERSION, then `yum install -y python3-pyparsing-VERSION` to overwrite install the adapted `pyparsing`

4.Modify the configuration file

```shell
[database]
connection = mysql+pymysql://aodh:AODH_DBPASS@controller/aodh

[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_id = default
user_domain_id = default
project_name = service
username = aodh
password = AODH_PASS

[service_credentials]
auth_type = password
auth_url = http://controller:5000/v3
project_domain_id = default
user_domain_id = default
project_name = service
username = aodh
password = AODH_PASS
interface = internalURL
region_name = RegionOne
```

5.Initialize the database

```shell
aodh-dbsync
```

6.Start Aodh services

```shell
systemctl enable openstack-aodh-api.service openstack-aodh-evaluator.service openstack-aodh-notifier.service openstack-aodh-listener.service

systemctl start openstack-aodh-api.service openstack-aodh-evaluator.service openstack-aodh-notifier.service openstack-aodh-listener.service
```

### Gnocchi Installation

1.Create the database

```shell
CREATE DATABASE gnocchi;

GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'localhost' IDENTIFIED BY 'GNOCCHI_DBPASS';

GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'%' IDENTIFIED BY 'GNOCCHI_DBPASS';
```

2.Create the corresponding Keystone resource objects

```shell
openstack user create --domain default --password-prompt gnocchi

openstack role add --project service --user gnocchi admin

openstack service create --name gnocchi --description "Metric Service" metric

openstack endpoint create --region RegionOne metric public http://controller:8041

openstack endpoint create --region RegionOne metric internal http://controller:8041

openstack endpoint create --region RegionOne metric admin http://controller:8041
```

3.Install Gnocchi

```shell
yum install openstack-gnocchi-api openstack-gnocchi-metricd python3-gnocchiclient
```

4.Modify the configuration file `/etc/gnocchi/gnocchi.conf`

```shell
[api]
auth_mode = keystone
port = 8041
uwsgi_mode = http-socket

[keystone_authtoken]
auth_type = password
auth_url = http://controller:5000/v3
project_domain_name = Default
user_domain_name = Default
project_name = service
username = gnocchi
password = GNOCCHI_PASS
interface = internalURL
region_name = RegionOne

[indexer]
url = mysql+pymysql://gnocchi:GNOCCHI_DBPASS@controller/gnocchi

[storage]
# coordination_url is not required but specifying one will improve
# performance with better workload division across workers.
coordination_url = redis://controller:6379
file_basepath = /var/lib/gnocchi
driver = file
```

5.Initialize the database

```shell
gnocchi-upgrade
```

6.Start Gnocchi services

```shell
systemctl enable openstack-gnocchi-api.service openstack-gnocchi-metricd.service

systemctl start openstack-gnocchi-api.service openstack-gnocchi-metricd.service
```

### Ceilometer Installation

1.Create the corresponding Keystone resource objects

```shell
openstack user create --domain default --password-prompt ceilometer

openstack role add --project service --user ceilometer admin

openstack service create --name ceilometer --description "Telemetry" metering
```

2.Install Ceilometer

```shell
yum install openstack-ceilometer-notification openstack-ceilometer-central
```

3.Modify the configuration file `/etc/ceilometer/pipeline.yaml`

```shell
publishers:
    # set address of Gnocchi
    # + filter out Gnocchi-related activity meters (Swift driver)
    # + set default archive policy
    - gnocchi://?filter_project=service&archive_policy=low
```

4.Modify the configuration file `/etc/ceilometer/ceilometer.conf`

```shell
[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller

[service_credentials]
auth_type = password
auth_url = http://controller:5000/v3
project_domain_id = default
user_domain_id = default
project_name = service
username = ceilometer
password = CEILOMETER_PASS
interface = internalURL
region_name = RegionOne
```

5.Initialize the database

```shell
ceilometer-upgrade
```

6.Start Ceilometer services

```shell
systemctl enable openstack-ceilometer-notification.service openstack-ceilometer-central.service

systemctl start openstack-ceilometer-notification.service openstack-ceilometer-central.service
```

### Heat Installation

1.Create the **heat** database and grant the correct access permissions to the **heat** database. Replace **HEAT_DBPASS** with an appropriate password

```shell
CREATE DATABASE heat;
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' IDENTIFIED BY 'HEAT_DBPASS';
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' IDENTIFIED BY 'HEAT_DBPASS';
```

2.Create service credentials, create the **heat** user, and add the **admin** role to it

```shell
openstack user create --domain default --password-prompt heat
openstack role add --project service --user heat admin
```

3.Create the **heat** and **heat-cfn** services and their corresponding API endpoints

```shell
openstack service create --name heat --description "Orchestration" orchestration
openstack service create --name heat-cfn --description "Orchestration"  cloudformation
openstack endpoint create --region RegionOne orchestration public http://controller:8004/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne orchestration internal http://controller:8004/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne orchestration admin http://controller:8004/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne cloudformation public http://controller:8000/v1
openstack endpoint create --region RegionOne cloudformation internal http://controller:8000/v1
openstack endpoint create --region RegionOne cloudformation admin http://controller:8000/v1
```

4.Create additional information for stack management, including the **heat** domain and its corresponding domain admin user **heat_domain_admin**,
**heat_stack_owner** role, and **heat_stack_user** role

```shell
openstack user create --domain heat --password-prompt heat_domain_admin
openstack role add --domain heat --user-domain heat --user heat_domain_admin admin
openstack role create heat_stack_owner
openstack role create heat_stack_user
```

5.Install software packages

```shell
yum install openstack-heat-api openstack-heat-api-cfn openstack-heat-engine
```

6.Modify the configuration file `/etc/heat/heat.conf`

```shell
[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
heat_metadata_server_url = http://controller:8000
heat_waitcondition_server_url = http://controller:8000/v1/waitcondition
stack_domain_admin = heat_domain_admin
stack_domain_admin_password = HEAT_DOMAIN_PASS
stack_user_domain_name = heat

[database]
connection = mysql+pymysql://heat:HEAT_DBPASS@controller/heat

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = heat
password = HEAT_PASS

[trustee]
auth_type = password
auth_url = http://controller:5000
username = heat
password = HEAT_PASS
user_domain_name = default

[clients_keystone]
auth_uri = http://controller:5000
```

7.Initialize **heat** database tables

```shell
su -s /bin/sh -c "heat-manage db_sync" heat
```

8.Start services

```shell
systemctl enable openstack-heat-api.service openstack-heat-api-cfn.service openstack-heat-engine.service
systemctl start openstack-heat-api.service openstack-heat-api-cfn.service openstack-heat-engine.service
```

## Quick Deployment Based on OpenStack SIG Development Tool oos

`oos` (openEuler OpenStack SIG) is a command-line tool provided by the OpenStack SIG. The `oos env` series of commands provide ansible scripts for one-click deployment of OpenStack (`all in one` or three-node `cluster`). Users can use these scripts to quickly deploy an OpenStack environment based on openEuler RPM. The `oos` tool supports two deployment methods: connecting to a cloud provider (currently only supports Huawei Cloud provider) and host management. The following uses connecting to Huawei Cloud to deploy an `all in one` OpenStack environment as an example to explain how to use the `oos` tool.

1.Install the `oos` tool

```shell
yum install openstack-sig-tool
```

2.Configure the Huawei Cloud provider information

Open the `/usr/local/etc/oos/oos.conf` file and modify the configuration to your Huawei Cloud resource information:

```shell
[huaweicloud]
ak = 
sk = 
region = ap-southeast-3
root_volume_size = 100
data_volume_size = 100
security_group_name = oos
image_format = openEuler-%%(release)s-%%(arch)s
vpc_name = oos_vpc
subnet1_name = oos_subnet1
subnet2_name = oos_subnet2
```

3.Configure OpenStack environment information

Open the `/usr/local/etc/oos/oos.conf` file and modify the configuration according to the current machine environment and requirements. The content is as follows:

```shell
[environment]
mysql_root_password = root
mysql_project_password = root
rabbitmq_password = root
project_identity_password = root
enabled_service = keystone,neutron,cinder,placement,nova,glance,horizon,aodh,ceilometer,cyborg,gnocchi,kolla,heat,swift,trove,tempest
neutron_provider_interface_name = br-ex
default_ext_subnet_range = 10.100.100.0/24
default_ext_subnet_gateway = 10.100.100.1
neutron_dataplane_interface_name = eth1
cinder_block_device = vdb
swift_storage_devices = vdc
swift_hash_path_suffix = ash
swift_hash_path_prefix = has
glance_api_workers = 2
cinder_api_workers = 2
nova_api_workers = 2
nova_metadata_api_workers = 2
nova_conductor_workers = 2
nova_scheduler_workers = 2
neutron_api_workers = 2
horizon_allowed_host = *
kolla_openeuler_plugin = false
```

**Key Configuration**

| Configuration Item   | Explanation |
|---|---|
| enabled_service  |  List of services to install. Users can add or remove according to their needs |
| neutron_provider_interface_name  | neutron L3 bridge name  |
| default_ext_subnet_range  | neutron private network IP range  |
| default_ext_subnet_gateway  | neutron private network gateway  |
| neutron_dataplane_interface_name  | Network interface used by neutron. It is recommended to use a new network interface to avoid conflicts with existing network interfaces and prevent disconnection of all-in-one hosts  |
| cinder_block_device  |  Volume device name used by cinder |
| swift_storage_devices  | Volume device name used by swift |
| kolla_openeuler_plugin | Whether to enable the kolla plugin. If set to True, kolla will support deploying openEuler containers |

4.Create an openEuler 24.03-LTS-SP4 x86_64 virtual machine on Huawei Cloud for deploying `all in one` OpenStack

```shell
# sshpass is used during `oos env create` to configure password-free access to the target virtual machine
dnf install sshpass
oos env create -r 24.03-lts-sp4 -f small -a x86 -n test-oos all_in_one
```

For specific parameters, you can use the `oos env create --help` command to view

5.Deploy OpenStack `all in one` environment

```shell
oos env setup test-oos -r wallaby
```

For specific parameters, you can use the `oos env setup --help` command to view

6.Initialize the tempest environment

If users want to run tempest tests in this environment, execute the `oos env init` command. This will automatically create the OpenStack resources needed by tempest

```shell
oos env init test-oos
```

After the command executes successfully, a `mytest` directory will be generated in the user's home directory. You can enter it and run the tempest run command.

If deploying the OpenStack environment through host management, the overall logic is the same as connecting to Huawei Cloud above. Steps 1, 3, 5, and 6 remain unchanged. Remove Step 2 for Huawei Cloud provider information configuration. Step 4 is changed from creating a virtual machine on Huawei Cloud to managing the host.

```shell
# sshpass is used during `oos env create` to configure password-free access to the target host
dnf install sshpass
oos env manage -r 24.03-lts-sp4 -i TARGET_MACHINE_IP -p TARGET_MACHINE_PASSWD -n test-oos
```

Replace `TARGET_MACHINE_IP` with the target machine IP and `TARGET_MACHINE_PASSWD` with the target machine password. For specific parameters, you can use the `oos env manage --help` command to view.
