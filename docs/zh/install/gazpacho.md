# OpenStack Gazpacho 部署指南（三节点）

[TOC]

本文档是基于 openEuler 24.03 LTS SP4 的 OpenStack **2026.1 Gazpacho** 部署指南，在阅读过程中，如果您有任何疑问或者发现任何问题，请咨询部署维护人员。

**约定**

本章节描述文档中的一些通用约定。

| 名称 | 定义 |
|:----:|:----:|
| MYSQL_ROOT_PASS | mariadb 数据库 root 密码，示例值 `RootMysql2026` |
| RABBIT_PASS | rabbitmq 的密码，示例值 `RbtOpenStack2026` |
| ADMIN_PASS | OpenStack admin 用户密码，示例值 `Admin2026` |
| KEYSTONE_DBPASS / KEYSTONE_PASS | keystone 服务数据库密码 / keystone 用户密码，示例值 `Keystone2026` |
| GLANCE_DBPASS / GLANCE_PASS | glance 服务数据库密码 / keystone 用户密码，示例值 `Glance2026` |
| PLACEMENT_DBPASS / PLACEMENT_PASS | placement 服务数据库密码 / keystone 用户密码，示例值 `Placement2026` |
| NOVA_DBPASS / NOVA_PASS | nova 服务数据库密码 / keystone 用户密码，示例值 `Nova2026` |
| NEUTRON_DBPASS / NEUTRON_PASS | neutron 服务数据库密码 / keystone 用户密码，示例值 `Neutron2026` |
| CINDER_DBPASS / CINDER_PASS | cinder 服务数据库密码 / keystone 用户密码，示例值 `Cinder2026` |
| SWIFT_PASS | swift keystone 用户密码，示例值 `Swift2026` |
| HEAT_DBPASS / HEAT_PASS | heat 服务数据库密码 / keystone 用户密码，示例值 `Heat2026` |
| IRONIC_DBPASS / IRONIC_PASS | ironic 服务数据库密码 / keystone 用户密码，示例值 `Ironic2026` |
| TROVE_DBPASS / TROVE_PASS | trove 服务数据库密码 / keystone 用户密码，示例值 `Trove2026` |
| CYBORG_DBPASS / CYBORG_PASS | cyborg 服务数据库密码 / keystone 用户密码，示例值 `Cyborg2026` |
| AODH_DBPASS / AODH_PASS | aodh 服务数据库密码 / keystone 用户密码，示例值 `Aodh2026` |
| GNOCCHI_DBPASS / GNOCCHI_PASS | gnocchi 服务数据库密码 / keystone 用户密码，示例值 `Gnocchi2026` |
| CEILOMETER_DBPASS / CEILOMETER_PASS | ceilometer 服务数据库密码 / keystone 用户密码，示例值 `Ceilometer2026` |
| METADATA_SECRET | metadata proxy 的 secret 密码，示例值 `Metadata2026` |
| PROVIDER_INTERFACE_NAME | 物理网络接口的名称（provider 网络），示例值 `ens34` |
| OVERLAY_INTERFACE_IP_ADDRESS | 各节点管理网 IP 地址（vxlan 隧道端点），示例值分别为 `192.168.0.2/192.168.0.3/192.168.0.4` |

> 说明：密码示例均为 `xxx2026` 系列，与单节点验证环境保持一致；如环境不同请按需替换。

OpenStack 提供多种部署方式，本文档采用 **基于 RPM 手动部署** 方式，服务范围与 openEuler OpenStack SIG 文档一致，共 **15 个 OpenStack 服务 + Tempest 测试工具**：

| 类别 | 服务 |
|:----:|:----:|
| 核心（7） | Keystone、Glance、Placement、Nova、Neutron、Cinder、Horizon |
| 可选（8） | Swift、Heat、Ironic、Trove、Cyborg、Aodh、Gnocchi、Ceilometer |
| 测试 | Tempest（功能测试，可选） |

## 基于RPM部署

### 环境准备

本文档基于 OpenStack 经典的三节点环境进行部署，三个节点分别是控制节点(Controller)、计算节点(Compute)、存储节点(Storage)。其中存储节点一般只部署存储服务，在资源有限的情况下，可以不单独部署该节点，把存储节点上的服务部署到计算节点即可。

