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

## Running individual unit tests

To run unit tests, run the command
```
make check test
```

To run an individual unit test - ex: TestReconcileCephNFS_addSecurityConfigsToPod in the cephNFS package
```
$ pwd
/home/sprabhu/rook/pkg/operator/ceph/nfs

$ go test -run 'TestReconcileCephNFS_addSecurityConfigsToPod'
--- FAIL: TestReconcileCephNFS_addSecurityConfigsToPod (0.00s)
    --- FAIL: TestReconcileCephNFS_addSecurityConfigsToPod/security.sssd.sidecar_with_configMap (0.00s)
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x28 pc=0x190e29f]

goroutine 59 [running]:
testing.tRunner.func1.2({0x1b2d200, 0x2ec4300})
	/usr/local/go/src/testing/testing.go:1526 +0x24e
testing.tRunner.func1()
	/usr/local/go/src/testing/testing.go:1529 +0x39f
panic({0x1b2d200, 0x2ec4300})
	/usr/local/go/src/runtime/panic.go:884 +0x213
github.com/rook/rook/pkg/operator/ceph/nfs.generateSssdSidecarResources(0xc00042d800, 0xc00035a9c0)
	/home/sprabhu/rook/pkg/operator/ceph/nfs/security.go:168 +0xb3f
github.com/rook/rook/pkg/operator/ceph/nfs.addSSSDConfigsToPod(0xc00025faf0, 0xc000261800, 0xc00042d9e8)
	/home/sprabhu/rook/pkg/operator/ceph/nfs/security.go:64 +0x705
github.com/rook/rook/pkg/operator/ceph/nfs.(*ReconcileCephNFS).addSecurityConfigsToPod(0x1bab580?, 0xc00042d800, 0x1db0f3a?)
	/home/sprabhu/rook/pkg/operator/ceph/nfs/security.go:39 +0x116
github.com/rook/rook/pkg/operator/ceph/nfs.TestReconcileCephNFS_addSecurityConfigsToPod.func3(0xc0004bcb60)
	/home/sprabhu/rook/pkg/operator/ceph/nfs/security_test.go:172 +0xb26
testing.tRunner(0xc0004bcb60, 0xc00007c990)
	/usr/local/go/src/testing/testing.go:1576 +0x10b
created by testing.(*T).Run
	/usr/local/go/src/testing/testing.go:1629 +0x3ea
exit status 2
FAIL	github.com/rook/rook/pkg/operator/ceph/nfs	0.024s
```
