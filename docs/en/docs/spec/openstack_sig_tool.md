
# openEuler OpenStack Development Platform

The openEuler OpenStack SIG was established in 2021, jointly invested and maintained by developers from companies such as China Unicom, China Telecom, Huawei, and UnionTech. It aims to provide native OpenStack on openEuler and build an open and reliable cloud computing technology stack, making it a benchmark SIG for openEuler. However, OpenStack itself is technically complex and contains numerous services. The development threshold is high, and there are high requirements for contributors' technical capabilities. Labor costs remain high, and there are various problems in actual development and contribution. To solve the problems faced by the SIG, there is an urgent need for an openEuler+OpenStack solution to reduce developer barriers, lower investment costs, improve development efficiency, and ensure the SIG's continuous activity and sustainable development.

## 1. Overview

### 1.1 Current Status

Currently, with the continuous development of the SIG, we have encountered the following problems:

1. OpenStack technology is complex, involving various technologies of cloud IaaS layer such as computing, networking, storage, images, and authentication. It is difficult for developers to master all aspects, and the submitted code logic and quality are concerning.
2. OpenStack is written in Python, and Python software dependency issues are difficult to handle. Taking OpenStack Wallaby version as an example, it involves 400+ core Python software packages. The dependency levels and dependency versions of each software are complex and intricate, making selection difficult and hard to form a closed loop.
3. There are many OpenStack software packages, and the workload for RPM Spec development is enormous. With the continuous evolution of openEuler and OpenStack versions, the N:N adaptation relationship will cause workload to multiply, and labor costs will increase.
4. The testing threshold for OpenStack is too high. Not only do developers need to be familiar with OpenStack, but they also need to have some understanding and mastery of Linux underlying technologies such as virtualization, virtual bridges, and block storage. Deploying an OpenStack environment takes too long, and functional testing is extremely difficult. Moreover, there are many test scenarios, such as X86 and ARM64 architecture testing, bare metal and virtual machine type testing, OVS and OVN bridge testing, LVM and Ceph storage testing, etc., which further increase labor costs and technical barriers.

### 1.2 Solution

To address the above problems faced by the SIG, standardization, tool-based approaches, and automation are imperative. This design document aims to provide an end-to-end available development solution for the openEuler OpenStack SIG. From technical standards to technical implementation, it proposes strict standard requirements and design solutions to meet the daily development needs of SIG developers, reduce development costs, decrease labor investment, lower development barriers, thereby improving development efficiency, improving SIG software quality, developing the SIG ecosystem, and attracting more developers to join the SIG. The main actions are as follows:

1. Output RPM SPEC development standards for OpenStack service software and dependency library software. Developers and Reviewers must strictly comply with standards for development and implementation.
2. Provide OpenStack Python software dependency analysis functionality, one-click generation of dependency topology and results, ensuring dependency closure and avoiding software dependency risks.
3. Provide OpenStack RPM spec generation functionality, one-click generation of RPM specs for common software, shortening development time and reducing investment costs.
4. Provide automated deployment and testing platform functionality, enabling one-click deployment of specified OpenStack versions on any openEuler version, rapid testing and iteration.
5. Provide openEuler Gitee repository automated processing capabilities to meet the need for batch software modifications, such as creating code branches, creating repositories, submitting Pull Requests, etc.

The above solutions can be unified into one system platform, called OpenStack SIG Tool (hereinafter referred to as oos), which is the openEuler OpenStack development platform. The specific architecture is as follows:

```ini
            ┌────────────────────┐        ┌─────────────────────┐
            │         CLI        │        │         GUI         │
            └─────┬─────────┬────┘        └──────────┬──────────┘
                  │         │                        │
          Built-in│         └───────────┬────────────┘
                  │                     │REST
┌─────────────────▼─────────────────────▼────────────────────────────────┐
│                       OpenStack Develop Platform                       │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
          ┌────────────────────┬────┴─────────────┬────────────────┐
          │                    │                  │                │
┌─────────▼─────────┐  ┌───────▼───────┐  ┌───────▼───────┐  ┌─────▼─────┐
│Dependency Analysis│  │SPEC Generation│  │Deploy and Test│  │Code Action│
└───────────────────┘  └───────────────┘  └───────────────┘  └───────────┘
```

This architecture has two main modes:

1. Client/Server mode
     In this mode, oos is deployed as a Web Server, and Client calls oos through REST.
     - Advantages: Provides asynchronous invocation capability, supports concurrent processing, supports record persistence.
     - Disadvantages: Has certain installation and deployment costs, and usage is relatively inflexible.

