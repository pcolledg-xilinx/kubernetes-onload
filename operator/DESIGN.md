# Kubernetes Onload #

## Conceptual Overview ##

The purpose of the Kubernetes Onload Operator is to streamline configuration of OpenOnload/EnterpriseOnload components in Kubernetes/OpenShift and to handle the software upgrade lifecycle of all its components, both kernel backend and application software for use in user pods.

The Kubernetes Onload Operator is not required to deploy Onload in a cluster. The Operator's subordinate resources can be applied to a cluster individually. However, any manually applied resources must be removed before deploying the Operator.

Deployment simply involves installing the Onload Operator and Kernel Module Management Operator from the OperatorHub and creating an `Onload` CR specific to the cluster's requirements. Air gapped environments will require local copies of the core resources. The Node Feature Discovery Operator can also be installed to identify Solarflare cards.

As an open source product, Onload can be customised. Official release artefacts can simply be replaced by custom artefacts prepared by the user.

## Operator ##

The stable release of the Operator shall be [available on OperatorHub](https://operatorhub.io/contribute).

### Resources ###

The Operator shall define an `Onload` CRD. A user shall then create their own CR of this kind to configure:

* The location of an Onload release (required)
* The location of an sfptpd release (default: null)
* Whether out-of-tree sfc kernel modules are enabled (default: true)
* Whether an MC CR is managed for boot-time sfc kernel module loading (default: false)

The Operator shall create and own a `Module` CR named `onload`. Based on this CR, the Kernel Module Management Operator will deploy on each node one of:

* [Onload Driver Container](#onload-driver-container) for Onload kernel modules

The Operator shall create and own `DaemonSet`s to deploy one each of the following pods on the same nodes selected for the above resources:

* [Onload Device Plugin Container](#onload-device-plugin-container) (required)
* [Onload Control Plane Container](#onload-control-plane-container) (required)
* [sfptpd](#sfptpd-container) (optional)

If sfc is enabled, the Operator shall create and own a `Module` CR named `sfc`. Based on this CR, the Kernel Module Management Operator will deploy on each node one of:

* [Solarflare Driver Container](#solarflare-driver-container) for sfc kernel modules

### Upgrade Lifecycle ###

1. User may test new release with KMM `PreflightValidator` CR (could also be integrated into upgrade with designated sacrificial node)
1. User shall change versions in `Onload` CR.
1. Operator reconciler shall detect change of versions in user `Onload` CR.
1. Operator shall freeze KMM state by labeling all selected nodes with `kmm.node.kubernetes.io/version-module.<module-namespace>.<module-name>: $existingVersion`
1. Operator shall iterate each selected node. May honour Disruption Budgets.
    1. Operator shall remove node from Onload Device Plugin DaemonSet, removing node's Device Plugin pod so resource `amd.com/onload` is no longer advertised, preventing new accelerated pods being scheduled on node.
    1. Operator shall list all pods on node which use resource `amd.com/onload` and calls Eviction API on them.
    1. Operator shall remove node label `kmm.node.kubernetes.io/version-module.<module-namespace>.<module-name>`, triggering KMM to change ModuleLoader pod version on node.
        1. Operator may validate driver has cleanly reloaded and on failure evict ModuleLoader to force retry.
    1. Operator shall return node to Onload Device Plugin DaemonSet, scheduling a new Onload Device Plugin pod on node. Kubernetes can schedule user accelerated pods again on node due to presence of `amd.com/onload` resource.

Ref:
* [Kernel Module Management > Preflight validation for Modules](https://kmm.sigs.k8s.io/documentation/preflight_validation/)
* [Kernel Module Management > Ordered upgrade of kernel module without reboot](https://kmm.sigs.k8s.io/documentation/ordered_upgrade/)
* [Kubernetes > API-initiated Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)

## Components ##

Components shall be available as container images on quay.io for minor official releases and for specific stock RHCOS kernel versions.

As open source products, all components can also be customised and/or built by the user. This is particularly useful for users wishing to use pre-release features. Documentation shall describe the process, using toolchains which are either in-cluster OpenShift Build Configs or CLI with Buildah (cf. Docker).

Driver Container images shall be tagged with their software version, kernel version (including variant), and architecture. The architecture is by default `amd64`. While Onload software does support other architectures, the Operator follows standard Kubernetes practices, being documented and supported only for `amd64`. Official support for Device Containers on other architectures is not planned.

### SFPTPD Container ###

May be deployed if PTP network hardware is present.

* Runs sfptpd -- _Implemented_
* Reports metrics from sfptpd

### Solarflare Driver Container ###

May be deployed if sfc hardware is present.

* Loads `sfc.ko` and dependencies -- _Implemented_

### Onload Driver Container ###

* Loads `onload.ko` and dependencies -- _Implemented_

### Onload Device Plugin Container ###

* Copies user software to host volume -- _Implemented_
* Advertises `/dev` mounts -- _Implemented_
* Advertises Onload user binaries
* May mount devices on Kernel Shared Memory devices.

### Solarflare Device Plugin Container ###

May be deployed if sfc hardware is present.

* Configures sfc firmware features with `sfboot`
* Configures sfc firmware binaries with `sfupdate`

### Onload Control Plane Container ###

* Hosts the Kernel-started Onload Control Plane binary -- _Implemented_

### Onload Metrics Container ###

* Publishes `orm_json` output

### Troubleshooting Container for sfc ###

May be deployed on demand.

* Outputs diagnostic reporting bundle script (featuring `sfreport`) to pod log
* Provides privileged shell for interactive use by user

ON-14959 & ON-15040

### Troubleshooting Container for Onload ###

May be deployed on demand.

* Outputs diagnostic reporting bundle script (featuring `onload_stackdump`, `orm_json`) to pod log
* Provides privileged shell for interactive use by user

ON-15024 & ON-15040

### Support Bundle ###

Shall provide user with an all-in-one command which runs above troubleshooting containers and additional commands. Output shall be a `tar.gz`

ON-15040

## TODO ##

* Hugepages - ON-15042
* Network Namespaces
* ef_vi allocation
* Manage Multus `NetworkAttachmentDefinition`
* Other CNIs (OVN-Kubernetes, Calico, cf. OKTA)
* TCPDirect
* Signed kernel modules
* Signed containers
* onload-user libc variants
* SRIOV (non-Multus)
* Kubernetes native differing requirements - ON-15041

_Copyright (c) 2023 Advanced Micro Devices, Inc._