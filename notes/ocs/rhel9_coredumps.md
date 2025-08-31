RHEL 9 has coredumps disabled by default. To enable it for the SMB service, first obtain the smb service service name in systemd 
```
# systemctl list-units|grep smb
  ceph-81007864-7c60-11f0-8388-525400762293-init@smb.mysmb.0.0.mycephfs11.dyudcn.service                                               loaded active exited    Ceph Init Containers for smb.mysmb.0.0.mycephfs11.dyudcn on 81007864-7c60-11f0-8388-525400762293
  ceph-81007864-7c60-11f0-8388-525400762293-sidecar@smb.mysmb.0.0.mycephfs11.dyudcn:configwatch.service                                loaded active running   Ceph sidecar smb.mysmb.0.0.mycephfs11.dyudcn:configwatch for 81007864-7c60-11f0-8388-525400762293
  ceph-81007864-7c60-11f0-8388-525400762293-sidecar@smb.mysmb.0.0.mycephfs11.dyudcn:ctdbd.service                                      loaded active running   Ceph sidecar smb.mysmb.0.0.mycephfs11.dyudcn:ctdbd for 81007864-7c60-11f0-8388-525400762293
  ceph-81007864-7c60-11f0-8388-525400762293-sidecar@smb.mysmb.0.0.mycephfs11.dyudcn:ctdbNodes.service                                  loaded active running   Ceph sidecar smb.mysmb.0.0.mycephfs11.dyudcn:ctdbNodes for 81007864-7c60-11f0-8388-525400762293
  ceph-81007864-7c60-11f0-8388-525400762293-sidecar@smb.mysmb.0.0.mycephfs11.dyudcn:proxy.service                                      loaded active running   Ceph sidecar smb.mysmb.0.0.mycephfs11.dyudcn:proxy for 81007864-7c60-11f0-8388-525400762293
  ceph-81007864-7c60-11f0-8388-525400762293-sidecar@smb.mysmb.0.0.mycephfs11.dyudcn:smbmetrics.service                                 loaded active running   Ceph sidecar smb.mysmb.0.0.mycephfs11.dyudcn:smbmetrics for 81007864-7c60-11f0-8388-525400762293
  ceph-81007864-7c60-11f0-8388-525400762293@smb.mysmb.0.0.mycephfs11.dyudcn.service                                                    loaded active running   Ceph smb.mysmb.0.0.mycephfs11.dyudcn for 81007864-7c60-11f0-8388-525400762293
```
Now look for the main service name. The service names contain the cluster name(mysmb in this case). The main service name does not contain the terms "sidecar" or "init" in the service name. 
In this case the main service name is ceph-81007864-7c60-11f0-8388-525400762293@smb.mysmb.0.0.mycephfs11.dyudcn.service, the last one listed in the output above.

Now use the systemctl edit command to modify the service file.
```
# systemctl edit ceph-81007864-7c60-11f0-8388-525400762293@smb.mysmb.0.0.mycephfs11.dyudcn.service
```
This will open an editor to allow you to modify the service file.

Add the following lines to the file
```
[Service]
LimitCORE=infinity
```
Save and exit. 

Now restart the smbd service
```
systemctl restart ceph-81007864-7c60-11f0-8388-525400762293@smb.mysmb.0.0.mycephfs11.dyudcn.service
```

To test the creation of the core file, you can kill the running smbd process with SIGSEGV to test creation of the coredump
```
# ps -fax|grep smbd
3520166 pts/1    S+     0:00              \_ grep --color=auto smbd
3516535 ?        Ss     0:00  \_ /run/podman-init -- samba-container run --setup=users --setup=smb_ctdb --wait-for=ctdb smbd
3516537 ?        S      0:00      \_ /usr/sbin/smbd --foreground --debug-stdout --no-process-group
3517809 ?        S      0:00          \_ /usr/sbin/smbd --foreground --debug-stdout --no-process-group
3517810 ?        S      0:00          \_ /usr/sbin/smbd --foreground --debug-stdout --no-process-group
# kill -s SIGSEGV 3517810
# coredumpctl list
TIME                            PID UID GID SIG     COREFILE EXE              SIZE
..
Fri 2025-08-29 16:41:17 UTC 3517810   0   0 SIGABRT present  /usr/sbin/smbd 763.0K

```
