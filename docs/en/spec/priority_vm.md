
# High and Low Priority VM Co-location

Virtual machine co-location refers to deploying and migrating virtual machines with different resource requirements (CPU, IO, Memory, etc.) to the same compute node through scheduling, thereby fully utilizing the node's resources. In single-node resource scheduling and allocation, high and low priorities are distinguished—when high priority VMs and low priority VMs compete for resources, resources are preferentially allocated to high priority VMs, strictly ensuring their QoS.

There are various scenarios for VM co-location, such as dynamically adjusting node resources through dynamic resource scheduling, or dynamically adjusting the distribution of VMs on nodes based on user usage patterns. High and low priority VM scheduling is one implementation method.

Introducing high and low priority VM technology in OpenStack Nova can satisfy VM co-location requirements to a certain extent. This document mainly introduces the design and implementation of high and low priority VM scheduling for OpenStack Nova VM creation functionality.

## Implementation Plan

Introduce the concept of high and low priorities in Nova's VM creation and migration process, with new high and low priority attributes for VM objects. During scheduling, high priority VMs will be scheduled to nodes with sufficient resources as much as possible. Such nodes need to meet at least the requirements that memory is not oversubscribed and CPUs used by high priority VMs are not oversubscribed.

This feature is implemented based on OpenStack Yoga version, running on openEuler 22.09 innovation release. It also introduces the Train version from openEuler 22.03 LTS SP1.

### Overall Architecture

When users create a flavor or create a VM, they can specify its priority attribute. However, the priority attribute does not affect Nova's existing resource model and node scheduling strategy—Nova still selects compute nodes and creates VMs according to the normal process.

The high and low priority VM feature mainly affects the resource scheduling and allocation policy at the single-node level after VM creation. When high priority VMs and low priority VMs compete for resources, resources are preferentially allocated to high priority VMs, strictly ensuring their QoS.

Nova has the following changes for the high and low priority VM feature:

1. VM objects and flavors have new high and low priority attribute configurations. Combined with business scenarios, the high priority attribute can only be set for pinned CPU type VMs, and the low priority attribute can only be set for non-pinned CPU type VMs.
2. For VMs with priority attributes, the libvirt XML configuration needs to be modified so that the single-node QoS management component (named Skylark) can perceive them, enabling automatic resource allocation and QoS management.
3. The CPU pinning range for low priority VMs has changed to fully utilize idle resources from high priority VMs.

### Resource Model

* VM objects have a new optional attribute `priority`, which can be set to `high` or `low`, representing high and low priority respectively.

* Flavor extra_specs add a new `hw:cpu_priority` field, marking it as a high or low priority VM flavor, with value `high` or `low`.

Parameter limits and rules:

1. `priority=high` must be used together with `hw:cpu_policy=dedicated`, otherwise an error is reported.
2. `priority=low` must be used together with `hw:cpu_policy=shared` (default value), otherwise an error is reported.
3. The priority configuration for VM objects and the priority configuration for flavors are both optional. When neither is configured, it represents a normal VM. When both are configured, the VM object's priority attribute takes precedence.

Normal VMs can coexist with VMs that have priority attributes because the priority attribute does not affect Nova's existing resource model and node scheduling strategy. When normal VMs compete with high priority VMs for resources, the Skylark component does not intervene. When normal VMs compete with low priority VMs for resources, the Skylark component prioritizes resource allocation for normal VMs.

### API

In the create VM API, the optional parameter `os:scheduler_hints.priority` can be set to `high` or `low` to set the VM object's priority.

```ini
POST v2/servers (v2.1 default version)
{
    "OS-SCH-HNT:scheduler_hints": {"priority": "high"}
}
```

### Scheduler

Remains unchanged

### Compute

#### Resource Reporting

Remains unchanged

#### Resource Allocation and Binding

High and low priority machines are created and CPU allocation is based on the priority flag:

* High priority VMs can only be pinned CPU type VMs, with one-to-one binding to CPUs specified in `cpu_dedicated_set`
* Low priority VMs can only be non-pinned CPU type VMs, defaulting to range binding to CPUs specified in `cpu_shared_set`.

