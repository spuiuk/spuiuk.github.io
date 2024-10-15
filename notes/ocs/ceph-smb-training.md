# Ceph SMB Debugging 

## Ceph Status

### Ceph Status
#### get status
```
ceph -s
```
#### list hosts
```
ceph orch host ls
```
#### list daemons running
```
ceph orch ps
```
#### list config options 
``` 
ceph config ls
```
specifically for the mgr service
```
ceph config ls|grep mgr
```
obtain the image which will be used my the smb service.
```
ceph config get mgr mgr/cephadm/container_image_samba
```

### Ceph Orchestrator
#### list enabled modules
```
ceph mgr module ls
```
#### enable smb mgr module
```
ceph mgr module enable smb
```

### Ceph Volume
#### list volumes
```
ceph fs volume ls
```

```
ceph fs subvolume ls <volume_name>
```
#### create volume
```
ceph fs volume create mycephfs
```

```
ceph fs subvolume create mycephfs smb1 --mode 0777
ceph fs subvolume create mycephfs smb2 --mode 0777
```

### Ceph SMB
#### list ceph smb clusters
```
ceph smb cluster ls
```
#### list shares for a smb cluster
```
ceph smb share ls mysmb
```
#### create a smb cluster - imperative method
```
ceph smb cluster create mysmb user --define-user-pass test1%x
```
#### create a share on a smb cluster - imperative method
```
ceph smb share create mysmb share1 mycephfs / --subvolume=smb1
```
#### Create a smb cluster - declarative method
The following file defines a smb cluster mysmb and a share mysmb-root
```
resources:
  - resource_type: ceph.smb.cluster
    cluster_id: mysmb
    auth_mode: user
    user_group_settings:
      - source_type: resource
        ref: ug1

  - resource_type: ceph.smb.usersgroups
    users_groups_id: ug1
    values:
      users:
        - name: test1
          password: x
        - name: test2
          password: x
      groups: []

  - resource_type: ceph.smb.share
    cluster_id: mysmb
    share_id: share2
    intent: present
    name: share2
    readonly: false
    browseable: true
    cephfs:
      volume: mycephfs
      subvolume: smb2
      path: /
      provider: samba-vfs
```

```
ceph smb apply -i /path/to/resources.yaml
```
More information available at 
https://docs.ceph.com/en/latest/mgr/smb/
#### smb cluster - obtain resource information
```
ceph smb show
```
#### Connect to the smb share exported by the ceph
``` 
mount -t cifs -o username=user1,password=passwd //localhost/share1/ /mnt
```
or
```
smbclient -U user1%passwd //localhost/share1
```
#### Find the container exporting the smb share
``` # ceph orch ps
NAME                             HOST        PORTS             STATUS        REFRESHED  AGE  MEM USE  MEM LIM  VERSION                IMAGE ID      CONTAINER ID  
..
smb.mysmb.0.0.mycephfs11.euuvip  mycephfs11  *:445,9922        running (2h)     6m ago   2h        -        -  4.22.0                 ff5d5b5b9bd9  96319e9d7ce6  
```
The last column give you the container id. in this case - "96319e9d7ce6"

You could also list all containers and just grep for the cluster name.
```
# podman ps -a|grep mysmb
96319e9d7ce6  quay.io/samba.org/samba-server:devbuilds-centos-amd64                                              run --setup=users...  3 hours ago     Up 3 hours                 ceph-71a16204-8af1-11ef-91fd-525400082db0-smb-mysmb-0-0-mycephfs11-euuvip
c27cacf4ebe8  quay.io/samba.org/samba-server:devbuilds-centos-amd64                                              update-config --w...  3 hours ago     Up 3 hours                 ceph-71a16204-8af1-11ef-91fd-525400082db0-smb-mysmb-0-0-mycephfs11-euuvip-configwatch
1ba3acea3bd1  quay.io/samba.org/samba-server:devbuilds-centos-amd64                                              run ctdbd --setup...  3 hours ago     Up 3 hours                 ceph-71a16204-8af1-11ef-91fd-525400082db0-smb-mysmb-0-0-mycephfs11-euuvip-ctdbd
6201c9b194e7  quay.io/samba.org/samba-metrics:latest                                                             --port=9922           3 hours ago     Up 3 hours                 ceph-71a16204-8af1-11ef-91fd-525400082db0-smb-mysmb-0-0-mycephfs11-euuvip-smbmetrics
ae6a2523c52c  quay.io/samba.org/samba-server:devbuilds-centos-amd64                                              --debug ctdb-moni...  11 minutes ago  Up 11 minutes              ceph-71a16204-8af1-11ef-91fd-525400082db0-smb-mysmb-0-0-mycephfs11-euuvip-ctdbNodes
```

