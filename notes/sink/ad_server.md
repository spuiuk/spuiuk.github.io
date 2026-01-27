# Test Samba AD Server

This document describes how we could setup a Samba based test AD server. This containerised AD server was first created for use as a test AD server by the "Samba In Kubernetes" project.

## Creating an AD server

### Test Samba AD Setup on an existing Ceph cluster

The aim of this setup is to create an AD server on one of the nodes of the Ceph server. This avoids the need to have a separate machine running an AD server.

Select a node on the Ceph cluster - eg: mycephfs11

Create a podman network on this node
```
podman network create --subnet 10.90.0.0/24 --gateway 10.90.0.1 --driver bridge --ignore --disable-dns podman_utils
```
This create a network "podman_utils" which we can use for creating an AD server on. This uses the subnet 10.90.0.0/24.

You can inspect the newly created podman network with the command
```
podman network inspect podman_utils
```

To run the Samba AD server
```
podman run --detach --rm --name samba-ad-server --cap-add=SYS_ADMIN --network=podman_utils quay.io/samba.org/samba-ad-server:latest
```
obtain the IP address of the samba AD server using the command
```
podman container inspect -f "{{.NetworkSettings.Networks.podman_utils.IPAddress}}" samba-ad-server
```
Remember this ip address for use in the latter steps.


On mycephfs11 podman creates a new interface podman1 for the new podman network. Add this interface to the public zone. This zone has --add-forward set allowing packets to be forwarded between the interfaces.
```
sudo firewall-cmd --permanent --zone=public --add-interface=podman1 && sudo firewall-cmd --reload
```

On the other nodes of the ceph cluster, add a route to the samba-ad-server with the command. Use the ip address as obtained from the earlier command. Also replace mycephfs11 with your own cluster node hostname.
```
sudo route add -host 10.90.0.2 gw mycephfs11

```

### For a Kubernetes Setup
Install the samba-ad server from the samba-operator repo https://github.com/samba-in-kubernetes/samba-operator
```
cd $SMBOPERATORDIR
./tests/test-deploy-ad-server.sh
```

This creates an ad server pod which provides DNS, LDAP and Kerberos services for our use. The DNS server offers dns lookups for the domain domain1.sink.test and the kerberos server for the realm DOMAIN1.SINK.TEST

The install script also modifies the k8s cluster coredns configuration to forward lookups to the domain domain1.sink.test to the newly created ad server. This means that we have a ready made LDAP and Kerberos server we can use for our internal testing.

## To setup the host as a client for the LDAP access on the AD.

1) From the running ad server container, read the file /var/lib/samba/private/tls/ca.pem. The is the ca cert which is required for tls connections.
```
$ kc get pods|grep ^samba-ad-server
samba-ad-server-569f9c54c4-lzhx4   1/1     Running   0
$ kc exec -it samba-ad-server-569f9c54c4-lzhx4 -- cat /var/lib/samba/private/tls/ca.pem
-----BEGIN CERTIFICATE-----
MIIFsDCCA5igAwIBAgIEeqB8YzANBgkqhkiG9w0BAQsFADB4MR0wGwYDVQQKExRT
..
V/w9SbR/kq9iXYObMckDa+yEby4=
-----END CERTIFICATE-----
```

2) Within the client container create file /etc/openldap/certs/ad-ca.pem with the contents from from the cert above.

3) Edit /etc/openldap/ldap.conf and add the line
```
TLS_CACERT /etc/openldap/certs/ad-ca.pem
```
This sets the right certificate to use for ldap search commands.

4) Quick ldap search
```
# ldapsearch -H ldap://dc1.domain1.sink.test -D "CN=Administrator,CN=Users,DC=domain1,DC=sink,DC=test" -w "Passw0rd" -x -Z -b "DC=domain1,DC=sink,DC=test"
```
where
-H ldap://dc1.domain1.sink.test : The ldap server URI
-D "CN=Administrator,CN=Users,DC=domain1,DC=sink,DC=test" : Bind DN
-w "Passw0rd" : password
-x : Simple auth
-Z : Use TLS
-b "DC=domain1,DC=sink,DC=test" : base dn for search

## To setup the client for kerberos authentication against the AD

