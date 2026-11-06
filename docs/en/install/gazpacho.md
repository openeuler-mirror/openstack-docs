# OpenStack Gazpacho Deployment Guide (Three Nodes)

[TOC]

This document is the OpenStack **2026.1 Gazpacho** deployment guide based on openEuler 24.03 LTS SP4. If you have any questions or find any issues while reading, please contact the deployment and maintenance personnel.

**Conventions**

This section describes common conventions used in this document.

| Name | Definition |
|:----:|:----:|
| MYSQL_ROOT_PASS | MariaDB root password, example value `RootMysql2026` |
| RABBIT_PASS | RabbitMQ password, example value `RbtOpenStack2026` |
| ADMIN_PASS | OpenStack admin user password, example value `Admin2026` |
| KEYSTONE_DBPASS / KEYSTONE_PASS | Keystone service database password / keystone user password, example value `Keystone2026` |
| GLANCE_DBPASS / GLANCE_PASS | Glance service database password / keystone user password, example value `Glance2026` |
| PLACEMENT_DBPASS / PLACEMENT_PASS | Placement service database password / keystone user password, example value `Placement2026` |
| NOVA_DBPASS / NOVA_PASS | Nova service database password / keystone user password, example value `Nova2026` |
| NEUTRON_DBPASS / NEUTRON_PASS | Neutron service database password / keystone user password, example value `Neutron2026` |
| CINDER_DBPASS / CINDER_PASS | Cinder service database password / keystone user password, example value `Cinder2026` |
| SWIFT_PASS | Swift keystone user password, example value `Swift2026` |
| HEAT_DBPASS / HEAT_PASS | Heat service database password / keystone user password, example value `Heat2026` |
| IRONIC_DBPASS / IRONIC_PASS | Ironic service database password / keystone user password, example value `Ironic2026` |
| TROVE_DBPASS / TROVE_PASS | Trove service database password / keystone user password, example value `Trove2026` |
| CYBORG_DBPASS / CYBORG_PASS | Cyborg service database password / keystone user password, example value `Cyborg2026` |
| AODH_DBPASS / AODH_PASS | Aodh service database password / keystone user password, example value `Aodh2026` |
| GNOCCHI_DBPASS / GNOCCHI_PASS | Gnocchi service database password / keystone user password, example value `Gnocchi2026` |
| CEILOMETER_DBPASS / CEILOMETER_PASS | Ceilometer service database password / keystone user password, example value `Ceilometer2026` |
| METADATA_SECRET | Secret password for the metadata proxy, example value `Metadata2026` |
| PROVIDER_INTERFACE_NAME | Name of the physical network interface (provider network), example value `ens34` |
| OVERLAY_INTERFACE_IP_ADDRESS | Management network IP address of each node (VXLAN tunnel endpoint), example values `192.168.0.2/192.168.0.3/192.168.0.4` |

> Note: The example passwords use the `xxx2026` pattern and are consistent with the single-node verification environment. Replace them as needed for your environment.

OpenStack supports multiple deployment methods. This document uses **manual RPM-based deployment**. The service scope is consistent with the openEuler OpenStack SIG documentation and covers **15 OpenStack services plus the Tempest test tool**:

| Category | Services |
|:----:|:----:|
| Core (7) | Keystone, Glance, Placement, Nova, Neutron, Cinder, Horizon |
| Optional (8) | Swift, Heat, Ironic, Trove, Cyborg, Aodh, Gnocchi, Ceilometer |
| Testing | Tempest (functional testing, optional) |

## RPM-based Deployment

### Environment Preparation

This document is based on a classic three-node OpenStack environment. The three nodes are the controller node, compute node, and storage node. The storage node usually runs only storage services. If resources are limited, you can deploy the storage services on the compute node instead.

