## Mounting nfs shares created by the Ceph-CSI driver

I use the following alias when using kubectl with the rook-ceph namespace in the commands below.
```
$ alias kr=kubectl -n rook-ceph'
```

### First, we setup a NodePort.
```
$ kr get services
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
..
rook-ceph-nfs-my-nfs-a    ClusterIP   10.102.88.161    <none>        2049/TCP            26h

$ kr expose service rook-ceph-nfs-my-nfs-a --type=NodePort --port=2049 --name=node-nfs

$ kr get services
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
node-nfs                  NodePort    10.108.69.151    <none>        2049:31573/TCP      135m
..
rook-ceph-nfs-my-nfs-a    ClusterIP   10.102.88.161    <none>        2049/TCP            28h
```
We can determine the NodePort. In this case, it is 31573.

### Then get the ip address for a node on the cluster.

```
$ minikube ip -n minikube-m02
192.168.39.86
```

### Next we attempt to obtain the share.

For a mounted PV, we can use the information from the description for the PV.

```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
pvc-717cfefa-a119-4271-b71f-e5659284b9d0   1Gi        RWO            Delete           Bound    default/nfs-pvc   rook-nfs                31h
$ kubectl describe pv pvc-717cfefa-a119-4271-b71f-e5659284b9d0
Name:            pvc-717cfefa-a119-4271-b71f-e5659284b9d0
Labels:          <none>
Annotations:     pv.kubernetes.io/provisioned-by: rook-ceph.nfs.csi.ceph.com
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    rook-nfs
Status:          Bound
Claim:           default/nfs-pvc
Reclaim Policy:  Delete
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        1Gi
Node Affinity:   <none>
Message:         
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            rook-ceph.nfs.csi.ceph.com
    FSType:            
    VolumeHandle:      0001-0009-rook-ceph-0000000000000001-8318502f-2847-11ed-a12d-62dcd71a5499
    ReadOnly:          false
    VolumeAttributes:      clusterID=rook-ceph
                           fsName=myfs
                           nfsCluster=my-nfs
                           pool=myfs-replicated
                           server=rook-ceph-nfs-my-nfs-a
                           share=/0001-0009-rook-ceph-0000000000000001-8318502f-2847-11ed-a12d-62dcd71a5499
                           storage.kubernetes.io/csiProvisionerIdentity=1661852183359-8081-rook-ceph.nfs.csi.ceph.com
                           subvolumeName=csi-vol-8318502f-2847-11ed-a12d-62dcd71a5499
                           subvolumePath=/volumes/csi/csi-vol-8318502f-2847-11ed-a12d-62dcd71a5499/2f263dad-c5b7-470c-9cb2-be980acc023e
Events:                <none>
```
The VolumeAttributes can be used to identify the share. In this case, it is share=/0001-0009-rook-ceph-0000000000000001-8318502f-2847-11ed-a12d-62dcd71a5499

So we have the following
- node ip address: 192.168.39.86
- node port: 31573
- share: /0001-0009-rook-ceph-0000000000000001-8318502f-2847-11ed-a12d-62dcd71a5499

From your host machine, attempt to mount the nfs share.
```
# mount -t nfs4 -o port=31573 192.168.39.86:/0001-0009-rook-ceph-0000000000000001-8318502f-2847-11ed-a12d-62dcd71a5499/ /mnt
```
