The tests were run on an AD server as per instructions at
https://spuiuk.github.io/notes/sink/ad_server.html
The tests were run on a generic fedora container to ensure that the lookups work.

There are three methods available to connect from sssd to the active directory.
https://sssd.io/docs/ad/ad-introduction.html

These are -

1) use realmd - This needs dbus. So I did not proceed with this option

2) manually joining ad domain using sssd-ad.
https://sssd.io/docs/ad/ad-provider-manual.html

Steps:
- dnf install -y krb5-workstation samba-client samba-common-tools cyrus-sasl-gssapi openldap-clients
- create krb5.conf
- test with kinit -V bwayne
- add smb.conf
- net ads join -U Administrator - use Administrator password
- Grab client ticket - kinit -k CLIENT-AD\$
- then run a ldapsearch - ldapsearch -H ldap://dc1.domain1.sink.test -Y GSSAPI -N -b "dc=domain1,dc=sink,dc=test" "(&(objectClass=user)(sAMAccountName=aduser))"
- This confirms that both kinit and ldapsearch work properly
Create sssd.conf - ensure permission set to 0600
manually modify /etc/nsswitch.conf

This step involves joing the Active Directory domain. This may not be feasible for CephNFS instances.

3) Manually with LDAP
https://sssd.io/docs/ad/ad-ldap-provider.html

This seems to be the best method for us as this is similar to what we currently use. However This requires a ca certificate to be copied over from the AD server.

Steps:
- Copy over ca cert from the Samba AD server to the client at the location /etc/openldap/certs/ad-ca.pem.
- dnf install -y openldap-clients sssd sssd-ldap
- Test with ldapsearch - ldapsearch -H ldap://dc1.domain1.sink.test -D "CN=Administrator,CN=Users,DC=domain1,DC=sink,DC=test" -w "Passw0rd" -Z -b "DC=domain1,DC=sink,DC=test"
- Add /etc/sssd/sssd.conf

```
[sssd]
# Only the nss service is required for the SSSD sidecar.
services = nss
domains = domain1.sink.test
config_file_version = 2

[domain/domain1.sink.test]
id_provider = ldap

ldap_uri = ldap://dc1.domain1.sink.test/
ldap_search_base = CN=Users,DC=domain1,DC=sink,DC=test
ldap_id_use_start_tls = True
cache_credentials = True
ldap_tls_cacert = /etc/openldap/certs/ad-ca.pem
ldap_tls_reqcert = allow
ldap_default_bind_dn = CN=Administrator,CN=Users,DC=domain1,DC=sink,DC=test
ldap_default_authtok = Passw0rd
ldap_schema = AD
ldap_id_mapping = True

```
- start the sssd daemon
- test using the id command

```
# id bwayne user1 user2
uid=125801107(bwayne) gid=125800513(Domain Users) groups=125800513(Domain Users),125801103(supervisors),125801105(characters),125801104(employees)
uid=125801112(user1) gid=125800513(Domain Users) groups=125800513(Domain Users),125801106(bulk)
uid=125801113(user2) gid=125800513(Domain Users) groups=125800513(Domain Users),125801106(bulk)
```

In this case, we use id mapping to obtain the uids, gids. These are mostly unique and designed to be replicated across multiple clients. More information is available in the id mapping section in the sssd-ldap man page.