2. Built-in mode
     In this mode, oos does not need to be deployed and provides services as a built-in CLI. Users directly call various functions through CLI.
     - Advantages: No deployment needed, can be used anytime and anywhere.
     - Disadvantages: No persistence capability, does not support concurrency, single user.

## 2. Detailed Design

### 2.1 OpenStack Spec Standards

Spec standards are one or more spec templates. For each keyword and build section of RPM spec, relevant content is strictly regulated. Developers must meet the standard requirements when writing specs; otherwise, code will not be allowed to be merged. The standard content is discussed publicly by SIG maintainers to form conclusions and is regularly reviewed and updated. Anyone has the right to raise questions and suggestions about standards. Maintainers are responsible for explanation and updates. There are currently two types of standards:

1. Service software standards
  These software take OpenStack core services such as Nova, Neutron, and Cinder as examples. They generally have high customization requirements and vary greatly in content, requiring manual writing. Standards should clearly specify the software layering method, build method, package composition content, testing methods, version number rules, etc.

2. Common dependency software standards
  These software generally have low customization and similar content structure, suitable for one-click generation by automated tools. We only need to define the generation rules for related tools in the standards.

#### 2.1.1 Service Software Standards

Each OpenStack service usually contains several sub-services. For these sub-services, we also need to split them into sub-RPM packages during packaging. This chapter specifies the principles for RPM package splitting of OpenStack services for openEuler SIG.

##### 2.1.1.1 General Principles

Adopt a layered architecture. The RPM package structure is shown in the figure below, using openstack-nova as an example:

```ini
Level | Package                                                                       | Example
      |                                                                               |
 ┌─┐  |                       ┌──────────────┐        ┌────────────────────────┐      | ┌────────────────────┐ ┌────────────────────────┐
 │1│  |                       │ Root Package │        │ Doc Package (Optional) │      | │ openstack-nova.rpm │ │ openstack-nova-doc.rpm │
 └─┘  |                       └────────┬─────┘        └────────────────────────┘      | └────────────────────┘ └────────────────────────┘
      |                                │                                              |
      |          ┌─────────────────────┼───────────────────────────┐                  |
      |          │                     │                           │                  |
 ┌─┐  | ┌────────▼─────────┐ ┌─────────▼────────┐                  │                  | ┌────────────────────────────┐ ┌────────────────────────┐
 │2│  | │ Service1 Package │ │ Service2 Package │                  │                  | │ openstack-nova-compute.rpm │ │ openstack-nova-api.rpm │
 └─┘  | └────────┬─────────┘ └────────┬─────────┘                  │                  | └────────────────────────────┘ └────────────────────────┘
      |          |                    │                            │                  |
      |          └──────────┬─────────┘                            │                  |
      │                     │                                      │                  |
 ┌─┐  |             ┌───────▼────────┐                             │                  | ┌───────────────────────────┐
 │3│  |             │ Common Package │                             │                  | │ openstack-nova-common.rpm │
 └─┘  |             └───────┬────────┘                             │                  | └───────────────────────────┘
      |                     │                                      │                  |
      |                     │                                      │                  |
      │                     │                                      │                  |
 ┌─┐  |            ┌────────▼────────┐            ┌────────────────▼────────────────┐ | ┌──────────────────┐ ┌────────────────────────┐
 │4│  |            │ Library Package ◄------------| Library Test Package (Optional) │ | │ python2-nova.rpm │ │ python2-nova-tests.rpm │
 └─┘  |            └─────────────────┘            └─────────────────────────────────┘ | └──────────────────┘ └────────────────────────┘
```

As shown in the figure, it is divided into 4 levels:

1. Root Package is the main RPM package and does not contain any files in principle. It is only used for service collection. Users can use this RPM to install all sub-RPM packages with one click.
   If the project has doc-related files, they can also be packaged separately (optional).
2. Service Package is the sub-service RPM package, containing the service's systemd service startup files, its own configuration files, etc.
3. Common Package is the RPM package for shared dependencies, containing common configuration files and system configuration files required by various sub-services.
4. Library Package is the Python source code package, containing the project's Python code.
   If the project has test-related files, they can also be packaged separately (optional).

Projects covered by this principle:

* openstack-nova
* openstack-cinder
* openstack-glance
* openstack-placment
* openstack-ironic

##### 2.1.1.2 Special Cases