1) Create file /etc/krb5.conf with the following contents
```
[logging]
default = FILE:/var/log/krb5libs.log
kdc = FILE:/var/log/krb5kdc.log
admin_server = FILE:/var/log/kadmind.log

[libdefaults]
default_realm = DOMAIN1.SINK.TEST
dns_lookup_realm = false
dns_lookup_kdc = false

[realms]

DOMAIN1.SINK.TEST = {
    kdc = dc1.domain1.sink.test
    admin_server = dc1.domain1.sink.test
    default_domain = default.svc.cluster.local
}

[domain_realm]
 default.svc.cluster.local = DOMAIN1.SINK.TEST
 .default.svc.cluster.local = DOMAIN1.SINK.TEST
```

2) Use kinit with test username 'bwayne' and password '1115Rose.' and obtain a ticket
```
 # kinit bwayne
Password for bwayne@DOMAIN1.SINK.TEST:
```

The kerberos server has to the following users/passwords created for our use
```
Administrator/Passw0rd
bwayne/1115Rose.
ckent/1115Rose.
bbanner/1115Rose.
pparker/1115Rose.
user0/1115Rose.
user1/1115Rose.
user2/1115Rose.
user3/1115Rose.
user4/1115Rose.
user5/1115Rose.
user6/1115Rose.
user7/1115Rose.
user8/1115Rose.
user9/1115Rose.
```

To manage the users/passwords on the test ad server, follow instructions at
https://wiki.samba.org/index.php/User_and_Group_management


## Creating a Ceph-SMB service with AD authentication

Ensure you can ping the AD server using ping
```
ping 10.90.0.2
```
Next, create the SMB cluster with the command
```
ceph smb cluster create mysmb --auth_mode=active-directory --domain-realm=DOMAIN1.SINK.TEST --domain-join-user-pass=Administrator%Passw0rd --custom-dns=10.90.0.2
```
We use the AD server as the custom DNS server for the SMB service.
- --auth_mode=active_directory - Uses active directory for authentication.
- --domain-realm=DOMAIN1.SINK.TEST - the realm used for the active directory.
- --domain-join-user-pass=Administrator%Passw0rd - The Admin username/password to join the Domain Controller.
- --custom-dns=10.90.0.2 - We use the IP address of the AD server here. The Samba-AD server will also act as the DNS server.

Next create the share - this triggers the creation of the containers which serve SMB shares.

```
ceph smb share create mysmb mysmb-root mycephfs --subvolume=smbshares --path=/
```

Give it a few minutes to come up. You can confirm the creation of the service with the command
```
# ceph orch ps
NAME                             HOST        PORTS             STATUS         REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID
..
smb.mysmb.0.0.mycephfs12.keovjq  mycephfs12  *:445,9922,4379   running (99s)    97s ago  99s        -        -  4.23.4   def55552456a  ae2c45020fd9

```
As can be seen, the SMB service is run on node mycephfs12.

Use smbclient(provided by package samba-client) to test this Ceph-SMB service.
```
# smbclient -U bwayne@DOMAIN1.SINK.TEST%1115Rose. //mycephfs12/mysmb-root
Try "help" to get a list of possible commands
smb: \> ls
  .                                   D        0  Sun Jan 25 16:14:12 2026
  ..                                  D        0  Sun Jan 25 16:14:12 2026

        5922816 blocks of size 1024. 5922816 blocks available
smb: \> put /etc/passwd passwd
putting file /etc/passwd as \passwd (83.5 kB/s) (average 83.5 kB/s)
smb: \>
```
Here we test the smb share by pushing /etc/passwd from the local machine to the SMB share.
For other username/password which can be used, look at the previous section of this document.

The samba configuration file set for this service is as follows
```
[global]
    load printers = No
    printing = bsd
    printcap name = /dev/null
    disable spoolss = Yes
    smbd profiling level = on
    workgroup = DOMAIN1
    idmap config * : backend = autorid
    idmap config * : range = 2000-9999999
    netbios name = MYSMB
    security = ads
    realm = domain1.sink.test

[mysmb-root]
    path = /volumes/_nogroup/smbshares/c4474e54-b571-481e-a4bd-5521f0a5876f/
    vfs objects = acl_xattr ceph_snapshots ceph_new
    acl_xattr:security_acl_name = user.NTACL
    ceph_new:config_file = /etc/ceph/ceph.conf
    ceph_new:filesystem = mycephfs
    ceph_new:user_id = smb.fs.cluster.mysmb
    read only = No
    browseable = Yes
    kernel share modes = no
    x:ceph:id = mysmb.mysmb-root
    smbd profiling share = yes
    ceph_new:proxy = yes
```