First prepare three openEuler 24.03 LTS SP4 environments. Download the appropriate image and install it according to your environment: [ISO image](https://repo.openeuler.org/openEuler-24.03-LTS-SP4/ISO/), [qcow2 image](https://repo.openeuler.org/openEuler-24.03-LTS-SP4/virtual_machine_img/).

The installation follows the topology below:

```text
controller: 192.168.0.2   (management network ens33; provider network ens34)
compute:    192.168.0.3   (management network ens33; provider network ens34)
storage:    192.168.0.4   (management network ens33; storage data disk /dev/sdb)
```

If the IP addresses or NIC names in your environment are different, modify the corresponding configuration files accordingly.

The three-node service topology in this document is as follows:

| Node | Deployed Services |
|:----:|:----:|
| Controller | mariadb / rabbitmq / memcached, Keystone, Glance, Placement, Nova (api/cond/sched/vncproxy), Neutron (server + ovs/dhcp/l3/metadata agents), Cinder (api/scheduler), Swift (proxy), Heat, Horizon, Ironic (api/cond), Trove, Cyborg (api/cond), Aodh, Gnocchi, Ceilometer (api/central), Tempest |
| Compute | Nova (nova-compute), Neutron (ovs agent), Cyborg (agent), Ceilometer (compute) |
| Storage | Cinder (cinder-volume, LVM backend), Swift (account/container/object) |

Before formal deployment, perform the following configuration and checks on each node:

#### Configure the openEuler 24.03 LTS SP4 Yum Repository (Gazpacho Multi-Version Repository)

This document uses the Gazpacho multi-version repository (EPOL Multi-Version), which provides all Gazpacho service packages. **All three nodes** must be configured.

```shell
cat > /etc/yum.repos.d/openstack-gazpacho.repo << EOF
[openstack-gazpacho]
name=openstack-gazpacho
baseurl=https://eulermaker.openeuler.openatom.cn/api/ems3/repositories/openEuler_24.03_LTS_SP4_Epol_Multi-Version_OpenStack_Gazpacho/openEuler%3A24.03-LTS-SP4/\$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.openeuler.org/openEuler-24.03-LTS-SP4/OS/\$basearch/RPM-GPG-KEY-openEuler
EOF
```

**Notes**:

- RPMs in the Gazpacho multi-version repository are signed, and `gpgcheck=1` works correctly (verified).
- In addition to the Gazpacho repository, base packages such as `openstack-neutron-openvswitch`, `targetcli`, and `scsi-target-utils` require the EPOL or everything repository. If EPOL is already enabled in the default openEuler repository, no additional configuration is needed.
- If a package cannot be found, verify that EPOL is enabled in `/etc/yum.repos.d/openEuler.repo`.

```shell
dnf makecache
```

#### Modify the Hostname and Hosts File

Set the hostname on each node. For example, on the controller node:

```shell
hostnamectl set-hostname controller
```

Then add the following entries to `/etc/hosts` on each node:

```text
192.168.0.2   controller
192.168.0.3   compute
192.168.0.4   storage
```

#### Clock Synchronization

Nodes in a cluster must have consistent time. This document uses `chrony`.

**Controller node**:

```shell
dnf install chrony
# Add the following line to /etc/chrony.conf: allow 192.168.0.0/24
sed -i '/^allow/a allow 192.168.0.0/24' /etc/chrony.conf
systemctl enable --now chronyd
```

**Other nodes** (compute/storage):

```shell
dnf install chrony
# Add "server controller iburst" to /etc/chrony.conf and comment out the pool line
echo "server controller iburst" >> /etc/chrony.conf
sed -i 's/^pool /#pool /' /etc/chrony.conf
systemctl enable --now chronyd
```

Verification (on other nodes): `chronyc sources` should show `^* controller`.

#### Install the Database

Install the database on the controller node. MariaDB is recommended.

```shell
dnf install mariadb-config mariadb mariadb-server python3-PyMySQL
```

Create the configuration file `/etc/my.cnf.d/openstack.cnf` with the following content:

```ini
[mysqld]
bind-address = 192.168.0.2
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

1. Start the server.

    ```shell
    systemctl enable --now mariadb
    ```

2. Initialize the database and follow the prompts.

    ```shell
    mysql_secure_installation
    ```

    Example:

    ```shell
    NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
        SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

    In order to log into MariaDB to secure it, we'll need the current
    password for the root user. If you've just installed MariaDB, and
    haven't set the root password yet, you should just press enter here.

    Enter current password for root (enter for none):

    # Press Enter here because the database is being initialized.

    OK, successfully used password, moving on...

    Setting the root password or using the unix_socket ensures that nobody
    can log into the MariaDB root user without the proper authorisation.

    You already have your root account protected, so you can safely answer 'n'.

    # Enter N as prompted.

    Switch to unix_socket authentication [Y/n] N
    Enabled successfully!
    Reloading privilege tables..
    ... Success!


    You already have your root account protected, so you can safely answer 'n'.

    # Enter Y to change the password.

    Change the root password? [Y/n] Y
    New password:
    Re-enter new password:
    Password updated successfully!
    Reloading privilege tables..
    ... Success!


    By default, a MariaDB installation has an anonymous user, allowing anyone
    to log into MariaDB without having to have a user account created for
    them.  This is intended only for testing, and to make the installation
    go a bit smoother.  You should remove them before moving into a
    production environment.

    # Enter Y to remove anonymous users.

    Remove anonymous users? [Y/n] Y
    ... Success!

    Normally, root should only be allowed to connect from 'localhost'.  This
    ensures that someone cannot guess at the root password from the network.

    # Enter Y to disallow remote root login.

    Disallow root login remotely? [Y/n] Y
    ... Success!

    By default, MariaDB comes with a database named 'test' that anyone can
    access.  This is also intended only for testing, and should be removed
    before moving into a production environment.

    # Enter Y to remove the test database.

    Remove test database and access to it? [Y/n] Y
    - Dropping test database...
    ... Success!
    - Removing privileges on test database...
    ... Success!

    Reloading the privilege tables will ensure that all changes made so far
    will take effect immediately.

    # Enter Y to reload the privilege tables.

    Reload privilege tables now? [Y/n] Y
    ... Success!

    Cleaning up...

    All done!  If you've completed all of the above steps, your MariaDB
    installation should now be secure.
    ```

    ***Note***

    **The new root password is the `MYSQL_ROOT_PASS` value defined in the conventions (example: RootMysql2026). All subsequent database initialization uses this password.**

3. Verify that you can log in to MariaDB with the password set in step 2.

    ```shell
    mysql -uroot -p
    ```

#### Install the Message Queue

Install the message queue on the controller node. RabbitMQ is recommended.

```shell
dnf install rabbitmq-server
systemctl enable --now rabbitmq-server
rabbitmqctl add_user openstack RABBIT_PASS
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### Install the Cache Service

Install the cache service on the controller node. Memcached is recommended.

```shell
dnf install memcached python3-memcached
# /etc/sysconfig/memcached: OPTIONS="-l 127.0.0.1,::1,controller"
systemctl enable --now memcached
```

### Deploy Services

#### Keystone

Keystone is the OpenStack identity service and the entry point for the entire OpenStack platform. It provides tenant isolation, user authentication, and service discovery, and is mandatory.

**Controller node**

1. Create the keystone database and grant permissions.

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE keystone;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
    MariaDB [(none)]> exit
    ```

    ***Note***

    **Replace `KEYSTONE_DBPASS` (example: Keystone2026) with the password for the Keystone database.**

2. Install the packages.

    ```shell
    dnf install openstack-keystone httpd mod_wsgi
    ```

3. Configure keystone.

    ```shell
    vim /etc/keystone/keystone.conf

    [database]
    connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

    [token]
    provider = fernet
    ```

    ***Explanation***

    The `[database]` section configures the database connection. The `[token]` section configures the token provider.

4. Synchronize the database.

    ```shell
    su -s /bin/sh -c "keystone-manage db_sync" keystone
    ```

5. Initialize the Fernet key repository.

    ```shell
    keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
    keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
    ```

6. Bootstrap the service.

    ```shell
    keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
      --bootstrap-admin-url http://controller:5000/v3/ \
      --bootstrap-internal-url http://controller:5000/v3/ \
      --bootstrap-public-url http://controller:5000/v3/ \
      --bootstrap-region-id RegionOne
    ```

    ***Note***

    **Replace `ADMIN_PASS` (example: Admin2026) with the password for the admin user.**

7. Configure the Apache HTTP server.

    ```shell
    # Modify /etc/httpd/conf/httpd.conf (add the line if it does not exist): ServerName controller
    ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
    systemctl enable --now httpd.service
    ```

8. Create the environment variable file.

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

9. Install the OpenStack command-line client.

    All subsequent services use the `openstack` command to create users, services, and endpoints. Install the command-line client first.

    ```shell
    dnf install python3-openstackclient
    ```

10. Create the domain, projects, users, and roles.

    ```shell
    source ~/.admin-openrc

    # Create the service project (the default domain is created during bootstrap).
    openstack project create --domain default --description "Service Project" service

    # Create the demo project, user, and role (optional).
    openstack project create --domain default --description "Demo Project" myproject
    openstack user create --domain default --password-prompt myuser
    openstack role create myrole
    openstack role add --project myproject --user myuser myrole
    ```

11. Verify the deployment.

    - Unset the temporary `OS_AUTH_URL` and `OS_PASSWORD` environment variables:

        ```shell
        source ~/.admin-openrc
        unset OS_AUTH_URL OS_PASSWORD
        ```

    - Request a token for the admin user:

        ```shell
        openstack --os-auth-url http://controller:5000/v3 \
          --os-project-domain-name Default --os-user-domain-name Default \
          --os-project-name admin --os-username admin token issue
        ```

    - Request a token for the myuser user:

        ```shell
        openstack --os-auth-url http://controller:5000/v3 \
          --os-project-domain-name Default --os-user-domain-name Default \
          --os-project-name myproject --os-username myuser token issue
        ```

#### Glance

Glance is the OpenStack image service. It uploads and downloads virtual machine and bare metal images, and is mandatory.

**Controller node**

1. Create the glance database and grant permissions.

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE glance;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
    MariaDB [(none)]> exit
    ```

    ***Note:***

    **Replace `GLANCE_DBPASS` (example: Glance2026) with the password for the glance database.**

2. Initialize the glance resource objects.

    - Import the environment variables:

        ```shell
        source ~/.admin-openrc
        ```

    - Create the glance user. The command prompts for a password; enter the `GLANCE_PASS` value (example: Glance2026):

        ```shell
        openstack user create --domain default --password-prompt glance
        User Password:
        Repeat User Password:
        ```

    - Add the glance user to the service project and assign the admin role:

        ```shell
        openstack role add --project service --user glance admin
        ```

    - Create the glance service entity:

        ```shell
        openstack service create --name glance --description "OpenStack Image" image
        ```

    - Create the glance API endpoints:

        ```shell
        openstack endpoint create --region RegionOne image public http://controller:9292
        openstack endpoint create --region RegionOne image internal http://controller:9292
        openstack endpoint create --region RegionOne image admin http://controller:9292
        ```

3. Install the packages.

    ```shell
    dnf install openstack-glance
    ```

4. Modify the glance configuration file.

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

    The `[database]` section configures the database connection.

    The `[keystone_authtoken]` and `[paste_deploy]` sections configure the identity service endpoint.

    The `[glance_store]` section configures local file system storage and the image file location.

    ***Important note***

    - **`[paste_deploy] flavor = keystone` is mandatory.** Without it, Glance uses `unauthenticated-context` (no token validation), and modification operations such as `openstack image set` return 403 (the modify_image policy requires a role).
    - **Load the metadata definitions** (otherwise the metadefs API returns 404):

    ```shell
    su -s /bin/sh -c "glance-manage db_load_metadefs" glance
    ```

5. Synchronize the database.

    ```shell
    su -s /bin/sh -c "glance-manage db_sync" glance
    ```

6. Start the service.

    ```shell
    systemctl enable openstack-glance-api.service
    systemctl start openstack-glance-api.service
    ```

7. Verify the deployment.

    - Import the environment variables:

        ```shell
        source ~/.admin-openrc
        ```

    - Download an image:

        ```shell
        x86 image:
        wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

        arm image:
        wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-aarch64-disk.img
        ```

        ***Note***

        **If you are using a Kunpeng architecture environment, download the aarch64 image.**

    - Upload the image to the Image service:

        ```shell
        openstack image create --disk-format qcow2 --container-format bare \
                            --file cirros-0.6.2-x86_64-disk.img --public cirros
        ```

    - Confirm the image upload and verify its properties:

        ```shell
        openstack image list
        ```

#### Placement

Placement is the OpenStack resource scheduling component. It is generally not user-facing and is called by components such as Nova. It is installed on the controller node.

Before installing and configuring Placement, create the database, service credentials, and API endpoints.

1. Create the database.

    - Log in to the database service as root:

        ```shell
        mysql -u root -p
        ```

    - Create the placement database:

        ```sql
        MariaDB [(none)]> CREATE DATABASE placement;
        ```

    - Grant database access:

        ```sql
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
          IDENTIFIED BY 'PLACEMENT_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
          IDENTIFIED BY 'PLACEMENT_DBPASS';
        ```

        Replace `PLACEMENT_DBPASS` (example: Placement2026) with the password for the placement database.

    - Exit the database client:

        ```shell
        exit
        ```

2. Configure the user and endpoints.

    - Source the admin credentials to obtain admin command-line permissions:

        ```shell
        source ~/.admin-openrc
        ```

    - Create the placement user and set the password (`PLACEMENT_PASS`, example: Placement2026):

        ```shell
        openstack user create --domain default --password-prompt placement

        User Password:
        Repeat User Password:
        ```

    - Add the placement user to the service project and assign the admin role:

        ```shell
        openstack role add --project service --user placement admin
        ```

    - Create the placement service entity:

        ```shell
        openstack service create --name placement \
          --description "Placement API" placement
        ```

    - Create the Placement API endpoints:

        ```shell
        openstack endpoint create --region RegionOne \
          placement public http://controller:8778
        openstack endpoint create --region RegionOne \
          placement internal http://controller:8778
        openstack endpoint create --region RegionOne \
          placement admin http://controller:8778
        ```

3. Install and configure the component.

    - Install the packages:

        ```shell
        dnf install openstack-placement-api
        ```

    - Edit `/etc/placement/placement.conf` and complete the following:

        - In the `[placement_database]` section, configure the database connection:

            ```ini
            [placement_database]
            connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
            ```

            Replace `PLACEMENT_DBPASS` with the placement database password.

        - In the `[api]` and `[keystone_authtoken]` sections, configure the identity service endpoint:

            ```ini
            [api]
            auth_strategy = keystone

            [keystone_authtoken]
            auth_url = http://controller:5000/v3
            memcached_servers = controller:11211
            auth_type = password
            project_domain_name = Default
            user_domain_name = Default
            project_name = service
            username = placement
            password = PLACEMENT_PASS
            ```

            Replace `PLACEMENT_PASS` with the placement user password.

    - Synchronize the database to populate the Placement database:

        ```shell
        su -s /bin/sh -c "placement-manage db sync" placement
        ```

4. Start the service.

    Restart the httpd service:

    ```shell
    systemctl restart httpd
    ```

5. Verify the deployment.

    - Source the admin credentials to obtain admin command-line permissions:

        ```shell
        source ~/.admin-openrc
        ```

    - Run the status check:

        ```shell
        placement-status upgrade check
        ```

        ```text
        +----------------------------------------------------------------------+
        | Upgrade Check Results                                                |
        +----------------------------------------------------------------------+
        | Check: Missing Root Provider IDs                                     |
        | Result: Success                                                      |
        | Details: None                                                        |
        +----------------------------------------------------------------------+
        | Check: Incomplete Consumers                                          |
        | Result: Success                                                      |
        | Details: None                                                        |
        +----------------------------------------------------------------------+
        | Check: Policy File JSON to YAML Migration                            |
        | Result: Failure                                                      |
        | Details: Your policy file is JSON-formatted which is deprecated. You |
        |   need to switch to YAML-formatted file. Use the                     |
        |   ``oslopolicy-convert-json-to-yaml`` tool to convert the            |
        |   existing JSON-formatted files to YAML in a backwards-              |
        |   compatible manner: https://docs.openstack.org/oslo.policy/         |
        |   latest/cli/oslopolicy-convert-json-to-yaml.html.                   |
        +----------------------------------------------------------------------+
        ```

        The `Policy File JSON to YAML Migration` check returns Failure because JSON-formatted policy files in Placement have been `deprecated` since the Wallaby release. You can use the [oslopolicy-convert-json-to-yaml](https://docs.openstack.org/oslo.policy/latest/cli/oslopolicy-convert-json-to-yaml.html) tool as suggested to convert the existing JSON policy file to YAML.

        ```shell
        oslopolicy-convert-json-to-yaml  --namespace placement \
          --policy-file /etc/placement/policy.json \
          --output-file /etc/placement/policy.yaml
        mv /etc/placement/policy.json{,.bak}
        ```

        Note: This issue can be ignored in the current environment and does not affect operation.

    - Run commands against the placement API:

        - Install the osc-placement plugin:

            ```shell
            dnf install python3-osc-placement
            ```

        - List the available resource classes and traits:

            ```shell
            openstack --os-placement-api-version 1.2 resource class list --sort-column name
            +----------------------------+
            | name                       |
            +----------------------------+
            | DISK_GB                    |
            | FPGA                       |
            | ...                        |

            openstack --os-placement-api-version 1.6 trait list --sort-column name
            +---------------------------------------+
            | name                                  |
            +---------------------------------------+
            | COMPUTE_ACCELERATORS                  |
            | COMPUTE_ARCH_AARCH64                  |
            | ...                                   |
            ```

#### Nova

Nova is the OpenStack compute service. It creates and provisions virtual machines.

**Controller node**

Perform the following operations on the controller node.

1. Create the databases.

    - Log in to the database service as root:

        ```shell
        mysql -u root -p
        ```

    - Create the `nova_api`, `nova`, and `nova_cell0` databases:

        ```sql
        MariaDB [(none)]> CREATE DATABASE nova_api;
        MariaDB [(none)]> CREATE DATABASE nova;
        MariaDB [(none)]> CREATE DATABASE nova_cell0;
        ```

    - Grant database access:

        ```sql
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
        ```

        Replace `NOVA_DBPASS` (example: Nova2026) with the password for the nova databases.

    - Exit the database client:

        ```sql
        exit
        ```

2. Configure the user and endpoints.

    - Source the admin credentials to obtain admin command-line permissions:

        ```shell
        source ~/.admin-openrc
        ```

    - Create the nova user and set the password (`NOVA_PASS`, example: Nova2026):

        ```shell
        openstack user create --domain default --password-prompt nova

        User Password:
        Repeat User Password:
        ```

    - Add the nova user to the service project and assign the admin role:

        ```shell
        openstack role add --project service --user nova admin
        ```

    - Create the nova service entity:

        ```shell
        openstack service create --name nova \
          --description "OpenStack Compute" compute
        ```

    - Create the Nova API endpoints:

        ```shell
        openstack endpoint create --region RegionOne \
          compute public http://controller:8774/v2.1
        openstack endpoint create --region RegionOne \
          compute internal http://controller:8774/v2.1
        openstack endpoint create --region RegionOne \
          compute admin http://controller:8774/v2.1
        ```

3. Install and configure the component.

    - Install the packages:

        ```shell
        dnf install openstack-nova-api openstack-nova-conductor \
          openstack-nova-novncproxy openstack-nova-scheduler
        ```

    - Edit `/etc/nova/nova.conf` and complete the following:

        - In the `[DEFAULT]` section, enable the compute and metadata APIs, configure the RabbitMQ message queue endpoint, set `my_ip` to the controller management IP, and explicitly define `log_dir`:

            ```ini
            [DEFAULT]
            enabled_apis = osapi_compute,metadata
            transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
            my_ip = 192.168.0.2
            log_dir = /var/log/nova
            state_path = /var/lib/nova
            ```

        - In the `[api_database]` and `[database]` sections, configure the database connections:

            ```ini
            [api_database]
            connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

            [database]
            connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
            ```

        - In the `[api]` and `[keystone_authtoken]` sections, configure the identity service endpoint:

            ```ini
            [api]
            auth_strategy = keystone

            [keystone_authtoken]
            auth_url = http://controller:5000/v3
            memcached_servers = controller:11211
            auth_type = password
            project_domain_name = Default
            user_domain_name = Default
            project_name = service
            username = nova
            password = NOVA_PASS
            ```

        - In the `[vnc]` section, enable and configure the remote console endpoint:

            ```ini
            [vnc]
            enabled = true
            server_listen = $my_ip
            server_proxyclient_address = $my_ip
            ```

        - In the `[glance]` section, configure the image service API address:

            ```ini
            [glance]
            api_servers = http://controller:9292
            ```

        - In the `[oslo_concurrency]` section, configure the lock path:

            ```ini
            [oslo_concurrency]
            lock_path = /var/lib/nova/tmp
            ```

        - In the `[placement]` section, configure the placement service endpoint:

            ```ini
            [placement]
            region_name = RegionOne
            project_domain_name = Default
            project_name = service
            auth_type = password
            user_domain_name = Default
            auth_url = http://controller:5000/v3
            username = placement
            password = PLACEMENT_PASS
            ```

        - In the `[neutron]` section, configure the neutron service endpoint:

            ```ini
            [neutron]
            auth_url = http://controller:5000
            auth_type = password
            project_domain_name = Default
            user_domain_name = Default
            region_name = RegionOne
            project_name = service
            username = neutron
            password = NEUTRON_PASS
            service_metadata_proxy = true
            metadata_proxy_shared_secret = METADATA_SECRET
            ```

        - In the `[service_user]` and `[keystone_authtoken]` sections, configure service-to-service authentication:

            ```ini
            [service_user]
            send_service_user_token = true
            username = nova
            password = NOVA_PASS
            user_domain_name = Default
            project_name = service
            project_domain_name = Default
            auth_type = password
            auth_url = http://controller:5000/v3

            [keystone_authtoken]
            service_token_roles = service
            service_token_roles_required = true
            ```

    - Synchronize the databases:

        - Synchronize the nova-api database:

            ```shell
            su -s /bin/sh -c "nova-manage api_db sync" nova
            ```

        - Register the cell0 database:

            ```shell
            su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
            ```

        - Create cell1:

            ```shell
            su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
            ```

        - Synchronize the nova database:

            ```shell
            su -s /bin/sh -c "nova-manage db sync" nova
            ```

        - Verify that cell0 and cell1 are registered correctly:

            ```shell
            su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
            ```

4. Grant the service role to the nova, cinder, and neutron users (required for service token validation).

    ```shell
    source ~/.admin-openrc
    openstack role create service
    openstack role add --user nova --project service service
    openstack role add --user cinder --project service service
    openstack role add --user neutron --project service service
    ```

    ***Important notes***

    - **The `[neutron]` authentication section is mandatory.** Without it, creating a VM reports `Unknown auth type: None` (nova needs the complete authentication section to call neutron).
    - **`[service_user]` and `service_token_roles = service` are mandatory.** Nova must carry a service token when calling neutron/cinder. `service_token_roles` must be `service` (and the nova user must be granted the service role); otherwise detaching a volume makes cinder report 409 ConflictNovaUsingAttachment, and updating a port makes neutron return policy 403.
    - **`[vnc] server_proxyclient_address` must not be empty** (use `$my_ip`). Otherwise nova fails to start with `ConfigFileValueError`.

5. Start the services.

    ```shell
    systemctl enable openstack-nova-api.service openstack-nova-scheduler.service \
      openstack-nova-conductor.service openstack-nova-novncproxy.service
    systemctl start openstack-nova-api.service openstack-nova-scheduler.service \
      openstack-nova-conductor.service openstack-nova-novncproxy.service
    ```

**Compute node**

1. Install the prerequisites.

    ```shell
    dnf install hostname openstack-nova-compute
    ```

    ***Note***

    - **The `hostname` command must be installed.** The `$my_ip` resolution in the nova configuration depends on the hostname command, which may not be installed on a minimal openEuler image.
    - **libvirtd must be started.** nova-compute depends on `/var/run/libvirt/libvirt-sock`. If libvirtd is not started, the compute node cannot register.

    ```shell
    systemctl enable --now libvirtd
    ```

2. Edit `/etc/nova/nova.conf`.

    ```ini
    [DEFAULT]
    enabled_apis = osapi_compute,metadata
    transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
    my_ip = 192.168.0.3
    compute_driver = libvirt.LibvirtDriver
    instances_path = /var/lib/nova/instances
    log_dir = /var/log/nova

    [api]
    auth_strategy = keystone

    [keystone_authtoken]
    auth_url = http://controller:5000/v3
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
    novncproxy_base_url = http://controller:6080/vnc_auto.html

    [glance]
    api_servers = http://controller:9292

    [oslo_concurrency]
    lock_path = /var/lib/nova/tmp

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
    project_domain_name = Default
    user_domain_name = Default
    region_name = RegionOne
    project_name = service
    username = neutron
    password = NEUTRON_PASS
    ```

    ***Note***

    - **The `[neutron]` authentication section is also mandatory on the compute node** (the compute node calls neutron to bind ports when creating VMs).

3. Check whether the compute node supports hardware acceleration for virtual machines (x86_64).

    ```shell
    egrep -c '(vmx|svm)' /proc/cpuinfo
    ```

    If the command returns 0, hardware acceleration is not supported and libvirt must be configured to use QEMU. If it returns 1 or greater, no additional configuration is required.

    ```ini
    [libvirt]
    virt_type = qemu
    ```

4. Check whether the compute node supports hardware acceleration for virtual machines (arm64).

    ```shell
    virt-host-validate
    ```

    If it reports FAIL (`/dev/kvm` does not exist), also configure `[libvirt] virt_type = qemu`.

5. Configure QEMU (arm64 only).

    Perform this step only when the processor is arm64.

    - Edit `/etc/libvirt/qemu.conf`:

        ```ini
        nvram = ["/usr/share/AAVMF/AAVMF_CODE.fd: \
                 /usr/share/AAVMF/AAVMF_VARS.fd", \
                 "/usr/share/edk2/aarch64/QEMU_EFI-pflash.raw: \
                 /usr/share/edk2/aarch64/vars-template-pflash.raw"]
        ```

    - Edit `/etc/qemu/firmware/edk2-aarch64.json`:

        ```json
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
        ```

6. Start the services.

    ```shell
    systemctl enable libvirtd.service openstack-nova-compute.service
    systemctl start libvirtd.service openstack-nova-compute.service
    ```

**Controller node (finalization)**

1. Add the compute node to the OpenStack cluster.

    - Source the admin credentials to obtain admin command-line permissions:

        ```shell
        source ~/.admin-openrc
        ```

    - Confirm that nova-compute has been discovered in the database:

        ```shell
        openstack compute service list --service nova-compute
        ```

    - Discover the compute node and add it to the cell database:

        ```shell
        su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
        ```

2. Verify the deployment.

    - List the service components and verify that each process started and registered successfully:

        ```shell
        openstack compute service list
        ```

    - List the API endpoints in the identity service and verify connectivity:

        ```shell
        openstack catalog list
        ```

    - List the images in the image service and verify connectivity:

        ```shell
        openstack image list
        ```

    - Check whether the cells are working and whether other prerequisites are met:

        ```shell
        nova-status upgrade check
        ```

#### Neutron

Neutron is the OpenStack networking service. It provides virtual switches, IP routing, DHCP, and related functions.

**Controller node**

1. Create the database, service credentials, and API endpoints.

    - Create the database:

        ```sql
        mysql -u root -p

        MariaDB [(none)]> CREATE DATABASE neutron;
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
        MariaDB [(none)]> exit;
        ```

        Replace `NEUTRON_DBPASS` (example: Neutron2026) with the password for the neutron database.

    - Create the user and service, and record the password entered for the neutron user. It is used as `NEUTRON_PASS` (example: Neutron2026):

        ```shell
        source ~/.admin-openrc
        openstack user create --domain default --password-prompt neutron
        openstack role add --project service --user neutron admin
        openstack service create --name neutron --description "OpenStack Networking" network
        ```

    - Deploy the Neutron API endpoints:

        ```shell
        openstack endpoint create --region RegionOne network public http://controller:9696
        openstack endpoint create --region RegionOne network internal http://controller:9696
        openstack endpoint create --region RegionOne network admin http://controller:9696
        ```

2. Install the packages.

    ```shell
    dnf install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch \
      openstack-neutron-rpc-server ebtables ipset
    ```

3. Configure Neutron.

    - Modify `/etc/neutron/neutron.conf`:

        ```ini
        [database]
        connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

        [DEFAULT]
        core_plugin = ml2
        service_plugins = router,qos,neutron.services.placement_report.plugin.PlacementReportPlugin
        allow_overlapping_ips = true
        transport_url = rabbit://openstack:RABBIT_PASS@controller
        auth_strategy = keystone
        notify_nova_on_port_status_changes = true
        notify_nova_on_port_data_changes = true

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
        service_token_roles = service
        service_token_roles_required = true

        [nova]
        auth_url = http://controller:5000
        auth_type = password
        project_domain_name = Default
        user_domain_name = Default
        region_name = RegionOne
        project_name = service
        username = nova
        password = NOVA_PASS

        [oslo_concurrency]
        lock_path = /var/lib/neutron/tmp

        [placement]
        auth_type = password
        auth_url = http://controller:5000/v3
        username = placement
        password = PLACEMENT_PASS
        user_domain_name = Default
        project_name = service
        project_domain_name = Default
        region_name = RegionOne
        ```

        ***Important notes***

        - **The placement_report entry in `service_plugins` must use the full class path** (`neutron.services.placement_report.plugin.PlacementReportPlugin`). The plugin is not registered as an entry point; using `placement_report` directly causes rpc-server to fail with `ImportError: Class not found`. placement_report synchronizes the bandwidth reported by the OVS agent to Placement (required by the QoS minimum bandwidth tests).
        - **The `[placement]` authentication section is mandatory.** Otherwise neutron cannot call the placement API and the resource provider is not created.

    - Modify `/etc/neutron/plugins/ml2/ml2_conf.ini`:

        ```ini
        [ml2]
        type_drivers = flat,vlan,vxlan
        tenant_network_types = vxlan
        mechanism_drivers = openvswitch,l2population
        extension_drivers = port_security,qos

        [ml2_type_flat]
        flat_networks = provider

        [ml2_type_vlan]
        network_vlan_ranges = provider:100:4000

        [ml2_type_vxlan]
        vni_ranges = 1:1000

        [securitygroup]
        enable_ipset = true
        ```

        ***Note***

        - **`[ml2_type_vlan] network_vlan_ranges` must be configured.** Otherwise creating a VLAN provider network returns 400 (used by tests such as QoS min_bw).

    - Modify `/etc/neutron/plugins/ml2/openvswitch_agent.ini`:

        ```ini
        [ovs]
        bridge_mappings = provider:br-provider
        local_ip = OVERLAY_INTERFACE_IP_ADDRESS
        resource_provider_bandwidths = br-provider:1000000:1000000
        resource_provider_packet_processing_without_direction = br-provider:100000000

        [agent]
        tunnel_types = vxlan

        [securitygroup]
        enable_security_group = true
        firewall_driver = openvswitch

        [placement]
        auth_type = password
        auth_url = http://controller:5000/v3
        username = placement
        password = PLACEMENT_PASS
        user_domain_name = Default
        project_name = service
        project_domain_name = Default
        region_name = RegionOne
        ```

        ***Important notes***

        - **Use `firewall_driver = openvswitch`** (OVS flow-based implementation). Do not use iptables-based drivers: iptables rules use the port UUID (36 characters) as the interface name, which exceeds IFNAMSIZ (15), causing iptables-restore to fail and security groups to stop working. Do not change it to NoopFirewallDriver either (security group tests will fail).
        - **The key in `resource_provider_bandwidths` is the bridge name (br-provider), not the physical network name**, and the option belongs in the **[ovs]** section (it has no effect in the [agent] section). Without it, the QoS minimum bandwidth tests have no allocation candidates.
        - **The `[placement]` authentication section is mandatory.** The OVS agent needs it to synchronize resource providers through placement_report.

    - Modify `/etc/neutron/l3_agent.ini` and `/etc/neutron/dhcp_agent.ini`:

        ```ini
        [DEFAULT]
        interface_driver = openvswitch
        ```

        ***Note***

        - **`interface_driver` must use the entry point name `openvswitch`**, not the full class name `neutron.agent.linux.interface.OVSInterfaceDriver` (stevedore fails to load it with `Could not load`, causing DHCP interface creation to fail and VMs to receive no IP address).

    - Append the following to `/etc/neutron/dhcp_agent.ini`:

        ```ini
        [DEFAULT]
        dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
        enable_isolated_metadata = true
        ```

    - Modify `/etc/neutron/metadata_agent.ini`:

        ```ini
        [DEFAULT]
        nova_metadata_host = 127.0.0.1
        metadata_proxy_shared_secret = METADATA_SECRET
        ```

        ***Note***

        - **`nova_metadata_host = 127.0.0.1`**: nova-metadata (uwsgi) listens only on 127.0.0.1:8775. Configuring it as a hostname (controller) causes the metadata agent to fail forwarding with Connection refused, and VMs cannot obtain keys (SSH is rejected).

4. Create the plugin.ini symbolic link (important).

    ```shell
    ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
    ```

    ***Important note***

    - **This symlink is mandatory.** The ExecStart of the Gazpacho neutron-rpc-server references `/etc/neutron/plugin.ini`. By default, that file is empty, so rpc-server loads no mechanism drivers. As a result, DHCP/router port binding fails (binding_failed), VM DHCP does not work, and floating IPs do not work. After linking to ml2_conf.ini, rpc-server can load the openvswitch mechanism driver.

5. Synchronize the database and start the services.

    ```shell
    su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
      --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
    systemctl restart openstack-nova-api
    systemctl enable --now neutron-server.service neutron-rpc-server.service \
      neutron-openvswitch-agent.service neutron-dhcp-agent.service \
      neutron-metadata-agent.service neutron-l3-agent.service
    ```

6. Create the provider bridge.

    ```shell
    ovs-vsctl add-br br-provider
    ovs-vsctl add-port br-provider PROVIDER_INTERFACE_NAME
    ```

    ***Note***

    - **br-provider is not created automatically.** You must run `add-br` manually. Provider networks (flat/VLAN) depend on it.
    - In the three-node environment, the provider NIC is connected directly to br-provider (the ext0 internal port workaround used in the single-node environment is not needed). The provider NICs on the controller and compute nodes only need to be connected to the same Layer 2 network.

7. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack network agent list
    # Each agent should show Alive.
    ```

**Compute node**

1. Install the packages.

    ```shell
    dnf install -y openstack-neutron-openvswitch ebtables ipset
    ```

2. Configure Neutron.

    - Modify `/etc/neutron/neutron.conf`:

        ```ini
        [DEFAULT]
        transport_url = rabbit://openstack:RABBIT_PASS@controller
        auth_strategy = keystone

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
        service_token_roles = service
        service_token_roles_required = true

        [oslo_concurrency]
        lock_path = /var/lib/neutron/tmp
        ```

    - Modify `/etc/neutron/plugins/ml2/openvswitch_agent.ini` (same as the controller node):

        ```ini
        [ovs]
        bridge_mappings = provider:br-provider
        local_ip = OVERLAY_INTERFACE_IP_ADDRESS
        resource_provider_bandwidths = br-provider:1000000:1000000
        resource_provider_packet_processing_without_direction = br-provider:100000000

        [agent]
        tunnel_types = vxlan

        [securitygroup]
        enable_security_group = true
        firewall_driver = openvswitch

        [placement]
        auth_type = password
        auth_url = http://controller:5000/v3
        username = placement
        password = PLACEMENT_PASS
        user_domain_name = Default
        project_name = service
        project_domain_name = Default
        region_name = RegionOne
        ```

3. Create the provider bridge (same as the controller node).

    ```shell
    ovs-vsctl add-br br-provider
    ovs-vsctl add-port br-provider PROVIDER_INTERFACE_NAME
    ```

4. Restart nova-compute and start the OVS agent.

    ```shell
    systemctl restart openstack-nova-compute.service
    systemctl enable --now neutron-openvswitch-agent.service
    ```

5. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack network agent list
    # Two OVS agents should be listed (controller + compute), both Alive.
    ```

#### Cinder

Cinder is the OpenStack block storage service. It creates, provisions, and backs up block devices.

**Controller node**

1. Initialize the database.

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE cinder;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
    MariaDB [(none)]> exit
    ```

2. Initialize the Keystone resource objects.

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt cinder    # Enter CINDER_PASS
    openstack role add --project service --user cinder admin
    openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
    openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
    openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
    openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
    ```

3. Install the packages.

    ```shell
    dnf install openstack-cinder-api openstack-cinder-scheduler
    ```

4. Modify the cinder configuration file `/etc/cinder/cinder.conf`.

    ```ini
    [DEFAULT]
    transport_url = rabbit://openstack:RABBIT_PASS@controller
    auth_strategy = keystone
    my_ip = 192.168.0.2

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
    service_token_roles = service
    service_token_roles_required = true

    [oslo_concurrency]
    lock_path = /var/lib/cinder/tmp

    [service_user]
    send_service_user_token = true
    username = cinder
    password = CINDER_PASS
    user_domain_name = Default
    project_name = service
    project_domain_name = Default
    auth_type = password
    auth_url = http://controller:5000/v3

    [nova]
    auth_type = password
    auth_url = http://controller:5000/v3
    username = cinder
    password = CINDER_PASS
    user_domain_name = Default
    project_name = service
    project_domain_name = Default
    region_name = RegionOne
    ```

    ***Important notes***

    - **`[oslo_concurrency] lock_path` is mandatory.** Without it, the LIO `create_iscsi_target` lock in the volume attach flow reports `RequiredOptError: lock_path`, and the volume remains in reserved/attaching state.
    - **`[service_user]` and `service_token_roles = service` are mandatory.** Cinder must carry a service token when calling nova (for example, update_server_volume after attaching a volume). The cinder user must also be granted the service role (see step 4 in the Nova section).
    - **The `[nova]` authentication section is mandatory.** When the volume migrate flow calls nova `update_server_volume`, the default token plugin lacks the project domain and reports `BadRequest: Expecting to find domain in project`.

5. Synchronize the database.

    ```shell
    su -s /bin/sh -c "cinder-manage db sync" cinder
    ```

6. Modify the nova configuration `/etc/nova/nova.conf`.

    ```ini
    [cinder]
    os_region_name = RegionOne
    ```

7. Start the services.

    ```shell
    systemctl restart openstack-nova-api
    systemctl enable --now openstack-cinder-api openstack-cinder-scheduler
    ```

**Storage node**

Prepare at least one disk on the storage node in advance as the Cinder storage backend. The following assumes that an unused disk named `/dev/sdb` exists on the storage node. Replace the device name according to your actual environment.

1. Install the packages.

    ```shell
    dnf install lvm2 device-mapper-persistent-data scsi-target-utils targetcli \
      python3-rtslib rpcbind nfs-utils openstack-cinder-volume
    ```

    ***Important notes***

    - **The missing rtslib_fb module must be fixed** (targetcli reports `No module named rtslib_fb.utils`). The python3-rtslib package in the Gazpacho repository provides the `rtslib` package, while targetcli 2.1.56 references `rtslib_fb`. The `rtslib_fb.py` file under `/usr/lib/python3.11/site-packages/` is a leftover single file. Replace it with a link to the rtslib package; otherwise the LIO target cannot be created and volume attachment fails:

    ```shell
    cd /usr/lib/python3.11/site-packages/
    mv rtslib_fb.py rtslib_fb.py.bak
    ln -s rtslib rtslib_fb
    targetcli ls    # The configuration tree should be displayed normally.
    ```

    - **The kernel target modules must be loaded** (required by LIO, and the configuration must persist):

    ```shell
    cat > /etc/modules-load.d/cinder.conf << EOF
    target_core_mod
    iscsi_target_mod
    target_core_file
    target_core_iblock
    EOF
    modprobe target_core_mod iscsi_target_mod target_core_file target_core_iblock
    ```

    - **iscsid must be started** (nova os-brick logs in to the iSCSI target through the iSCSI client service; if it is not started, volume attachment remains in attaching state):

    ```shell
    systemctl enable --now iscsid
    ```

2. Configure the LVM volume group.

    ```shell
    pvcreate /dev/sdb
    vgcreate cinder-volumes /dev/sdb
    ```

3. Modify the cinder configuration `/etc/cinder/cinder.conf`.

    ```ini
    [DEFAULT]
    transport_url = rabbit://openstack:RABBIT_PASS@controller
    auth_strategy = keystone
    my_ip = 192.168.0.4
    enabled_backends = lvm
    glance_api_servers = http://controller:9292

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
    service_token_roles = service
    service_token_roles_required = true

    [database]
    connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

    [lvm]
    volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
    volume_group = cinder-volumes
    target_protocol = iscsi
    target_helper = lioadm
    volume_backend_name = lvm

    [oslo_concurrency]
    lock_path = /var/lib/cinder/tmp

    [service_user]
    send_service_user_token = true
    username = cinder
    password = CINDER_PASS
    user_domain_name = Default
    project_name = service
    project_domain_name = Default
    auth_type = password
    auth_url = http://controller:5000/v3
    ```

    ***Important notes***

    - **`volume_backend_name = lvm` must be configured explicitly.** By default, the backend reports the name LVM, which does not match the volume type `volume_backend_name=lvm`. The scheduler then reports `No valid backend was found` and volume creation fails.
    - **`lock_path`, the kernel modules, iscsid, and rtslib_fb are all mandatory.** Missing any one of them causes volume attach/create failures (see T9/T16 in the troubleshooting quick reference).

4. Start the service.

    ```shell
    systemctl enable --now openstack-cinder-volume.service
    ```

5. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack volume service list
    # cinder-volume should show controller@lvm... with state up.
    openstack volume type create --property volume_backend_name=lvm lvm
    openstack volume create --type lvm --size 1 testvol
    openstack volume list
    ```

    ***Extension (optional): multiple backends***

    If you need multiple backends (required for volume_migrate/retype tests), prepare another disk `/dev/sdc` on the storage node and add a second backend:

    ```shell
    pvcreate /dev/sdc
    vgcreate cinder-volumes-2 /dev/sdc
    ```

    Change `enabled_backends = lvm,lvm2` in `/etc/cinder/cinder.conf` and add:

    ```ini
    [lvm2]
    volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
    volume_group = cinder-volumes-2
    target_protocol = iscsi
    target_helper = lioadm
    volume_backend_name = lvm2
    lvm_type = default
    ```

    ***Note***

    - **Both `[lvm2] volume_backend_name` and `lvm_type = default` must be configured.** Without backend_name, scheduling fails. Without lvm_type, cinder detects a thin pool (the new volume group has none) and volume creation ends in ERROR.
    - Restart `openstack-cinder-volume` and verify that `openstack volume service list` shows two volume services.

#### Horizon

Horizon is the OpenStack dashboard. It provides a web management interface.

**Controller node**

1. Install the packages.

    ```shell
    dnf install openstack-dashboard python3-rjsmin
    ```

    ***Important note***

    - **`python3-rjsmin` must be installed together with the dashboard.** Without it, Horizon template rendering reports `rjsmin.jsmin couldn't be imported`, and httpd fails to start. Because keystone also runs on httpd, the entire authentication service becomes unavailable.

2. Modify `/etc/httpd/conf.d/openstack-dashboard.conf`.

    ```shell
    # Change ServerName to controller. The original value is openstack-dashboard.example.com,
    # which does not match and causes 403.
    sed -i 's/ServerName .*/ServerName controller/' /etc/httpd/conf.d/openstack-dashboard.conf
    ```

3. Modify `/etc/openstack-dashboard/local_settings`.

    ```python
    OPENSTACK_HOST = "controller"
    ALLOWED_HOSTS = ['controller', '192.168.0.2']
    OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
    SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
    CACHES = {
        'default': {
            'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
            'LOCATION': 'controller:11211',
        }
    }
    OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
    OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
    OPENSTACK_KEYSTONE_DEFAULT_ROLE = "member"
    POLICY_FILES_PATH = "/etc/openstack-dashboard"

    OPENSTACK_API_VERSIONS = {
        "identity": 3,
        "image": 2,
        "volume": 3,
    }

    # Critical fixes. Without them, 400/403 errors occur, or a directory error occurs after login.
    CSRF_TRUSTED_ORIGINS = ['http://controller', 'http://192.168.0.2']
    LOGIN_URL = '/dashboard/auth/login/'
    LOGIN_REDIRECT_URL = '/dashboard/'
    LOGOUT_REDIRECT_URL = '/dashboard/auth/login/'
    ```

    ***Important notes***

    - **`ALLOWED_HOSTS` must include controller.** Otherwise accessing the dashboard returns 400.
    - **`CSRF_TRUSTED_ORIGINS` must be configured.** Otherwise the login form POST returns 403 (Django CSRF validation).
    - **`LOGIN_REDIRECT_URL = '/dashboard/'` and related settings must be configured.** Otherwise a successful login redirects to the root path `/` (no WSGI alias), causing a 403 directory error and failing the Tempest dashboard tests.
    - **Do not configure `WEBROOT`.** Configuring it causes the login page to fail with a 500 template error.

4. Start the service.

    ```shell
    systemctl restart httpd
    ```

5. Verify the deployment.

    ```shell
    curl -s -o /dev/null -w '%{http_code}\n' http://controller/dashboard/auth/login/
    # It should return 200. Log in at http://192.168.0.2/dashboard in a browser.
    ```

#### Swift

Swift is the OpenStack object storage service. It stores objects and containers.

**Controller node**

1. Initialize the Keystone resource objects.

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt swift    # Enter SWIFT_PASS
    openstack role add --project service --user swift admin
    openstack service create --name swift --description "OpenStack Object Storage" object-store
    openstack endpoint create --region RegionOne object-store public http://controller:8080/v1/AUTH_%\(project_id\)s
    openstack endpoint create --region RegionOne object-store internal http://controller:8080/v1/AUTH_%\(project_id\)s
    openstack endpoint create --region RegionOne object-store admin http://controller:8080/v1/AUTH_%\(project_id\)s
    ```

2. Install the packages.

    ```shell
    dnf install openstack-swift-proxy python3-swiftclient python3-keystoneclient \
      python3-keystonemiddleware memcached
    ```

3. Modify `/etc/swift/proxy-server.conf`.

    ```ini
    [DEFAULT]
    bind_port = 8080
    user = swift
    swift_dir = /etc/swift

    [pipeline:main]
    pipeline = catch_errors gatekeeper healthcheck proxy-logging cache listing_formats container_sync bulk tempurl ratelimit tempauth copy container-quotas account-quotas slo dlo versioned_writes symlink proxy-logging proxy-server

    [filter:keystoneauth]
    use = egg:swift#keystoneauth
    operator_roles = admin,user,member
    reseller_prefix = AUTH

    [filter:authtoken]
    paste.filter_factory = keystonemiddleware.auth_token:filter_factory
    www_authenticate_uri = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = swift
    password = SWIFT_PASS
    delay_auth_decision = True

    [filter:cache]
    use = egg:swift#memcache
    memcache_servers = controller:11211

    [app:proxy-server]
    use = egg:swift#proxy
    allow_account_management = true
    account_autocreate = true
    ```

    ***Important notes***

    - **`operator_roles = admin,user,member` must include member.** The Gazpacho package default contains only `admin,user`. Tempest dynamic users have only the member role, so access to Swift returns 403.
    - **The password in `[filter:authtoken]` must be the real SWIFT_PASS** (the template default is swift). Note that it belongs in the `[filter:authtoken]` section, not `[filter:keystoneauth]`.

4. Generate the rings and distribute them to each node.

    ```shell
    # Use the official ports for each storage type: account 6202 / container 6201 / object 6200.
    swift-ring-builder account.builder create 10 3 1
    swift-ring-builder container.builder create 10 3 1
    swift-ring-builder object.builder create 10 3 1
    # Example: add the corresponding device on storage node 192.168.0.4.
    swift-ring-builder account.builder add --region 1 --zone 1 --ip 192.168.0.4 --port 6202 --device sdb --weight 100
    swift-ring-builder container.builder add --region 1 --zone 1 --ip 192.168.0.4 --port 6201 --device sdb --weight 100
    swift-ring-builder object.builder add --region 1 --zone 1 --ip 192.168.0.4 --port 6200 --device sdb --weight 100
    swift-ring-builder account.builder rebalance
    swift-ring-builder container.builder rebalance
    swift-ring-builder object.builder rebalance
    # Distribute *.ring.gz and *.builder to /etc/swift/ on the storage node.
    scp /etc/swift/*.ring.gz /etc/swift/*.builder storage:/etc/swift/
    ```

    ***Important note***

    - **The ring and bind_port must correspond to the official ports (6202/6201/6200).** Do not use a single port for all services; otherwise container operations return 503.

5. Start the service.

    ```shell
    systemctl enable --now openstack-swift-proxy
    ```

**Storage node**

1. Install the required packages.

    ```shell
    dnf install openstack-swift-account openstack-swift-container openstack-swift-object
    dnf install xfsprogs rsync
    ```

2. Format the device `/dev/sdb` (add another disk `/dev/sdc` if disaster recovery is required) as XFS.

    ```shell
    mkfs.xfs /dev/sdb
    ```

3. Create the mount point directory structure.

    ```shell
    mkdir -p /srv/node/sdb
    ```

4. Find the UUID of the new partition.

    ```shell
    blkid
    ```

5. Edit `/etc/fstab` and add the following entry:

    ```bash
    UUID="<UUID-from-output-above>" /srv/node/sdb xfs noatime 0 2
    ```

6. Mount the device.

    ```shell
    mount /srv/node/sdb
    chown -R swift:swift /srv/node
    ```

    ***Note***

    **If disaster recovery is not required, create only one device and skip the rsync configuration below.**

7. (Optional) Create or edit `/etc/rsyncd.conf` with the following content:

    ```ini
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

    **Replace `MANAGEMENT_INTERFACE_IP_ADDRESS` with the management network IP address of the storage node.**

    Enable and start rsyncd:

    ```shell
    systemctl enable rsyncd.service
    systemctl start rsyncd.service
    ```

8. Modify the storage service configurations (`account-server.conf`, `container-server.conf`, and `object-server.conf`).

    ```ini
    [DEFAULT]
    bind_ip = 0.0.0.0
    bind_port = 6202   # account; container uses 6201; object uses 6200
    user = swift
    swift_dir = /etc/swift
    devices = /srv/node
    mount_check = false
    ```

    ***Important notes***

    - **`bind_ip = 0.0.0.0`**: the default is 127.0.0.1, and the proxy on the controller cannot connect (503 Connection refused).
    - **`devices = /srv/node` and the directory must exist** (`/srv/node/<device>`); otherwise 503 is returned because the device cannot be found.
    - **`mount_check = false`**: the default true checks whether the device is a mount point. If it is not mounted, 503 is returned.

9. Start the services.

    ```shell
    systemctl enable --now openstack-swift-account openstack-swift-container openstack-swift-object
    ```

10. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack container create test
    openstack object create test /etc/hosts
    openstack object list test
    # Success if no 503 is returned.
    ```

#### Heat

Heat is the OpenStack orchestration service. It orchestrates composite cloud applications based on descriptive templates and is also called the `Orchestration Service`. Heat services are generally installed on the `Controller` node.

**Controller node**

1. Create the **heat** database and grant the correct access permissions. Replace **HEAT_DBPASS** (example: Heat2026) with a suitable password.

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE heat;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' IDENTIFIED BY 'HEAT_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' IDENTIFIED BY 'HEAT_DBPASS';
    MariaDB [(none)]> exit;
    ```

2. Create the service credentials. Create the **heat** user and grant the **admin** role. The password entered when creating the user is `HEAT_PASS` (example: Heat2026).

    ```shell
    source ~/.admin-openrc

    openstack user create --domain default --password-prompt heat
    openstack role add --project service --user heat admin
    ```

3. Create the **heat** and **heat-cfn** services and their API endpoints.

    ```shell
    openstack service create --name heat --description "Orchestration" orchestration
    openstack service create --name heat-cfn --description "Orchestration" cloudformation
    openstack endpoint create --region RegionOne orchestration public http://controller:8004/v1/%\(tenant_id\)s
    openstack endpoint create --region RegionOne orchestration internal http://controller:8004/v1/%\(tenant_id\)s
    openstack endpoint create --region RegionOne orchestration admin http://controller:8004/v1/%\(tenant_id\)s
    openstack endpoint create --region RegionOne cloudformation public http://controller:8000/v1
    openstack endpoint create --region RegionOne cloudformation internal http://controller:8000/v1
    openstack endpoint create --region RegionOne cloudformation admin http://controller:8000/v1
    ```

4. Create the additional information required for stack management.

    Create the **heat** domain:

    ```shell
    openstack domain create --description "Stack projects and users" heat
    ```

    Create the **heat_domain_admin** user in the **heat** domain and record the password. It is used as `HEAT_DOMAIN_PASS` below:

    ```shell
    openstack user create --domain heat --password-prompt heat_domain_admin
    ```

    Grant the **admin** role to **heat_domain_admin**:

    ```shell
    openstack role add --domain heat --user-domain heat --user heat_domain_admin admin
    ```

    Create the **heat_stack_owner** role:

    ```shell
    openstack role create heat_stack_owner
    ```

    Create the **heat_stack_user** role:

    ```shell
    openstack role create heat_stack_user
    ```

5. Install the packages.

    ```shell
    dnf install openstack-heat-api openstack-heat-api-cfn openstack-heat-engine
    ```

6. Modify the configuration file `/etc/heat/heat.conf`.

    ```ini
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
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = heat
    password = HEAT_PASS

    [trustee]
    auth_type = password
    auth_url = http://controller:5000
    username = heat
    password = HEAT_PASS
    user_domain_name = Default

    [clients_keystone]
    auth_uri = http://controller:5000
    ```

7. Initialize the **heat** database tables.

    ```shell
    su -s /bin/sh -c "heat-manage db_sync" heat
    ```

8. Start the services.

    ```shell
    systemctl enable openstack-heat-api.service openstack-heat-api-cfn.service openstack-heat-engine.service
    systemctl start openstack-heat-api.service openstack-heat-api-cfn.service openstack-heat-engine.service
    ```

9. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack stack list
    openstack orchestration service list
    ```

#### Ironic

Ironic is the OpenStack bare metal service. It is recommended if you need bare metal provisioning; otherwise, it can be skipped.

Perform the following operations on the controller node.

1. Set up the database.

    The bare metal service stores information in a database. Create an **ironic** database that the **ironic** user can access. Replace **IRONIC_DBPASS** (example: Ironic2026) with a suitable password:

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE ironic CHARACTER SET utf8;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'localhost' IDENTIFIED BY 'IRONIC_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'%' IDENTIFIED BY 'IRONIC_DBPASS';
    MariaDB [(none)]> exit
    ```

2. Create the service user authentication.

    - Create the Bare Metal service user:

        Replace `IRONIC_PASS` (example: Ironic2026) with the ironic user password. If you need ironic-inspector (bare metal inspection service), also create `IRONIC_INSPECTOR_PASS`.

        ```shell
        source ~/.admin-openrc
        openstack user create --domain default --password-prompt ironic
        openstack role add --project service --user ironic admin
        openstack service create --name ironic \
          --description "Ironic baremetal provisioning service" baremetal

        # (Optional) ironic-inspector service user
        openstack service create --name ironic-inspector \
          --description "Ironic inspector baremetal provisioning service" baremetal-introspection
        openstack user create --domain default --password-prompt ironic-inspector
        openstack role add --project service --user ironic-inspector admin
        ```

    - Create the Bare Metal service endpoints:

        ```shell
        openstack endpoint create --region RegionOne baremetal admin http://controller:6385
        openstack endpoint create --region RegionOne baremetal public http://controller:6385
        openstack endpoint create --region RegionOne baremetal internal http://controller:6385
        # (Optional, for ironic-inspector)
        openstack endpoint create --region RegionOne baremetal-introspection internal http://controller:5050/v1
        openstack endpoint create --region RegionOne baremetal-introspection public http://controller:5050/v1
        openstack endpoint create --region RegionOne baremetal-introspection admin http://controller:5050/v1
        ```

3. Install the components.

    ```shell
    dnf install openstack-ironic-api openstack-ironic-conductor python3-ironicclient
    ```

4. Configure `/etc/ironic/ironic.conf`.

    ```ini
    [database]
    connection = mysql+pymysql://ironic:IRONIC_DBPASS@controller/ironic

    [DEFAULT]
    transport_url = rabbit://openstack:RABBIT_PASS@controller
    auth_strategy = keystone
    my_ip = 192.168.0.2
    enabled_boot_interfaces = pxe,ipxe
    default_boot_interface = ipxe
    enabled_deploy_interfaces = direct
    default_deploy_interface = direct
    enabled_network_interfaces = flat,noop,neutron
    default_network_interface = flat

    [keystone_authtoken]
    www_authenticate_uri = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = ironic
    password = IRONIC_PASS

    [neutron]
    auth_url = http://controller:5000
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    region_name = RegionOne
    project_name = service
    username = neutron
    password = NEUTRON_PASS
    ```

5. Synchronize the database and start the services.

    ```shell
    ironic-dbsync --config-file /etc/ironic/ironic.conf create_schema
    systemctl enable openstack-ironic-api.service openstack-ironic-conductor.service
    systemctl start openstack-ironic-api.service openstack-ironic-conductor.service
    ```

6. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack baremetal driver list
    ```

#### Trove

Trove is the OpenStack database service (DBaaS). It creates and manages database instances.

**Controller node**

1. Create the database, service credentials, and API endpoints.

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE trove;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'localhost' IDENTIFIED BY 'TROVE_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'%' IDENTIFIED BY 'TROVE_DBPASS';
    MariaDB [(none)]> exit
    ```

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt trove    # Enter TROVE_PASS
    openstack role add --project service --user trove admin
    openstack service create --name trove --description "Database Service" database
    openstack endpoint create --region RegionOne database public http://controller:8779/v1.0/%\(tenant_id\)s
    openstack endpoint create --region RegionOne database internal http://controller:8779/v1.0/%\(tenant_id\)s
    openstack endpoint create --region RegionOne database admin http://controller:8779/v1.0/%\(tenant_id\)s
    ```

2. Install the packages.

    ```shell
    dnf install openstack-trove python3-troveclient
    ```

3. Modify the configuration files.

    Edit `/etc/trove/trove.conf`:

    ```ini
    [DEFAULT]
    bind_host = 192.168.0.2
    log_dir = /var/log/trove
    network_driver = trove.network.neutron.NeutronDriver
    network_label_regex=.*
    nova_keypair = trove-mgmt
    default_datastore = mysql
    taskmanager_manager = trove.taskmanager.manager.Manager
    trove_api_workers = 5
    transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
    reboot_time_out = 300
    usage_timeout = 900
    agent_call_high_timeout = 1200

    [database]
    connection = mysql+pymysql://trove:TROVE_DBPASS@controller/trove

    [keystone_authtoken]
    auth_url = http://controller:5000/v3/
    auth_type = password
    project_domain_name = Default
    project_name = service
    user_domain_name = Default
    password = TROVE_PASS
    username = trove

    [service_credentials]
    auth_url = http://controller:5000/v3/
    region_name = RegionOne
    project_name = service
    project_domain_name = Default
    user_domain_name = Default
    username = trove
    password = TROVE_PASS

    [mariadb]
    tcp_ports = 3306,4444,4567,4568

    [mysql]
    tcp_ports = 3306

    [postgresql]
    tcp_ports = 5432
    ```

    **Explanation:**

    > `bind_host` in `[DEFAULT]` is the IP address of the Trove controller node.
    > `transport_url` is the RabbitMQ connection information. Replace `RABBIT_PASS` with the RabbitMQ password.
    > `connection` in `[database]` is the database information created for Trove in MySQL.
    > Replace `TROVE_PASS` in `[keystone_authtoken]` and `[service_credentials]` with the actual trove user password.

    Edit `/etc/trove/trove-guestagent.conf`. The guestagent is an independent Trove component that must be pre-installed in the VM image created by Trove through Nova. It reports heartbeats to Trove through RabbitMQ:

    ```ini
    [DEFAULT]
    log_file = trove-guestagent.log
    log_dir = /var/log/trove/
    ignore_users = os_admin
    control_exchange = trove
    transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
    rpc_backend = rabbit
    command_process_timeout = 60

    [service_credentials]
    auth_url = http://controller:5000/v3/
    region_name = RegionOne
    project_name = service
    password = TROVE_PASS
    project_domain_name = Default
    user_domain_name = Default
    username = trove
    ```

    **Explanation:**

    > `transport_url` is the RabbitMQ connection information. Replace `RABBIT_PASS` with the RabbitMQ password.
    > Replace `TROVE_PASS` in `[service_credentials]` with the actual trove user password.

4. Synchronize the database and start the services.

    ```shell
    su -s /bin/sh -c "trove-manage db_sync" trove
    systemctl enable openstack-trove-api.service openstack-trove-taskmanager.service \
      openstack-trove-conductor.service
    systemctl start openstack-trove-api.service openstack-trove-taskmanager.service \
      openstack-trove-conductor.service
    ```

5. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack database service list
    ```

#### Cyborg

Cyborg is the OpenStack accelerator management service. It discovers, manages, and attaches accelerator devices.

**Controller node**

1. Create the database, service credentials, and API endpoints.

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE cyborg;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON cyborg.* TO 'cyborg'@'localhost' IDENTIFIED BY 'CYBORG_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON cyborg.* TO 'cyborg'@'%' IDENTIFIED BY 'CYBORG_DBPASS';
    MariaDB [(none)]> exit
    ```

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt cyborg    # Enter CYBORG_PASS
    openstack role add --project service --user cyborg admin
    openstack service create --name cyborg --description "Acceleration Service" accelerator
    openstack endpoint create --region RegionOne accelerator public http://controller:9101
    openstack endpoint create --region RegionOne accelerator internal http://controller:9101
    openstack endpoint create --region RegionOne accelerator admin http://controller:9101
    ```

2. Install the packages.

    ```shell
    dnf install openstack-cyborg
    ```

3. Modify `/etc/cyborg/cyborg.conf`.

    ```ini
    [DEFAULT]
    transport_url = rabbit://openstack:RABBIT_PASS@controller
    use_syslog = False
    state_path = /var/lib/cyborg
    host = controller

    [api]
    host_ip = 0.0.0.0

    [database]
    connection = mysql+pymysql://cyborg:CYBORG_DBPASS@controller/cyborg

    [keystone_authtoken]
    www_authenticate_uri = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = cyborg
    password = CYBORG_PASS

    [service_catalog]
    project_domain_id = default
    user_domain_id = default
    project_name = service
    password = CYBORG_PASS
    username = cyborg
    auth_url = http://controller:5000/v3/
    auth_type = password

    [placement]
    project_domain_name = Default
    project_name = service
    user_domain_name = Default
    password = PLACEMENT_PASS
    username = placement
    auth_url = http://controller:5000/v3/
    auth_type = password
    auth_section = keystone_authtoken

    [nova]
    project_domain_name = Default
    project_name = service
    user_domain_name = Default
    password = NOVA_PASS
    username = nova
    auth_url = http://controller:5000/v3/
    auth_type = password
    auth_section = keystone_authtoken
    ```

    ***Important note***

    - **The `[placement]` authentication section is mandatory.** The PlacementClient of the cyborg conductor depends on it; otherwise the service fails.

4. Create the WSGI entry point and modify the unit (important).

    ```shell
    cat > /usr/lib/python3.11/site-packages/cyborg_wsgi.py << 'EOF'
    from oslo_config import cfg
    from cyborg import conf
    from cyborg import objects
    from cyborg.common import rpc
    from cyborg.api import app as app_module

    conf.CONF(project='cyborg')
    objects.register_all()
    rpc.init(cfg.CONF)
    application = app_module.setup_app()
    EOF

    # Change the ExecStart of /usr/lib/systemd/system/openstack-cyborg-api.service to:
    # ExecStart=/usr/bin/gunicorn -b 0.0.0.0:9101 --workers 4 --threads 4 cyborg_wsgi:application
    systemctl daemon-reload
    ```

    ***Important note***

    - **cyborg-api must be deployed with gunicorn and a wrapper entry point.** Initialize RPC/objects before creating the application; otherwise gunicorn crashes while parsing argv. The port is 9101.

5. Synchronize the database and start the services.

    ```shell
    cyborg-dbsync upgrade
    systemctl enable --now openstack-cyborg-api openstack-cyborg-conductor
    ```

6. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack accelerator device list
    curl -s -o /dev/null -w '%{http_code}\n' http://controller:9101
    # It should return 401/200 (the service is reachable).
    ```

**Compute node**

1. Install and configure the agent.

    ```shell
    dnf install openstack-cyborg
    # Configure /etc/cyborg/cyborg.conf the same as the controller node
    # ([keystone_authtoken]/[placement]/[DEFAULT] host=compute).
    systemctl enable --now openstack-cyborg-agent
    ```

#### Aodh

Aodh is the OpenStack alarming service. It provides threshold alarms and composite alarms.

**Controller node**

1. Create the database, service credentials, and API endpoints.

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE aodh;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'localhost' IDENTIFIED BY 'AODH_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'%' IDENTIFIED BY 'AODH_DBPASS';
    MariaDB [(none)]> exit
    ```

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt aodh    # Enter AODH_PASS
    openstack role add --project service --user aodh admin
    openstack service create --name aodh --description "Telemetry" alarming
    openstack endpoint create --region RegionOne alarming public http://controller:8042
    openstack endpoint create --region RegionOne alarming internal http://controller:8042
    openstack endpoint create --region RegionOne alarming admin http://controller:8042
    ```

2. Install the packages.

    ```shell
    dnf install openstack-aodh-api openstack-aodh-evaluator openstack-aodh-notifier \
      openstack-aodh-listener openstack-aodh-expirer python3-aodhclient \
      python3-observabilityclient
    ```

3. Modify `/etc/aodh/aodh.conf`.

    ```ini
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
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = aodh
    password = AODH_PASS

    [service_credentials]
    auth_type = password
    auth_url = http://controller:5000/v3
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = aodh
    password = AODH_PASS
    interface = internalURL
    region_name = RegionOne
    ```

4. Modify the aodh-api unit (important).

    ```shell
    # Add --port 8042 to ExecStart in /usr/lib/systemd/system/openstack-aodh-api.service
    # (before the "--" separator).
    # Example: ExecStart=... --port 8042 --config-file /etc/aodh/aodh.conf ...
    systemctl daemon-reload
    ```

    ***Important note***

    - **aodh-api must explicitly specify `--port 8042`.** By default it listens on 8000, which conflicts with heat-api-cfn (8000) and reports `Address already in use`.

5. Synchronize the database and start the services.

    ```shell
    aodh-dbsync
    systemctl enable --now openstack-aodh-api openstack-aodh-evaluator \
      openstack-aodh-notifier openstack-aodh-listener
    ```

6. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack alarm list
    curl -s -o /dev/null -w '%{http_code}\n' http://controller:8042
    ```

#### Gnocchi

Gnocchi is the OpenStack metric storage service. It provides a time-series database for Ceilometer/Aodh.

**Controller node**

1. Create the database, service credentials, and API endpoints.

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE gnocchi;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'localhost' IDENTIFIED BY 'GNOCCHI_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'%' IDENTIFIED BY 'GNOCCHI_DBPASS';
    MariaDB [(none)]> exit
    ```

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt gnocchi    # Enter GNOCCHI_PASS
    openstack role add --project service --user gnocchi admin
    openstack service create --name gnocchi --description "Metric Service" metric
    openstack endpoint create --region RegionOne metric public http://controller:8041
    openstack endpoint create --region RegionOne metric internal http://controller:8041
    openstack endpoint create --region RegionOne metric admin http://controller:8041
    ```

2. Install the packages.

    ```shell
    dnf install openstack-gnocchi-api openstack-gnocchi-metricd python3-gnocchiclient
    ```

3. Modify `/etc/gnocchi/gnocchi.conf`.

    ```ini
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

    [database]
    connection = mysql+pymysql://gnocchi:GNOCCHI_DBPASS@controller/gnocchi

    [indexer]
    url = mysql+pymysql://gnocchi:GNOCCHI_DBPASS@controller/gnocchi

    [storage]
    # coordination_url is not required but specifying one will improve
    # performance with better workload division across workers.
    # coordination_url = redis://controller:6379
    file_basepath = /var/lib/gnocchi
    driver = file
    ```

    ***Important note***

    - **`[indexer] url` must use MySQL.** Gazpacho 4.6.0 has no file indexer, and the default `file:///` causes gnocchi-upgrade to hang.

4. Upgrade the database and start the services.

    ```shell
    gnocchi-upgrade    # If it hangs, verify that [indexer] url uses MySQL.
    systemctl enable --now openstack-gnocchi-api openstack-gnocchi-metricd
    ```

5. Verify the deployment.

    ```shell
    source ~/.admin-openrc
    openstack metric list
    curl -s -o /dev/null -w '%{http_code}\n' http://controller:8041/v1/status
    ```

#### Ceilometer

Ceilometer is the data collection service in OpenStack.

**Controller node**

1. Create the service credentials and API endpoints.

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt ceilometer    # Enter CEILOMETER_PASS
    openstack role add --project service --user ceilometer admin
    openstack service create --name ceilometer --description "Telemetry" metering
    openstack endpoint create --region RegionOne metering public http://controller:8777
    openstack endpoint create --region RegionOne metering internal http://controller:8777
    openstack endpoint create --region RegionOne metering admin http://controller:8777
    ```

    ***Note***

    - The Gazpacho ceilometer has no independent API listener (8777 MISS is normal). Verify it through notifications and samples.

2. Install the Ceilometer packages.

    ```shell
    dnf install openstack-ceilometer-notification openstack-ceilometer-central python3-xmltodict
    ```

3. Edit the configuration file `/etc/ceilometer/pipeline.yaml`.

    ```yaml
    publishers:
        # set address of Gnocchi
        # + filter out Gnocchi-related activity meters (Swift driver)
        # + set default archive policy
        - gnocchi://?filter_project=service&archive_policy=low
    ```

4. Edit the configuration file `/etc/ceilometer/ceilometer.conf`.

    ```ini
    [DEFAULT]
    transport_url = rabbit://openstack:RABBIT_PASS@controller

    [service_credentials]
    auth_type = password
    auth_url = http://controller:5000/v3
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = ceilometer
    password = CEILOMETER_PASS
    interface = internalURL
    region_name = RegionOne
    ```

5. Synchronize the database.

    ```shell
    ceilometer-upgrade
    ```

6. Complete the Ceilometer installation on the controller node.

    ```shell
    systemctl enable openstack-ceilometer-notification.service openstack-ceilometer-central.service
    systemctl start openstack-ceilometer-notification.service openstack-ceilometer-central.service
    ```

**Compute node**

1. Install the Ceilometer packages.

    ```shell
    dnf install openstack-ceilometer-compute
    ```

2. Edit the configuration file `/etc/ceilometer/ceilometer.conf`.

    ```ini
    [DEFAULT]
    transport_url = rabbit://openstack:RABBIT_PASS@controller

    [service_credentials]
    auth_url = http://controller:5000
    project_domain_name = Default
    user_domain_name = Default
    auth_type = password
    username = ceilometer
    project_name = service
    password = CEILOMETER_PASS
    interface = internalURL
    region_name = RegionOne
    ```

3. Edit the configuration file `/etc/nova/nova.conf`.

    ```ini
    [DEFAULT]
    instance_usage_audit = True
    instance_usage_audit_period = hour

    [notifications]
    notify_on_state_change = vm_and_task_state

    [oslo_messaging_notifications]
    driver = messagingv2
    ```

4. Complete the installation.

    ```shell
    systemctl enable openstack-ceilometer-compute.service
    systemctl start openstack-ceilometer-compute.service

    # Restart the nova-compute service.
    systemctl restart openstack-nova-compute.service
    ```

#### Tempest

Tempest is the OpenStack integration test suite. It verifies the functional correctness of each service.

**Controller node**

1. Install the packages and initialize Tempest.

    ```shell
    dnf install openstack-tempest
    tempest init /opt/tempest
    cd /opt/tempest
    ```

2. Configure `/opt/tempest/etc/tempest.conf`.

    ```ini
    [auth]
    use_dynamic_credentials = true
    admin_username = admin
    admin_password = ADMIN_PASS
    admin_project_name = admin
    admin_domain_name = Default
    admin_project_domain_name = Default
    default_credentials_domain_name = Default

    [identity]
    uri_v3 = http://controller:5000/v3
    auth_version = v3
    domain_name = Default

    [identity-feature-enabled]
    api_v3 = true

    [compute]
    image_ref = <cirros image id>
    image_ref_alt = <cirros image id>
    flavor_ref = 1
    build_timeout = 900
    min_microversion = 2.1
    max_microversion = latest

    [compute-feature-enabled]
    snapshot = true

    [validation]
    run_validation = true
    image_ssh_user = cirros

    [network]
    public_network_id = <public network id>
    project_networks_reachable = false
    min_microversion = 1.29
    max_microversion = latest

    [network-feature-enabled]
    ipv6 = true
    ipv6_subnet_attributes = true
    qos_placement_physnet = provider
    update_port_qos = true
    provider_net_base_segmentation_id = 100

    [volume]
    catalog_type = volumev3
    min_microversion = 3.46
    max_microversion = latest
    backend_names = lvm,lvm2

    [volume-feature-enabled]
    multi_backend = true

    [placement]
    min_microversion = 1.29
    max_microversion = latest

    [service_available]
    neutron = true
    nova = true
    cinder = true
    glance = true
    keystone = true
    swift = true
    heat = true
    placement = true
    ```

    ***Important notes***

    - **`[auth] default_credentials_domain_name = Default` is mandatory.** The domain parameter of the dashboard test login form depends on it.
    - **`[compute] image_ref_alt` is mandatory.** It is required by the rebuild test; without it, rebuild reports `imageRef: None`.
    - **`[volume] catalog_type = volumev3` is mandatory.** Without it, volume tests report `EndpointNotFound`.
    - **`[volume] backend_names` belongs in the [volume] section** (do not put it in volume-feature-enabled). It is a comma-separated list that must match the Cinder backend `volume_backend_name`. If configured incorrectly, volume types are mapped to BACKEND_1/2 and scheduling fails.
    - **`project_networks_reachable = false`**: connectivity verification uses floating IPs. If the controller can route directly to tenant networks, you can set it to true.
    - **Snapshot/IPv6/QoS switches** (`snapshot=true`, `ipv6`, `qos_placement_physnet`, `update_port_qos`) enable the corresponding scenario test classes.

3. Run the tests.

    ```shell
    # Full functional tests (scenario)
    cd /opt/tempest
    tempest run --regex '^tempest.scenario' --concurrency 1
    # Single-service API tests
    tempest run --regex '^tempest.api.compute'
    ```

    ***Note***

    - VM tests require SSH access to the VM. Ensure that nova-api-metadata is running (`systemctl enable --now openstack-nova-metadata-api`), that `[neutron] service_metadata_proxy = true` is set in nova.conf, and that `metadata_proxy_shared_secret` matches metadata_agent.ini. Otherwise the VM cannot obtain keys and SSH is rejected.
    - In environments without KVM, VMs start slowly (QEMU), so SSH tests may take longer.

4. Install the Tempest extensions (optional).

    OpenStack services also provide Tempest test packages. You can install them to enrich the test coverage. Gazpacho provides extension tests for Cinder, Glance, Keystone, Ironic, and Trove. Run the following command:

    ```shell
    dnf install python3-cinder-tempest-plugin python3-glance-tempest-plugin \
      python3-ironic-tempest-plugin python3-keystone-tempest-plugin \
      python3-trove-tempest-plugin
    ```

---

*End of document. If you have any questions, please contact the deployment and maintenance personnel.*
