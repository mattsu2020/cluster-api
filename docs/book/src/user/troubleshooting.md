# Troubleshooting

<!-- TOC -->
* [Troubleshooting](#troubleshooting)
  * [General troubleshooting workflow](#general-troubleshooting-workflow)
    * [1. Verify the management cluster and providers](#1-verify-the-management-cluster-and-providers)
    * [2. Find the resource that is not progressing](#2-find-the-resource-that-is-not-progressing)
    * [3. Inspect Conditions and Events](#3-inspect-conditions-and-events)
    * [4. Inspect provider logs](#4-inspect-provider-logs)
    * [5. Inspect the workload cluster and Nodes](#5-inspect-the-workload-cluster-and-nodes)
    * [6. Inspect the infrastructure and bootstrap process](#6-inspect-the-infrastructure-and-bootstrap-process)
    * [7. Collect information for further investigation](#7-collect-information-for-further-investigation)
  * [Troubleshooting Quick Start with Docker (CAPD)](#troubleshooting-quick-start-with-docker-capd)
  * [Node bootstrap failures when using CABPK with cloud-init](#node-bootstrap-failures-when-using-cabpk-with-cloud-init)
  * [Labeling nodes with reserved labels such as `node-role.kubernetes.io` fails with kubeadm error during bootstrap](#labeling-nodes-with-reserved-labels-such-as-node-rolekubernetesio-fails-with-kubeadm-error-during-bootstrap)
  * [Cluster API with Docker - common issues with docker -](#cluster-api-with-docker---common-issues-with-docker---)
  * [Cluster API with Docker  - "too many open files"](#cluster-api-with-docker---too-many-open-files)
    * [MacOS and Docker Desktop -  "too many open files"](#macos-and-docker-desktop---too-many-open-files)
  * [Failed clusterctl init - 'failed to get cert-manager object'](#failed-clusterctl-init---failed-to-get-cert-manager-object)
  * [Failed clusterctl upgrade apply - 'failed to update cert-manager component'](#failed-clusterctl-upgrade-apply---failed-to-update-cert-manager-component)
  * [Clusterctl failing to start providers due to outdated image overrides](#clusterctl-failing-to-start-providers-due-to-outdated-image-overrides)
  * [Managed Cluster and co-authored slices](#managed-cluster-and-co-authored-slices)
  * [Failed to removed fields from lists using Server Side Apply](#failed-to-removed-fields-from-lists-using-server-side-apply)
  * [kubeadm join fails after upgrading to Kubernetes v1.36.1 / v1.35.5 / v1.34.8 / v1.33.12](#kubeadm-join-fails-after-upgrading-to-kubernetes-patch-releases)
<!-- TOC -->

## General troubleshooting workflow

Cluster API manages workload clusters through resources and controllers in a management cluster. Start troubleshooting
from the management cluster, even when the symptom is a workload cluster that cannot be reached. The resource `Conditions`,
Kubernetes Events, and provider controller logs usually identify the component where reconciliation stopped.

The examples below use `<cluster-name>` and `<namespace>` for the workload Cluster resource. Cluster API resources are
namespaced, so include `--namespace <namespace>` (or `-n <namespace>`) when the Cluster is not in the current namespace.

### 1. Verify the management cluster and providers

First, confirm that `kubectl` and `clusterctl` are using the expected management cluster:

```bash
kubectl config current-context
kubectl cluster-info
clusterctl get providers
```

`clusterctl get providers` lists the installed core, bootstrap, control plane, and infrastructure providers and their
target namespaces. Check that the controller Deployments in those namespaces are available. For example:

```bash
kubectl get deployments -n capi-system
kubectl get pods -n <provider-namespace>
```

If a provider Pod is not `Running` and ready, inspect it before troubleshooting an individual workload cluster:

```bash
kubectl describe pod -n <provider-namespace> <provider-pod>
kubectl logs -n <provider-namespace> <provider-pod> --all-containers --previous
```

The `--previous` flag shows logs from the previous container instance and is useful for `CrashLoopBackOff`. Omit it if
the container has not restarted.

### 2. Find the resource that is not progressing

Use `clusterctl describe cluster` for an overview of the Cluster and its resource hierarchy:

```bash
clusterctl describe cluster <cluster-name> --namespace <namespace>
```

Start with the first resource that reports `READY` as `False` or `Unknown`. The displayed reason and message often
identify the next resource to inspect. By default, the command groups Machines with the same state and hides bootstrap
and infrastructure objects when their state matches the owning Machine. Disable those shortcuts when more detail is
needed:

```bash
clusterctl describe cluster <cluster-name> --namespace <namespace> --grouping=false --echo
```

See the [`clusterctl describe cluster` command](../clusterctl/commands/describe-cluster.md) for more visualization
options.

### 3. Inspect Conditions and Events

Show all Conditions in the resource hierarchy:

```bash
clusterctl describe cluster <cluster-name> --namespace <namespace> --show-conditions all
```

For the resource that is not progressing, inspect its complete status and Events. Replace `<kind>` and `<name>` with
values such as `cluster` and the Cluster name, `machine` and a Machine name, or a provider-specific resource and name:

```bash
kubectl describe <kind> <name> --namespace <namespace>
kubectl get <kind> <name> --namespace <namespace> -o yaml
kubectl get events --namespace <namespace> --sort-by=.lastTimestamp
```

When reading a Condition, use these fields together:

* `type` identifies the aspect of the resource being reported.
* `status` indicates whether the condition is `True`, `False`, or `Unknown`.
* `reason` and `message` explain the current state and usually identify the next troubleshooting action.
* `lastTransitionTime` indicates when the state last changed.
* `observedGeneration`, when present, shows which resource generation the controller observed. If it is lower than
  `metadata.generation`, the controller has not processed the latest specification yet.

Follow object references in `spec`, `status`, and `metadata.ownerReferences`. For example, a Machine can point to a
bootstrap configuration and an infrastructure Machine. Inspect the referenced object when the Machine Condition reports
a bootstrap or infrastructure failure.

Events are best used with Conditions and logs. They can be repeated, aggregated, or removed over time, so the absence of
an Event does not prove that an error did not occur.

### 4. Inspect provider logs

The failing resource type determines which controller logs to inspect:

* `Cluster`, `Machine`, `MachineSet`, and `MachineDeployment`: core provider.
* `KubeadmConfig`: kubeadm bootstrap provider.
* `KubeadmControlPlane`: kubeadm control plane provider.
* Provider-specific infrastructure resources: the corresponding infrastructure provider.

Use the provider namespace from `clusterctl get providers`, find its controller Deployment, and read the manager
container logs around the time shown by the Condition or Event:

```bash
kubectl get deployments -n <provider-namespace>
kubectl logs -n <provider-namespace> deployment/<provider-controller-manager> \
  -c manager --since=30m
```

Search for the Cluster or failing resource name. Include timestamps if logs need to be correlated across controllers:

```bash
kubectl logs -n <provider-namespace> deployment/<provider-controller-manager> \
  -c manager --since=30m --timestamps | grep '<cluster-name>\|<resource-name>'
```

If the controller has restarted, also inspect the previous container logs with `--previous`. Errors from an external
cloud or infrastructure API can require checking credentials, quotas, permissions, networking, or service health outside
the management cluster.

### 5. Inspect the workload cluster and Nodes

Continue on the workload cluster only after the control plane is initialized and a kubeconfig is available:

```bash
clusterctl get kubeconfig <cluster-name> --namespace <namespace> > <cluster-name>.kubeconfig
kubectl --kubeconfig <cluster-name>.kubeconfig get nodes -o wide
kubectl --kubeconfig <cluster-name>.kubeconfig get pods -A
kubectl --kubeconfig <cluster-name>.kubeconfig get events -A --sort-by=.lastTimestamp
```

If the API server cannot be reached, return to the management cluster and inspect the control plane, infrastructure,
load balancer, and network configuration. If the API server is reachable but a Node is not ready, inspect the Node and
the system Pods scheduled to it:

```bash
kubectl --kubeconfig <cluster-name>.kubeconfig describe node <node-name>
kubectl --kubeconfig <cluster-name>.kubeconfig get pods -A \
  --field-selector spec.nodeName=<node-name>
```

### 6. Inspect the infrastructure and bootstrap process

When a Machine remains in provisioning or bootstrap fails, inspect the actual machine through the infrastructure
provider. Depending on the provider, this can mean a virtual machine, container, bare-metal host, or cloud instance.
Verify that it exists, is powered on, has the expected network connectivity, and can reach the workload cluster API
endpoint and required image registries.

If access to the machine is available, inspect the bootstrap and node service logs. The exact commands depend on the
operating system and bootstrap provider. For kubeadm with cloud-init, see
[Node bootstrap failures when using CABPK with cloud-init](#node-bootstrap-failures-when-using-cabpk-with-cloud-init).

### 7. Collect information for further investigation

If the cause is still unclear, collect enough information to reproduce the timeline before resources or Events are
deleted. At minimum, include:

* Cluster API and provider versions from `clusterctl get providers`.
* Kubernetes versions for the management and workload clusters.
* `clusterctl describe cluster` output with `--show-conditions all --grouping=false --echo`.
* YAML and Events for the resources that are not progressing.
* Relevant provider logs with timestamps and the time range in which the failure occurred.
* The infrastructure or bootstrap logs, when applicable.
* The steps that triggered the problem and whether it is reproducible.

Remove credentials, kubeconfig data, Secrets, cloud-init user data, tokens, and other sensitive values before sharing
the collected information.

## Troubleshooting Quick Start with Docker (CAPD)

<aside class="note warning">

<h1>Warning</h1>

If you've run the Quick Start before ensure that you've [cleaned up](./quick-start.md#clean-up) all resources before trying it again. Check `docker ps` to ensure there are no running containers left before beginning the Quick Start.

</aside>

This guide assumes you've completed the [apply the workload cluster](./quick-start.md#apply-the-workload-cluster) section of the Quick Start using Docker.

When running `clusterctl describe cluster capi-quickstart` to verify the created resources, we expect the output to be similar to this (**note: this is before installing the Calico CNI**).

```shell
NAME                                                           READY  SEVERITY  REASON                       SINCE  MESSAGE
Cluster/capi-quickstart                                        True                                          46m
├─ClusterInfrastructure - DockerCluster/capi-quickstart-94r9d  True                                          48m
├─ControlPlane - KubeadmControlPlane/capi-quickstart-6487w     True                                          46m
│ └─3 Machines...                                              True                                          47m    See capi-quickstart-6487w-d5lkp, capi-quickstart-6487w-mpmkq, ...
└─Workers
  └─MachineDeployment/capi-quickstart-md-0-d6dn6               False  Warning   WaitingForAvailableMachines  48m    Minimum availability requires 3 replicas, current 0 available
    └─3 Machines...                                            True                                          47m    See capi-quickstart-md-0-d6dn6-584ff97cb7-kr7bj, capi-quickstart-md-0-d6dn6-584ff97cb7-s6cbf, ...
```

Machines should be started, but Workers are not because Calico isn't installed yet. You should be able to see the containers running with `docker ps --all` and they should not be restarting.

If you notice Machines are failing to start/restarting your output might look similar to this:

```shell
clusterctl describe cluster capi-quickstart
NAME                                                           READY  SEVERITY  REASON                       SINCE  MESSAGE
Cluster/capi-quickstart                                        False  Warning   ScalingUp                    57s    Scaling up control plane to 3 replicas (actual 2)
├─ClusterInfrastructure - DockerCluster/capi-quickstart-n5w87  True                                          110s
├─ControlPlane - KubeadmControlPlane/capi-quickstart-6587k     False  Warning   ScalingUp                    57s    Scaling up control plane to 3 replicas (actual 2)
│ ├─Machine/capi-quickstart-6587k-fgc6m                        True                                          81s
│ └─Machine/capi-quickstart-6587k-xtvnz                        False  Warning   BootstrapFailed              52s    1 of 2 completed
└─Workers
  └─MachineDeployment/capi-quickstart-md-0-5whtj               False  Warning   WaitingForAvailableMachines  110s   Minimum availability requires 3 replicas, current 0 available
    └─3 Machines...                                            False  Info      Bootstrapping                77s    See capi-quickstart-md-0-5whtj-5d8c9746c9-f8sw8, capi-quickstart-md-0-5whtj-5d8c9746c9-hzxc2, ...
```

In the example above we can see that the Machine `capi-quickstart-6587k-xtvnz` has failed to start. The reason provided is `BootstrapFailed`.

To investigate why a machine fails to start you can inspect the conditions of the objects using `clusterctl describe --show-conditions all cluster capi-quickstart`. You can get more detailed information about the status of the machines using `kubectl describe machines`.

To inspect the underlying infrastructure - in this case Docker containers acting as Machines - you can access the logs using `docker logs <MACHINE-NAME>`. For example:

```shell
docker logs capi-quickstart-6587k-xtvnz
(...)
Failed to create control group inotify object: Too many open files
Failed to allocate manager object: Too many open files
[!!!!!!] Failed to allocate manager object.
Exiting PID 1...
```

To resolve this specific error please read [Cluster API with Docker  - "too many open files"](#cluster-api-with-docker----too-many-open-files).

## Node bootstrap failures when using CABPK with cloud-init

Failures during Node bootstrapping can have a lot of different causes. For example, Cluster API resources might be
misconfigured or there might be problems with the network. The following steps describe how bootstrap failures can
be troubleshooted systematically.

1. Access the Node via ssh.
2. Take a look at cloud-init logs via `less /var/log/cloud-init-output.log` or `journalctl -u cloud-init --since "1 day ago"`.
   (Note: cloud-init persists logs of the commands it executes (like kubeadm) only after they have returned.)
3. It might also be helpful to take a look at `journalctl --since "1 day ago"`.
4. If you see that kubeadm times out waiting for the static Pods to come up, take a look at:
   1. containerd: `crictl ps -a`, `crictl logs`, `journalctl -u containerd`
   2. Kubelet: `journalctl -u kubelet --since "1 day ago"`
      (Note: it might be helpful to increase the Kubelet log level by e.g. setting `--v=8` via
      `systemctl edit --full kubelet && systemctl restart kubelet`)
5. If Node bootstrapping consistently fails and the kubeadm logs are not verbose enough, the `kubeadm` verbosity
   can be increased via `KubeadmConfigSpec.Verbosity`.

## Labeling nodes with reserved labels such as `node-role.kubernetes.io` fails with kubeadm error during bootstrap

Self-assigning Node labels such as `node-role.kubernetes.io` using the kubelet `--node-labels` flag
(see `kubeletExtraArgs` in the [CABPK examples](https://github.com/kubernetes-sigs/cluster-api/tree/main/bootstrap/kubeadm))
is not possible due to a security measure imposed by the
[`NodeRestriction` admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)
that kubeadm enables by default.

Assigning such labels to Nodes must be done after the bootstrap process has completed:

```bash
kubectl label nodes <name> node-role.kubernetes.io/worker=""
```

For convenience, here is an example one-liner to do this post installation

```bash
# Kubernetes 1.19 (kubeadm 1.19 sets only the node-role.kubernetes.io/master label)
kubectl get nodes --no-headers -l '!node-role.kubernetes.io/master' -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}' | xargs -I{} kubectl label node {} node-role.kubernetes.io/worker=''
# Kubernetes >= 1.20 (kubeadm >= 1.20 sets the node-role.kubernetes.io/control-plane label)
kubectl get nodes --no-headers -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}' | xargs -I{} kubectl label node {} node-role.kubernetes.io/worker=''
```

## Cluster API with Docker - common issues with docker - 

When provisioning workload clusters using Cluster API with the Docker infrastructure provider,
provisioning might be stuck:

1. if there are stopped containers on your machine from previous runs. Clean unused containers with [docker rm -f ](https://docs.docker.com/engine/reference/commandline/rm/).

2. if the Docker space on your disk is being exhausted
    * Run [docker system df](https://docs.docker.com/engine/reference/commandline/system_df/) to inspect the disk space consumed by Docker resources.
    * Run [docker system prune --volumes](https://docs.docker.com/engine/reference/commandline/system_prune/) to prune dangling images, containers, volumes and networks.


## Cluster API with Docker  - "too many open files"
When creating many nodes using Cluster API and Docker infrastructure, either by creating large Clusters or a number of small Clusters, the OS may run into inotify limits which prevent new nodes from being provisioned.
If the error  `Failed to create inotify object: Too many open files` is present in the logs of the Docker Infrastructure provider this limit is being hit.

On Linux this issue can be resolved by increasing the inotify watch limits with:

```bash
sysctl fs.inotify.max_user_watches=1048576
sysctl fs.inotify.max_user_instances=8192
```

Newly created clusters should be able to take advantage of the increased limits.

### MacOS and Docker Desktop -  "too many open files"
This error was also observed in Docker Desktop 4.3 and 4.4 on MacOS. It can be resolved by updating to Docker Desktop for Mac 4.5 or using a version lower than 4.3.

[The upstream issue for this error is closed as of the release of Docker 4.5.0](https://github.com/docker/for-mac/issues/6071)

Note: The below workaround is not recommended unless upgrade or downgrade cannot be performed.

If using a version of Docker Desktop for Mac 4.3 or 4.4, the following workaround can be used:

Increase the maximum inotify file watch settings in the Docker Desktop VM:

1) Enter the Docker Desktop VM
```bash
nc -U ~/Library/Containers/com.docker.docker/Data/debug-shell.sock
```
2) Increase the inotify limits using sysctl
```bash
sysctl fs.inotify.max_user_watches=1048576
sysctl fs.inotify.max_user_instances=8192
```
3) Exit the Docker Desktop VM
```bash
exit
```

## Failed clusterctl init - 'failed to get cert-manager object'

When using older versions of Cluster API 0.4 and 1.0 releases - 0.4.6, 1.0.3 and older respectively - Cert Manager may not be downloadable due to a change in the repository location. This will cause `clusterctl init` to fail with the error:

```bash
clusterctl init --infrastructure docker
```
```bash
Fetching providers
Installing cert-manager Version="v1.11.0"
Error: action failed after 10 attempts: failed to get cert-manager object /, Kind=, /: Object 'Kind' is missing in 'unstructured object has no kind'
```

This error was fixed in more recent Cluster API releases on the 0.4 and 1.0 release branches. The simplest way to resolve the issue is to upgrade to a newer version of Cluster API for a given release. For who need to continue using an older release it is possible to override the repository used by `clusterctl init` in the clusterctl config file. The default location of this file is in `$XDG_CONFIG_HOME/cluster-api/clusterctl.yaml`.

To do so add the following to the file:
```yaml
cert-manager:
  url: "https://github.com/cert-manager/cert-manager/releases/latest/cert-manager.yaml"
```

Alternatively a Cert Manager yaml file can be placed in the [clusterctl overrides layer](../clusterctl/configuration.md#overrides-layer) which is by default in `$XDG_CONFIG_HOME/cluster-api/overrides`. A Cert Manager yaml file can be placed at e.g. `$XDG_CONFIG_HOME/cluster-api/overrides/cert-manager/v1.11.0/cert-manager.yaml`

More information on the clusterctl config file can be found at [its page in the book](../clusterctl/configuration.md#clusterctl-configuration-file)

## Failed clusterctl upgrade apply - 'failed to update cert-manager component'

Upgrading Cert Manager may fail due to a breaking change introduced in Cert Manager release v1.6.
An upgrade using `clusterctl` is affected when:

* using `clusterctl` in version `v1.1.4` or a more recent version.
* Cert Manager lower than version `v1.0.0` did run in the management cluster (which was shipped in Cluster API until including `v0.3.14`).

This will cause `clusterctl upgrade apply` to fail with the error:

```bash
clusterctl upgrade apply
```

```bash
Checking cert-manager version...
Deleting cert-manager Version="v1.5.3"
Installing cert-manager Version="v1.7.2"
Error: action failed after 10 attempts: failed to update cert-manager component apiextensions.k8s.io/v1, Kind=CustomResourceDefinition, /certificaterequests.cert-manager.io: CustomResourceDefinition.apiextensions.k8s.io "certificaterequests.cert-manager.io" is invalid: status.storedVersions[0]: Invalid value: "v1alpha2": must appear in spec.versions
```

The Cert Manager maintainers provide documentation to [migrate the deprecated API Resources](https://cert-manager.io/docs/installation/upgrading/remove-deprecated-apis/#upgrading-existing-cert-manager-resources) to the new storage versions to mitigate the issue.

More information about the change in Cert Manager can be found at [their upgrade notes from v1.5 to v1.6](https://cert-manager.io/docs/installation/upgrading/upgrading-1.5-1.6).

## Clusterctl failing to start providers due to outdated image overrides

clusterctl allows users to configure [image overrides](../clusterctl/configuration.md#image-overrides) via the clusterctl config file.
However, when the image override is pinning a provider image to a specific version, it could happen that this
conflicts with clusterctl behavior of picking the latest version of a provider.

E.g., if you are pinning KCP images to version v1.0.2 but then clusterctl init fetches yamls for version v1.1.0 or greater KCP will
fail to start with the following error:

```bash
invalid argument "ClusterTopology=false,KubeadmBootstrapFormatIgnition=false" for "--feature-gates" flag: unrecognized feature gate: KubeadmBootstrapFormatIgnition
```

In order to solve this problem you should specify the version of the provider you are installing by appending a
version tag to the provider name:

```bash
clusterctl init -b kubeadm:v1.0.2 -c kubeadm:v1.0.2 --core cluster-api:v1.0.2 -i docker:v1.0.2
```

Even if slightly verbose, pinning the version provides a better control over what is installed, as usually
required in an enterprise environment, especially if you rely on an internal repository with a separated
software supply chain or a custom versioning schema.

## Managed Cluster and co-authored slices

As documented in [#6320](https://github.com/kubernetes-sigs/cluster-api/issues/6320) managed topologies
assumes a slice to be either authored from templates or by the users/the infrastructure controllers.

In cases the slice is instead co-authored (templates provide some info, the infrastructure controller
fills in other info) this can lead to infinite reconcile.

A solution to this problem is being investigated, but in the meantime you should avoid co-authored slices.

## Failed to removed fields from lists using Server Side Apply

The Cluster API projects is continuously improving its API, including improving the support for Server Side Apply, 
which allows for a more granular ownership of list items. 

However, when transitioning from atomic lists to map lists, there are edge cases not supported
and this can lead to a SSA patches failing to remove an item in a list.

Note: the issue only occurs in a very specific scenario, most of the users are not affected
(e.g. client-side apply or "continuous" SSA with GitOps tools works as expected)

Example of fields transitioned from atomic lists to map lists are e.g.

- `cluster.spec.topology.variables`
- `cluster.spec.topology.workers.machineDeployments`

In case you face this issue, please use kubectl edit or kubectl apply with client-side apply
to remove the item from the list; after the item is removed everything should work as expected.

See [comment](https://github.com/kubernetes-sigs/cluster-api/issues/11857#issuecomment-2740339933) for more details.

## kubeadm join fails after upgrading to Kubernetes patch releases

When upgrading a cluster to any of the following Kubernetes patch releases,
`kubeadm join` completes but the control plane rollout gets stuck because the
API server cannot proxy requests to the kubelet:

- v1.36.1
- v1.35.5
- v1.34.8
- v1.33.12

**Cause:** These releases include a kubeadm security improvement
([kubernetes/kubernetes#138957](https://github.com/kubernetes/kubernetes/pull/138957))
that reduces the scope of the API server's kubelet client credentials.
A dedicated `ClusterRoleBinding` named `kubeadm:apiserver-kubelet-client` is now
required, binding the API server's certificate CN (`kube-apiserver-kubelet-client`)
to the `system:kubelet-api-admin` ClusterRole.

Without this binding, the API server cannot proxy or exec to kubelets on nodes
with the new certificates. KCP logs will show errors similar to:

```text
unable to upgrade connection: Forbidden (user=kube-apiserver-kubelet-client,
verb=create, resource=nodes, subresource(s)=[proxy])
```

CAPI releases prior to v1.11.11, v1.12.8, and v1.13.2 do not create this binding
during upgrades, causing the control plane rollout to get stuck.

**Fix:** Before upgrading Kubernetes to the above patch versions, upgrade CAPI to
one of the following releases which include the corresponding fix
([cluster-api#13664](https://github.com/kubernetes-sigs/cluster-api/pull/13664)):

- v1.13.2 or later
- v1.12.8 or later
- v1.11.11 or later

**If you are already in a broken state** and cannot upgrade CAPI first, manually
create the missing `ClusterRoleBinding` on the workload cluster:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubeadm:apiserver-kubelet-client
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubelet-api-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: kube-apiserver-kubelet-client
```

After applying this manifest to the workload cluster, retry the upgrade.
