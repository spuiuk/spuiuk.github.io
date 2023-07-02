## My build script for ceph-csi

This script is executed from within the ceph-csi repo directory.
```
#!/bin/bash -x

export ENV_CSI_IMAGE_NAME=quay.io/spuiuk/ceph-csi
make image-cephcsi

login_quay # helper script to login to quay.io repository

podman push ${ENV_CSI_IMAGE_NAME}:canary
```

## My testing script using e2e

This script is executed from within the ceph-csi repo directory.
```
make clean

export VM_DRIVER=kvm2
./scripts/minikube.sh up

export ROOK_VERSION="v1.11.9"
./scripts/minikube.sh deploy-rook

./scripts/install-snapshot.sh install

minikube image load quay.io/spuiuk/ceph-csi:test
minikube image tag quay.io/spuiuk/ceph-csi:test quay.io/cephcsi/cephcsi:canary
sleep 5
minikube image ls


make run-e2e E2E_ARGS="--deploy-rbd=false --deploy-nfs=true --test-cephfs=false --test-rbd=false --test-nfs=true --delete-namespace-on-failure=false"
```

## To deploy your test ceph-csi using the examples in the rook directory

- Create a new repository on quay.io for the cephcsi devel image. For example, https://quay.io/spuiuk/ceph-csi:test is where I host my ceph-csi images
- Make the necessary modifications for the ceph-csi repo, commit the patches and build the image with the script above.
- Use the build script above to build the image and push onto your test repository.
- Modify the rook example by copying ROOKDIR/deploy/examples/operator.yaml to ROOKDIR/deploy/examples/operator-devel.yaml and modifying it 
```
110c110,111
<   # ROOK_CSI_CEPH_IMAGE: "quay.io/cephcsi/cephcsi:v3.8.0"
---
>   ROOK_CSI_CEPH_IMAGE: "quay.io/spuiuk/ceph-csi:canary"
563c564,566
```

