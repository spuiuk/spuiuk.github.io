The smbshare is created using the yaml file
smbshare2.yaml
```
apiVersion: samba-operator.samba.org/v1alpha1
kind: SmbShare
metadata:
  name: smbshare2
spec:
  storage:
    pvc:
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
  readOnly: false
```

The client on which the tests are run are defined by smbclient.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: smbclient
  labels:
    app: samba-operator-test-smbclient
spec:
  containers:
    - name: client
      image: quay.io/samba.org/samba-client:latest
      command:
      - sleep
      - infinity
```

on the smbclient container
install the samba-test package. This provides the smbtorture testing command.

```
dnf install -y samba-test
```

List the shares available with
```
[root@smbclient /]# smbclient -L smbshare2 --user DOMAIN1\\sambauser --password=samba
dos charset 'CP850' unavailable - using ASCII

	Sharename       Type      Comment
	---------       ----      -------
	IPC$            IPC       IPC Service (Samba 4.16.6)
	smbshare2       Disk      
SMB1 disabled -- no workgroup available
```

To run a smb2.rw test.
```
smbtorture -U DOMAIN1\\sambauser --password=samba //smbshare2/smbshare2 smb2.rw
```

Additional tests available with smbtorture can be listed by simply running the smbtorture command without any parameters. However be aware that the tests themselves maybe marked as flapping or known fail in the Samba testing environment.

For the gluster environment, we run the following tests
https://github.com/gluster/samba-integration/tree/tests/testcases/smbtorture-test
The corresponding list of known fails and flapping tests are available at
https://github.com/gluster/samba-integration/tree/tests/testcases/smbtorture-test/selftest
