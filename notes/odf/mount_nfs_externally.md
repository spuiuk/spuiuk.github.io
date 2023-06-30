## Mounting CephNFS - nfs shares

I use the following alias when using kubectl with the rook-ceph namespace in the commands below.
```
$ alias kr=kubectl -n rook-ceph'
```

### Minikube 

If you start up minikube with the argument "--extra-config=apiserver.service-node-port-range=1-65535", you will be able to assign the Nodeports on a much wider range of ports which includes the standard NFS port 2049.

The minikube startup script I use is
```
if ! minikube status|grep Running
then
	/usr/local/bin/minikube start --driver=kvm2 --nodes=3 --extra-disks=2 --memory 4096 --cpus 2 --extra-config=apiserver.service-node-port-range=1-65535
fi
```

For our use case, it is important to run minikube with the kvm2 driver. This allows us easier access to the nodes using the libvirt network interfaces.

### Virtual machine setup

The kvm2 driver used by minikube setups a virtual network called mk-minikube which we can use to access the minikube nodes.

Start up a virtual machine on the same host as the minikube setup to use as a NFS client. I use Fedora 37 on the client machine and virt-manager to interact with the KVM hypervisor on the host machine.
Click on the virtual machine and go to the "Details" view. Click on "Add Hardware" and select the "Network" device. In the Network source drop down, select the 'mk-minikube' isolated network. Select "virtio" for the device model. Click Finish. You will need to stop the virtual machine and start it again for the client to show the new network interface. Once up, the network device should have an ip address in 192.168.39.0/24 network which is used by minikube for its nodes. You should now be able to directly contact the node from the client.

For easier access, you can also add a forward rule to the subnet 192.168.39.0/24 in the host firewall and setup a static route to the host machine in your network router. This will enable you to directly access the client and the node machines using the ip addresses in the minikube network subnet.

### Setup a NodePort to the NFS service.

The following yaml file exposes a CephNFS resource "my-nfs" over NodePort.
```
apiVersion: v1
kind: Service
metadata:
  name: my-nfs-ext
  labels:
    app: my-nfs-service
spec:
  type: NodePort
  ports:
    - port: 2049
      nodePort: 2049
      name: "nfs"
    - port: 2049
      nodePort: 2049
      protocol: UDP
      name: "nfsu"
  selector:
    ceph_nfs: my-nfs
```
Here we use the selector to point this NodePort to the service we intend to use. Here the selector is used to point to the CephNFS resource my-nfs.

Setup the service and confirm that the NodePort service is up and running and listening on port 2049.
```
$ kr create -f nfs-ext.yaml 
service/my-nfs-ext created
$ kr get services
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
my-nfs-ext                NodePort    10.103.117.112   <none>        2049:2049/TCP,2049:2049/UDP   3s
...
```
### Then get the ip address for a node on the cluster.

```
$ minikube ip 
192.168.39.8
```
We can then mount the NFS share over this ip address.

### Mount on the client vm.

From your client vm, attempt to mount the nfs share.
```
# mount -t nfs4 -o port=31573 192.168.39.8:/ /mnt
# cd /mnt
# ls -la
total 1
drwxr-xr-x. 3 nobody nobody 29 Apr 13 15:15 myfs
```