Some OpenStack components themselves contain only one service and do not have the concept of sub-services. For such services, only two levels are needed:

```ini
 Level | Package                                                         | Example
       |                                                                 |
  ┌─┐  |               ┌──────────────┐  ┌────────────────────────┐      | ┌────────────────────────┐ ┌────────────────────────────┐
  │1│  |               │ Root Package │  │ Doc Package (Optional) │      | │ openstack-keystone.rpm │ │ openstack-keystone-doc.rpm │
  └─┘  |               └───────┬──────┘  └────────────────────────┘      | └────────────────────────┘ └────────────────────────────┘
       |                       │                                         |
       |          ┌────────────┴───────────────────┐                     |
  ┌─┐  |  ┌───────▼─────────┐     ┌────────────────▼────────────────┐    | ┌──────────────────────┐ ┌────────────────────────────┐
  │2│  |  │ Library Package ◄-----| Library Test Package (Optional) │    | │ python2-keystone.rpm │ │ python2-keystone-tests.rpm │
  └─┘  |  └─────────────────┘     └─────────────────────────────────┘    | └──────────────────────┘ └────────────────────────────┘
```

1. Root Package RPM contains all files except Python source code, including service startup files, project configuration files, system configuration files, etc.
   If the project has doc-related files, they can also be packaged separately (optional).
2. Library Package is the Python source code package, containing the project's Python code.
   If the project has test-related files, they can also be packaged separately (optional).

Projects covered by this principle:

* openstack-keystone
* openstack-horizon

Some projects have several sub-RPM packages, but these sub-RPM packages are mutually exclusive. The structure of such services is as follows:

```ini
Level | Package                                                                           | Example
      |                                                                                   |
 ┌─┐  |                       ┌──────────────┐        ┌────────────────────────┐          | ┌───────────────────────┐ ┌───────────────────────────┐
 │1│  |                       │ Root Package │        │ Doc Package (Optional) │          | │ openstack-neutron.rpm │ │ openstack-neutron-doc.rpm │
 └─┘  |                       └────────┬─────┘        └────────────────────────┘          | └───────────────────────┘ └───────────────────────────┘
      |                                │                                                  |
      |          ┌─────────────────────┴───────────────────────────────┐                  |
      |          │                                                     │                  |
 ┌─┐  | ┌────────▼─────────┐ ┌──────────────────┐ ┌──────────────────┐ |                  | ┌──────────────────────────────┐ ┌───────────────────────────────────┐ ┌───────────────────────────────────┐
 │2│  | │ Service1 Package │ │ Service2 Package │ │ Service3 Package │ |                  | │ openstack-neutron-server.rpm │ │ openstack-neutron-openvswitch.rpm │ │ openstack-neutron-linuxbridge.rpm │
 └─┘  | └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘ |                  | └──────────────────────────────┘ └───────────────────────────────────┘ └───────────────────────────────────┘
      |          |                    |                    |           |                  |
      |          └────────────────────┼────────────────────┘           |                  |
      |                               │                                |                  |
 ┌─┐  |                       ┌───────▼────────┐                       |                  | ┌──────────────────────────────┐
 │3│  |                       │ Common Package │                       |                  | │ openstack-neutron-common.rpm │
 └─┘  |                       └───────┬────────┘                       |                  | └──────────────────────────────┘
      |                               │                                |                  |
      |                               │                                |                  |
      |                               │                                |                  |
 ┌─┐  |                      ┌────────▼────────┐      ┌────────────────▼────────────────┐ | ┌─────────────────────┐ ┌───────────────────────────┐
 │4│  |                      │ Library Package ◄------| Library Test Package (Optional) │ | │ python2-neutron.rpm │ │ python2-neutron-tests.rpm │
 └─┘  |                      └─────────────────┘      └─────────────────────────────────┘ | └─────────────────────┘ └───────────────────────────┘
```

As shown in the figure, Service2 and Service3 are mutually exclusive.

1. Root package only contains non-exclusive sub-packages, and exclusive sub-packages are provided separately.
   If the project has doc-related files, they can also be packaged separately (optional).
2. Service Package is the sub-service RPM package, containing the service's systemd service startup files, its own configuration files, etc.
   Exclusive Service packages are not included by Root packages, and users need to install them separately.
3. Common Package is the RPM package for shared dependencies, containing common configuration files and system configuration files required by various sub-services.
4. Library Package is the Python source code package, containing the project's Python code.
   If the project has test-related files, they can also be packaged separately (optional).

Projects covered by this principle:

* openstack-neutron

