# Testing patches on Rook

- Create a new repository on quay.io - For example, https://quay.io/repository/spuiuk/rook is what I use to host my rook test repo
- Create patches on the rook repo. Once this is done, use the following commmand to build the image
```
make build IMAGE=ceph
```
- Once this is done, you will end up with a image similar to localhost/build-f808c8a7/ceph-amd64. This is the image we need to push to our repo.
```
IMG=$(podman images |grep ^localhost/build-|awk '{print $3}')
podman login -u <username> -p <password> quay.io
podman push ${IMG} quay.io/spuiuk/rook:test
```
Note that I tag the image as test.

To use this image, we need to modify the operator.yaml file
Copy over deploy/examples/operator.yaml to deploy/examples/operator-devel.yaml and make the following modifications.
```
$ diff deploy/examples/operator.yaml deploy/examples/operator-devel.yaml
26c26
<   ROOK_LOG_LEVEL: "INFO"
---
>   ROOK_LOG_LEVEL: "DEBUG"
563c563,564
<           image: rook/ceph:master
---
>           image: quay.io/spuiuk/rook:test
>           imagePullPolicy: Always

```

Use this yaml instead of the operator.yaml when setting up the rook operator in your k8s cluster.
