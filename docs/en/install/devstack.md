# Installing OpenStack Using DevStack

[TOC]

Currently, the upstream DevStack project natively supports installing OpenStack on openEuler. openEuler 20.03 LTS SP2 has been verified and is backed by official upstream CI quality assurance. Other versions of openEuler require users to test on their own (verified on openEuler master branch as of 2022-04-25).

## Installation Steps

Prepare an openEuler environment, 20.03 LTS SP2 [virtual machine image address](https://repo.openeuler.org/openEuler-20.03-LTS-SP2/virtual_machine_img/), master [virtual machine image address](http://121.36.84.172/dailybuild/EBS-openEuler-Mainline/)

1. Configure yum repository

    **openEuler 20.03 LTS SP2**:

    The openEuler official repository is missing some RPM packages required by OpenStack, so you need to first configure the RPM repository prepared by OpenStack SIG in oepkg

    ```shell
    vi /etc/yum.repos.d/openeuler.repo

    [openstack]
    name=openstack
    baseurl=https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP2/budding-openeuler/openstack-master-ci/aarch64/
    enabled=1
    gpgcheck=0
    ```

    **openEuler master**:

    Use the master RPM repository:

    ```shell
    vi /etc/yum.repos.d/openeuler.repo

    [mainline]
    name=mainline
    baseurl=http://119.3.219.20:82/openEuler:/Mainline/standard_aarch64/
    gpgcheck=false

    [epol]
    name=epol
    baseurl=http://119.3.219.20:82/openEuler:/Epol/standard_aarch64/
    gpgcheck=false
    ```

2. Pre-installation preparation

    **openEuler 20.03 LTS SP2**:

    In some versions of openEuler official images, the EPOL-update URL may be incorrectly configured and needs to be modified

    ```shell
    vi /etc/yum.repos.d/openEuler.repo

    # Change the [EPOL-UPDATE] URL to
    baseurl=http://repo.openeuler.org/openEuler-20.03-LTS-SP2/EPOL/update/main/$basearch/
    ```

    **openEuler master**:

    ```shell
    yum remove python3-pip # System pip conflicts with devstack pip, need to remove it first
    # The master VM environment is missing some dependencies that devstack does not automatically install, need to manually install them
    yum install iptables tar wget python3-devel httpd-devel iscsi-initiator-utils libvirt python3-libvirt qemu memcached
    ```

3. Download devstack

    ```shell
    yum update
    yum install git
    cd /opt/
    git clone https://opendev.org/openstack/devstack.git
    ```

4. Initialize devstack environment configuration

    ```shell
    # Create stack user
    /opt/devstack/tools/create-stack-user.sh
    # Modify directory permissions
    chown -R stack:stack /opt/devstack
    chmod -R 755 /opt/devstack
    chmod -R 755 /opt/stack
    # Switch to the branch for the OpenStack version to be deployed, using yoga as an example; if not switched, the default installation is master version of OpenStack
    git checkout stable/yoga
    ```

5. Initialize devstack configuration file

    ```shell
    Switch to stack user
    su stack
    At this point, please verify if the stack user's PATH environment variable includes `/usr/sbin`; if not, execute
    PATH=$PATH:/usr/sbin
    Create new configuration file
    vi /opt/devstack/local.conf

    [[local|localrc]]
    DATABASE_PASSWORD=root
    RABBIT_PASSWORD=root
    SERVICE_PASSWORD=root
    ADMIN_PASSWORD=root
    OVN_BUILD_FROM_SOURCE=True
    ```

    openEuler does not provide OVN RPM packages, so you need to configure `OVN_BUILD_FROM_SOURCE=True` to compile OVN from source

    Additionally, if using an arm64 virtual machine environment, you need to configure libvirt nested virtualization; append the following configuration to `local.conf`:

    ```shell
    [[post-config|$NOVA_CONF]]
    [libvirt]
    cpu_mode=custom
    cpu_model=cortex-a72
    ```

    If installing Ironic, you need to install dependencies in advance:

    ```bash
    sudo dnf install syslinux-nonlinux
    ```

    **openEuler master special configuration**: Since devstack has not yet been adapted for the latest openEuler, we need to manually fix some issues:

    1. Modify devstack source code

        ```shell
        vi /opt/devstack/tools/fixup_stuff.sh
        Delete all echo statements in the fixup_openeuler method
        (echo '[openstack-ci]'
        echo 'name=openstack'
        echo 'baseurl=https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP2/budding-openeuler/openstack-master-ci/'$arch'/'
        echo 'enabled=1'
        echo 'gpgcheck=0') | sudo tee -a /etc/yum.repos.d/openstack-master.repo > /dev/null
        ```

    2. Modify requirements source code

        The Yoga version of keystone dependency `setproctitle`'s devstack default version does not support python3.10 and needs to be upgraded; manually download the requirements project and modify it

        ```shell
        cd /opt/stack
        git clone https://opendev.org/openstack/requirements --branch stable/yoga
        vi /opt/stack/requirements/upper-constraints.txt
        setproctitle===1.2.3
        ```

    3. OpenStack horizon has a BUG and cannot be installed normally. Here we temporarily do not install horizon; modify `local.conf` and add a new line:

        ```shell
        [[local|localrc]]
        disable_service horizon
        ```

        If there is indeed a need for horizon, the following issues need to be resolved:

        ```shell
        # 1. horizon depends on pyScss with default version 1.3.7, which does not support python3.10
        # Solution: Need to clone the `requirements` project in advance and modify the code
        vi /opt/stack/requirements/upper-constraints.txt
        pyScss===1.4.0

        # 2. horizon depends on httpd's mod_wsgi plugin, but currently openEuler's mod_wsgi build is abnormal (2022-04-25) (after solving, yum install mod_wsgi will work), cannot be installed from yum
        # Solution: Manually build mod_wsgi from source and configure, this process is complex and will be skipped here
        ```

    4. The dstat service depends on `pcp-system-tools` which has abnormal build (2022-04-25) (after solving, yum install pcp-system-tools will work), cannot be installed from yum; temporarily do not install dstat

        ```shell
        [[local|localrc]]
        disable_service dstat
        ```

6. Deploy OpenStack

    Go to the devstack directory and execute `./stack.sh`, wait for OpenStack installation and deployment to complete.