#### 2.1.2 Common Dependency Software Standards

A dependency library generally contains only one RPM package and does not need to be split.

```ini
 Level | Package                                         | Example
       |                                                 |
  ┌─┐  |  ┌─────────────────┐ ┌────────────────────────┐ | ┌──────────────────────────┐ ┌───────────────────────────────┐
  │1│  |  │ Library Package │ │ Help Package (Optional)│ | │ python2-oslo-service.rpm │ │ python2-oslo-service-help.rpm │
  └─┘  |  └─────────────────┘ └────────────────────────┘ | └──────────────────────────┘ └───────────────────────────────┘
```

**NOTE**

The openEuler community has requirements for naming Python2 and Python3 RPM packages. Python2 package prefix is *python2-*, and Python3 package prefix is *python3-*. Therefore, when building Library RPM packages, OpenStack developers must also comply with openEuler community standards.

### 2.2 Software Dependency Functionality

The software dependency analysis function provides users with the capability to analyze the full Python software dependency topology and corresponding software versions of the target OpenStack version with one click. It automatically compares with the target openEuler version and outputs corresponding software package development suggestions. This function includes two sub-functions:

- Dependency Analysis

     Parses the dependency tree of OpenStack Python packages, disassembles the dependency topology. The dependency tree is essentially a traversal of a directed graph. In theory, a normal Python dependency tree is a Directed Acyclic Graph (DAG). There are many methods for parsing DAGs, and the commonly used breadth-first search method can be used here. However, in some special scenarios, the Python dependency tree becomes a Directed Cyclic Graph. For example: Sphinx is a documentation generation project, but it also depends on Sphinx for its own documentation generation, which leads to the formation of dependency cycles. For such problems, we only need to manually break specific nodes on the cycle. Similar cases include some test dependency libraries. Another workaround is to skip non-core libraries such as documentation and testing, which not only avoids the formation of dependency cycles but also greatly reduces the number of software packages and development workload. Taking OpenStack Wallaby version as an example, the full dependency package count is about 700+. After removing documentation and testing, the dependency package count is about 300+. Therefore, we introduce the concept of `core` core. Users can select the software range to be analyzed based on their needs. Additionally, although OpenStack contains dozens of services, users may only need some of them. Therefore, we introduce a `projects` filter, allowing users to specify the software dependency range to be analyzed according to their needs.

- Dependency Comparison
    After dependency analysis is complete, corresponding openEuler development actions are also needed. Therefore, we provide RPM software package development suggestions based on the target openEuler version. There is an N:N mapping relationship between openEuler and OpenStack versions. One openEuler version can support multiple OpenStack versions, and one OpenStack version can be deployed on multiple openEuler versions. After users specify the target openEuler version and OpenStack version, this function automatically traverses the openEuler software repository, analyzes and outputs the operations required for all software packages involved in OpenStack, such as initializing repositories, creating openEuler branches, upgrading software packages, etc. This provides guidance for developers' subsequent development.

#### 2.2.1 Version Matching Standards

- Dependency Analysis

     Input: Target OpenStack version, target OpenStack service list, whether to analyze only core software.

     Output: All involved software packages and corresponding content of each package. The format is as follows:

     ```ini
     └──{OpenStack version name}_cached_file
          └──packageA.yaml
          └──packageB.yaml
          └──packageC.yaml
          ......
     ```

     Each software content format is as follows:

     ```ini
     {
        "name": "packageA",
        "version_dict": {
           "version": "0.3.7",
           "eq_version": "",
           "ge_version": "0.3.5",
           "lt_version": "",
           "ne_version": [],
           "upper_version": "0.3.7"},
           "deep": {
              "count": 1,
              "list": ["packageB", "packageC"]},
           "requires": {}
     }
     ```

     Key description:

     |       Key         | Description |
     |:-----------------:|:-----------:|
     |  name             | Software package name |
     | version_dict      | Software version requirements, including equals, greater than or equal, less than, not equal, etc. |
     | version_dict.deep | Indicates the depth of this software in the full dependency tree and the depth-first traversal path |
     | requires          | Contains the list of dependent software of this software |

