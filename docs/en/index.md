# openEuler OpenStack SIG

## SIG Work Objectives and Scope

- Provide native OpenStack on openEuler, building an open and reliable cloud computing technology stack.
- Hold regular meetings, collect developer and vendor needs, and discuss OpenStack community development.

## Organizational Meetings

Public meeting time: Monthly meeting, some Wednesday afternoon from 3:00-4:00 PM (Beijing Time) in the middle to late month.

Meeting link: Announced via WeChat group messages and mailing list.

Meeting notes: <https://etherpad.openeuler.org/p/sig-openstack-meetings>

## OpenStack Version Support List

OpenStack SIG collects OpenStack version requirements through user feedback and other methods. After open discussion among SIG members, the OpenStack version roadmap is determined. Versions under planning may be adjusted due to requirement changes, manpower changes, etc. OpenStack SIG welcomes more developers and vendors to participate and jointly improve openEuler's OpenStack support.

● - Supported
○ - Planned/Under Development
▲ - Supported on some openEuler versions

|                         | Queens | Rocky | Train | Ussuri | Victoria | Wallaby | Xena | Yoga | Antelope |
|:-----------------------:|:------:|:-----:|:-----:|:------:|:--------:|:-------:|:----:|:----:|:--------:|
| openEuler 20.03 LTS SP1 |        |       |   ●   |        |          |         |      |      |          |
| openEuler 20.03 LTS SP2 |    ●   |   ●   |       |        |          |         |      |      |          |
| openEuler 20.03 LTS SP3 |    ●   |   ●   |   ●   |        |          |         |      |      |          |
| openEuler 20.03 LTS SP4 |        |       |   ●   |        |          |         |      |      |          |
| openEuler 21.03         |        |       |       |        |     ●    |         |      |      |          |
| openEuler 21.09         |        |       |       |        |          |    ●    |      |      |          |
| openEuler 22.03 LTS     |        |       |   ●   |        |          |    ●    |      |      |          |
| openEuler 22.03 LTS SP1 |        |       |   ●   |        |          |    ●    |      |      |          |
| openEuler 22.03 LTS SP2 |        |       |   ●   |        |          |    ●    |      |      |          |
| openEuler 22.03 LTS SP3 |        |       |   ●   |        |          |    ●    |      |      |          |
| openEuler 22.03 LTS SP4 |        |       |   ●   |        |          |    ●    |      |      |          |
| openEuler 22.09         |        |       |       |        |          |         |      |   ●  |     ●    |
| openEuler 24.03 LTS     |        |       |       |        |          |    ●    |      |      |     ●    |
| openEuler 24.03 LTS SP1 |        |       |       |        |          |    ●    |      |      |     ●    |
| openEuler 24.03 LTS SP2 |        |       |       |        |          |    ●    |      |      |     ●    |
| openEuler 24.03 LTS SP4 |        |       |       |        |          |    ●    |      |      |     ●    |

