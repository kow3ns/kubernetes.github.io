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
This tutorial provides an introduction to the 
[Stateful Set](/docs/concepts/controllers/statefulsets.md) concept. It 
demonstrates how to create, delete, scale, and update the container image of a 
Stateful Set. It also demonstrates how Stateful Sets interact with the 
[Pod](/docs/user-guide/pods/single-container/) concept.
{% endcapture %}

{% capture prerequisites %}
Before you begin this tutorial, you should familiarize yourself with the 
following Kubernetes concepts.

* [Pods](/docs/user-guide/pods/single-container/)
* [Cluster DNS](/docs/admin/dns/)
* [Headless Services](/docs/user-guide/services/#headless-services)
* [Persistent Volumes](/docs/user-guide/volumes/)
* [Persistent Volume Provisioning](http://releases.k8s.io/{{page.githubbranch}}/examples/experimental/persistent-volume-provisioning/)
* [Stateful Sets](/docs/concepts/controllers/statefulsets.md)
* [Kubeclt](/docs/user-guide/kubectl)
{% endcapture %}

{% capture objectives %}
Stateful Sets are intended to be used with stateful applications and stateful 
distributed systems. However, the administration of stateful applications and 
distributed systems on Kubernetes is a broad, complex topic. In order to 
demonstrate the basic features of a Stateful Set, and to not conflate the former 
topic with the latter, a simple web application will be used as a running 
example throughout this tutorial.

After this tutorial, you will be familiar with the following.

* How to create a Stateful Set
* How to delete a Stateful Set
* How to scale a Stateful Set
* Howe to update the container image of a Stateful Set's Pods
* How a Stateful Set schedules Pods
* The stick identity of a Stateful Set's Pods
{% endcapture %}

{% capture lessoncontent %}
### Creating a Stateful Set 

Begin by creating a Stateful Set using the example below. It is similar to the 
example presented in the
[Stateful Sets](/docs/concepts/controllers/statefulsets.md) concept. It creates 
a [Headless Services](/docs/user-guide/services/#headless-services), `nginx`, to 
control the domain of the Stateful Set, `web`. 

{% include code.html language="yaml" file="web.yaml" ghlink="/docs/tutorials/stateful-application/web.yaml" %}

Use [`kubeclt create`](/docs/user-guide/kubectl/kubectl_create/) to create the 
Headless Service and the Stateful Set.

```shell
$ kubectl create -f web.yml 
service "nginx" created
statefulset "web" created
```

The command above creates two Pods, each running a 
[NGINX](https://www.nginx.com) webserver.

Use [`kubectl get`](/docs/user-guide/kubectl/kubectl_get/) to verify that the 
Headless Service and Stateful Set were successfully created.

```shell
$ kubectl get service nginx
NAME      CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     None         <none>        80/TCP    12s

$ kubectl get statefulset web
NAME      DESIRED   CURRENT   AGE
web       2         1         20s
```

### Pods in a Stateful Set

Use [`kubectl get`](/docs/user-guide/kubectl/kubectl_get/) to view the Status of
the Stateful Set's Pods. The `-l app=nginx` parameter will ensure that only the 
Pods created by the Stateful Set will be retrieved.

```shell
kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          10m
web-1     1/1       Running   0          9m
```

After a few seconds, this command should display the above, indicating that 
the Stateful Set has successfully created two Pods, `web-0` and `web-1`.

#### Pod Identity

As mentioned in the [Stateful Sets](/docs/concepts/controllers/statefulsets.md) 
concept, the Pods in a Stateful Set have a sticky, unique identity. Let's 
explore what this means in the context of the `web` Stateful Set.

##### Ordinal Index

Above, when you got Stateful Set's Pods using, you saw two Pods, `web-0` and 
`web-1`. The Pods names take the form `$(statefulset name)-$(ordinal index)`.
The ordinal index, assinged by the Stateful Set controller, is unique to the
Pod, and it uniquely identifies the Pod in the Set throughout the Pod's 
lifecycle.

##### Network Identity

Each Pod has a stable hostname based on its ordinal index. Use
[`kubectl exec`](/docs/user-guide/kubectl/kubectl_exec/) to execute the hostname 
command in each Pod and retrieve its hostname. 

```shell
$ for i in 0 1; do kubectl exec web-$i -- sh -c 'hostname'; done
web-0
web-1
```

If the Pods are rescheduled, their hostnames will not change. Pod `web-0` will 
always have hostname web-0 and Pod `web-1` will always have hostname web-1.

Use [`kubectl run`](/docs/user-guide/kubectl/kubectl_run/) to execute 
a container that provides the nslookup command from the dnsutils package. Using 
nslookup on the Pods' hostnames you can examine their in-cluster DNS 
addresses.

```shell
$ kubectl run -i --tty --image busybox dns-test --restart=Never /bin/sh 
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.2.6

$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.3.4
```

The CNAME of the headless serivce points to SRV records (one for each Pod whose 
readiness check has passed). The SRV records point to A record entries that 
contain the Pods IP addresses. As long as the Headless Service exists, when 
`web-0` and `web-1` are (re)scheduled, they will always have the same SRV 
records after their readiness checks succeed. However, the IP Addresses in 
their A records may change. This will be explored further in the 
[Deleting Pods](#deleting-pods) section.

##### Stable Storage

Get the Pods' Persistent Volumes Claims. These are created based on the 
`volumeClaimsTemplate` field in the Stateful Set's `spec`. 

```shell
$kubectl get pvc -l app=nginx
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-883e5172-b03b-11e6-adfc-42010a800002   1Gi        RWO           1h
www-web-1   Bound     pvc-88445cb0-b03b-11e6-adfc-42010a800002   1Gi        RWO           1h
```

The Persisteng Volume Cliams have resulted in the provisioning of two 
Persistent Volumes. Use the identifiers of the Persistent Volume Cliams to get 
the Persistent Volumes.

```shell
$ kubectl get pv pvc-883e5172-b03b-11e6-adfc-42010a800002 pvc-88445cb0-b03b-11e6-adfc-42010a800002
NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM               REASON    AGE
pvc-883e5172-b03b-11e6-adfc-42010a800002   1Gi        RWO           Delete          Bound     default/www-web-0             1h
pvc-88445cb0-b03b-11e6-adfc-42010a800002   1Gi        RWO           Delete          Bound     default/www-web-1             1h

```

The containers NGINX webservers, by default, will serve an index file at 
`/usr/share/nginx/html/index.html`. The `volumeMounts` field in the 
Stateful Sets `spec` ensures that the `/usr/share/nginx/html` directory is 
backed by a Persistent Volume.

Write the Pods' hostnames to their `index.html` files and verify that the NGINX 
webservers serve the hostnames.

```shell
$ for i in 0 1; do
  kubectl exec web-$i -- sh -c 'echo $(hostname) > /usr/share/nginx/html/index.html';
done

$ for i in 0 1; do kubectl exec -it web-$i -- curl localhost; done
web-0
web-1
```

Because this data has been written to a Persistent Volume, and because the 
Persistent Volumes stick to the Pods, `web-0` will continue to server web-0 and 
`web-1` will continue to serve web-1 until the index.html files are modified.
This is true even if the Pods are rescheduled to different nodes. This will be 
demonstrated in detail in the [Deleting Pods](#deleting-pods) section.

### Deleting Stateful Sets

There are three different types of deletion that pertain to Stateful Sets, and 
each has different semantics.

1. You can delete a Stateful Set's Pods (In this case the Stateful Set will 
reschedule the Pods).
2. You can delete a Stateful Set and orphan its Pods.
3. You can delete a Stateful Set and its Pods (a cascading delete).
We will examine all three cases.

#### Deleting Pods

When a Stateful Sets Pods are deleted, the Stateful Set controller will 
reschedule the Pods (though potentially not on the same node). Use
[`kubectl delete`](/docs/user-guide/kubectl/kubectl_delete/) to delete all the 
Pods in the Stateful Set.

```shell
$ kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```
Use [`kubectl get`](/docs/user-guide/kubectl/kubectl_get/), with the `-w` parameter, 
to watch the Pods being relaunched (You may want to do this in a separate 
terminal while you delete the pods). 

```shell
$kubectl get -w pod -l app=nginx
NAME      READY     STATUS              RESTARTS   AGE
web-0     0/1       ContainerCreating   0          2s
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          3s
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         6s
```
Once all the Pods are Running agian, examine their hostnames and in-cluster 
DNS entries.

```shell
$ for i in 0 1; do kubectl exec web-$i -- sh -c 'hostname'; done
web-0
web-1

$ kubectl run -i --tty --image busybox dns-test --restart=Never /bin/sh 
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.2.9

$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.3.6
```

The Pods' ordinals, hostnames, SRV records, and A record names have not changed, 
but the IP addresses associated with the Pods may have changed. In the cluster 
used for this turorial, they have. This is why it is important not to configure 
other applications to connect to Pods in a Stateful Set by IP address.

If you need to find and connect to the active members of a Stateful Set, you 
should query the CNAME of the Headless Service 
(e.g. `nginx.default.svc.cluster.local`). The SRV records associated with the 
CNAME will contains only the Pods in the Stateful Set that are Running and 
Ready.

Alternatively, if you only need a predefined set of addresses, for instance if 
your application already implements connection logic that tests for 
liveness and readiness, you should use the SRV records of the Pods in the 
Stateful Set (e.g `web-0.nginx.default.svc.cluster.local`, 
`web-1.nginx.default.svc.cluster.local`).

Examine the data served by the Pods' webservers.

```shell
$ for i in 0 1; do kubectl exec -it web-$i -- curl localhost; done
web-0
web-1
```

Even though the Pods have been rescheduled, they are still serving the data 
you entered prior to deleting the Pods. This is because the Persistent Volumes 
associated with the Stateful Sets `volumeCliamsTemplate` have been remounted 
to the mount points indicated by the `volumeMounts` section.

#### Non-Cascading Delete

Use [`kubectl delete`](/docs/user-guide/kubectl/kubectl_delete/) to delete the 
Stateful Set. Make sure to supply the `--cascade=false` parameter to the 
command. This parameter tells Kubernetes to only delete the Stateful Set, and to 
not delete any of its Pods.

```shell
$ kubectl delete statefulset web --cascade=false
```

Get the Pods to examine their status.

```shell
$ kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          56m
web-1     1/1       Running   0          56m
```

Even though `web` has been deleted, all of the Pods are still running. Delete 
`web-0`.

```shell
$ kubectl delete pod web-0
pod "web-0" deleted
```
This time, when you get the Pods that the Stateful Set created, you will find
only `web-1`. As the `web` Stateful Set has been deleted, `web-0` will not 
be rescheduled.

```shell
$ kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-1     1/1       Running   0          59m
```

Recreate the Stateful Set using the same command you did in the 
[Creating a Stateful Set](#creating-a-stateful-set) section. Note that, unless
you deleted the `nginx` Service ( which you should not have ), you will see 
an error indicating that the Service already exists.

```shell
kubectl create -f web.yaml 
statefulset "web" created
Error from server (AlreadyExists): error when creating "web.yaml": services "nginx" already exists
```

Ignore the error. It only indicates that an attempt was made to create the nginx
Headless Service even though that Service already exists. Get the Stateful Set's
Pods one more time.

```shell
kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          4m
web-1     1/1       Running   0          1h
```

Notice that `web-0` was recreated, but `web-1` was left untouched. When the 
Stateful Set was recreated, it repaired the state of the set by relaunching 
`web-0`,but, since `web-1` was already running, it adopdted this Pod.

Let's take another look at the contents of the `index.html` file served by the 
Pods' webservers.

```shell
$ for i in 0 1; do kubectl exec -it web-$i -- curl localhost; done
web-0
web-1
```

Even though we deleted both the Stateful Set and the `web-0` Pod, it still 
serves the hostname you originally entered into its `index.html`. This is because 
the Stateful Set never deletes the Persistent Volumes associated with a Pod. 
When you recreated the Stateful Set and it relaunched `web-0`, its original 
Persistent Volume was remounted.

#### Cascading Delete

Use [`kubectl delete`](/docs/user-guide/kubectl/kubectl_delete/) to delete the 
Stateful Set. This time, ommit the --cascade=false parameter.

```shell
$ kubectl delete statefulset web
statefulset "web" deleted
```

Attempt to get the Stateful Set's Pods. 

```shell
$ kubectl get pods -l app=nginx
No resources found.
```

This time no resources are found. A cascading delete will delete the Stateful 
Set, along with all of its Pods.

Note that, while a cascading delete will delete the Stateful Set and its Pods, 
it will not delete the Headless Service associated with the Sateful Set. You
must delete the `nginx` Service manually.

```shell
$ kubectl delete service nginx
service "nginx" deleted
```

Recreate the Stateful Set and Headless Service one more time, and inspect 
contents served by the Pods' webservers.

```shell
kubectl create -f web.yaml 
service "nginx" created
statefulset "web" created

$ for i in 0 1; do kubectl exec -it web-$i -- curl localhost; done
web-0
web-1
```

Even though you completely deleted the Stateful Set, and all of its Pods, the 
Pods are recreated with their Persistent Volumes mounted. 

### Updating a Stateful Set

You can't update any field of the Stateful Set except for `replicas` and 
`containers` in the podTemplate. Updating `replicas` will scale the Set.
Updating `containers` can be used to update the Stateful Set's Pods, but the 
updates are only applied when the Pods are rescheduled.

#### Scaling a Stateful Set

You can scale a Stateful Set by updating the `replicas` field. Stateful Sets
offer the following garuantees with respect to deployment and sclaing.

* For a Stateful Set with N replicas, when Pods are being deployed, they are created sequentially, in order from {0..N-1}. 
* When Pods are being deleted, they are terminated in reverse order, from {N-1..0}.
* Before a scaling operation is applied to a Pod, all of its predecessors must be Running and Ready. 
* Before a Pod is terminated, all of its successors must be completely shutdown.

Use [`kubectl scale`](/docs/user-guide/kubectl/kubectl_scale/) to scale the 
replicas, and use [`kubectl get`](/docs/user-guide/kubectl/kubectl_get/) to watch 
the scaling operation progress (You may want to watch the progress from a 
separate terminal window).

```shell
$ kubectl scale statefulset web --replicas=5
statefulset "web" scaled

$kubectl get pods -w -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          2h
web-1     1/1       Running   0          2h
NAME      READY     STATUS    RESTARTS   AGE
web-2     0/1       Pending   0          0s
web-2     0/1       Pending   0         0s
web-2     0/1       ContainerCreating   0         0s
web-2     1/1       Running   0         19s
web-3     0/1       Pending   0         0s
web-3     0/1       Pending   0         0s
web-3     0/1       ContainerCreating   0         0s
web-3     1/1       Running   0         18s
web-4     0/1       Pending   0         0s
web-4     0/1       Pending   0         0s
web-4     0/1       ContainerCreating   0         0s
web-4     1/1       Running   0         19s
```

As the Stateful Set controller scaled the number of replicas, it 
created each Pod sequentially, waiting each Pod's predecessor to be Running and 
Ready before launching the subsequent Pod.

Note that you can also use 
[`kubectl patch`](/docs/user-guide/kubectl/kubectl_pacth/) to achieve the same 
effect.

```shell
$ kubectl patch statefulset web -p '{"spec":{"replicas":3}}'
"web" patched
```

Scale the Stateful Set back down to 3 replicas and watch the scale down 
process.

```shell
$ kubectl scale statefulset web --replicas=3
statefulset "web" scaled

$ kubectl get pods -w -l app=nginx
NAME      READY     STATUS              RESTARTS   AGE
web-0     1/1       Running             0          3h
web-1     1/1       Running             0          3h
web-2     1/1       Running             0          55s
web-3     1/1       Running             0          36s
web-4     0/1       ContainerCreating   0          18s
NAME      READY     STATUS    RESTARTS   AGE
web-4     1/1       Running   0          19s
web-4     1/1       Terminating   0         24s
web-4     1/1       Terminating   0         24s
web-3     1/1       Terminating   0         42s
web-3     1/1       Terminating   0         42s


```

The controller deleted one Pod at a time in reverse order, and it waited 
for each to be completely shutdown 
(past its [terminationGracePeriodSeconds](/docs/user-guide/pods/index#termination-of-pods)) 
before deleting the next.

Get the Statefuls Sets Persistent Volume Claims. Note that we still have five 
Persistent Volume Claims and five Persistent Volumes.

```shell
$ kubectl get pvc -l app=nginx
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-883e5172-b03b-11e6-adfc-42010a800002   1Gi        RWO           23h
www-web-1   Bound     pvc-88445cb0-b03b-11e6-adfc-42010a800002   1Gi        RWO           23h
www-web-2   Bound     pvc-0b18f8ce-b0fe-11e6-adfc-42010a800002   1Gi        RWO           32m
www-web-3   Bound     pvc-0b1d1f40-b0fe-11e6-adfc-42010a800002   1Gi        RWO           32m
www-web-4   Bound     pvc-0b1f794c-b0fe-11e6-adfc-42010a800002   1Gi        RWO           32m
```

Just as you saw in 
[Deleting Pods in a Stateful Set](#deleting-pods-in-a-stateful-set), the 
Persitent Volumes mounted to the Pods of a Stateful Set are not deleted when 
the Stateful Set's Pods are deleted. This property can be used to facilliate 
container image upgrades.

#### Upgrading Container Images

Stateful Set currently *does not* support automated image upgrade. However, you 
can update the `image` field of any container in the podTemplate and delete 
Stateful Set's Pods one by one, the Stateful Set controller will recreate 
each Pod with the new image.

Use [`kubectl patch`](/docs/user-guide/kubectl/kubectl_pacth/) to update the 
container image for the Stateful Set and then delete `web-0`.

```shell
$ kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"gcr.io/google_containers/nginx-slim:0.7"}]'
"web" patched

$ kubectl delete pod web-0
pod "web-0" deleted
```

Get the Pods to view their container images.

```shell{% raw %}
$ for p in 0 1 2; do kubectl get po web-$p --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
gcr.io/google_containers/nginx-slim:0.7
gcr.io/google_containers/nginx-slim:0.8
gcr.io/google_containers/nginx-slim:0.8
{% endraw %}```

To complete the image update, delete the remianing two Pods, and watch the 
upgrade progress.

```shell
$ kubectl delete pod web-1 web-2
pod "web-1" deleted
pod "web-2" deleted

$ kubectl get pods -w -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          8m
web-1     1/1       Running   0          4h
web-2     1/1       Running   0          23m
NAME      READY     STATUS        RESTARTS   AGE
web-1     1/1       Terminating   0          4h
web-1     1/1       Terminating   0         4h
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-2     1/1       Terminating   0         23m
web-2     1/1       Terminating   0         23m
web-1     1/1       Running   0         4s
web-2     0/1       Pending   0         0s
web-2     0/1       Pending   0         0s
web-2     0/1       ContainerCreating   0         0s
web-2     1/1       Running   0         36s
```

Once all the Pods are Running and Ready, get the Pods to view the container 
images.

```shell{% raw %}
$ for p in 0 1 2; do kubectl get po web-$p --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
gcr.io/google_containers/nginx-slim:0.7
gcr.io/google_containers/nginx-slim:0.7
gcr.io/google_containers/nginx-slim:0.7
{% endraw %}```
{% endcapture %}

{% capture cleanup %}
* Follow the steps in the [Cascading Delete](#cascading-delete) section to 
delete the Stateful Set. Be sure you remember to delete the associated 
Headless Service.
{% endcapture %}

{% include templates/tutorial.md %}