- Dependency Comparison
     Input: Dependency analysis results, target openEuler version and base comparison baseline.

     Output: A table containing the analysis results and handling suggestions for each software. Each row represents one software. All column names and definition standards are as follows:

     |       Column         | Description |
     |:-----------------:|:-----------:|
     |  Project Name     | Software package name |
     | openEuler Repo    | Software source repository name on openEuler |
     | Repo version | Source version on openEuler |
     | Required (Min) Version | Required minimum version |
     | lt Version | Required less-than version |
     | ne Version | Required not-equal version |
     | Upper Version | Required maximum version |
     | Status | Development suggestion |
     | Requires | Software dependency list |
     | Depth | Software dependency tree depth |

     The suggestions included in `Status` are:
     - "OK": Current version can be used directly, no processing needed.
     - "Need Create Repo": This software package does not exist in the openEuler system, and a new repository needs to be created in the src-openeuler repo on Gitee.
     - "Need Create Branch": The required branch does not exist in the repository, and developers need to create and initialize it.
     - "Need Init Branch": The branch exists but contains no source package of any version, and developers need to initialize this branch.
     - "Need Downgrade": Downgrade the software package.
     - "Need Upgrade": Upgrade the software package.

     Developers perform subsequent development actions based on `Status` suggestions.

#### 2.2.2 API and CLI Definition

1. Create Dependency Analysis
     - CLI: `oos dependence analysis create`
     - endpoint: `/dependence/analysis`
     - type: POST
     - sync OR async: async
     - request body:

          ```ini
          {
               "release"[required]: Enum("OpenStack Release"),
               "runtime"[optional][Default: "3.10"]: Enum("Python version"),
               "core"[optional][Default: False]: Boolean,
               "projects"[optional][Default: None]: List("OpenStack service")
          }
          ```

     - response body:

          ```ini
          {
               "ID": UUID,
               "status": Enum("Running", "Error")
          }
          ```

2. Get Dependency Analysis
     - CLI: `oos dependence analysis show`, `oos dependence analysis list`
     - endpoint: `/dependence/analysis/{UUID}`, `/dependence/analysis`
     - type: GET
     - sync OR async: sync
     - request body: None
     - response body:

          ```ini
          {
               "ID": UUID,
               "status": Enum("Running", "Error", "OK")
          }
          ```

3. Delete Dependency Analysis
     - CLI: `oos dependence analysis delete`
     - endpoint: `/dependence/analysis/{UUID}`
     - type: DELETE
     - sync OR async: sync
     - request body: None
     - response body:

          ```ini
          {
               "ID": UUID,
               "status": Enum("Error", "OK")
          }
          ```

4. Create Dependency Comparison
     - CLI: `oos dependence generate`
     - endpoint: `/dependence/generate`
     - type: POST
     - sync OR async: async
     - request body:

          ```ini
          {
               "analysis_id"[required]: UUID,
               "compare"[optional][Default: None]: {
                    "token"[required]: GITEE_TOKEN_ID,
                    "compare-from"[optional][Default: master]: Enum("openEuler project branch"),
                    "compare-branch"[optional][Default: master]: Enum("openEuler project branch")

               }
          }
          ```

     - response body:

          ```ini
          {
               "ID": UUID,
               "status": Enum("Running", "Error")
          }
          ```

5. Get Dependency Comparison
     - CLI: `oos dependence generate show`, `oos dependence generate list`
     - endpoint: `/dependence/generate/{UUID}`, `/dependence/generate`
     - type: GET
     - sync OR async: sync
     - request body: None
     - response body:

          ```ini
          {
               "ID": UUID,
               "data" RAW(result data file)
          }
          ```

6. Delete Dependency Comparison
     - CLI: `oos dependence generate delete`
     - endpoint: `/dependence/generate/{UUID}`
     - type: DELETE
     - sync OR async: sync
     - request body: None
     - response body:

          ```ini
          {
               "ID": UUID,
               "status": Enum("Error", "OK")
          }
          ```

### 2.3 Software SPEC Generation Functionality

A large number of Python libraries that OpenStack depends on are developer-oriented. These libraries do not provide user services externally but only provide code-level calls. Their RPM content is simple and has a fixed format, making it suitable for tool-based approaches to improve development efficiency.

#### 2.3.1 SPEC Generation Standards

SPEC writing is generally divided into several stages, each with corresponding standard requirements:

1. Fill in conventional items, including Name, Version, Release, Summary, License, etc. This information is provided by the target software's PyPI information.
2. Fill in sub-software package information, including software package name, build dependencies, install dependencies, description, etc. This information is also provided by the target software's PyPI information. Among them, software package names need to have obvious Python prefixes, such as being prefixed with `python3-`.
3. Fill in build process information, including %prep, %build %install %check content. This content has a fixed format, and corresponding RPM macro commands can be generated.
4. RPM package file packaging stage. In this stage, through file search methods, bin, lib, doc, etc. are placed in corresponding directories.

