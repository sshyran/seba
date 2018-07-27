## CORD Storage charts

These charts implement persistent storage within kubernetes.

These charts should provide a
[StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/)
named `cord-block` which,
[PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
can be created from.

As they create a `StorageClass` with the same name, their use is mutually
exclusive - you can't use both the Local charts and the Rook Ceph charts at the
same time.

## Local

For limited testing-only use, the `local-pv` chart provides a
[local](https://kubernetes.io/docs/concepts/storage/volumes/#local),
non-distributed PersistentVolume on one of the k8s nodes.

You must set the `volumeHostName` variable to the node name (`kubectl get
nodes` will list nodes) that the local PersistentVolume will be created on.

## Rook Ceph

[Rook](https://rook.github.io/) provides an abstraction layer for Ceph and
other distributed persistent data storage systems.

There are 3 Rook charts included with CORD:

 - `rook-operator`, which runs the volume provisioning portion of Rook (and is
   a thin wrapper around the upstream [rook-ceph
   chart](https://rook.github.io/docs/rook/v0.8/helm-operator.html)
 - `rook-cluster`, which creates the `cord-block` StorageClass
 - `rook-tools`, which provides a toolbox container for troubleshooting
   problems with Rook/Ceph

To create persistent volumes, you will need to load the first 2, with the third
only needed if something goes wrong.

### Rook Node Prerequisties

By default, all the nodes running k8s are expected to have a directory named
`/mnt/ceph` where the Ceph OSD data is stored.  In a production deployment,
this would ideally be located on a different block storage device.

There should be at least 3 nodes available to provide data redundancy

### Loading Rook Charts

First, add the `rook-beta` repo to helm, then load the `rook-operator` chart
into the `rook-ceph-system` namespace:

```shell
cd helm-charts/storage
helm repo add rook-beta https://charts.rook.io/beta
helm dep update rook-operator
helm install --namespace rook-ceph-system -n rook-operator rook-operator
```

Check that it's running (it will start the `rook-ceph-agent` and `rook-discover` DaemonSets):

```shell
$ kubectl -n rook-ceph-system get pods
NAME                                  READY     STATUS    RESTARTS   AGE
rook-ceph-agent-m26xw                 1/1       Running   0          51s
rook-ceph-agent-pwh7q                 1/1       Running   0          51s
rook-ceph-agent-rdfl8                 1/1       Running   0          51s
rook-ceph-operator-687b7bb6ff-ls4dw   1/1       Running   0          1m
rook-discover-gtxx6                   1/1       Running   0          51s
rook-discover-j8cpx                   1/1       Running   0          51s
rook-discover-q9jb7                   1/1       Running   0          51s
```

Next, load the `rook-cluster` chart:

```shell
helm install -n rook-cluster rook-cluster
```

Check that the cluster is running:

```shell
$ kubectl -n rook-ceph get pods
NAME                                  READY     STATUS      RESTARTS   AGE
rook-ceph-mgr-a-75654fb698-tx45f      1/1       Running     0          1m
rook-ceph-mon0-ms98s                  1/1       Running     0          2m
rook-ceph-mon1-qpkgx                  1/1       Running     0          2m
rook-ceph-mon2-j6cq6                  1/1       Running     0          2m
rook-ceph-osd-id-0-6cc589c899-jnfd4   1/1       Running     0          1m
rook-ceph-osd-id-1-6fc9bd8455-2ctk4   1/1       Running     0          1m
rook-ceph-osd-id-2-7b45b5657b-cg57j   1/1       Running     0          1m
rook-ceph-osd-prepare-node1-sjqpb     0/1       Completed   0          1m
rook-ceph-osd-prepare-node2-7qj5h     0/1       Completed   0          1m
rook-ceph-osd-prepare-node3-gqsnf     0/1       Completed   0          1m

$ kubectl -n rook-ceph get sc
NAME            PROVISIONER                    AGE
cord-block      ceph.rook.io/block             38s
...
```

Now you can create PersistentVolumeClaims on `cord-block` and they'll be
fulfilled by Rook.

### Troubleshooting Rook

Checking the `rook-ceph-operator` logs can be enlightening:

```shell
kubectl -n rook-ceph-system logs -f rook-ceph-operator-...
```

The [Rook toolbox container](https://rook.io/docs/rook/v0.8/toolbox.html) has
been containerized as the `rook-tools` chart, and provides a variety of tools
for debugging Rook and Ceph.

Load the `rook-tools` chart:

```shell
helm install -n rook-cluster rook-cluster
```

Once the container is running (check with `kubectl -n rook-ceph get pods`),
exec into it to run the tools:

```shell
kubectl -n rook-ceph exec -it rook-ceph-tools bash
```

### Cleaning up after Rook

The `rook-operator` chart will leave a DaemonSets and a few other things behind
after it's removed. Clean these up using these commands:

```shell
kubectl -n rook-ceph-system delete daemonset rook-ceph-agent
kubectl -n rook-ceph-system delete daemonset rook-discover
helm delete --purge rook-operator
```

If you have other charts that create `PersistentVolumeClaims`, you may need to
clean them up manually:

```shell
$ kubectl --all-namespaces get pvc
```

Files may be left behind in the Ceph storage directory, or as Rook configuration.

If you've used the `automation-tools/kubespray-installer` scripts to set up a
test environment as described above, you can delete all these files with the
following commands:

```shell
cd cord/automation-tools/kubespray-installer
ansible -i inventories/test/inventory.cfg -b -m shell -a "rm -rf /var/lib/rook && rm -rf /mnt/ceph/*" all
```

