## Running tcpdump on kubernetes pods

I have been investigating an issue with a smbtorture test failure with the samba-operator in kubernetes. To further debug the problem, I wanted to look into the tcpdump. This however was tricky on my setup with minikube. I thought about sshing into a minikube node and run tcpdump on the node itself. However this wasn't very convinient. The minikube node is not designed to be used by a user and doesn't have many tools available nor did it have a package manager to install any additional packages such as tcpdump.

On investigating, I came across [ephemeral containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/).
These are intended to be used to troubleshoot on pods which have restricted packages available on them. Ephemeral containers are not meant for building applications and aren't meant to be automatically started.

Ensure that both server and client are of version 1.23 or later. Use command _kubectl version_ to confirm.

The Ephemeral containers is a beta feature in version 1.23 and to use it in minikube, we need to start minikube with the parameter [--feature-gates=EphemeralContainers=true](https://minikube.sigs.k8s.io/docs/handbook/config/)

My minikube startup command for the samba-operator environment is set to
```
minikube start --nodes=3 --extra-disks=2 --memory 4096 --cpus 2 --feature-gates=EphemeralContainers=true
```

The current pods running on my cluster
```
$ kubectl get pods -o wide
NAME                         READY   STATUS    RESTARTS   AGE    IP           NODE           NOMINATED NODE   READINESS GATES
smbclient                    1/1     Running   0          14h    10.244.2.3   minikube-m03   <none>           <none>
smbshare2-555956d4fd-4qzpj   1/1     Running   0          146m   10.244.1.5   minikube-m02   <none>           <none>
```

To connect to the samba container on smbshare2-555956d4fd-4qzpj
```
$ kubectl debug -it smbshare2-555956d4fd-4qzpj --image=nicolaka/netshoot --target=samba -- /bin/bash
```

or on smbclient
```
$ kubectl debug -it smbclient --image=nicolaka/netshoot -- /bin/bash
```

This opens up a bash shell on the ephemeral container which is connected to the pod you would like to debug.
The [netshoot image](https://github.com/nicolaka/netshoot) is a container image with several networking troubleshooting tools available within it.

Any files copied thus created can be copied over using _kubectl cp_ command.

For tcpdump, an easier option to capture data is to use the command in the following manner.
```
$ kubectl debug -iq smbclient --image=nicolaka/netshoot -- tcpdump -s0 host 10.244.2.3 and host 10.244.1.5 -w - > out.pcap
```

Here the -q parameter is required to disable any informational messages. This is needed because these messages is included in out.pcap which corrupts the file rendering it unusable.

The out.pcap file created on your local machine can then be processed as required.

Note that the ephemeral container will continue to be listed in the pod description until the pod is rescheduled or deleted.


Notes:

The version of kubectl provided with F34 contains an old version of kubectl which doesn't support epehemeral containers.
```
$ rpm -q kubernetes-client
kubernetes-client-1.20.5-1.fc34.x86_64
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.5", GitCommit:"6b1d87acf3c8253c123756b9e61dac642678305f", GitTreeState:"archive", BuildDate:"2021-03-30T00:00:00Z", GoVersion:"go1.16", Compiler:"gc", Platform:"linux/amd64"}
```

This causes the following issue when using the old client
```
$ kubectl debug -it smbshare3-1 --image=busybox --target=samba
Defaulting debug container name to debugger-64p2g.
error: error updating ephemeral containers: EphemeralContainers in version "v1" cannot be handled as a Pod: no kind "EphemeralContainers" is registered for version "v1" in scheme "k8s.io/kubernetes/pkg/api/legacyscheme/scheme.go:30"
```

More information on using ephemeral containers for tcpdump.
https://downey.io/blog/kubernetes-ephemeral-debug-container-tcpdump/