**NOTE**: Outside of common standards, there are also some exceptions that need special explanation:

1. If the software package name itself already contains words like `python`, there is no need to add `python-` or `python3-` prefix.
2. During software build and installation stages, based on the software's installation method, macro commands include `%py3_build` or `pyproject_build`, which require manual review.
3. If the software itself contains compiled code such as C language, the `BuildArch: noarch` keyword needs to be removed, and attention should be paid to the difference between RPM macros `%{python3_sitelib}` and `%{python3_sitearch}` in the %files stage.

#### 2.3.2 API and CLI Definition

1. Create SPEC
     - CLI: `oos spec create`
     - endpoint: `/spec`
     - type: POST
     - sync OR async: async
     - request body:

          ```ini
          {
               "name"[required]: String,
               "version"[optional][Default: "latest"]: String,
               "arch"[optional][Default: False]: Boolean,
               "check"[optional][Default: True]: Boolean,
               "pyproject"[optional][Default: False]: Boolean,
          }
          ```

     - response body:

          ```ini
          {
               "ID": UUID,
               "status": Enum("Running", "Error")
          }
          ```

2. Get SPEC
     - CLI: `oos spec show`, `oos spec list`
     - endpoint: `/spec/{UUID}`, `/spec/`
     - type: GET
     - sync OR async: sync
     - request body: None
     - response body:

          ```ini
          {
               "ID": UUID,
               "status": Enum("Running", "Error", "OK")
          }
          ```

3. Update SPEC
     - CLI: `oos spec update`
     - endpoint: `/spec/{UUID}`
     - type: POST
     - sync OR async: async
     - request body:

          ```ini
          {
               "name"[required]: String,
               "version"[optional][Default: "latest"]: String,
          }
          ```

     - response body:

          ```ini
          {
               "ID": UUID,
               "status": Enum("Running", "Error")
          }
          ```

4. Delete SPEC
     - CLI: `oos spec delete`
     - endpoint: `/spec/{UUID}`
     - type: DELETE
     - sync OR async: sync
     - request body: None
     - response body:

          ```ini
          {
               "ID": UUID,
               "status": Enum("Error", "OK")
          }
          ```

### 2.4 Automated Deployment and Testing Functionality

OpenStack has diverse deployment scenarios, complex deployment processes, and high technical thresholds for deployment. To solve the problems of high thresholds, low efficiency, and high labor costs, the openEuler OpenStack development platform needs to provide automated deployment and testing functionality.

- Automated Deployment

     Provides one-click deployment capability for OpenStack based on openEuler, including support for deployment functionality across different architectures, services, and scenarios. It provides the capability to quickly provision and configure openEuler environments based on different environments. It also provides `plugin` capabilities to facilitate users in extending supported deployment backends and scenarios.

- Automated Testing

     Provides one-click testing capability for OpenStack based on openEuler, including support for testing across different scenarios. It provides users with the capability to customize testing, standardizes test reports, and supports reporting and persisting test results.

#### 2.4.1 Automated Deployment

Automated deployment mainly includes two parts: openEuler environment preparation and OpenStack deployment.

- openEuler Environment Preparation

  Provides the capability to quickly provision openEuler environments. Supported provisioning methods include `creating public cloud resources` and `managing existing environments`. The specific design is as follows:

  ```ini
  **NOTE**
     OpenStack on openEuler is primarily supported in RPM + systemd mode, and container mode is not supported for now.
  ```

  - Create Public Cloud Resources

    Creating public cloud resources primarily supports virtual machines (bare metal operations are complex on cloud, and ecosystem satisfaction is insufficient, so no support for now). Uses a plugin-based approach to provide multi-cloud support capability. Huawei Cloud is used as the reference implementation and is prioritized. Support for other clouds is continuously advanced based on user needs. Based on scenarios, supports all-in-one and three-node topologies.
     1. Create Environment
          - CLI: `oos env create`
          - endpoint: `/environment`
          - type: POST
          - sync OR async: async
          - request body:

               ```ini
               {
                    "name"[required]: String,
                    "type"[required]: Enum("all-in-one", "cluster"),
                    "release"[required]: Enum("openEuler_Release"),
                    "flavor"[required]: Enum("small", "medium", "large"),
                    "arch"[required]: Enum("x86", "arm64"),
               }
               ```

          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

     2. Query Environment
          - CLI: `oos env list`
          - endpoint: `/environment`
          - type: GET
          - sync OR async: async
          - request body: None
          - response body:

               ```ini
               {
                    "ID": UUID,
                    "Provider": String,
                    "Name": String,
                    "IP": IP_ADDRESS,
                    "Flavor": Enum("small", "medium", "large"),
                    "openEuler_release": String,
                    "OpenStack_release": String,
                    "create_time": TIME,
               }
               ```

     3. Delete Environment
          - CLI: `oos env delete`
          - endpoint: `/environment/{UUID}`
          - type: DELETE
          - sync OR async: sync
          - request body: None
          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("Error", "OK")
               }
               ```

  - Manage Existing Environments

    Users can also directly use existing openEuler environments for OpenStack deployment. They need to bring existing environments under management into the platform. After management, environments and created projects can be directly queried or deleted.
    1. Manage Environment
          - CLI: `oos env manage`
          - endpoint: `/environment/manage`
          - type: POST
          - sync OR async: sync
          - request body:

               ```ini
               {
                    "name"[required]: String,
                    "ip"[required]: IP_ADDRESS,
                    "release"[required]: Enum("openEuler_Release"),
                    "password"[required]: String,
               }
               ```

          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("Error", "OK")
               }
               ```

