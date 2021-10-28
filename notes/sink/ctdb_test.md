## Break down of [samba-container/examples/kubernetes/samba-ctdb-dm-sset.yml](https://github.com/samba-in-kubernetes/samba-container/blob/77a5104c6dea44869d753de455bf9477d4cb5404/examples/kubernetes/samba-ctdb-dm-sset.yml)

### ConfigMap
A configMap with the name **samba-container-config-swc** is created with a config.json containing information pertaining to the samba configuration. This for the test containers is mounted at mount point /etc/samba-container.
- [Ref](https://matthewpalmer.net/kubernetes-app-developer/articles/ultimate-configmap-guide-kubernetes.html)

### Secret
A secret with the name **ad-join-secret** is created. This contains the username/password of the AD joining secret. This is mounted as volume with name samba-join-data at mount point /etc/join-data
- [Ref](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)

### PersistentVolumeClaim
A persistentVolumeClaim with the name **ctdb-shared-swc** with mode RedWriteMany is created on storage class rook-cephfs with size 1G. This is shared as volume ctdb-shared which is mounted at /var/lib/ctdb/shared. The persistentVolume is created by dynamic provisioning through rook-cephfs storage class.

A persistentVolumeClaim with the name **samba-share-data-swc** with mode ReadWriteMany is created on storageclass rook-cephfs with size 2G. This is shared as volume samba-share-data which is mounted at /share. The persistentVolume is created using dynamic provisioning through cephfs storage class.

- [Ref1](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/)
- [Ref2](https://medium.com/avmconsulting-blog/persistent-volumes-pv-and-claims-pvc-in-kubernetes-bd76923a61f6)

### Service
 A kubernetes headless service of the name **sssamba3-swc** is created. This means a DNS lookup on the service name will result in the ip address of all the pods being returned. This exposes port 445 with the name smb to pods with label app:clustered-samba-swc.

eg: from one of the containers in the pod.
```
root@clustered-samba-swc-0 /]# host  sssamba3-swc
sssamba3-swc.default.svc.cluster.local has address 10.244.1.15
sssamba3-swc.default.svc.cluster.local has address 10.244.1.16
sssamba3-swc.default.svc.cluster.local has address 10.244.1.17
```
- [Ref1](https://www.youtube.com/watch?v=T4Z7visMM4E)
- [Ref2](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/)
- [Ref3](https://stackoverflow.com/questions/52707840/what-is-a-headless-service-what-does-it-do-accomplish-and-what-are-some-legiti)

### StatefulSet
A statefulset called **clustered-samba-swc**. Replica count is set to 3. This results in the following pods created

```
NAME                               READY   STATUS    RESTARTS   AGE
clustered-samba-swc-0              4/4     Running   0          6d
clustered-samba-swc-1              4/4     Running   0          6d
clustered-samba-swc-2              4/4     Running   0          6d
```
- [Ref](https://medium.com/stakater/k8s-deployments-vs-statefulsets-vs-daemonsets-60582f0c62d4)

##### InitContainers:
All initial containers for this stateful set run the container image - **quay.io/samba.org/samba-server:latest**

**init**: Initialize the entire container environment.\
```samba-container --config=/etc/samba-container/config.json --id=demo --skip-if-file=/var/lib/ctdb/shared/nodes init```

**import**: Import configuration parameters from the sambacc config to samba's registry config.\
```samba-container --config=/etc/samba-container/config.json --id=demo --skip-if-file=/var/lib/ctdb/shared/nodes import```

**must-join**: If possible, perform an unattended domain join. Otherwise, exit or block until a join has been perfmed by another process.\
```samba-container --config=/etc/samba-container/config.json --id=demo --skip-if-file=/var/lib/ctdb/shared/nodes must-join --files --join-file=/etc/join-data/join.json```

**ctdb-migrate**: Migrate standard samba databases to CTDB databases.\
```samba-container --config=/etc/samba-container/config.json --id=demo --skip-if-file=/var/lib/ctdb/shared/nodes ctdb-migrate --dest-dir=/var/lib/ctdb/persistent```

**ctdb-set-node**: Set up the current node in the ctdb and sambacc nodes files.\
```samba-container --config=/etc/samba-container/config.json --id=demo ctdb-set-node --hostname=$(HOSTNAME) --take-node-number-from-hostname=after-last-dash```

**ctdb-must-have-node**: Block until the current node is present in the ctdb nodes file.\
```samba-container --config=/etc/samba-container/config.json --id=demo ctdb-must-have-node --hostname=$(HOSTNAME) --take-node-number-from-hostname=after-last-dash```

##### Containers:
All containers run in this stateful set run quay.io/samba.org/samba-server:latest

**ctdb**: Run a specified server process - ctdb.\
```samba-container --config=/etc/samba-container/config.json --id=demo --debug-delay=2 run ctdbd --setup=smb_ctdb --setup=ctdb_config --setup=ctdb_etc --setup=ctdb_nodes```

**ctdb-manage-nodes**: Run a long lived procees to monitor the node state file for new nodes. When a new node is found, if the current node is in the correct state, this node will add it to CTDB.\
```samba-container --config=/etc/samba-container/config.json --id=demo ctdb-manage-nodes --hostname=$(HOSTNAME) --take-node-number-from-hostname=after-last-dash```

**smb**: Run a specified server process - smbd.\
```samba-container --config=/etc/samba-container/config.json --id=demo --debug-delay=12 run smbd --setup=nsswitch --setup=smb_ctdb```

**winbind**: Run a specified server process - winbind.\
```samba-container --config=/etc/samba-container/config.json --id=demo --debug-delay=10 run winbindd --setup=nsswitch --setup=smb_ctdb```
