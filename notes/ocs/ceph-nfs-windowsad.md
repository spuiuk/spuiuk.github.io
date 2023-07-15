Setting up Ceph NFS-Ganesha and using a Windows AD for user id lookups.



To use a Windows AD(or FreeIPA), we need to modify the sssd configuration file to point to the Windows AD server as well as provide the credentials to connect to the LDAP server. This is done by setting a config-map with the sssd.conf file.
If your LDAP/AD server doesn't allow anonymous binds to connect to and search the directory, we will have to setup authentication. This can either be a) password based authentication or 2) Kerberos authentication.
When using a password, it is important to setup TLS for which you will require the CA certificate file to connect to the AD server.

The configMap setup. In this case, we use password based authentication and provide a CA certificate along with it. If using GSSAPI, comment out the block with the password information and uncomment the block for GSSAPI based authentication.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nfs-sssd-config
  labels:
    app: ldap-nfs
  namespace: rook-ceph
data:
  sssd.conf: |
    [sssd]
    # Only the nss service is required for the SSSD sidecar.
    services = nss
    domains = domain1.sink.test
    config_file_version = 2
    
    [domain/domain1.sink.test]
    id_provider = ldap
    ldap_uri = ldap://dc1.domain1.sink.test/
    ldap_search_base = CN=Users,DC=domain1,DC=sink,DC=test
    cache_credentials = True
    ldap_schema = AD
    ldap_id_mapping = True
    
    # recommended options for speeding up LDAP lookups:
    enumerate = false
    ignore_group_members = true

    ## Using GSSAPI to connect to the LDAP Server
    # ldap_sasl_mech = GSSAPI
    ## Use a Krb5 principal which can lookup ldap searches
    # ldap_sasl_authid = bwayne@DOMAIN1.SINK.TEST

    ## Using password to connect to the LDAP server
    ## Use TLS when using password to authenticate
    ldap_default_bind_dn = CN=Administrator,CN=Users,DC=domain1,DC=sink,DC=test
    ldap_default_authtok = Passw0rd
    ldap_tls_cacert = /etc/sssd/rook-additional/ca-certs/ad-ca.pem
    ldap_tls_reqcert = hard
    ldap_id_use_start_tls = True

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: windowsad-ca-cert
  labels:
    app: ldap-nfs
  namespace: rook-ceph
data:
  ad-ca.pem: |
    -----BEGIN CERTIFICATE-----
    MIIFsDCCA5igAwIBAgIE3B/uYzANBgkqhkiG9w0BAQsFADB4MR0wGwYDVQQKExRT
    YW1iYSBBZG1pbmlzdHJhdGlvbjE3MDUGA1UECxMuU2FtYmEgLSB0ZW1wb3Jhcnkg
    YXV0b2dlbmVyYXRlZCBDQSBjZXJ0aWZpY2F0ZTEeMBwGA1UEAxMVREMxLmRvbWFp
    bjEuc2luay50ZXN0MB4XDTIzMDIxNjEyMjE0OFoXDTI1MDExNjEyMjE0OFoweDEd
    MBsGA1UEChMUU2FtYmEgQWRtaW5pc3RyYXRpb24xNzA1BgNVBAsTLlNhbWJhIC0g
    dGVtcG9yYXJ5IGF1dG9nZW5lcmF0ZWQgQ0EgY2VydGlmaWNhdGUxHjAcBgNVBAMT
    FURDMS5kb21haW4xLnNpbmsudGVzdDCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCC
    AgoCggIBALC+AHB8oTx//ewY5FDb3h1PXwqWRJjHG33ckHZm/vYoxUVPmtN7MBPj
    Z9ABomx5jWKU20Or46Hc+PpGP+kbPe8ax7ZbhcVkWaC6MdjEGxZtn3Jhg0Ws4/qJ
    rKPRByJH+Lm1nzLgTxo6k3zj1rPb7TQePcvN2qUB2Qe8QAdmb8ZzWzaXQ0/wNPMc
    K3WLX84BoyVCqKvcCDkrDYyBEhVWgs2SrjPXr6LkD9HFktNyyAH3m8VAEkwg0wwd
    U8+f64bZMz1qiqTJqAXvrJ4vBzepbezcCJBY9a92eNanAxKP+OYgq2xHH8UaBHY0
    H9bSUDRSvUWoPVO9g4MBu7Y/0ECMB01lIvEDVDeOayUvsl3ehzOxGlZAB6kO87EF
    OalZmdTd/X0I3WDTU0z2HYZAmcm6dtFYx1LxwMR79r6XibokwOBdiDiWjG5yO7xO
    EHEDFnGBGDteWu+4botorATVs+Dq6k///UC1sc73n/mlx0hrJGiLx3MI8dmbRL2X
    4gYInEMsFDd4+zUOg/iZfBUQTb6b64jrWeqRLQ/DsHV5aNGA7k9dIkHbFjetpkpj
    SRp4nb3Q0NH62rOJrhL31JlGi7EU2TBTQGsso4KuGxeK9WULDhRz5frQ01o2LKjp
    ou3THXaMKWS5+zXyR3ZDZzdo3K1+F7n25nHCE2fPD6bqaw02zkMxAgMBAAGjQjBA
    MA8GA1UdEwEB/wQFMAMBAf8wDgYDVR0PAQH/BAQDAgEGMB0GA1UdDgQWBBQ+RG2j
    K6SGqXf2tgdCsAL1CR3uCTANBgkqhkiG9w0BAQsFAAOCAgEAPHeeDHunkwcV7mno
    zLdDMOyR8kSPutK9HLogiJa5SVO/4pdKs29JJ0v9E3hRbMPwE0q+Tdj3NFgSQyo2
    TFKEl48uv2ggLF7fbLk+Zdnz4QVWSxc5AJmgEMdAzcWivohy6LEoux7Oh6VZg2Io
    7xKloAUpEmaRUHp3zKZAg9ly6t6JW61fyh3hbfHNQr4KkeVMM380QQNIqzOEq6vs
    ph1rD8L5FrnYdfmfGMxJF+69nIs3zykRHOstKOMAbq0uuehZDHC9Y6JDH49AohtY
    NS9V5tKlaOmPu22TDLLad1KtkDOyPPSSWUaArvHM5rshDubcy+nwxi3LQVmFF+MD
    fevmptwnQOXThBSrapO1dyF730S8iS7swHIojRv4EZmCjYoZhSD+1G4ZveetNqmA
    ZTlaWeJStKOc4kZ0vDkBp7ne6vYQ3zdtb6zyI4uUYRxn83CFg3rCBWNtNn6Kely2
    w9nJpYwv6GUvzh7q0ADbEfwLVKPASSdP9296tfAqAaJX8aDKG7n0Cs6hAfjfHDV4
    k84Bejr2A/GpH/3hqT7TjI/QqF9AhKZ175LcLyTteYj7dUI6orfF14MlmaLgiPny
    wsFRzJK9/x4b0/SykBgL5mB6qYNUkav88saUl/3EiffCi6UaIH1Zq36I1LPSlKwj
    /X5mCIYaQ9E8akCumdi0+fRzLC4=
    -----END CERTIFICATE-----        
