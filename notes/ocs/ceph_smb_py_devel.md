This note documents the steps I took to create my first patch for the ceph-smb project.
The Pull Request is at
https://github.com/ceph/ceph/pull/59844

Before we start, we go through the usual steps to create a new branch on which you work
```
git checkout -b mgr_smb_add_public_addrs_cli
```
The code I am modifying is at src/pybind/mgr/smb/ and are python code.

To avoid tox errors caused by missing rook modules, you will need to run the following command
```
(cd src/pybind/mgr/rook/ && ./generate_rook_ceph_client.sh )
```

With the latest upstream version of ceph, you will need Python version 3.12. Please make sure that this is set correctly.

Once the changes are completed, I ran the tox tests to confirm that the python sources are correctly formatted.
From the repo root, run
```
cd src/pybind/mgr && tox -e mypy,flake8,check-black -x testenv.basepython=python3.12
```

After making the necessary modifications, we need to test out the changes.
I run my tests using the test cluster set up with the repo
https://github.com/spuiuk/cephfs_smb/
This uses cephadm which in turn uses ceph container images.
Therefore to test my changes, I need to create a ceph container image.

To do this, from the ceph repo root, run the command
```
# remove the existing ceph_test image.
podman rmi ceph_test
# generate a new ceph_test image
cd build/
../src/script/cpatch.py --target ceph_test --pull
# Now push the image to my container repositories at quay.io
podman push ceph_test:latest quay.io/spuiuk/ceph_test
```
The new test image is saved at quay.io/spuiuk/ceph_test - modify this to use your own container repository target. You can now use this ceph container to test by setting the image for ceph-base under IMAGES in extra_vars.yml file before calling cephadm bootstrap. Please read the [Readme file](https://github.com/spuiuk/cephfs_smb/blob/main/README.md) for more details.

With the cephfs_smb project, create a file devel.mk with the following content
```
CUSTOM_VERSION:
  ..
  images:
    ceph-base: quay.io/spuiuk/ceph_test
    ..
```
This image is passed to cephadm for use and testing.

Create the cluster as described in the cephfs_smb [README file](https://github.com/spuiuk/cephfs_smb/blob/main/README.md) and test your changes.

For example, for my test patch, I ran the following from within the test cluster.
```
To test, I create a smb cluster and share with the following commands

ceph smb cluster create mysmb2 user --define_user_pass test1%x --clustering default --public_addrs "192.168.145.31/24"
ceph smb share create mysmb2 mysmb2-root mycephfs "/" --subvolume=smbshares

I finally tested the samba share using smbclient

smbclient -U test1%x //mycephfs11/mysmb2-root/

I was also able to exec into the container and confirm that the public ip address was setup as expected.
```
