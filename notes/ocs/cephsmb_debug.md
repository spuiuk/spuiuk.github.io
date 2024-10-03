## SMB share information

I am running the tests on a QE system which has a SMB cluster and share setup for me. To obtain the information of the share, I run the following
```
# ceph smb show
{
  "resources": [
    {
      "resource_type": "ceph.smb.cluster",
      "cluster_id": "smb1",
      "auth_mode": "user",
      "intent": "present",
      "user_group_settings": [
        {
          "source_type": "resource",
          "ref": "smb1amwleddx"
        }
      ],
      "placement": {
        "label": "smb"
      },
      "clustering": "default",
      "public_addrs": []
    },
    {
      "resource_type": "ceph.smb.share",
      "cluster_id": "smb1",
      "share_id": "share1",
      "intent": "present",
      "name": "share1",
      "readonly": false,
      "browseable": true,
      "cephfs": {
        "volume": "cephfs",
        "path": "/",
        "subvolumegroup": "smb",
        "subvolume": "sv1",
        "provider": "samba-vfs"
      }
    },
    {
      "resource_type": "ceph.smb.join.auth",
      "auth_id": "join1-admin",
      "intent": "present",
      "auth": {
        "username": "Administrator",
        "password": "Redhat@123"
      }
    },
    {
      "resource_type": "ceph.smb.usersgroups",
      "users_groups_id": "smb1amwleddx",
      "intent": "present",
      "values": {
        "users": [
          {
            "name": "user1",
            "password": "passwd"
          }
        ],
        "groups": []
      },
      "linked_to_cluster": "smb1"
    },
    {
      "resource_type": "ceph.smb.usersgroups",
      "users_groups_id": "ug1",
      "intent": "present",
      "values": {
        "users": [
          {
            "name": "user1",
            "password": "passwd"
          }
        ],
        "groups": []
      }
    }
  ]
}
```
Here, we can see from 'resource_type = ceph.smb.share' that the clusterid is smb1 which exports share share1.
We can also check 'resource_type = ceph.smb.cluster' for clusterid smb1, the system uses user authentication with details in smb1amwleddx.
So we look up 'resource_type = ceph.smb.usersgroups' with name smb1amwleddx. We see user/password used is 'user1/passwd'

To summarise:
Clusterid - smb1
Share - share1
user - user1/passwd

## client access to the filesystem
```
mount -t cifs -o username=user1,password=passwd //localhost/share1/ /mnt
smbclient -U user1%passwd //localhost/share1
```

## Obtaining debug information

### Exec into the samba container
To look into the container running the samba process
```
# ceph orch ps
NAME                         HOST     PORTS             STATUS          REFRESHED  AGE  MEM USE  MEM LIM  VERSION          IMAGE ID      CONTAINER ID
...
smb.smb1.0.0.argo012.dmvfnv  argo012  *:445,9922        running (8h)       9m ago   8h        -        -  4.21.0           9df7f1b0ddf1  37b562c0612b
smb.smb1.1.0.argo013.pdqanl  argo013  *:445,9922        running (8h)      13s ago   8h        -        -  4.21.0           9df7f1b0ddf1  b1611f182cba
smb.smb1.2.0.argo014.uxvxbv  argo014  *:445,9922        running (8h)      13s ago   8h        -        -  4.21.0           9df7f1b0ddf1  01acd983b50f
```
The last column gives the container id. This share has a clustered setup with 3 nodes.

We login to the first container with
```
# podman exec -it 37b562c0612b /bin/bash
```
run ps -fax|grep smb to confirm that this is the container being hit right now.
```
# ps -fax|grep smb|wc -l
608
```

### Samba logs
to get the logs, from the host machine
```
podman logs 37b562c0612b
```

### Samba configuration
To get the latest samba configuration, using the samba container id from the ceph orch command, you can run
```
# podman exec -it 37b562c0612b net conf list
[global]
  load printers = No
  printing = bsd
  printcap name = /dev/null
  disable spoolss = Yes
  netbios name = smb1

[share1]
  path = /volumes/smb/sv1/5ffc0edb-9b23-4d4b-948f-f4f448cd31f8/
  vfs objects = acl_xattr ceph_new
  acl_xattr:security_acl_name = user.NTACL
  ceph_new:config_file = /etc/ceph/ceph.conf
  ceph_new:filesystem = cephfs
  ceph_new:user_id = smb.fs.cluster.smb1
  read only = No
  browseable = Yes
  kernel share modes = no
  x:ceph:id = smb1.share1
  ceph_new:proxy = yes
```
you can also run net conf list from within the samba container.

