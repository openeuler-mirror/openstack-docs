
# openEuler OpenStack Development Platform Requirements Specification

## Background

Currently, with the continuous development of the SIG, we have encountered the following problems:

1. OpenStack technology is complex, involving various technologies of cloud IaaS layer such as computing, networking, storage, images, and authentication. It is difficult for developers to master all aspects, and the submitted **code logic and quality are concerning**.
2. OpenStack is written in Python, and Python software dependency issues are difficult to handle. Taking OpenStack Wallaby version as an example, it involves 400+ core Python software packages. The dependency levels and dependency versions of each software are **complex and intricate, making selection difficult**, and hard to form a closed loop.
3. There are many OpenStack software packages, and the workload for RPM Spec development is enormous. With the continuous evolution of openEuler and OpenStack versions, the N:N adaptation relationship will cause **workload to multiply, and labor costs will increase**.
4. The testing threshold for OpenStack is too high. Not only do developers need to be familiar with OpenStack, but they also need to have some understanding and mastery of Linux underlying technologies such as virtualization, virtual bridges, and block storage. Deploying an OpenStack environment takes too long, and functional testing is extremely difficult. Moreover, there are many test scenarios, such as X86, ARM64 architecture testing, bare metal and virtual machine type testing, OVS and OVN bridge testing, LVM and Ceph storage testing, etc., which further increase **labor costs and technical barriers**.

To address the above problems, there is a need to provide a development platform in openEuler OpenStack to solve the pain points encountered in the development process.

## Goals

Design and develop an openEuler open-source development platform strongly related to OpenStack. Through standardization, tool-based approaches, and automation, meet the daily development needs of SIG developers, reduce development costs, decrease labor investment, lower development barriers, thereby improving development efficiency, improving SIG software quality, developing the SIG ecosystem, and attracting more developers to join the SIG.

## Scope

**User scope**: openEuler OpenStack SIG developers

**Business scope**: openEuler OpenStack SIG daily development activities

**Programming languages**: Python, Ansible, Jinja, JavaScript

**IT technologies**: Web services, RESTful specifications, CLI standards, frontend GUI, database usage

## Functions

The OpenStack development platform overall adopts a Client/Server architecture, providing platform capabilities to SIG, and client-side is open to specified user whitelist.

To facilitate users outside the whitelist, this platform also provides CLI mode. In this mode, no additional server-side communication is needed, and it can be used out of the box locally.

1. Output RPM SPEC development standards for OpenStack service software and dependency library software. Developers and Reviewers must strictly comply with standards for development and implementation.
2. Provide OpenStack Python software dependency analysis functionality, one-click generation of dependency topology and results, ensuring dependency closure and avoiding software dependency risks.
3. Provide OpenStack RPM spec generation functionality, one-click generation of RPM specs for common software, shortening development time and reducing investment costs.
4. Provide automated deployment and testing platform functionality, enabling one-click deployment of specified OpenStack versions on any openEuler version, rapid testing and iteration.
5. Provide openEuler Gitee repository automated processing capabilities to meet the need for batch software modifications, such as creating code branches, creating repositories, submitting Pull Requests, etc.

### SPEC Development Standards Formulation

【Function Points】

    1. Constrain SPEC format and content standards for OpenStack service-level projects.
    2. Define the framework for SPEC of OpenStack dependency library-level projects.

【Prerequisites】: Consensus among all OpenStack SIG Maintainers, no disagreement among participating vendors.

【Participants】: China Telecom, China Unicom, UnionTech Software

【Input】: RPM SPEC writing standards

【Output】: Service-level and dependency library-level SPEC templates; software layering standards.

【Impact on other functions】: This function is a prerequisite for the following software functions. The `SPEC auto-generation function` mentioned below must follow this standard.

### Dependency Analysis Requirements

【Function Points】

    1. Automatically generate OpenStack dependency tables based on specified openEuler versions.
    2. Handle common dependency issues such as dependency cycles, missing versions, and inconsistent names.

【Prerequisites】: N/A

【Participants】: OpenStack SIG core developers

【Input】: openEuler version number, OpenStack version number, target dependency scope (core/testing/documentation)