|            | Queens | Rocky | Train | Victoria | Wallaby | Yoga | Antelope |
|:---------: |:------:|:-----:|:-----:|:--------:|:-------:|:----:|:--------:|
|  Keystone  |    ●   |   ●   |   ●   |     ●    |    ●    |   ●  |    ●     |
|   Glance   |    ●   |   ●   |   ●   |     ●    |    ●    |   ●  |    ●     |
|    Nova    |    ●   |   ●   |   ●   |     ●    |    ●    |   ●  |    ●     |
|   Cinder   |    ●   |   ●   |   ●   |     ●    |    ●    |   ●  |    ●     |
|  Neutron   |    ●   |   ●   |   ●   |     ●    |    ●    |   ●  |    ●     |
|  Tempest   |    ●   |   ●   |   ●   |     ●    |    ●    |   ●  |    ●     |
|  Horizon   |    ●   |   ●   |   ●   |     ●    |    ●    |   ●  |    ●     |
|   Ironic   |    ●   |   ●   |   ●   |     ●    |    ●    |   ●  |    ●     |
| Placement  |        |       |   ●   |     ●    |    ●    |   ●  |    ●     |
|   Trove    |    ●   |   ●   |   ●   |          |    ●    |   ●  |    ●     |
|   Kolla    |    ●   |   ●   |   ●   |          |    ●    |   ●  |    ●     |
|   Rally    |    ▲   |   ▲   |       |          |         |      |          |
|   Swift    |        |       |   ●   |          |    ●    |   ●  |    ●     |
|    Heat    |        |       |   ●   |          |    ▲    |   ●  |     ●    |
| Ceilometer |        |       |   ●   |          |    ▲    |   ●  |    ●     |
|    Aodh    |        |       |   ●   |          |    ▲    |   ●  |    ●     |
|   Cyborg   |        |       |   ●   |          |    ▲    |   ●  |    ●     |
|   Gnocchi  |        |       |   ●   |          |    ●    |   ●  |    ●     |
| OpenStack-helm |    |       |       |          |         |   ●  |    ●     |
|  Barbican  |        |       |       |          |    ▲    |      |    ●     |
|  Octavia   |        |       |       |          |    ▲    |      |    ●     |
|  Designate |        |       |       |          |    ▲    |      |    ●     |
|  Manila    |        |       |       |          |    ▲    |      |    ●     |
|  Masakari  |        |       |       |          |    ▲    |      |    ●     |
|  Mistral   |        |       |       |          |    ▲    |      |    ●     |
|  Senlin    |        |       |       |          |    ▲    |      |    ●     |
|  Zaqar     |        |       |       |          |    ▲    |      |    ●     |

Note:

1. openEuler 20.03 LTS SP2 does not support Rally
2. Heat, Ceilometer, Swift, Aodh and Cyborg are only supported on openEuler 22.03 LTS and above
3. Barbican, Octavia, Designate, Manila, Masakari, Mistral, Senlin and Zaqar are only supported on openEuler 22.03 LTS SP2 and above

## oepkg Software Repository Address List

Support for Queens, Rocky, and Train versions is hosted on SIG's officially certified third-party software platform oepkg:

- 20.03-LTS-SP1 Train: <https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP1/contrib/openstack/train/>
<https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP1/contrib/openstack/train/>
    This Train version is not pure native code and includes related code for smart NIC support. Users should review it themselves before use.

- 20.03-LTS-SP2 Rocky: <https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP2/budding-openeuler/openstack/queens/>

- 20.03-LTS-SP3 Rocky: <https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP3/budding-openeuler/openstack/rocky/>

- 20.03-LTS-SP2 Queens: <https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP2/budding-openeuler/openstack/queens/>

- 20.03-LTS-SP3 Rocky: <https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP3/budding-openeuler/openstack/rocky/>

Additionally, although 20.03-LTS-SP1 has Queens and Rocky version packages, they have not been verified. Please use with caution:

- 20.03-LTS-SP1 Queens: <https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP1/contrib/openstack/queens/>

- 20.03-LTS-SP1 Rocky: <https://repo.oepkgs.net/openEuler/rpm/openEuler-20.03-LTS-SP1/contrib/openstack/rocky/>

## Maintainer Join and Exit

Following the open source and open philosophy, OpenStack SIG also has certain norms and requirements for maintainer member joining and exiting.

### How to Become a Maintainer

As the direct person in charge of SIG, maintainers have rights in code merging, roadmap planning, and nominating maintainers. At the same time, they also have obligations in software quality guardianship and version development. If you want to become a maintainer of OpenStack SIG, you need to meet the following requirements:

1. Continuously participate in OpenStack SIG development contributions for no less than one openEuler release cycle (usually 3 months)
2. Continuously participate in OpenStack SIG code review, and your review ranking should not be lower than the SIG average
3. Attend OpenStack SIG regular meetings on time (usually bi-weekly), a typical openEuler release cycle includes 6 meetings, and the number of absences should not exceed 2 times

Bonus items:

1. Actively participate in various activities organized by OpenStack SIG, such as online sharing, offline meetups, or summits.
2. Help SIG expand its operational scope and conduct joint technology innovation, such as proactively open-sourcing new projects and attracting new developers and vendors to join the SIG.

SIG maintainers will hold closed-door meetings each quarter to review current contribution data. After contributors meet the relevant requirements and reach consensus through discussion, and are willing to serve as maintainers, SIG will submit relevant applications to openEuler TC.

### Maintainer Exit