#### exec into the container
```
podman exec -it 96319e9d7ce6 /bin/bash
```
#### obtain the smb configuration
```
# podman exec -it 96319e9d7ce6 net conf list
[global]
  load printers = No
  printing = bsd
  printcap name = /dev/null
  disable spoolss = Yes
  netbios name = mysmb

[share1]
  path = /volumes/_nogroup/smb1/22e03c9d-65dd-448a-971e-c68c9437fbed/
  vfs objects = acl_xattr ceph_new
  acl_xattr:security_acl_name = user.NTACL
  ceph_new:config_file = /etc/ceph/ceph.conf
  ceph_new:filesystem = mycephfs
  ceph_new:user_id = smb.fs.cluster.mysmb
  read only = No
  browseable = Yes
  kernel share modes = no
  x:ceph:id = mysmb.share1

[share2]
  path = /volumes/_nogroup/smb2/66c2b496-82e4-43d2-bf5d-c0dc5b275711/
  vfs objects = acl_xattr ceph_new
  acl_xattr:security_acl_name = user.NTACL
  ceph_new:config_file = /etc/ceph/ceph.conf
  ceph_new:filesystem = mycephfs
  ceph_new:user_id = smb.fs.cluster.mysmb
  read only = No
  browseable = Yes
  kernel share modes = no
  x:ceph:id = mysmb.share2
```
#### Look at the samba logs
```
podman logs -f 96319e9d7ce6
```

### Container startup

#### Systemd service script
The systemd start up scripts for the smb service can be found in /etc/systemd/system.
```
# ls /etc/systemd/system/|grep mysmb
ceph-71a16204-8af1-11ef-91fd-525400082db0-init@smb.mysmb.0.0.mycephfs11.euuvip.service
ceph-71a16204-8af1-11ef-91fd-525400082db0-sidecar@smb.mysmb.0.0.mycephfs11.euuvip:configwatch.service
ceph-71a16204-8af1-11ef-91fd-525400082db0-sidecar@smb.mysmb.0.0.mycephfs11.euuvip:ctdbd.service
ceph-71a16204-8af1-11ef-91fd-525400082db0-sidecar@smb.mysmb.0.0.mycephfs11.euuvip:ctdbNodes.service
ceph-71a16204-8af1-11ef-91fd-525400082db0-sidecar@smb.mysmb.0.0.mycephfs11.euuvip:smbmetrics.service
ceph-71a16204-8af1-11ef-91fd-525400082db0@smb.mysmb.0.0.mycephfs11.euuvip.service.d
```
Here you can see the init service and the sidecars.

Looking into any of the ctdb service files.
```
# cat ceph-71a16204-8af1-11ef-91fd-525400082db0-sidecar@smb.mysmb.0.0.mycephfs11.euuvip:ctdbd.service|grep ExecStart
ExecStart=/bin/bash /var/lib/ceph/71a16204-8af1-11ef-91fd-525400082db0/smb.mysmb.0.0.mycephfs11.euuvip/sidecar-ctdbd.run start
```
You can see the start up script for the ctdb service.
```
# cd /var/lib/ceph/71a16204-8af1-11ef-91fd-525400082db0/smb.mysmb.0.0.mycephfs11.euuvip/
# cat sidecar-ctdbd.run
```
You can see the various volumes which are mounted in the ctdb service. These provide the configuration files, space for sockets, runtime files.