首先准备三个 openEuler 24.03 LTS SP4 环境，根据您的环境，下载对应的镜像并安装即可：[ISO镜像](https://repo.openeuler.org/openEuler-24.03-LTS-SP4/ISO/)、[qcow2镜像](https://repo.openeuler.org/openEuler-24.03-LTS-SP4/virtual_machine_img/)。

下面的安装按照如下拓扑进行：

```text
controller：192.168.0.2   （管理网 ens33；provider 网络 ens34）
compute：   192.168.0.3   （管理网 ens33；provider 网络 ens34）
storage：   192.168.0.4   （管理网 ens33；存储数据盘 /dev/sdb）
```

如果您的环境 IP 或网卡不同，请按照您的环境修改相应的配置文件。

本文档的三节点服务拓扑如下：

| 节点 | 部署服务 |
|:----:|:----:|
| Controller | mariadb / rabbitmq / memcached、Keystone、Glance、Placement、Nova(api/cond/sched/vncproxy)、Neutron(server + ovs/dhcp/l3/metadata agents)、Cinder(api/scheduler)、Swift(proxy)、Heat、Horizon、Ironic(api/cond)、Trove、Cyborg(api/cond)、Aodh、Gnocchi、Ceilometer(api/central)、Tempest |
| Compute | Nova(nova-compute)、Neutron(ovs agent)、Cyborg(agent)、Ceilometer(compute) |
| Storage | Cinder(cinder-volume，LVM 后端)、Swift(account/container/object) |

在正式部署之前，需要对每个节点做如下配置和检查：

#### 配置 openEuler 24.03 LTS SP4 yum 源（Gazpacho 多版本源）

本文档使用 Gazpacho 多版本源（EPOL Multi-Version），源内提供全部 Gazpacho 服务包。**三个节点**均需配置。

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

**注意**：

- Gazpacho 多版本源中 RPM 均带签名，`gpgcheck=1` 可正常校验（已验证）。
- 除 Gazpacho 源外，`openstack-neutron-openvswitch`、`targetcli`、`scsi-target-utils` 等基础包需要 EPOL 或 everything 源；如默认 openEuler 源已启用 EPOL 则无需额外配置。
- 如遇包找不到，请确认 `/etc/yum.repos.d/openEuler.repo` 中 EPOL 已启用。

```shell
dnf makecache
```

#### 修改主机名以及映射

每个节点分别修改主机名，以 controller 为例：

```shell
hostnamectl set-hostname controller
```

然后修改每个节点的 `/etc/hosts` 文件，新增如下内容:

```text
192.168.0.2   controller
192.168.0.3   compute
192.168.0.4   storage
```

#### 时钟同步

集群环境时刻要求每个节点的时间一致，一般由时钟同步软件保证。本文使用 `chrony` 软件。

**Controller节点**：

```shell
dnf install chrony
# /etc/chrony.conf 新增一行：allow 192.168.0.0/24
sed -i '/^allow/a allow 192.168.0.0/24' /etc/chrony.conf
systemctl enable --now chronyd
```

**其他节点**（compute/storage）：

```shell
dnf install chrony
# /etc/chrony.conf 新增 server controller iburst，并注释掉 pool 行
echo "server controller iburst" >> /etc/chrony.conf
sed -i 's/^pool /#pool /' /etc/chrony.conf
systemctl enable --now chronyd
```

验证（其他节点）：`chronyc sources` 应显示 `^* controller`。

#### 安装数据库

数据库安装在控制节点，这里推荐使用 mariadb。

```shell
dnf install mariadb-config mariadb mariadb-server python3-PyMySQL
```

新增配置文件 `/etc/my.cnf.d/openstack.cnf`，内容如下：

```ini
[mysqld]
bind-address = 192.168.0.2
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

1. 启动服务器

    ```shell
    systemctl enable --now mariadb
    ```

2. 初始化数据库，根据提示进行即可

    ```shell
    mysql_secure_installation
    ```

    示例如下：

    ```shell
    NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
        SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

    In order to log into MariaDB to secure it, we'll need the current
    password for the root user. If you've just installed MariaDB, and
    haven't set the root password yet, you should just press enter here.

    Enter current password for root (enter for none):

    #这里输入密码，由于我们是初始化DB，直接回车就行

    OK, successfully used password, moving on...

    Setting the root password or using the unix_socket ensures that nobody
    can log into the MariaDB root user without the proper authorisation.

    You already have your root account protected, so you can safely answer 'n'.

    # 这里根据提示输入N

    Switch to unix_socket authentication [Y/n] N

    Enabled successfully!
    Reloading privilege tables..
    ... Success!


    You already have your root account protected, so you can safely answer 'n'.

    # 输入Y，修改密码

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

    # 输入Y，删除匿名用户

    Remove anonymous users? [Y/n] Y
    ... Success!

    Normally, root should only be allowed to connect from 'localhost'.  This
    ensures that someone cannot guess at the root password from the network.

    # 输入Y，关闭root远程登录权限

    Disallow root login remotely? [Y/n] Y
    ... Success!

    By default, MariaDB comes with a database named 'test' that anyone can
    access.  This is also intended only for testing, and should be removed
    before moving into a production environment.

    # 输入Y，删除test数据库

    Remove test database and access to it? [Y/n] Y
    - Dropping test database...
    ... Success!
    - Removing privileges on test database...
    ... Success!

    Reloading the privilege tables will ensure that all changes made so far
    will take effect immediately.

    # 输入Y，重载配置

    Reload privilege tables now? [Y/n] Y
    ... Success!

    Cleaning up...

    All done!  If you've completed all of the above steps, your MariaDB
    installation should now be secure.
    ```

    ***注意***

    **新设置的 root 密码即为约定中的 `MYSQL_ROOT_PASS`（示例 RootMysql2026），后续所有数据库初始化均使用该密码。**

3. 验证，根据第二步设置的密码，检查是否能登录 mariadb

    ```shell
    mysql -uroot -p
    ```

#### 安装消息队列

消息队列安装在控制节点，这里推荐使用 rabbitmq。

```shell
dnf install rabbitmq-server
systemctl enable --now rabbitmq-server
rabbitmqctl add_user openstack RABBIT_PASS
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### 安装缓存服务

缓存服务安装在控制节点，这里推荐使用 Memcached。

```shell
dnf install memcached python3-memcached
# /etc/sysconfig/memcached：OPTIONS="-l 127.0.0.1,::1,controller"
systemctl enable --now memcached
```

### 部署服务

#### Keystone

Keystone 是 OpenStack 提供的鉴权服务，是整个 OpenStack 的入口，提供了租户隔离、用户认证、服务发现等功能，必须安装。

**Controller节点**

1. 创建 keystone 数据库并授权

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE keystone;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
    MariaDB [(none)]> exit
    ```

    ***注意***

    **替换 `KEYSTONE_DBPASS`（示例 Keystone2026），为 Keystone 数据库设置密码**

2. 安装软件包

    ```shell
    dnf install openstack-keystone httpd mod_wsgi
    ```

3. 配置 keystone 相关配置

    ```shell
    vim /etc/keystone/keystone.conf

    [database]
    connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

    [token]
    provider = fernet
    ```

    ***解释***

    [database] 部分，配置数据库入口；[token] 部分，配置 token provider

4. 同步数据库

    ```shell
    su -s /bin/sh -c "keystone-manage db_sync" keystone
    ```

5. 初始化 Fernet 密钥仓库

    ```shell
    keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
    keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
    ```

6. 启动服务

    ```shell
    keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
      --bootstrap-admin-url http://controller:5000/v3/ \
      --bootstrap-internal-url http://controller:5000/v3/ \
      --bootstrap-public-url http://controller:5000/v3/ \
      --bootstrap-region-id RegionOne
    ```

    ***注意***

    **替换 `ADMIN_PASS`（示例 Admin2026），为 admin 用户设置密码**

7. 配置 Apache HTTP server

    ```shell
    # /etc/httpd/conf/httpd.conf 修改（不存在则新增）：ServerName controller
    ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
    systemctl enable --now httpd.service
    ```

8. 创建环境变量配置

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

9. 安装 OpenStack 命令行客户端

    后续所有服务都需要使用 `openstack` 命令创建用户、服务与 endpoint，需要先安装命令行客户端。

    ```shell
    dnf install python3-openstackclient
    ```

10. 依次创建 domain、projects、users、roles

    ```shell
    source ~/.admin-openrc

    # 创建 service 项目（default domain 在 bootstrap 时已创建）
    openstack project create --domain default --description "Service Project" service

    # 创建 demo 项目/用户/角色（可选）
    openstack project create --domain default --description "Demo Project" myproject
    openstack user create --domain default --password-prompt myuser
    openstack role create myrole
    openstack role add --project myproject --user myuser myrole
    ```

11. 验证

    - 取消临时环境变量 OS_AUTH_URL 和 OS_PASSWORD：

        ```shell
        source ~/.admin-openrc
        unset OS_AUTH_URL OS_PASSWORD
        ```

    - 为 admin 用户请求 token：

        ```shell
        openstack --os-auth-url http://controller:5000/v3 \
          --os-project-domain-name Default --os-user-domain-name Default \
          --os-project-name admin --os-username admin token issue
        ```

    - 为 myuser 用户请求 token：

        ```shell
        openstack --os-auth-url http://controller:5000/v3 \
          --os-project-domain-name Default --os-user-domain-name Default \
          --os-project-name myproject --os-username myuser token issue
        ```

#### Glance

Glance 是 OpenStack 提供的镜像服务，负责虚拟机、裸机镜像的上传与下载，必须安装。

**Controller节点**

1. 创建 glance 数据库并授权

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE glance;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
    MariaDB [(none)]> exit
    ```

    ***注意:***

    **替换 `GLANCE_DBPASS`（示例 Glance2026），为 glance 数据库设置密码**

2. 初始化 glance 资源对象

    - 导入环境变量

        ```shell
        source ~/.admin-openrc
        ```

    - 创建 glance 用户时，命令行会提示输入密码，请输入自定义的密码（即 `GLANCE_PASS`，示例 Glance2026）：

        ```shell
        openstack user create --domain default --password-prompt glance
        User Password:
        Repeat User Password:
        ```

    - 添加 glance 用户到 service project 并指定 admin 角色：

        ```shell
        openstack role add --project service --user glance admin
        ```

    - 创建 glance 服务实体：

        ```shell
        openstack service create --name glance --description "OpenStack Image" image
        ```

    - 创建 glance API 服务 endpoints：

        ```shell
        openstack endpoint create --region RegionOne image public http://controller:9292
        openstack endpoint create --region RegionOne image internal http://controller:9292
        openstack endpoint create --region RegionOne image admin http://controller:9292
        ```

3. 安装软件包

    ```shell
    dnf install openstack-glance
    ```

4. 修改 glance 配置文件

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

    ***解释:***

    [database] 部分，配置数据库入口

    [keystone_authtoken] [paste_deploy] 部分，配置身份认证服务入口

    [glance_store] 部分，配置本地文件系统存储和镜像文件的位置

    ***注意（关键）***

    - **`[paste_deploy] flavor = keystone` 必须配置**：缺省时走 `unauthenticated-context`（无 token 校验），导致 `openstack image set` 等修改操作 403（modify_image 策略需要角色）。
    - **加载元数据定义**（否则 metadefs API 404）：

    ```shell
    su -s /bin/sh -c "glance-manage db_load_metadefs" glance
    ```

5. 同步数据库

    ```shell
    su -s /bin/sh -c "glance-manage db_sync" glance
    ```

6. 启动服务

    ```shell
    systemctl enable openstack-glance-api.service
    systemctl start openstack-glance-api.service
    ```

7. 验证

    - 导入环境变量

        ```shell
        source ~/.admin-openrc
        ```

    - 下载镜像

        ```shell
        x86镜像下载：
        wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

        arm镜像下载：
        wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-aarch64-disk.img
        ```

        ***注意***

        **如果您使用的环境是鲲鹏架构，请下载 aarch64 版本的镜像**

    - 向 Image 服务上传镜像：

        ```shell
        openstack image create --disk-format qcow2 --container-format bare \
                            --file cirros-0.6.2-x86_64-disk.img --public cirros
        ```

    - 确认镜像上传并验证属性：

        ```shell
        openstack image list
        ```

#### Placement

Placement 是 OpenStack 提供的资源调度组件，一般不面向用户，由 Nova 等组件调用，安装在控制节点。

安装、配置 Placement 服务前，需要先创建相应的数据库、服务凭证和 API endpoints。

1. 创建数据库

    - 使用 root 用户访问数据库服务：

        ```shell
        mysql -u root -p
        ```

    - 创建 placement 数据库：

        ```sql
        MariaDB [(none)]> CREATE DATABASE placement;
        ```

    - 授权数据库访问：

        ```sql
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
          IDENTIFIED BY 'PLACEMENT_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
          IDENTIFIED BY 'PLACEMENT_DBPASS';
        ```

        替换 `PLACEMENT_DBPASS`（示例 Placement2026）为 placement 数据库访问密码。

    - 退出数据库访问客户端：

        ```shell
        exit
        ```

2. 配置用户和 Endpoints

    - source admin 凭证，以获取 admin 命令行权限：

        ```shell
        source ~/.admin-openrc
        ```

    - 创建 placement 用户并设置用户密码（即 `PLACEMENT_PASS`，示例 Placement2026）：

        ```shell
        openstack user create --domain default --password-prompt placement

        User Password:
        Repeat User Password:
        ```

    - 添加 placement 用户到 service project 并指定 admin 角色：

        ```shell
        openstack role add --project service --user placement admin
        ```

    - 创建 placement 服务实体：

        ```shell
        openstack service create --name placement \
          --description "Placement API" placement
        ```

    - 创建 Placement API 服务 endpoints：

        ```shell
        openstack endpoint create --region RegionOne \
          placement public http://controller:8778
        openstack endpoint create --region RegionOne \
          placement internal http://controller:8778
        openstack endpoint create --region RegionOne \
          placement admin http://controller:8778
        ```

3. 安装及配置组件

    - 安装软件包：

        ```shell
        dnf install openstack-placement-api
        ```

    - 编辑 `/etc/placement/placement.conf` 配置文件，完成如下操作：

        - 在 `[placement_database]` 部分，配置数据库入口：

            ```ini
            [placement_database]
            connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
            ```

            替换 `PLACEMENT_DBPASS` 为 placement 数据库的密码。

        - 在 `[api]` 和 `[keystone_authtoken]` 部分，配置身份认证服务入口：

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

            替换 `PLACEMENT_PASS` 为 placement 用户的密码。

    - 数据库同步，填充 Placement 数据库：

        ```shell
        su -s /bin/sh -c "placement-manage db sync" placement
        ```

4. 启动服务

    重启 httpd 服务：

    ```shell
    systemctl restart httpd
    ```

5. 验证

    - source admin 凭证，以获取 admin 命令行权限

        ```shell
        source ~/.admin-openrc
        ```

    - 执行状态检查：

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

        这里可以看到 `Policy File JSON to YAML Migration` 的结果为 Failure。这是因为在 Placement 中，JSON 格式的 policy 文件从 Wallaby 版本开始已处于 `deprecated` 状态。可以参考提示，使用 [oslopolicy-convert-json-to-yaml](https://docs.openstack.org/oslo.policy/latest/cli/oslopolicy-convert-json-to-yaml.html) 工具将现有的 JSON 格式 policy 文件转化为 YAML 格式。

        ```shell
        oslopolicy-convert-json-to-yaml  --namespace placement \
          --policy-file /etc/placement/policy.json \
          --output-file /etc/placement/policy.yaml
        mv /etc/placement/policy.json{,.bak}
        ```

        注：当前环境中此问题可忽略，不影响运行。

    - 针对 placement API 运行命令：

        - 安装 osc-placement 插件：

            ```shell
            dnf install python3-osc-placement
            ```

        - 列出可用的资源类别及特性：

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

Nova 是 OpenStack 的计算服务，负责虚拟机的创建、发放等功能。

**Controller节点**

在控制节点执行以下操作。

1. 创建数据库

    - 使用 root 用户访问数据库服务：

        ```shell
        mysql -u root -p
        ```

    - 创建 `nova_api`、`nova` 和 `nova_cell0` 数据库：

        ```sql
        MariaDB [(none)]> CREATE DATABASE nova_api;
        MariaDB [(none)]> CREATE DATABASE nova;
        MariaDB [(none)]> CREATE DATABASE nova_cell0;
        ```

    - 授权数据库访问：

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

        替换 `NOVA_DBPASS`（示例 Nova2026）为 nova 相关数据库访问密码。

    - 退出数据库访问客户端：

        ```sql
        exit
        ```

2. 配置用户和 Endpoints

    - source admin 凭证，以获取 admin 命令行权限：

        ```shell
        source ~/.admin-openrc
        ```

    - 创建 nova 用户并设置用户密码（即 `NOVA_PASS`，示例 Nova2026）：

        ```shell
        openstack user create --domain default --password-prompt nova

        User Password:
        Repeat User Password:
        ```

    - 添加 nova 用户到 service project 并指定 admin 角色：

        ```shell
        openstack role add --project service --user nova admin
        ```

    - 创建 nova 服务实体：

        ```shell
        openstack service create --name nova \
          --description "OpenStack Compute" compute
        ```

    - 创建 Nova API 服务 endpoints：

        ```shell
        openstack endpoint create --region RegionOne \
          compute public http://controller:8774/v2.1
        openstack endpoint create --region RegionOne \
          compute internal http://controller:8774/v2.1
        openstack endpoint create --region RegionOne \
          compute admin http://controller:8774/v2.1
        ```

3. 安装及配置组件

    - 安装软件包：

        ```shell
        dnf install openstack-nova-api openstack-nova-conductor \
          openstack-nova-novncproxy openstack-nova-scheduler
        ```

    - 编辑 `/etc/nova/nova.conf` 配置文件，完成如下操作：

        - 在 `[DEFAULT]` 部分，启用计算和元数据的 API，配置 RabbitMQ 消息队列入口，使用 controller 节点管理 IP 配置 my_ip，显式定义 log_dir：

            ```ini
            [DEFAULT]
            enabled_apis = osapi_compute,metadata
            transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
            my_ip = 192.168.0.2
            log_dir = /var/log/nova
            state_path = /var/lib/nova
            ```

        - 在 `[api_database]` 和 `[database]` 部分，配置数据库入口：

            ```ini
            [api_database]
            connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

            [database]
            connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
            ```

        - 在 `[api]` 和 `[keystone_authtoken]` 部分，配置身份认证服务入口：

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

        - 在 `[vnc]` 部分，启用并配置远程控制台入口：

            ```ini
            [vnc]
            enabled = true
            server_listen = $my_ip
            server_proxyclient_address = $my_ip
            ```

        - 在 `[glance]` 部分，配置镜像服务 API 的地址：

            ```ini
            [glance]
            api_servers = http://controller:9292
            ```

        - 在 `[oslo_concurrency]` 部分，配置 lock path：

            ```ini
            [oslo_concurrency]
            lock_path = /var/lib/nova/tmp
            ```

        - 在 `[placement]` 部分，配置 placement 服务的入口：

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

        - 在 `[neutron]` 部分，配置 neutron 服务入口：

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

        - 在 `[service_user]` 和 `[keystone_authtoken]` 部分，配置服务间认证：

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

    - 数据库同步：

        - 同步 nova-api 数据库：

            ```shell
            su -s /bin/sh -c "nova-manage api_db sync" nova
            ```

        - 注册 cell0 数据库：

            ```shell
            su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
            ```

        - 创建 cell1 cell：

            ```shell
            su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
            ```

        - 同步 nova 数据库：

            ```shell
            su -s /bin/sh -c "nova-manage db sync" nova
            ```

        - 验证 cell0 和 cell1 注册正确：

            ```shell
            su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
            ```

4. 授予 nova/cinder 用户 service 角色（service token 校验需要）

    ```shell
    source ~/.admin-openrc
    openstack role create service
    openstack role add --user nova --project service service
    openstack role add --user cinder --project service service
    openstack role add --user neutron --project service service
    ```

    ***注意（关键）***

    - **`[neutron]` 认证段必须配置**：缺省时创建虚机报 `Unknown auth type: None`（nova 调 neutron 需要完整认证段）。
    - **`[service_user]` + `service_token_roles = service` 必须配置**：nova 调用 neutron/cinder 时需携带 service token；`service_token_roles` 必须为 `service`（并给 nova 用户授予 service 角色），否则挂卷 detach 时 cinder 报 409 ConflictNovaUsingAttachment、更新 port 时 neutron policy 403。
    - **`[vnc] server_proxyclient_address` 必须非空**（用 `$my_ip`），否则 nova 启动报 `ConfigFileValueError`。

5. 启动服务

    ```shell
    systemctl enable openstack-nova-api.service openstack-nova-scheduler.service \
      openstack-nova-conductor.service openstack-nova-novncproxy.service
    systemctl start openstack-nova-api.service openstack-nova-scheduler.service \
      openstack-nova-conductor.service openstack-nova-novncproxy.service
    ```

**Compute节点**

1. 前置依赖

    ```shell
    dnf install hostname openstack-nova-compute
    ```

    ***注意***

    - **`hostname` 命令必须安装**：nova 配置中 `$my_ip` 取地址依赖 hostname 命令，openEuler 最小化镜像可能未装。
    - **libvirtd 必须启动**：nova-compute 依赖 `/var/run/libvirt/libvirt-sock`，未启动则 compute 无法注册。

    ```shell
    systemctl enable --now libvirtd
    ```

2. 编辑 `/etc/nova/nova.conf` 配置文件

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

    ***注意***

    - **`[neutron]` 认证段同样必须配置**（compute 节点创建虚机时调 neutron 绑定 port）。

3. 确认计算节点是否支持虚拟机硬件加速（x86_64）

    ```shell
    egrep -c '(vmx|svm)' /proc/cpuinfo
    ```

    返回 0 则不支持硬件加速，需配置 libvirt 使用 QEMU；返回 1 或更大则无需额外配置。

    ```ini
    [libvirt]
    virt_type = qemu
    ```

4. 确认计算节点是否支持虚拟机硬件加速（arm64）

    ```shell
    virt-host-validate
    ```

    显示 FAIL 时（`/dev/kvm` 不存在），同样配置 `[libvirt] virt_type = qemu`。

5. 配置 qemu（仅 arm64）

    仅当处理器为 arm64 架构时需要执行此操作。

    - 编辑 `/etc/libvirt/qemu.conf`：

        ```ini
        nvram = ["/usr/share/AAVMF/AAVMF_CODE.fd: \
                 /usr/share/AAVMF/AAVMF_VARS.fd", \
                 "/usr/share/edk2/aarch64/QEMU_EFI-pflash.raw: \
                 /usr/share/edk2/aarch64/vars-template-pflash.raw"]
        ```

    - 编辑 `/etc/qemu/firmware/edk2-aarch64.json`：

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

6. 启动服务

    ```shell
    systemctl enable libvirtd.service openstack-nova-compute.service
    systemctl start libvirtd.service openstack-nova-compute.service
    ```

**Controller节点（收尾）**

1. 添加计算节点到 openstack 集群

    - source admin 凭证，以获取 admin 命令行权限：

        ```shell
        source ~/.admin-openrc
        ```

    - 确认 nova-compute 服务已识别到数据库中：

        ```shell
        openstack compute service list --service nova-compute
        ```

    - 发现计算节点，将计算节点添加到 cell 数据库：

        ```shell
        su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
        ```

2. 验证

    - 列出服务组件，验证每个流程都成功启动和注册：

        ```shell
        openstack compute service list
        ```

    - 列出身份服务中的 API 端点，验证与身份服务的连接：

        ```shell
        openstack catalog list
        ```

    - 列出镜像服务中的镜像，验证与镜像服务的连接：

        ```shell
        openstack image list
        ```

    - 检查 cells 是否运作成功，以及其他必要条件是否已具备：

        ```shell
        nova-status upgrade check
        ```

#### Neutron

Neutron 是 OpenStack 的网络服务，提供虚拟交换机、IP 路由、DHCP 等功能。

**Controller节点**

1. 创建数据库、服务凭证和 API 服务端点

    - 创建数据库：

        ```sql
        mysql -u root -p

        MariaDB [(none)]> CREATE DATABASE neutron;
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
        MariaDB [(none)]> exit;
        ```

        替换 `NEUTRON_DBPASS`（示例 Neutron2026）为 neutron 数据库访问密码。

    - 创建用户和服务，并记住创建 neutron 用户时输入的密码，用于配置 `NEUTRON_PASS`（示例 Neutron2026）：

        ```shell
        source ~/.admin-openrc
        openstack user create --domain default --password-prompt neutron
        openstack role add --project service --user neutron admin
        openstack service create --name neutron --description "OpenStack Networking" network
        ```

    - 部署 Neutron API 服务端点：

        ```shell
        openstack endpoint create --region RegionOne network public http://controller:9696
        openstack endpoint create --region RegionOne network internal http://controller:9696
        openstack endpoint create --region RegionOne network admin http://controller:9696
        ```

2. 安装软件包

    ```shell
    dnf install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch \
      openstack-neutron-rpc-server ebtables ipset
    ```

3. 配置 Neutron

    - 修改 `/etc/neutron/neutron.conf`

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

        ***注意（关键）***

        - **`service_plugins` 中 placement_report 必须用完整类路径**（`neutron.services.placement_report.plugin.PlacementReportPlugin`）：该 plugin 未注册 entry point，直接写 `placement_report` 会导致 rpc-server 启动失败 `ImportError: Class not found`。placement_report 负责把 ovs agent 上报的带宽同步到 Placement（QoS minimum bandwidth 测试依赖）。
        - **`[placement]` 认证段必须配置**：否则 neutron 无法调用 placement API（resource provider 不创建）。

    - 修改 `/etc/neutron/plugins/ml2/ml2_conf.ini`

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

        ***注意***

        - **`[ml2_type_vlan] network_vlan_ranges` 必须配置**：否则创建 vlan 类型 provider 网络返回 400（QoS min_bw 测试等会用到 vlan 网络）。

    - 修改 `/etc/neutron/plugins/ml2/openvswitch_agent.ini`

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

        ***注意（关键）***

        - **`firewall_driver = openvswitch`**（OVS 流表实现）：不要用 iptables 类 driver——iptables 规则用 port UUID（36 字符）作接口名，超过 IFNAMSIZ(15) 会导致 iptables-restore 失败、安全组失效；也不要改成 NoopFirewallDriver（安全组测试会失败）。
        - **`resource_provider_bandwidths` 的 key 是网桥名（br-provider）而非物理网络名**，且该配置项在 **[ovs] 段**（[agent] 段无效）；不配则 QoS minimum bandwidth 测试无 allocation 候选。
        - **`[placement]` 认证段必须配置**：ovs agent 通过 placement_report 同步 resource provider 需要。

    - 修改 `/etc/neutron/l3_agent.ini`、`/etc/neutron/dhcp_agent.ini`

        ```ini
        [DEFAULT]
        interface_driver = openvswitch
        ```

        ***注意***

        - **`interface_driver` 必须用入口点名 `openvswitch`**，不能写完整类名 `neutron.agent.linux.interface.OVSInterfaceDriver`（stevedore 加载失败 `Could not load`，DHCP 接口创建异常导致虚机拿不到 IP）。

    - 修改 `/etc/neutron/dhcp_agent.ini`（追加）

        ```ini
        [DEFAULT]
        dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
        enable_isolated_metadata = true
        ```

    - 修改 `/etc/neutron/metadata_agent.ini`

        ```ini
        [DEFAULT]
        nova_metadata_host = 127.0.0.1
        metadata_proxy_shared_secret = METADATA_SECRET
        ```

        ***注意***

        - **`nova_metadata_host = 127.0.0.1`**：nova-metadata（uwsgi）仅监听 127.0.0.1:8775，配置成 hostname（controller）会导致 metadata agent 转发 Connection refused、虚机拿不到密钥（SSH 拒绝）。

4. 创建 plugin.ini 的符号链接（关键）

    ```shell
    ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
    ```

    ***注意（关键）***

    - **该 symlink 必须做**：Gazpacho 的 neutron-rpc-server 的 ExecStart 引用 `/etc/neutron/plugin.ini`，该文件缺省为空文件 → rpc-server 加载的 mechanism drivers 为空 → DHCP/router port 绑定失败（binding_failed）→ 虚机 DHCP 不通、floating IP 不通。symlink 到 ml2_conf.ini 后 rpc-server 才能加载 openvswitch mech driver。

5. 同步数据库并启动

    ```shell
    su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
      --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
    systemctl restart openstack-nova-api
    systemctl enable --now neutron-server.service neutron-rpc-server.service \
      neutron-openvswitch-agent.service neutron-dhcp-agent.service \
      neutron-metadata-agent.service neutron-l3-agent.service
    ```

6. 创建 provider 网桥

    ```shell
    ovs-vsctl add-br br-provider
    ovs-vsctl add-port br-provider PROVIDER_INTERFACE_NAME
    ```

    ***注意***

    - **br-provider 不会自动创建**，必须手动 `add-br`；provider 网络（flat/vlan）依赖它。
    - 三节点环境 provider 网卡直接接入 br-provider（无需单节点环境中的 ext0 内部端口变通）；controller 与各 compute 节点的 provider 网卡接入同一二层网络即可互相通信。

7. 验证

    ```shell
    source ~/.admin-openrc
    openstack network agent list
    # 各 agent 应显示 Alive
    ```

**Compute节点**

1. 安装软件包

    ```shell
    dnf install -y openstack-neutron-openvswitch ebtables ipset
    ```

2. 配置 Neutron

    - 修改 `/etc/neutron/neutron.conf`

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

    - 修改 `/etc/neutron/plugins/ml2/openvswitch_agent.ini`（与 Controller 节点一致）

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

3. 创建 provider 网桥（同 Controller 节点）

    ```shell
    ovs-vsctl add-br br-provider
    ovs-vsctl add-port br-provider PROVIDER_INTERFACE_NAME
    ```

4. 重启 nova-compute 并启动 ovs agent

    ```shell
    systemctl restart openstack-nova-compute.service
    systemctl enable --now neutron-openvswitch-agent.service
    ```

5. 验证

    ```shell
    source ~/.admin-openrc
    openstack network agent list
    # ovs agent 应显示两个（controller + compute），均 Alive
    ```

#### Cinder

Cinder 是 OpenStack 的存储服务，提供块设备的创建、发放、备份等功能。

**Controller节点**

1. 初始化数据库

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE cinder;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
    MariaDB [(none)]> exit
    ```

2. 初始化 Keystone 资源对象

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt cinder    # 输入 CINDER_PASS
    openstack role add --project service --user cinder admin
    openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
    openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
    openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
    openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
    ```

3. 安装软件包

    ```shell
    dnf install openstack-cinder-api openstack-cinder-scheduler
    ```

4. 修改 cinder 配置文件 `/etc/cinder/cinder.conf`

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

    ***注意（关键）***

    - **`[oslo_concurrency] lock_path` 必须配置**：缺省时挂卷 attach 流程中 LIO `create_iscsi_target` 的锁报 `RequiredOptError: lock_path`，卷卡在 reserved/attaching。
    - **`[service_user]` + `service_token_roles = service` 必须配置**：cinder 调用 nova（挂卷后的 update_server_volume 等）需携带 service token；同时需要给 cinder 用户授予 service 角色（见 Nova 章节步骤 5）。
    - **`[nova]` 认证段必须配置**：卷迁移（migrate）流程调 nova `update_server_volume` 时，缺省 token 插件缺少 project domain，报 `BadRequest: Expecting to find domain in project`。

5. 数据库同步

    ```shell
    su -s /bin/sh -c "cinder-manage db sync" cinder
    ```

6. 修改 nova 配置 `/etc/nova/nova.conf`

    ```ini
    [cinder]
    os_region_name = RegionOne
    ```

7. 启动服务

    ```shell
    systemctl restart openstack-nova-api
    systemctl enable --now openstack-cinder-api openstack-cinder-scheduler
    ```

**Storage节点**

Storage 节点要提前准备至少一块硬盘，作为 cinder 的存储后端，下文默认 storage 节点已经存在一块未使用的硬盘，设备名称为 `/dev/sdb`，用户在配置过程中，请按照真实环境信息进行名称替换。

1. 安装软件包

    ```shell
    dnf install lvm2 device-mapper-persistent-data scsi-target-utils targetcli \
      python3-rtslib rpcbind nfs-utils openstack-cinder-volume
    ```

    ***注意（关键）***

    - **必须修复 rtslib_fb 缺失**（targetcli 报 `No module named rtslib_fb.utils`）：Gazpacho 源中 python3-rtslib 提供的是 `rtslib` 包，而 targetcli 2.1.56 引用 `rtslib_fb`；`/usr/lib/python3.11/site-packages/` 下的 `rtslib_fb.py` 是单文件残留，需替换为指向 rtslib 包的链接，否则 LIO target 无法创建、挂卷失败：

    ```shell
    cd /usr/lib/python3.11/site-packages/
    mv rtslib_fb.py rtslib_fb.py.bak
    ln -s rtslib rtslib_fb
    targetcli ls    # 应能正常输出配置树
    ```

    - **必须加载内核 target 模块**（LIO 依赖，且需持久化）：

    ```shell
    cat > /etc/modules-load.d/cinder.conf << EOF
    target_core_mod
    iscsi_target_mod
    target_core_file
    target_core_iblock
    EOF
    modprobe target_core_mod iscsi_target_mod target_core_file target_core_iblock
    ```

    - **必须启动 iscsid**（nova os-brick 登录 iSCSI target 依赖 iscsi 客户端服务，未启动则挂卷卡 attaching）：

    ```shell
    systemctl enable --now iscsid
    ```

2. 配置 lvm 卷组

    ```shell
    pvcreate /dev/sdb
    vgcreate cinder-volumes /dev/sdb
    ```

3. 修改 cinder 配置 `/etc/cinder/cinder.conf`

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

    ***注意（关键）***

    - **`volume_backend_name = lvm` 必须显式配置**：缺省时后端上报的名称为 LVM，与卷类型 `volume_backend_name=lvm` 不匹配，scheduler 报 `No valid backend was found`，卷创建失败。
    - **`lock_path`、内核模块、iscsid、rtslib_fb 四项缺一不可**，任何一项缺失都会导致挂卷/建卷失败（详见坑速查 T9/T16）。

4. 启动服务

    ```shell
    systemctl enable --now openstack-cinder-volume.service
    ```

5. 验证

    ```shell
    source ~/.admin-openrc
    openstack volume service list
    # cinder-volume 应显示 controller@lvm... 等状态 up
    openstack volume type create --property volume_backend_name=lvm lvm
    openstack volume create --type lvm --size 1 testvol
    openstack volume list
    ```

    ***扩展（可选）：多后端***

    如需多后端（volume_migrate/retype 测试需要），在 storage 节点再准备一块盘 `/dev/sdc`，追加第二后端：

    ```shell
    pvcreate /dev/sdc
    vgcreate cinder-volumes-2 /dev/sdc
    ```

    `/etc/cinder/cinder.conf` 修改 `enabled_backends = lvm,lvm2` 并新增：

    ```ini
    [lvm2]
    volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
    volume_group = cinder-volumes-2
    target_protocol = iscsi
    target_helper = lioadm
    volume_backend_name = lvm2
    lvm_type = default
    ```

    ***注意***

    - **`[lvm2] volume_backend_name` 与 `lvm_type = default` 都要配**：backend_name 不配则调度失败；lvm_type 不配时 cinder 会检测 thin pool（新卷组无 thin pool）导致卷创建 ERROR。
    - 重启 `openstack-cinder-volume`，验证 `openstack volume service list` 出现两个 volume 服务。

#### Horizon

Horizon 是 OpenStack 的 Dashboard，提供 Web 管理界面。

**Controller节点**

1. 安装软件包

    ```shell
    dnf install openstack-dashboard python3-rjsmin
    ```

    ***注意（关键）***

    - **`python3-rjsmin` 必须一并安装**：缺省时 horizon 模板渲染报 `rjsmin.jsmin couldn't be imported`，httpd 启动失败——而 keystone 也运行在 httpd 上，会导致整个认证服务不可用。

2. 修改 `/etc/httpd/conf.d/openstack-dashboard.conf`

    ```shell
    # 将 ServerName 改为 controller（原为 openstack-dashboard.example.com，不匹配会导致 403）
    sed -i 's/ServerName .*/ServerName controller/' /etc/httpd/conf.d/openstack-dashboard.conf
    ```

3. 修改 `/etc/openstack-dashboard/local_settings`

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

    # 关键修复项（缺省会导致 400/403/登录后 403 目录错误）
    CSRF_TRUSTED_ORIGINS = ['http://controller', 'http://192.168.0.2']
    LOGIN_URL = '/dashboard/auth/login/'
    LOGIN_REDIRECT_URL = '/dashboard/'
    LOGOUT_REDIRECT_URL = '/dashboard/auth/login/'
    ```

    ***注意（关键）***

    - **`ALLOWED_HOSTS` 必须含 controller**：否则访问 dashboard 报 400。
    - **`CSRF_TRUSTED_ORIGINS` 必须配置**：否则登录表单 POST 报 403（Django CSRF 校验）。
    - **`LOGIN_REDIRECT_URL = '/dashboard/'` 等必须配置**：否则登录成功后重定向到根路径 `/`（无 WSGI 别名，403 目录错误），tempest dashboard 测试失败。
    - **不要配置 `WEBROOT`**：配置后 login 页渲染报 500（模板错误）。

4. 启动服务

    ```shell
    systemctl restart httpd
    ```

5. 验证

    ```shell
    curl -s -o /dev/null -w '%{http_code}\n' http://controller/dashboard/auth/login/
    # 应返回 200；浏览器访问 http://192.168.0.2/dashboard 登录
    ```

#### Swift

Swift 是 OpenStack 的对象存储服务，提供对象、容器的存储。

**Controller节点**

1. 初始化 Keystone 资源对象

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt swift    # 输入 SWIFT_PASS
    openstack role add --project service --user swift admin
    openstack service create --name swift --description "OpenStack Object Storage" object-store
    openstack endpoint create --region RegionOne object-store public http://controller:8080/v1/AUTH_%\(project_id\)s
    openstack endpoint create --region RegionOne object-store internal http://controller:8080/v1/AUTH_%\(project_id\)s
    openstack endpoint create --region RegionOne object-store admin http://controller:8080/v1/AUTH_%\(project_id\)s
    ```

2. 安装软件包

    ```shell
    dnf install openstack-swift-proxy python3-swiftclient python3-keystoneclient \
      python3-keystonemiddleware memcached
    ```

3. 修改 `/etc/swift/proxy-server.conf`

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

    ***注意（关键）***

    - **`operator_roles = admin,user,member` 必须含 member**：Gazpacho 源打包默认只有 `admin,user`，而 tempest 动态用户只有 member 角色，访问 swift 报 403。
    - **`[filter:authtoken]` 的密码必须是真实 SWIFT_PASS**（模板默认是 swift），且注意是在 `[filter:authtoken]` 段（不是 `[filter:keystoneauth]` 段）。

4. 生成 ring 并分发到各节点

    ```shell
    # 每类型 storage 用官方端口：account 6202 / container 6201 / object 6200
    swift-ring-builder account.builder create 10 3 1
    swift-ring-builder container.builder create 10 3 1
    swift-ring-builder object.builder create 10 3 1
    # 以 storage 节点 192.168.0.4 为例，每个 builder 添加对应设备
    swift-ring-builder account.builder add --region 1 --zone 1 --ip 192.168.0.4 --port 6202 --device sdb --weight 100
    swift-ring-builder container.builder add --region 1 --zone 1 --ip 192.168.0.4 --port 6201 --device sdb --weight 100
    swift-ring-builder object.builder add --region 1 --zone 1 --ip 192.168.0.4 --port 6200 --device sdb --weight 100
    swift-ring-builder account.builder rebalance
    swift-ring-builder container.builder rebalance
    swift-ring-builder object.builder rebalance
    # 将 *.ring.gz、*.builder 分发到 storage 节点的 /etc/swift/ 下
    scp /etc/swift/*.ring.gz /etc/swift/*.builder storage:/etc/swift/
    ```

    ***注意（关键）***

    - **ring 与 bind_port 必须一一对应官方端口（6202/6201/6200）**：不能统一用一个端口，否则 container 操作 503。

5. 启动服务

    ```shell
    systemctl enable --now openstack-swift-proxy
    ```

**Storage节点**

1. 安装支持的程序包

    ```shell
    dnf install openstack-swift-account openstack-swift-container openstack-swift-object
    dnf install xfsprogs rsync
    ```

2. 将设备 `/dev/sdb`（如需要容灾可再加一块 `/dev/sdc`）格式化为 XFS

    ```shell
    mkfs.xfs /dev/sdb
    ```

3. 创建挂载点目录结构

    ```shell
    mkdir -p /srv/node/sdb
    ```

4. 找到新分区的 UUID

    ```shell
    blkid
    ```

5. 编辑 `/etc/fstab` 文件并将以下内容添加到其中

    ```bash
    UUID="<UUID-from-output-above>" /srv/node/sdb xfs noatime 0 2
    ```

6. 挂载设备

    ```shell
    mount /srv/node/sdb
    chown -R swift:swift /srv/node
    ```

    ***注意***

    **如果用户不需要容灾功能，以上步骤只需要创建一个设备即可，同时可以跳过下面的 rsync 配置。**

7. （可选）创建或编辑 `/etc/rsyncd.conf` 文件以包含以下内容:

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

    **替换 `MANAGEMENT_INTERFACE_IP_ADDRESS` 为存储节点上管理网络的 IP 地址**

    启动 rsyncd 服务并配置它在系统启动时启动:

    ```shell
    systemctl enable rsyncd.service
    systemctl start rsyncd.service
    ```

8. 修改各 storage 服务配置（account-server.conf / container-server.conf / object-server.conf）

    ```ini
    [DEFAULT]
    bind_ip = 0.0.0.0
    bind_port = 6202   # account；container 为 6201；object 为 6200
    user = swift
    swift_dir = /etc/swift
    devices = /srv/node
    mount_check = false
    ```

    ***注意（关键）***

    - **`bind_ip = 0.0.0.0`**：默认 127.0.0.1，proxy 从 controller 连不上（503 Connection refused）。
    - **`devices = /srv/node` 且目录必须存在**（`/srv/node/<device>`），否则 503 找不到设备。
    - **`mount_check = false`**：默认 true 会检查 device 是否挂载点，未挂载即 503。

9. 启动服务

    ```shell
    systemctl enable --now openstack-swift-account openstack-swift-container openstack-swift-object
    ```

10. 验证

    ```shell
    source ~/.admin-openrc
    openstack container create test
    openstack object create test /etc/hosts
    openstack object list test
    # 无 503 即成功
    ```

#### Heat

Heat 是 OpenStack 的自动编排服务，基于描述性的模板来编排复合云应用，也称为 `Orchestration Service`。Heat 的各服务一般安装在 `Controller` 节点上。

**Controller节点**

1. 创建 **heat** 数据库，并授予 **heat** 数据库正确的访问权限，替换 **HEAT_DBPASS**（示例 Heat2026）为合适的密码

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE heat;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' IDENTIFIED BY 'HEAT_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' IDENTIFIED BY 'HEAT_DBPASS';
    MariaDB [(none)]> exit;
    ```

2. 创建服务凭证，创建 **heat** 用户，并为其增加 **admin** 角色（创建用户时输入的密码即为 `HEAT_PASS`，示例 Heat2026）

    ```shell
    source ~/.admin-openrc

    openstack user create --domain default --password-prompt heat
    openstack role add --project service --user heat admin
    ```

3. 创建 **heat** 和 **heat-cfn** 服务及其对应的 API 端点

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

4. 创建 stack 管理的额外信息

    创建 **heat** domain：

    ```shell
    openstack domain create --description "Stack projects and users" heat
    ```

    在 **heat** domain 下创建 **heat_domain_admin** 用户，并记下输入的密码，用于配置下面的 `HEAT_DOMAIN_PASS`：

    ```shell
    openstack user create --domain heat --password-prompt heat_domain_admin
    ```

    为 **heat_domain_admin** 用户增加 **admin** 角色：

    ```shell
    openstack role add --domain heat --user-domain heat --user heat_domain_admin admin
    ```

    创建 **heat_stack_owner** 角色：

    ```shell
    openstack role create heat_stack_owner
    ```

    创建 **heat_stack_user** 角色：

    ```shell
    openstack role create heat_stack_user
    ```

5. 安装软件包

    ```shell
    dnf install openstack-heat-api openstack-heat-api-cfn openstack-heat-engine
    ```

6. 修改配置文件 `/etc/heat/heat.conf`

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

7. 初始化 **heat** 数据库表

    ```shell
    su -s /bin/sh -c "heat-manage db_sync" heat
    ```

8. 启动服务

    ```shell
    systemctl enable openstack-heat-api.service openstack-heat-api-cfn.service openstack-heat-engine.service
    systemctl start openstack-heat-api.service openstack-heat-api-cfn.service openstack-heat-engine.service
    ```

9. 验证

    ```shell
    source ~/.admin-openrc
    openstack stack list
    openstack orchestration service list
    ```

#### Ironic

Ironic 是 OpenStack 的裸金属服务，如果用户需要进行裸机部署则推荐使用该组件。否则，可以不用安装。

在控制节点执行以下操作。

1. 设置数据库

    裸金属服务在数据库中存储信息，创建一个 **ironic** 用户可以访问的 **ironic** 数据库，替换 **IRONIC_DBPASS**（示例 Ironic2026）为合适的密码：

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE ironic CHARACTER SET utf8;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'localhost' IDENTIFIED BY 'IRONIC_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'%' IDENTIFIED BY 'IRONIC_DBPASS';
    MariaDB [(none)]> exit
    ```

2. 创建服务用户认证

    - 创建 Bare Metal 服务用户：

        替换 `IRONIC_PASS`（示例 Ironic2026）为 ironic 用户密码；如需使用 ironic-inspector（裸机检查服务），再创建 `IRONIC_INSPECTOR_PASS`。

        ```shell
        source ~/.admin-openrc
        openstack user create --domain default --password-prompt ironic
        openstack role add --project service --user ironic admin
        openstack service create --name ironic \
          --description "Ironic baremetal provisioning service" baremetal

        # （可选）ironic-inspector 服务用户
        openstack service create --name ironic-inspector \
          --description "Ironic inspector baremetal provisioning service" baremetal-introspection
        openstack user create --domain default --password-prompt ironic-inspector
        openstack role add --project service --user ironic-inspector admin
        ```

    - 创建 Bare Metal 服务访问入口：

        ```shell
        openstack endpoint create --region RegionOne baremetal admin http://controller:6385
        openstack endpoint create --region RegionOne baremetal public http://controller:6385
        openstack endpoint create --region RegionOne baremetal internal http://controller:6385
        # （可选，对应 ironic-inspector）
        openstack endpoint create --region RegionOne baremetal-introspection internal http://controller:5050/v1
        openstack endpoint create --region RegionOne baremetal-introspection public http://controller:5050/v1
        openstack endpoint create --region RegionOne baremetal-introspection admin http://controller:5050/v1
        ```

3. 安装组件

    ```shell
    dnf install openstack-ironic-api openstack-ironic-conductor python3-ironicclient
    ```

4. 配置 `/etc/ironic/ironic.conf`

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

5. 同步数据库并启动

    ```shell
    ironic-dbsync --config-file /etc/ironic/ironic.conf create_schema
    systemctl enable openstack-ironic-api.service openstack-ironic-conductor.service
    systemctl start openstack-ironic-api.service openstack-ironic-conductor.service
    ```

6. 验证

    ```shell
    source ~/.admin-openrc
    openstack baremetal driver list
    ```

#### Trove

Trove 是 OpenStack 的数据库服务（DBaaS），提供数据库实例的创建、管理等功能。

**Controller节点**

1. 创建数据库、服务凭证和 API 服务端点

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE trove;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'localhost' IDENTIFIED BY 'TROVE_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'%' IDENTIFIED BY 'TROVE_DBPASS';
    MariaDB [(none)]> exit
    ```

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt trove    # 输入 TROVE_PASS
    openstack role add --project service --user trove admin
    openstack service create --name trove --description "Database Service" database
    openstack endpoint create --region RegionOne database public http://controller:8779/v1.0/%\(tenant_id\)s
    openstack endpoint create --region RegionOne database internal http://controller:8779/v1.0/%\(tenant_id\)s
    openstack endpoint create --region RegionOne database admin http://controller:8779/v1.0/%\(tenant_id\)s
    ```

2. 安装软件包

    ```shell
    dnf install openstack-trove python3-troveclient
    ```

3. 修改配置文件

    编辑 `/etc/trove/trove.conf`：

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

    **解释：**

    > `[DEFAULT]` 分组中 `bind_host` 配置为 Trove 控制节点的 IP。
    > `transport_url` 为 RabbitMQ 连接信息，`RABBIT_PASS` 替换为 RabbitMQ 的密码。
    > `[database]` 分组中的 `connection` 为前面在 mysql 中为 Trove 创建的数据库信息。
    > `[keystone_authtoken]` 与 `[service_credentials]` 中 `TROVE_PASS` 替换为实际 trove 用户的密码。

    编辑 `/etc/trove/trove-guestagent.conf`（guestagent 是 trove 中一个独立组件，需要预先内置到 Trove 通过 Nova 创建的虚拟机镜像中，负责通过 RabbitMQ 向 Trove 上报心跳）：

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

    **解释：**

    > `transport_url` 为 RabbitMQ 连接信息，`RABBIT_PASS` 替换为 RabbitMQ 的密码。
    > `[service_credentials]` 中 `TROVE_PASS` 替换为实际 trove 用户的密码。

4. 同步数据库并启动

    ```shell
    su -s /bin/sh -c "trove-manage db_sync" trove
    systemctl enable openstack-trove-api.service openstack-trove-taskmanager.service \
      openstack-trove-conductor.service
    systemctl start openstack-trove-api.service openstack-trove-taskmanager.service \
      openstack-trove-conductor.service
    ```

5. 验证

    ```shell
    source ~/.admin-openrc
    openstack database service list
    ```

#### Cyborg

Cyborg 是 OpenStack 的加速器管理服务，负责加速器设备的发现、管理与挂载。

**Controller节点**

1. 创建数据库、服务凭证和 API 服务端点

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE cyborg;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON cyborg.* TO 'cyborg'@'localhost' IDENTIFIED BY 'CYBORG_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON cyborg.* TO 'cyborg'@'%' IDENTIFIED BY 'CYBORG_DBPASS';
    MariaDB [(none)]> exit
    ```

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt cyborg    # 输入 CYBORG_PASS
    openstack role add --project service --user cyborg admin
    openstack service create --name cyborg --description "Acceleration Service" accelerator
    openstack endpoint create --region RegionOne accelerator public http://controller:9101
    openstack endpoint create --region RegionOne accelerator internal http://controller:9101
    openstack endpoint create --region RegionOne accelerator admin http://controller:9101
    ```

2. 安装软件包

    ```shell
    dnf install openstack-cyborg
    ```

3. 修改 `/etc/cyborg/cyborg.conf`

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

    ***注意（关键）***

    - **`[placement]` 认证段必须配置**：cyborg conductor 的 PlacementClient 依赖该段，缺省会导致服务异常。

4. 创建 WSGI 入口并修改 unit（关键）

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

    # 修改 /usr/lib/systemd/system/openstack-cyborg-api.service 的 ExecStart 为：
    # ExecStart=/usr/bin/gunicorn -b 0.0.0.0:9101 --workers 4 --threads 4 cyborg_wsgi:application
    systemctl daemon-reload
    ```

    ***注意（关键）***

    - **cyborg-api 必须用 gunicorn + 包装入口部署**：先初始化 RPC/objects 再创建 application，否则 gunicorn 下 argv 解析崩溃；端口为 9101。

5. 同步数据库并启动

    ```shell
    cyborg-dbsync upgrade
    systemctl enable --now openstack-cyborg-api openstack-cyborg-conductor
    ```

6. 验证

    ```shell
    source ~/.admin-openrc
    openstack accelerator device list
    curl -s -o /dev/null -w '%{http_code}\n' http://controller:9101
    # 应返回 401/200（服务可达）
    ```

**Compute节点**

1. 安装并配置 agent

    ```shell
    dnf install openstack-cyborg
    # /etc/cyborg/cyborg.conf 配置同 Controller 节点（[keystone_authtoken]/[placement]/[DEFAULT] host=compute）
    systemctl enable --now openstack-cyborg-agent
    ```

#### Aodh

Aodh 是 OpenStack 的告警服务，提供阈值告警、组合告警等功能。

**Controller节点**

1. 创建数据库、服务凭证和 API 服务端点

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE aodh;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'localhost' IDENTIFIED BY 'AODH_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'%' IDENTIFIED BY 'AODH_DBPASS';
    MariaDB [(none)]> exit
    ```

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt aodh    # 输入 AODH_PASS
    openstack role add --project service --user aodh admin
    openstack service create --name aodh --description "Telemetry" alarming
    openstack endpoint create --region RegionOne alarming public http://controller:8042
    openstack endpoint create --region RegionOne alarming internal http://controller:8042
    openstack endpoint create --region RegionOne alarming admin http://controller:8042
    ```

2. 安装软件包

    ```shell
    dnf install openstack-aodh-api openstack-aodh-evaluator openstack-aodh-notifier \
      openstack-aodh-listener openstack-aodh-expirer python3-aodhclient \
      python3-observabilityclient
    ```

3. 修改 `/etc/aodh/aodh.conf`

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

4. 修改 aodh-api 的 unit（关键）

    ```shell
    # /usr/lib/systemd/system/openstack-aodh-api.service 的 ExecStart 加 --port 8042（在 "--" 分隔符之前）
    # 示例：ExecStart=... --port 8042 --config-file /etc/aodh/aodh.conf ...
    systemctl daemon-reload
    ```

    ***注意（关键）***

    - **aodh-api 必须显式指定 `--port 8042`**：默认监听 8000，与 heat-api-cfn（8000）冲突，报 `Address already in use`。

5. 同步数据库并启动

    ```shell
    aodh-dbsync
    systemctl enable --now openstack-aodh-api openstack-aodh-evaluator \
      openstack-aodh-notifier openstack-aodh-listener
    ```

6. 验证

    ```shell
    source ~/.admin-openrc
    openstack alarm list
    curl -s -o /dev/null -w '%{http_code}\n' http://controller:8042
    ```

#### Gnocchi

Gnocchi 是 OpenStack 的指标存储服务，为 Ceilometer/Aodh 提供时间序列数据库。

**Controller节点**

1. 创建数据库、服务凭证和 API 服务端点

    ```sql
    mysql -u root -p

    MariaDB [(none)]> CREATE DATABASE gnocchi;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'localhost' IDENTIFIED BY 'GNOCCHI_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'%' IDENTIFIED BY 'GNOCCHI_DBPASS';
    MariaDB [(none)]> exit
    ```

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt gnocchi    # 输入 GNOCCHI_PASS
    openstack role add --project service --user gnocchi admin
    openstack service create --name gnocchi --description "Metric Service" metric
    openstack endpoint create --region RegionOne metric public http://controller:8041
    openstack endpoint create --region RegionOne metric internal http://controller:8041
    openstack endpoint create --region RegionOne metric admin http://controller:8041
    ```

2. 安装软件包

    ```shell
    dnf install openstack-gnocchi-api openstack-gnocchi-metricd python3-gnocchiclient
    ```

3. 修改 `/etc/gnocchi/gnocchi.conf`

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

    ***注意（关键）***

    - **`[indexer] url` 必须用 mysql**：Gazpacho 4.6.0 无 file indexer，默认 `file:///` 会导致 gnocchi-upgrade 卡死。

4. 升级数据库并启动

    ```shell
    gnocchi-upgrade    # 如卡住请确认 [indexer] url 为 mysql
    systemctl enable --now openstack-gnocchi-api openstack-gnocchi-metricd
    ```

5. 验证

    ```shell
    source ~/.admin-openrc
    openstack metric list
    curl -s -o /dev/null -w '%{http_code}\n' http://controller:8041/v1/status
    ```

#### Ceilometer

Ceilometer 是 OpenStack 中负责数据收集的服务。

**Controller节点**

1. 创建服务凭证和 API 服务端点

    ```shell
    source ~/.admin-openrc
    openstack user create --domain default --password-prompt ceilometer    # 输入 CEILOMETER_PASS
    openstack role add --project service --user ceilometer admin
    openstack service create --name ceilometer --description "Telemetry" metering
    openstack endpoint create --region RegionOne metering public http://controller:8777
    openstack endpoint create --region RegionOne metering internal http://controller:8777
    openstack endpoint create --region RegionOne metering admin http://controller:8777
    ```

    ***注意***

    - Gazpacho 的 ceilometer 无独立 API 监听（8777 MISS 属正常），通过通知/样本验证。

2. 安装 Ceilometer 软件包

    ```shell
    dnf install openstack-ceilometer-notification openstack-ceilometer-central python3-xmltodict
    ```

3. 编辑配置文件 `/etc/ceilometer/pipeline.yaml`

    ```yaml
    publishers:
        # set address of Gnocchi
        # + filter out Gnocchi-related activity meters (Swift driver)
        # + set default archive policy
        - gnocchi://?filter_project=service&archive_policy=low
    ```

4. 编辑配置文件 `/etc/ceilometer/ceilometer.conf`

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

5. 数据库同步

    ```shell
    ceilometer-upgrade
    ```

6. 完成控制节点 Ceilometer 安装

    ```shell
    systemctl enable openstack-ceilometer-notification.service openstack-ceilometer-central.service
    systemctl start openstack-ceilometer-notification.service openstack-ceilometer-central.service
    ```

**Compute节点**

1. 安装 Ceilometer 软件包

    ```shell
    dnf install openstack-ceilometer-compute
    ```

2. 编辑配置文件 `/etc/ceilometer/ceilometer.conf`

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

3. 编辑配置文件 `/etc/nova/nova.conf`

    ```ini
    [DEFAULT]
    instance_usage_audit = True
    instance_usage_audit_period = hour

    [notifications]
    notify_on_state_change = vm_and_task_state

    [oslo_messaging_notifications]
    driver = messagingv2
    ```

4. 完成安装

    ```shell
    systemctl enable openstack-ceilometer-compute.service
    systemctl start openstack-ceilometer-compute.service

    # 重启 nova-compute 服务
    systemctl restart openstack-nova-compute.service
    ```

#### Tempest

Tempest 是 OpenStack 的集成测试套件，用于验证各服务的功能正确性。

**Controller节点**

1. 安装软件包并初始化

    ```shell
    dnf install openstack-tempest
    tempest init /opt/tempest
    cd /opt/tempest
    ```

2. 配置 `/opt/tempest/etc/tempest.conf`

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
    image_ref = <cirros镜像id>
    image_ref_alt = <cirros镜像id>
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
    public_network_id = <public网络id>
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

    ***注意（关键）***

    - **`[auth] default_credentials_domain_name = Default` 必须配置**：dashboard 测试登录表单的 domain 参数依赖它。
    - **`[compute] image_ref_alt` 必须配置**：rebuild 测试需要，缺省时 rebuild 报 `imageRef: None`。
    - **`[volume] catalog_type = volumev3` 必须配置**：缺省时 volume 类测试报 `EndpointNotFound`。
    - **`[volume] backend_names` 在 [volume] 段**（不要写到 volume-feature-enabled 段），逗号分隔，与 cinder 后端 volume_backend_name 一致；配错时卷类型映射为 BACKEND_1/2，调度失败。
    - **`project_networks_reachable = false`**：连通性验证走 floating IP；如 controller 可直接路由到租户网络可设为 true。
    - **快照/IPv6/QoS 相关开关**（snapshot=true、ipv6、qos_placement_physnet、update_port_qos）用于启用对应 scenario 测试类。

3. 运行测试

    ```shell
    # 全部功能测试（scenario）
    cd /opt/tempest
    tempest run --regex '^tempest.scenario' --concurrency 1
    # 单服务 API 测试
    tempest run --regex '^tempest.api.compute'
    ```

    ***注意***

    - 虚机类测试依赖虚机 SSH 可达：需确保 nova-api-metadata 已启动（`systemctl enable --now openstack-nova-metadata-api`）、nova.conf `[neutron] service_metadata_proxy = true` 且 `metadata_proxy_shared_secret` 与 metadata_agent.ini 一致，否则虚机拿不到密钥、SSH 拒绝。
    - 无 KVM 环境下虚机启动慢（qemu），SSH 类测试需耐心等待。

4. 安装 tempest 扩展（可选）

    OpenStack 各个服务本身也提供了一些 tempest 测试包，用户可以安装这些包来丰富 tempest 的测试内容。在 Gazpacho 中，提供了 Cinder、Glance、Keystone、Ironic、Trove 的扩展测试，用户可以执行如下命令进行安装使用：

    ```shell
    dnf install python3-cinder-tempest-plugin python3-glance-tempest-plugin \
      python3-ironic-tempest-plugin python3-keystone-tempest-plugin \
      python3-trove-tempest-plugin
    ```

---

*文档结束。如有疑问请联系部署维护人员。*