When SIG maintainers cannot continue to serve as maintainers due to personal reasons (job changes, business adjustments, etc.), they may apply to exit voluntarily.

SIG maintainers also review the current maintainer list every six months. If contributors are found to be no longer suitable to serve as maintainers (insufficient contributions, inactivity, etc.), after reaching consensus through discussion, they will submit relevant applications to openEuler TC.

### Maintainer List

| Name | Gitee ID | Email | Company |
|---|---|---|---|
| `Chen Shuo` | [`joec88`](https://atomgit.com/joec88)|<joseph.chn1988@gmail.com>|China Unicom|
| `Li Kunshan` | [`liksh`](https://atomgit.com/liksh)|<li_kunshan@163.com>|China Unicom|
| `Huang Tianhua` | [`huangtianhua`](https://atomgit.com/huangtianhua)|<huangtianhua223@gmail.com>|Huawei|
| `Wang Xiyuan` | [`xiyuanwang`](https://atomgit.com/xiyuanwang)|<wangxiyuan1007@gmail.com>|Huawei|
| `Zhang Fan` | [`zh-f`](https://atomgit.com/zh-f)|<zh.f@outlook.com>|China Telecom|
| `Zhang Ying` | [`zhangy1317`](https://atomgit.com/zhangy1317)|<zhangy1317@foxmail.com>|China Unicom|
| `Han Guangyu` | [`han-guangyu`](https://atomgit.com/han-guangyu)|<hanguangyu@uniontech.com>|UnionTech Software|
| `Wang Dongxing` | [`desert-sailor`](https://atomgit.com/desert-sailor)|<dongxing.wang_a@thundersoft.com>|Thundersoft|
| `Zheng Ting` | [`tzing_t`](https://atomgit.com/tzing_t)|<zhengting13@huawei.com>|Huawei|

## How to Contribute

OpenStack SIG follows the four Open principles of the OpenStack community (Open source, Open Design, Open Development, Open Community) and welcomes developers, users, and vendors to participate in SIG contributions in various open source ways, including but not limited to:

1. [Submit Issues](https://atomgit.com/openeuler/openstack/issues/new)
    If you encounter any problems using OpenStack, you can submit ISSUES to SIG, including usage questions, software package bugs, feature requests, etc.
2. Participate in Technical Discussions
   Discuss OpenStack technology with SIG members in real-time through mailing lists, WeChat groups, online meetings, etc.
3. Participate in SIG Software Development and Testing
    1. OpenStack SIG follows the openEuler version development rhythm and releases different versions of OpenStack every few months. Each version includes hundreds of RPM packages, and developers can participate in the development of these RPM packages.
    2. OpenStack SIG includes some vendor-donated and self-developed projects. Developers can participate in the development of related projects.
    3. After openEuler new versions are released, users can test the corresponding OpenStack, and related bugs and issues can be submitted to SIG.
    4. OpenStack SIG also provides a series of tools and documents to improve development efficiency. Users can help optimize and improve them.
4. Technology Prediction, Joint Innovation
   OpenStack SIG welcomes various forms of joint innovation and invites developers to create cloud computing technology for China in an open-source way, using SIG as a platform. If you have ideas or development intentions, welcome to join SIG.

Of course, contribution forms are not limited to these. Any OpenStack-related or open source-related matters can be brought to SIG. OpenStack SIG welcomes your participation.

## Project List

Complete project list of SIG: <https://atomgit.com/openeuler/openstack/blob/master/tools/oos/etc/openeuler_sig_repo.yaml>

OpenStack contains many projects. For easy management, a unified entry project is set up. Users and developers with any questions about OpenStack SIG and various OpenStack subprojects can submit Issues in this project.

- <https://atomgit.com/openeuler/openstack>

SIG has also jointly created a series of self-developed projects with major vendors and developers:

- <https://atomgit.com/openeuler/openstack-kolla-ansible-plugin>
- <https://atomgit.com/openeuler/openstack-kolla-plugin>
- <https://atomgit.com/openeuler/hostha>
- <https://atomgit.com/openeuler/opensd>

## Communication Group

Add the assistant and reply "join group" to enter the openEuler sig-OpenStack communication group.
![assistant](img/install/wechat_group_assistant.jpg)
