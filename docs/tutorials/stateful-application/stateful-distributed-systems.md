---
assignees:
- bprashanth
- enisoc
- erictune
- foxish
- janetkuo
- kow3ns
- smarterclayton
---

{% capture overview %}
This tutorial demonstrates how to deploy 
[Apache Zookeeper](https://zookeeper.apache.org) on Kubernetes using 
[StatefulSets](/docs/concepts/controllers/statefulsets/). It demonstrates how 
StatefulSets interact with other Kubernetes concepts to allow users to deploy and 
manage [CP Distributed Systems](https://people.eecs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf).
{% endcapture %}

{% capture prerequisites %}

Before starting this tutorial, you should be familiar with the following 
Kubernetes concepts.

* [Pods](/docs/user-guide/pods/single-container/)
* [Cluster DNS](/docs/admin/dns/)
* [Headless Services](/docs/user-guide/services/#headless-services)
* [PersistentVolumes](/docs/user-guide/volumes/)
* [PersistentVolume Provisioning](http://releases.k8s.io/{{page.githubbranch}}/examples/experimental/persistent-volume-provisioning/)
* [StatefulSets](/docs/concepts/controllers/statefulsets/)
* [PodDisruptionBudgets](/docs/admin/disruptions/#specifying-a-poddisruptionbudget)
* [Interpod Affinity](/docs/user-guide/node-selection/)
* [kubectl CLI](/docs/user-guide/kubectl)

You should have a passing familiarity with 
[Apache Zookeeper](https://zookeeper.apache.org), but an in depth knowledge of 
ZooKeeper internals is not required.

You will require a cluster with at least three nodes. Each node will require
at least 2 CPUs and 8 GiB of Memory.

This tutorial assumes that your cluster is configured to dynamically provision 
PersistentVolumes. If your cluster is not configured to do so, you
will have to manually provision thee 20 GiB volumes prior to starting this 
tutorial.



{% endcapture %}

{% capture objectives %}

After this tutorial, you will know the following.

* How to deploy a fixed size StatefulSet.
* How to configure the Pods of a StatefulSet using ConfigMaps.
* How to spread the Pods of a StatefulSet to be tolerant to node failure.
* How to use PodDistruptionBugets to protect the Pods in a StatefulSet during 
  [evictions](/docs/admin/disruptions/).

{% endcapture %}

{% capture lessoncontent %}
### ZooKeeper Basics

You do not need an in depth knowledge of ZooKeeper to complete this tutorial, 
but a very basic treatment is nessecary to understand the concepts presented 
here.

#### What is ZooKeeper?
Quoting the [ZooKeeper Overview](https://zookeeper.apache.org/doc/current/zookeeperOver.html#ch_DesignOverview)
"ZooKeeper is a distributed, open-source coordination service for distributed 
applications. It exposes a simple set of primitives that distributed 
applications can build upon to implement higher level services for 
synchronization, configuration maintenance, and groups and naming."

To simplify the above, ZooKeeper allows you to read, write, and observe updates 
to data. Data are organized in a file system like hierarchy and
replicated to all ZooKeeper servers in the Ensemble. All operations on data are 
strongly consistent. 

#### ZooKeeper Guarantees
By consistent, we mean that operations in ZooKeeper are linearizable. In fact, as described in the 
[Zookeeper Guarantees](https://zookeeper.apache.org/doc/current/zookeeperOver.html#Guarantees),
ZooKeeper provides the following to clients.

* Sequential Consistency - Updates from a client will be applied in the order that they were sent.
* Atomicity - Updates either succeed or fail. No partial results.
* Single System Image - A client will see the same view of the service regardless of the server that it connects to.
* Reliability - Once an update has been applied, it will persist from that time forward until a client overwrites the update.
* Timeliness - The clients view of the system is guaranteed to be up-to-date within a certain time bound.

#### ZooKeeper Architecture Basics

To achieve its gaurantees ZooKeeper uses a protocol called 
[Zab](https://pdfs.semanticscholar.org/b02c/6b00bd5dbdbd951fddb00b906c82fa80f0b3.pdf) 
to achieve goals similar to the 
[Raft](https://www.usenix.org/system/files/conference/atc14/atc14-paper-ongaro.pdf) 
protocol used by [etcd](https://coreos.com/etcd/docs/latest/).
In a ZooKeeper ensemble (a group of servers), the servers use the Zab protocol 
to elect a leader. 

All mutations must be observed by a quorum (without respect
to weighted quorums this may be thought of as a majority) of the ensemble that 
contains the leader before they are acknowledged. 

Even though ZooKeeper servers serve all data from memory, all mutations are 
committed to a WAL (Write Ahead Log) on disk. Periodically, in order to prevent 
the WAL from growing without bound, the in memory data is snapshotted to disk 
allowing the process to safely recover without previous entries to the WAL. This
allows those entries to be discarded, keeping the WAL from growing without 
bound.

###Creating a ZooKeeper Ensemble

The example below contains a Headless Service (`zk-headless`), 
a ConfigMap (`zk-config`), a PodDisruptionBudget (`zk-budget`), and a 
StatefulSet (`zk`).

{% include code.html language="yaml" file="zookeeper.yaml" ghlink="/docs/tutorials/stateful-application/zookeeper.yaml" %}

Download the example and save it to a file named `zookeeper.yaml`

Open a command terminal, and use 
[`kubectl get`](/docs/user-guide/kubectl/kubectl_get/)  to watch for the 
creation of the Pods in the `zk` StatefulSet.

```shell
kubectl get pods -w -l app=zk
```

Open a second command terminal, and use 
[`kubectl create`](/docs/user-guide/kubectl/kubectl_create/)  to create the 
manifest contained in zookeeper.yaml.

```shell
kubectl create -f zookeeper.yaml
```

You will see the following output in the second terminal indicating that the 
Headless Service, ConfigMap, PodDisruptionBudget, and StatefulSet were created.

```shell
service "zk-headless" created
configmap "zk-config" created
poddisruptionbudget "zk-budget" created
statefulset "zk" created
```

In the first terminal, you should see output like the following. 

```shell
kubectl get pods -w -l app=zk
NAME      READY     STATUS              RESTARTS   AGE
zk-0      0/1       ContainerCreating   0          19s
NAME      READY     STATUS    RESTARTS   AGE
zk-0      0/1       Running   0          41s
zk-0      1/1       Running   0         57s
zk-1      0/1       Pending   0         0s
zk-1      0/1       Pending   0         0s
zk-1      0/1       ContainerCreating   0         0s
zk-1      0/1       Running   0         33s
zk-1      1/1       Running   0         50s
zk-2      0/1       Pending   0         0s
zk-2      0/1       Pending   0         0s
zk-2      0/1       ContainerCreating   0         0s
zk-2      0/1       Running   0         33s
```

Wait until all of the Pods are [Running and Ready](/docs/user-guide/pod-states).
Use `CTRL-C` to send a `SIGTERM` to the `kubectl get` command.

#### Sanity Test

The most basic sanity test is to write some data to one ZooKeeper server and 
to read the data from another. The image used for this ensemble contains 
`zkCli.sh` which can be used for this purpose.

[`kubectl exec`](/docs/user-guide/kubectl/kubectl_create/) command to write 
`world` to the path `/hello` to the zk-0 Pod.

```shell
kubectl exec zk-0 zkCli.sh create /hello world
```

The ouptut of this command should end with the statement below.

```shell
Created /hello
```

Get the data from the `zk-1` Pod.

```shell
kubectl exec zk-0 zkCli.sh get /hello
```

You should see `world` in the console output as below.

```
WatchedEvent state:SyncConnected type:None path:null
world
cZxid = 0x100000009
ctime = Wed Nov 30 00:14:26 UTC 2016
mZxid = 0x100000009
mtime = Wed Nov 30 00:14:26 UTC 2016
pZxid = 0x100000009
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0
```

#### About The Image

You may have noticed that the image used for the ZooKeeper ensemble took some
time to load. This is because the image is based off of the latest LTS version 
of Ubuntu, which is much larger than the smaller Apline or Busy Box images.

When deploying a JVM based distributed system, you may be tempted to, and 
ultimately decide to, deploy images based on OpenJDK running on Alpine or 
Busy Box, but you should be aware that this requires the JVM to be bulit against
mucl or ulibc. The most common distributions of the JVM, on Linux, are
built and tested against glibc.

Prior to optimizing for a tiny image, you should verify that JVM version you 
are using is well tested against libc version of the image, and that the 
appliation you are deploying is well tested against that JVM version.

We chose the latest LTS Ubunutu image because it supported OS for commercial 
Hadoop distributions, and because the JVM version avaiable through its software
repositories is up to date and contains the latest security patches.

### Configuring the Ensemble

### Understanding the code
Here's something interesting about the code you ran in the preceding steps.
{% endcapture %}

{% capture cleanup %}
* Delete this.
* Stop this.
{% endcapture %}

{% capture whatsnext %}
* Learn more about [this](...).
* See this [related tutorial](...).
{% endcapture %}

{% include templates/tutorial.md %}