Additionally, a new configuration item `cpu_priority_mix_enable` is added to the `compute` block in `nova.conf`, with a default value of False. When set to True, low priority VMs can use CPUs bound by high priority VMs, meaning low priority VMs can range bind CPUs specified in both `cpu_shared_set` and `cpu_dedicated_set`.

#### VM XML

High and low priority machines are created and VMs are tagged based on the priority flag.

* A new `<resource>` attribute fragment is added to the Libvirt XML, including two values: `/high_prio_machine` and `/low_prio_machine`, representing high and low priority VMs respectively. This fragment itself has no function in Nova, it only indicates to the `Skylark` QoS service the high and low priority attributes of the VM.

### Example

Suppose a compute node has 14 cores, with cpu_dedicated_set=0-11 (12 cores total) and cpu_shared_set=12-13 (2 cores total), with cpu_allocation_ratio=8:

1. From the scheduler's perspective, high priority VMs have 12 available cores, and from the compute's perspective, 12 cores can be pinned—consistent with Nova's original logic.
2. From the scheduler's perspective, low priority VMs have 2 * 8 = 16 available cores, and from the compute's perspective, 2 cores can be pinned (when cpu_priority_mix_enable=False)—consistent with Nova's original logic.
3. From the scheduler's perspective, low priority VMs have 2 * 8 = 16 available cores, and from the compute's perspective, 2+12=14 cores can be pinned (when cpu_priority_mix_enable=True)—different from Nova's original logic.

### Parameter Configuration Recommendations

First determine the global oversubscription ratio and extreme oversubscription ratio.

    Definition of global oversubscription ratio: The ratio of all allocatable vCPU quantities (high and low combined) to all available physical cores. This is a calculated theoretical value—for example, in the above scenario, the global oversubscription ratio is (12 + 2 * 8) / 14 = 2.
    Significance of global oversubscription ratio: In the high and low priority scenario, the global oversubscription ratio mainly affects the QoS of low priority VMs under normal conditions (when high priority VM vCPUs are not simultaneously peaking). Setting a reasonable global oversubscription ratio can reduce situations where underlying resources are sufficient but scheduling fails.

    Definition of extreme oversubscription ratio: That is cpu_allocation_ratio. It only affects the oversubscription capability of shared cores.
    Significance of extreme oversubscription ratio: In the high and low priority scenario, the extreme oversubscription ratio mainly affects the QoS of low priority VMs under extreme conditions (when all high priority VM vCPUs are simultaneously peaking).

After users select appropriate global and extreme oversubscription ratios based on business characteristics and QoS goals, they can configure reasonable cpu_dedicated_set and cpu_shared_set according to the following calculation formula.
    Calculation formula:

    ```
    Expected global oversubscription ratio = (extreme oversubscription ratio * shared core count + dedicated core count) / total compute node core count
    ```

    Using the compute node in the above example, with 14 total cores and assuming an extreme oversubscription ratio of 8, the calculation yields:

    ```
    When dedicated core count is 12 and shared core count is 2, expected global oversubscription = (8*2+12)/14 = 2

    When dedicated core count is 4 and shared core count is 10, expected global oversubscription = (8*10+4)/14 = 6
    ```

## Development Schedule

Developers:

* `Wang Xiyuan`<wangxiyuan1007@gmail.com>
* `Guo Lei`<guolei_yewu@cmss.chinamobile.com>
* `Ma Ganlin`<maganlin_yewu@cmss.chinamobile.com>
* `Han Guangyu`<hanguangyu@uniontech.com>
* `Zhang Ying`<zhangy1317@foxmail.com>
* `Zhang Fan`<zh.f@outlook.com>

Timeline:

* 2022-04-01 to 2022-05-30: Complete development
* 2022-06-01 to 2022-07-30: Testing, integration, and code refresh
* 2022-08-01 to 2022-08-30: Complete RPM package build
* 2022-09-30: Introduce openEuler 22.09 Yoga version
* 2022-12-30: Introduce openEuler 22.03 LTS SP1 Train version