- OpenStack Deployment

     Provides the capability to deploy specified OpenStack versions on created/managed openEuler environments.
     1. Deploy OpenStack
          - CLI: `oos env setup`
          - endpoint: `/environment/setup`
          - type: POST
          - sync OR async: async
          - request body:

               ```ini
               {
                    "target"[required]: UUID(environment),
                    "release"[required]: Enum("OpenStack_Release"),
               }
               ```

          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

     2. Initialize OpenStack Resources
          - CLI: `oos env init`
          - endpoint: `/environment/init`
          - type: POST
          - sync OR async: async
          - request body:

               ```ini
               {
                    "target"[required]: UUID(environment),
               }
               ```

          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

     3. Uninstall Deployed OpenStack
          - CLI: `oos env clean`
          - endpoint: `/environment/clean`
          - type: POST
          - sync OR async: async
          - request body:

               ```ini
               {
                    "target"[required]: UUID(environment),
               }
               ```

          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

#### Automated Testing

After successful environment deployment, the SIG development platform provides automated testing functionality based on the deployed OpenStack environment. It mainly includes the following important contents:

OpenStack itself provides a complete testing framework. It includes `unit testing` and `functional testing`. Among them, `unit testing` is already included by RPM spec in section 2.3. The %check stage of spec can define the unit testing method for each project. Normally, it only needs to add `pytest` or `stestr`. `Functional testing` is provided by the OpenStack Tempest service. During the `oos env init` stage of automated deployment mentioned above, oos automatically installs Tempest and generates the default configuration file.

- CLI: `oos env test`
- endpoint: `/environment/test`
- type: POST
- sync OR async: async
- request body:

     ```ini
     {
          "target"[required]: UUID(environment),
     }
     ```

- response body:

     ```ini
     {
          "ID": UUID,
          "status": Enum("Running", "Error")
     }
     ```

After test execution is complete, oos outputs test reports. By default, oos uses the `subunit2html` tool to generate HTML format Tempest test result files.

### 2.5 openEuler Automated Development Functionality

There are many software packages involved in OpenStack. As versions continue to evolve and supported services continue to improve, the list of software packages maintained by the SIG will be continuously refreshed. To reduce repetitive development actions, oos also encapsulates some easy-to-use code development platform automation capabilities, such as automatic code submission capabilities based on Gitee. The functions are as follows:

```ini
     ┌───────────────────────────────────────────────────┐
     │                     Code Action                   │
     └─────────────────────┬─────────────────────────────┘
                           │
           ┌───────────────┼───────────────────┐
           │               │                   │
     ┌─────▼─────┐  ┌──────▼──────┐  ┌─────────▼─────────┐
     │Repo Action│  │Branch Action│  │Pull Request Action│
     └───────────┘  └─────────────┘  └───────────────────┘
```

1. `Repo Action` provides automation functionality related to software repositories:

     1. Auto-create Repository
          - CLI: `oos repo create`
          - endpoint: `/repo`
          - type: POST
          - sync OR async: async
          - request body:

               ```ini
               {
                    "project"[required]: String,
                    "repo"[required]: String,
                    "push"[optional][Default: "False"]: Boolean,
               }
               ```

          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