### To find the proxy daemon container
```
# podman ps -a|grep smb-smb1|grep proxy
bb61199e434d  cp.stg.icr.io/cp/ibm-ceph/ceph-8-rhel9@sha256:64331e5e55c57b4b93c505180f2c76cf41014aeee36723e37f534e8e31408950                                      22 minutes ago  Up 22 minutes                         ceph-eb6df304-53c9-11ef-983a-0cc47af3ea56-smb-smb1-0-0-argo012-dmvfnv-proxy
```

### get the proxy daemon logs
```
podman logs bb61199e434d
```

### exec into the proxy container
```
podman exec -it bb61199e434d /bin/bash
```


## To run loading tests

### delete any pending test files created before redoing tests - this is needed if previous tests have failed.
```
smbclient -U user1%passwd //localhost/share1 -c "deltree loadtest"
```

### run an individual loading test
I create a file mytox.ini in the sit-test-cases repo with the following contents
```
[tox]
envlist = mytest
isolated_build = True

[testenv]
setenv =
    TEST_INFO_FILE = test-info.yml
    PYTHONPATH = /root/samba_cephfs/sit-test-cases:./lib64/python3.6/site-packages/
deps =
    pytest
    pyyaml
    pytest-randomly
    pytest-timeout
    iso8601
    smbprotocol
allowlist_externals = pytest

[testenv:mytest]
commands = pytest -vrfEsxXpP testcases/loading/test_loading.py
```

So we first create test-info.yml within the sit-test-cases repo with the following content.
```
server: localhost

users:
  user1: passwd

shares:
  share1:
```

We are now ready to run the test. from /root/sit-test-cases run
```
tox -c mytox.ini
```

### Changing test parameters
To change the number of connections, edit testcases/loading/test_loading.py
variables to set
total_processes: int = 50 # 50 processes
per_process_threads: int = 12 # each running 12 threads
This makes a total of 50x12 = 600
test_runtime: int = 300 # run the test for 300 seconds.

For example, to change the tests to run 600 tests for 300 seconds, I make the following change.
```
diff --git a/testcases/loading/test_loading.py b/testcases/loading/test_loading.py
index 0358757..2b75978 100644
--- a/testcases/loading/test_loading.py
+++ b/testcases/loading/test_loading.py
@@ -24,11 +24,11 @@ test_info_file = os.getenv("TEST_INFO_FILE")
 test_info: dict = testhelper.read_yaml(test_info_file)

 # total number of processes
-total_processes: int = 10
+total_processes: int = 50
 # each with this number of threads
-per_process_threads: int = 5
+per_process_threads: int = 12
 # running the connection test for this many seconds
-test_runtime: int = 30
+test_runtime: int = 300
 # size of test files
 test_file_size = 16 * 1024 * 1024  # 4k size
 # number of files each thread creates
```

## Disabling proxy service

We may want to disable proxy service in the course of debugging. The following steps describe how this can be done.

Set provider to samba-vfs/new to disable proxy for the share.
(samba-vfs/proxied in case you want to explicitly request for proxy)

For example - consider the following share resource
```
- resource_type: ceph.smb.share
  cluster_id: smb1
  share_id: share1
  cephfs:
    volume: cephfs
    subvolumegroup: smb
    subvolume: sv1
    path: /
    provider: samba-vfs/new
```
This will result in a share called share1 within cluster smb1 with proxy disabled.

### Old method I used

Below is a far more round about way which shouldn't be used but leaving this here since it could be useful for debugging in the future.

Ensure the smb share is up and running.

