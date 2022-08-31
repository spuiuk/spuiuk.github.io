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

We can do that by looking into the NFS-Ganesha configuration file for the share.

First identify the pod containing the share. Also identify the tools pod which we will require.
```
$ kr get pods
NAME                                                     READY   STATUS      RESTARTS      AGE
..
rook-ceph-nfs-my-nfs-a-5787f9fdd9-v95fr                  2/2     Running     0             30h
..
rook-ceph-tools-7c8ddb978b-cm9hr                         1/1     Running     0             30h
```

To read the ganesha configuration file, use the nfsganesha pod identified in the previous command:
```
$ kr exec -it rook-ceph-nfs-my-nfs-a-5787f9fdd9-v95fr  -- cat /etc/ganesha/ganesha.conf
Defaulted container "nfs-ganesha" out of: nfs-ganesha, dbus-daemon, generate-minimal-ceph-conf (init)

NFS_CORE_PARAM {
	Enable_NLM = false;
	Enable_RQUOTA = false;
	Protocols = 4;
}

MDCACHE {
	Dir_Chunk = 0;
}

EXPORT_DEFAULTS {
	Attr_Expiration_Time = 0;
}

NFSv4 {
	Delegations = false;
	RecoveryBackend = 'rados_cluster';
	Minor_Versions = 1, 2;
}

RADOS_KV {
	ceph_conf = '/etc/ceph/ceph.conf';
	userid = nfs-ganesha.my-nfs.a;
	nodeid = my-nfs.a;
	pool = ".nfs";
	namespace = "my-nfs";
}

RADOS_URLS {
	ceph_conf = '/etc/ceph/ceph.conf';
	userid = nfs-ganesha.my-nfs.a;
	watch_url = 'rados://.nfs/my-nfs/conf-nfs.my-nfs';
}

%url	rados://.nfs/my-nfs/conf-nfs.my-nfs
```
Note the rados url at the bottom. This identifies the
- pool: .nfs
- Namespace: my-nfs
- configuration: conf-nfs.my-nfs

Now exec into the tools pod which we identified in the previous get pods command.
```
$ kr exec -it rook-ceph-tools-7c8ddb978b-cm9hr -- /bin/bash
```
Use the rados command to get the object containing the configuration using arguments from the url above.
```
$ rados -p .nfs -N my-nfs get conf-nfs.my-nfs  /tmp/conf-nfs.my-nfs
$ cat /tmp/conf-nfs.my-nfs
%url "rados://.nfs/my-nfs/export-1"
```
similarly fetch export-1
```
bash-4.4$ rados -p .nfs -N my-nfs get export-1  /tmp/export-1       
bash-4.4$ cat /tmp/export-1
EXPORT {
    FSAL {
        name = "CEPH";
        user_id = "nfs.my-nfs.1";
        filesystem = "myfs";
        secret_access_key = "AQCl2g1jzhM7LRAALLETml8eoxC3C9p0ltoMaw==";
    }
    export_id = 1;
    path = "/volumes/csi/csi-vol-8318502f-2847-11ed-a12d-62dcd71a5499/2f263dad-c5b7-470c-9cb2-be980acc023e";
    pseudo = "/0001-0009-rook-ceph-0000000000000001-8318502f-2847-11ed-a12d-62dcd71a5499";
    access_type = "RW";
    squash = "none";
    attr_expiration_time = 0;
    security_label = true;
    protocols = 4;
    transports = "TCP";
}
```

The directive pseudo identifies the share: /0001-0009-rook-ceph-0000000000000001-8318502f-2847-11ed-a12d-62dcd71a5499

So we have the following
- node ip address: 192.168.39.86
- node port: 31573
- share: /0001-0009-rook-ceph-0000000000000001-8318502f-2847-11ed-a12d-62dcd71a5499

From your host machine, attempt to mount the nfs share.
```
# mount -t nfs4 -o port=31573 192.168.39.86:/0001-0009-rook-ceph-0000000000000001-8318502f-2847-11ed-a12d-62dcd71a5499/ /mnt
```