2. `Branch Action` provides automation functionality related to software branches:

     1. Auto-create Branch
          - CLI: `oos repo branch-create`
          - endpoint: `/repo/branch`
          - type: POST
          - sync OR async: async
          - request body:

               ```ini
               {
                    "branches"[required]: {
                         "branch-name"[required]: String,
                         "branch-type"[optional][Default: "None"]: Enum("protected"),
                         "parent-branch"[required]: String
                    }
               }
               ```

          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

3. `Pull Request Action` provides automation functionality related to code PRs:

     1. Add PR comments, facilitating users to perform routine comments such as `retest`, `/lgtm`, etc.
          - CLI: `oos repo pr-comment`
          - endpoint: `/repo/pr/comment`
          - type: POST
          - sync OR async: sync
          - request body:

               ```ini
               {
                    "repo"[required]: String,
                    "pr_number"[required]: Int,
                    "comment"[required]: String
               }
               ```

          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("OK", "Error")
               }
               ```

     2. Fetch all SIG PRs, facilitating maintainers to get the current development status of the SIG and improve review efficiency.
          - CLI: `oos repo pr-fetch`
          - endpoint: `/repo/pr/fetch`
          - type: POST
          - sync OR async: async
          - request body:

               ```ini
               {
                    "repo"[optional][Default: "None"]: List[String]
               }
               ```

          - response body:

               ```ini
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

## 3. Quality, Security, and Compliance

SIG open-source software needs to comply with various requirements from the openEuler community for its software, and should also comply with the export standards of OpenStack community software.

### 3.1 Quality and Security

- Software Quality (Serviceability)
     1. The corresponding software code must include unit tests with coverage not less than 80%.
     2. End-to-end functional tests must be provided, covering all above-mentioned interfaces and core scenario tests.
     3. Based on the openEuler community CI, build CI/CD processes. All Pull Requests must have CI to ensure code quality, and regularly release versions with release intervals not exceeding 3 months.
     4. Based on the Gitee ISSUE system, handle problems discovered and reported by users. The closure rate should be greater than 80%, and the closure cycle should not exceed 1 week.

- Software Security
     1. Data Security: The software does not connect to the network throughout the process, and persistent storage does not contain user sensitive information.
     2. Network Security: OOS uses HTTP protocol for communication under REST architecture, but the software is designed for use in intranet environments. It is not recommended to expose it on public IP addresses. If necessary, it is recommended to add access IP whitelist restrictions.
     3. System Security: Based on openEuler security mechanisms, regularly release CVE fixes or security patches.
     4. Application Layer Security: Not involved, no application-level security services are provided, such as password policies, access control, etc.
     5. Management Security: The software provides log generation and periodic backup mechanisms to facilitate users' regular audits.

- Reliability

     This software is oriented toward openEuler community OpenStack development activities, and does not involve service deployment or commercial production. All code is open and transparent, and does not involve private functions or code. Therefore, features such as node redundancy and disaster recovery backup are not provided.

### 3.2 Compliance

1. License Compliance

     This platform uses Apache2.0 License, which does not restrict downstream forked software from being closed source or for commercial use. However, downstream software must mark the code source and retain the original License.

2. Legal Compliance

     This platform is jointly developed and maintained by open-source developers, and does not involve trade secrets or non-public code from commercial companies. All contributors must comply with the openEuler community contribution guidelines to ensure their contributions are compliant and legal. The SIG and the community itself do not bear corresponding responsibilities.

     If non-compliant source code is found, the SIG has the right and obligation to promptly delete it without needing the contributor's permission. It also has the right to prohibit non-compliant code or developers from continuing to contribute.

     If developers have non-public code to contribute, they must first comply with their company's open-source processes and regulations, and contribute code publicly according to openEuler community open-source specifications.

## 4. Implementation Plan

|       Time         | Content | Status|
|:-----------------:|:-----------:|:-----------:|
|  2021.06           | Complete software overall framework writing, implement CLI Built-in mechanism, at least one API available | Done |
|  2021.12           | Complete CLI Built-in mechanism with full functionality available | Done |
|  2022.06           | Complete quality hardening, ensure functionality, formally introduce OOS in openEuler OpenStack community development process | Done |
|  2022.12           | Continuously improve OOS, ensure usability and robustness, automation coverage exceeding 80%, reduce development labor investment | Done |
|  2023.06           | Complete REST framework, CI/CD processes, enrich Plugin mechanism, introduce more backend support | Working in progress |
|  2023.12           | Complete frontend GUI functionality | Planning |
