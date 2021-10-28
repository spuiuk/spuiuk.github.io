## Manually Setting up a CTDB - Samba test clustered

### Minikube
We use minikube to setup the kubernetes environment. The command used is

```minikube start --nodes=3 --extra-disks=2 --memory 4096 --cpus 2```

- --nodes=3 sets up a 3 node cluster
- --extra-disks=2 which sets up 2 disks per node
- --memory=4096 and 4Gb of memory per node
- --cpus 2 with 2 cpus.

This could take a while as minikube downloads the required images to start up the kubernetes instance.

Once complete, you can test the setup using the commands

``` minikube status```

To get a list of all the pods running, use the command

```kubectl get pods -A```

### AD Server
Next we create an AD
```
git clone https://github.com/samba-in-kubernetes/samba-operator.git
cd samba-operator
./tests/test-deploy-ad-server.sh
```

The install of the AD setup ends by adding a zone entry to the coredns server. You can check this by looking at the coredns configmap
```
kubectl -n kube-system edit cm coredns
````
Look for an entry similar to
```
    domain1.sink.test:53 {
        errors
        cache 30
        forward . 10.244.0.10
    }
```

This forwards all lookups for domain domain1.sink.test to the AD server we created earlier.

### Rook - Ceph

We then use rook to create a Ceph cluster.
- [Ref](https://rook.io/docs/rook/v1.7/quickstart.html)

Switch back to the base directory and clone the rook repo.

```
git clone --single-branch --branch release-1.7 https://github.com/rook/rook.git
```
We create a Rook cluster using example manifests.
```
kubectl create -f rook/cluster/examples/kubernetes/ceph/crds.yaml
kubectl create -f rook/cluster/examples/kubernetes/ceph/common.yaml
kubectl create -f rook/cluster/examples/kubernetes/ceph/operator.yaml
kubectl create -f rook/cluster/examples/kubernetes/ceph/cluster.yaml
```
This could take a while. Run the command to Watch while all the required containers are created ie. all containers are either Running or Completed.\
```kubectl get pod -n rook-ceph -w```

We then add the required tools
```
kubectl create -f rook/cluster/examples/kubernetes/ceph/toolbox.yaml
kubectl create -f rook/cluster/examples/kubernetes/ceph/pool.yaml
kubectl create -f rook/cluster/examples/kubernetes/ceph/filesystem.yaml
```
and finally, setup the cephfs storage class
```
kubectl apply -f rook/cluster/examples/kubernetes/ceph/csi/cephfs/storageclass.yaml
```

You can confirm the creation of the storage class with\
```kubectl get sc```

You will need to set rook-cephfs storage class as default storage class.
Use the following commands
```
kubectl patch storageclass rook-cephfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Confirm that the cephfs storage class is set to default with the command
```
$ kubectl get storageclass
NAME                    PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-cephfs (default)   rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           true                   15s
standard                k8s.io/minikube-hostpath        Delete          Immediate           false                  24m
```
### Samba operator

The Samba Operator is not required to manually create the ctdb-samba test cluster. However for development purposes, we may want to install it on the test machine.  Back in the base directory, we first clone the samba-operator
```
git clone https://github.com/samba-in-kubernetes/samba-operator.git
```
Install the operator with
```
cd samba-operator/
make deploy
```

check for the samba-operator-system namespace
```
$ kubectl get namespaces
NAME                    STATUS   AGE
..
samba-operator-system   Active   12h
```

The operator also installs a few new api resource called smbshares.
```
$ kubectl api-resources
NAME                              SHORTNAMES        APIVERSION                             NAMESPACED   KIND
..
smbcommonconfigs                                    samba-operator.samba.org/v1alpha1      true         SmbCommonConfig
smbsecurityconfigs                                  samba-operator.samba.org/v1alpha1      true         SmbSecurityConfig
smbshares                                           samba-operator.samba.org/v1alpha1      true         SmbShare
..
```
### CTDB Samba cluster setup

We then start our test samba-ctdb-dm StatefulSet.

Clone the samba-container repo in the base directory
```
git clone https://github.com/samba-in-kubernetes/samba-container.git
cd samba-container
kubectl create -f ./examples/kubernetes/samba-ctdb-dm-sset.yml
```

The creation of the new pods could take some time but once done, you should see the resources similar to the one below.
```
$ kubectl get pvc
NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ctdb-shared-swc        Bound    pvc-8960d4b6-0d4c-45a9-8d7e-380be58783a5   1Gi        RWX            rook-cephfs    12h
samba-share-data-swc   Bound    pvc-e7d0d410-6ed1-4853-87ce-ebd9c8867d27   2Gi        RWX            rook-cephfs    12h

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
pvc-8960d4b6-0d4c-45a9-8d7e-380be58783a5   1Gi        RWX            Delete           Bound    default/ctdb-shared-swc        rook-cephfs             12h
pvc-e7d0d410-6ed1-4853-87ce-ebd9c8867d27   2Gi        RWX            Delete           Bound    default/samba-share-data-swc   rook-cephfs             12h

$ kubectl get statefulset
NAME                  READY   AGE
clustered-samba-swc   0/3     12h

$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
clustered-samba-swc-0              4/4     Running   0          12h
clustered-samba-swc-1              4/4     Running   0          12h
clustered-samba-swc-2              4/4     Running   0          12h
samba-ad-server-86b7dd9856-hgtm4   1/1     Running   0          12h
```
For more information on the yaml file **samba-contianer/examples/kubernetes/samba-ctdb-dm-sset.yml** is available in [ctdb_test.md](ctdb_test.md)
