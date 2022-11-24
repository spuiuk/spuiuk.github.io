# Test Samba AD Server

Install the samba-ad server from the samba-operator repo https://github.com/samba-in-kubernetes/samba-operator
```
cd $SMBOPERATORDIR
./tests/test-deploy-ad-server.sh
```

This creates an ad server pod which provides DNS, LDAP and Kerberos services for our use. The DNS server offers dns lookups for the domain domain1.sink.test and the kerberos server for the realm DOMAIN1.SINK.TEST

The install script also modifies the k8s cluster coredns configuration to forward lookups to the domain domain1.sink.test to the newly created ad server. This means that we have a ready made LDAP and Kerberos server we can use for our internal testing.

## To setup LDAP

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

## To setup kerberos authentication on a container

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