```
We first create the configMap nfs-sssd-config providing the sssd.conf file. The sssd.conf file is the sssd configuration which will be used by the CephNFS instance. The configuration includes the following directives
- ldap_uri: Points to the Windows AD server.
- ldap_search_base: The search base to use for user lookups.
- ldap_sasl_mech: The SASL mechanism for authenticating with the LDAP server. This is used with Kerberos auth and is set to GSSAPI.
- ldap_sasl_authid: The kerberos principal used to authenticate with the LDAP server. This has to be provided in the keytab.
- ldap_tls_cacert: The CA certificate to use. This is the ad-ca.pen file which we upload in this config map.
- ldap_default_bind_dn: In this case, we use the Administrator account. This should be replaced by a user allowed to look up userids.
- ldap_default_authok: The password to use for the bind_dn.
- ldap_id_mapping: We set this to True to enable. The sssd server will generate the UIDs in this case. This may or may not be required for your setup.

More information about these settings are available at [the sssd website](https://sssd.io/docs/ad/ad-ldap-provider.html)

The second configMap windowsad-ca-cert contains the ca cert file ad-ca.pem which is copied over from the Windows AD server. This is only required with password based authentication.

We can now start out CephNFS setup.

```
#https://rook.io/docs/rook/v1.10/CRDs/ceph-nfs-crd/
apiVersion: ceph.rook.io/v1
kind: CephNFS
metadata:
  name: ldap-nfs
  namespace: rook-ceph
spec:
  # Settings for the NFS server
  server:
    active: 1
    placement:
    annotations:
      type: ldap-lookup-nfs
    labels:
      app: ldap-nfs
    resources:
    logLevel: NIV_INFO

  security:
    sssd:
      sidecar:
        image: registry.access.redhat.com/rhel7/sssd:latest

        sssdConfigFile:
          volumeSource:
            configMap:
              name: nfs-sssd-config
              defaultMode: 0600 # mode must be 0600
        additionalFiles:
          - subPath: ca-certs
            volumeSource:
              configMap:
                name: windowsad-ca-cert
                defaultMode: 0600 # mode must be 0600
        debugLevel: 0
        resources: {}
```

Here we setup the ldap-nfs CephNFS resource in the rook-ceph namespace.
The sssdConfigFile specifies that the configMap ldap-nfs-sssd-config is to be used. The file sssd.conf in the configMap will be used for the sssd configuration.
We then use the configMap windows-ca-cert configMap mounted under /etc/sssd/rook-additional/ca-certs/ as additionalFiles. This location will contain the CA cert file ad-ca.pem which is referenced in the sssd.conf and used to connect to the WindowsAD server.