【Output】: Full dependency library information for the specified OpenStack version, including minimum/maximum dependency versions, belonging openEuler SIG, RPM package name, dependency level, sub-dependency tree, etc. Can be output in Excel format.

【Impact on other functions】: N/A

### Spec Auto-generation Requirements

【Function Points】

    1. One-click generate RPM SPEC for OpenStack dependency library software.
    2. Support various Python software build systems, such as setuptools, pyproject, etc.

【Prerequisites】: Must comply with `SPEC Development Standards`

【Participants】: OpenStack SIG core developers

【Input】: Specified software name and target version

【Output】: RPM SPEC file for the corresponding software

【Impact on other functions】: Generated SPECs can be pushed to openEuler community with one click through the `Code Submission Function` mentioned below.

### Automated Deployment and Testing Requirements

【Function Points】

    1. One-click rapid deployment of OpenStack single/multi-node environments with specified OpenStack versions, topologies, and functions.
    2. One-click resource pre-configuration and functional testing based on deployed OpenStack environments.
    3. Support multi-cloud and host management functionality, support plugin customization.

【Prerequisites】: N/A

【Participants】: OpenStack SIG core developers, various cloud platform related developers

【Input】: Target OpenStack version, compute/network/storage driver scenarios

【Output】: An OpenStack environment that can execute OpenStack Tempest tests with one click; Tempest test report.

【Impact on other functions】: N/A

### One-click Code Processing Requirements

【Function Points】

    1. One-click perform various operations on Repos, Branches, and PRs of projects belonging to openEuler OpenStack.
    2. Operations include: create/delete source repositories; create/delete openEuler branches; submit software Update PRs; add review comments on PRs.

【Prerequisites】: Submit PR function depends on the `SPEC Generation` function mentioned above.

【Participants】: OpenStack SIG core developers

【Input】: Specified software name, openEuler release name, target Spec file, review comment content.

【Output】: Software repository creation PR; software branch creation PR; software upgrade PR; new review comments on PRs.

【Impact on other functions】: N/A

## Non-functional Requirements

### Testing Requirements

1. The corresponding software code must include unit tests with coverage not less than 80%.
2. End-to-end functional tests must be provided, covering all above-mentioned interfaces and core scenario tests.
3. Based on the openEuler community CI, build CI/CD processes. All Pull Requests must have CI to ensure code quality, and regularly release versions with release intervals not exceeding 3 months.

### Security

1. Data Security: The software does not connect to the network throughout the process, and persistent storage does not contain user sensitive information.
2. Network Security: OOS uses HTTP protocol for communication under REST architecture, but the software is designed for use in intranet environments. It is not recommended to expose it on public IP addresses. If necessary, it is recommended to add access IP whitelist restrictions.
3. System Security: Based on openEuler security mechanisms, regularly release CVE fixes or security patches.
4. Application Layer Security: Not involved, no application-level security services are provided, such as password policies, access control, etc.
5. Management Security: The software provides log generation and periodic backup mechanisms to facilitate users' regular audits.

### Reliability

This software is oriented toward openEuler community OpenStack development activities, and does not involve service deployment or commercial production. All code is open and transparent, and does not involve private functions or code. Therefore, features such as node redundancy and disaster recovery backup are not provided.

### Open Source Compliance

This platform uses Apache2.0 License, which does not restrict downstream forked software from being closed source or for commercial use. However, downstream software must mark the code source and retain the original License.

## Implementation Plan

|       Time         | Content |
|:-----------------:|:-----------:|
|  2021.06           | Complete software overall framework writing, implement CLI Built-in mechanism, at least one API available |
|  2021.12           | Complete CLI Built-in mechanism with full functionality available |
|  2022.06           | Complete quality hardening, ensure functionality, formally introduce OOS in openEuler OpenStack community development process |
|  2022.12           | Continuously improve OOS, ensure usability and robustness, automation coverage exceeding 80%, reduce development labor investment |
|  2023.06           | Complete REST framework, CI/CD processes, enrich Plugin mechanism, introduce more backend support |
|  2023.12           | Complete frontend GUI functionality |