```
# ceph orch ps|grep ^smb\.
smb.smb1.argo012.nimmwn    argo012  *:445,9922        running (54s)    45s ago  54s        -        -  4.21.0           9df7f1b0ddf1  cb30a8f7a3ab
smb.smb1.argo013.hblxgk    argo013  *:445,9922        running (51s)    46s ago  51s        -        -  4.21.0           9df7f1b0ddf1  d42ebc96763b
smb.smb1.argo014.zvmqsd    argo014  *:445,9922        running (48s)    46s ago  48s        -        -  4.21.0           9df7f1b0ddf1  fd479bb49af2
```
Here the cluster name is smb1. The container id running the service is in the last column. On my machine, this is cb30a8f7a3ab.

You can first read the configuration file from the smb container
```
# podman exec -it cb30a8f7a3ab net conf list
[global]
  load printers = No
  printing = bsd
  printcap name = /dev/null
  disable spoolss = Yes
  netbios name = smb1

[share1]
  path = /volumes/smb/sv1/5ffc0edb-9b23-4d4b-948f-f4f448cd31f8/
  vfs objects = acl_xattr ceph_new
  acl_xattr:security_acl_name = user.NTACL
  ceph_new:config_file = /etc/ceph/ceph.conf
  ceph_new:filesystem = cephfs
  ceph_new:user_id = smb.fs.cluster.smb1
  read only = No
  browseable = Yes
  kernel share modes = no
  x:ceph:id = smb1.share1
  ceph_new:proxy = yes

```

Download the smb  service definition with the command
```
# rados -p .smb -N smb1 get config.smb /tmp/config.smb
# cat /tmp/config.smb
{"samba-container-config": "v0", "configs": {"smb1": {"instance_name": "smb1", "instance_features": [], "globals": ["default", "smb1"], "shares": ["share1"]}}, "globals": {"default": {"options": {"load printers": "No", "printing": "bsd", "printcap name": "/dev/null", "disable spoolss": "Yes"}}, "smb1": {"options": {}}}, "shares": {"share1": {"options": {"path": "/volumes/smb/sv1/5ffc0edb-9b23-4d4b-948f-f4f448cd31f8/", "vfs objects": "acl_xattr ceph_new", "acl_xattr:security_acl_name": "user.NTACL", "ceph_new:config_file": "/etc/ceph/ceph.conf", "ceph_new:filesystem": "cephfs", "ceph_new:user_id": "smb.fs.cluster.smb1", "read only": "No", "browseable": "Yes", "kernel share modes": "no", "x:ceph:id": "smb1.share1", "ceph_new:proxy": "yes"}}}}
```
modify the json file so that ceph_new:proxy is set to 'no'

```
# cat /tmp/config.smb
{"samba-container-config": "v0", "configs": {"smb1": {"instance_name": "smb1", "instance_features": [], "globals": ["default", "smb1"], "shares": ["share1"]}}, "globals": {"default": {"options": {"load printers": "No", "printing": "bsd", "printcap name": "/dev/null", "disable spoolss": "Yes"}}, "smb1": {"options": {}}}, "shares": {"share1": {"options": {"path": "/volumes/smb/sv1/5ffc0edb-9b23-4d4b-948f-f4f448cd31f8/", "vfs objects": "acl_xattr ceph_new", "acl_xattr:security_acl_name": "user.NTACL", "ceph_new:config_file": "/etc/ceph/ceph.conf", "ceph_new:filesystem": "cephfs", "ceph_new:user_id": "smb.fs.cluster.smb1", "read only": "No", "browseable": "Yes", "kernel share modes": "no", "x:ceph:id": "smb1.share1", "ceph_new:proxy": "no"}}}}

# rados -p .smb -N smb1 put config.smb /tmp/config.smb
```

The config watch container should be able to detect this change and accordingly set the new configuation

```
# wait for a few seconds

# podman exec -it cb30a8f7a3ab net conf list
[global]
  load printers = No
  printing = bsd
  printcap name = /dev/null
  disable spoolss = Yes
  netbios name = smb1

[share1]
  path = /volumes/smb/sv1/5ffc0edb-9b23-4d4b-948f-f4f448cd31f8/
  vfs objects = acl_xattr ceph_new
  acl_xattr:security_acl_name = user.NTACL
  ceph_new:config_file = /etc/ceph/ceph.conf
  ceph_new:filesystem = cephfs
  ceph_new:user_id = smb.fs.cluster.smb1
  read only = No
  browseable = Yes
  kernel share modes = no
  x:ceph:id = smb1.share1
  ceph_new:proxy = no
```
