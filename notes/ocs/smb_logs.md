To obtain the logs for the smbd service, first identify the systemd service name for the smbd service.

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
Now look for the main service name. The service names contain the cluster name(mysmb in this case). The main service name does not contain   the terms "sidecar" or "init" in the service name.
In this case the main service name is ceph-81007864-7c60-11f0-8388-525400762293@smb.mysmb.0.0.mycephfs11.dyudcn.service, the last one listed in the output above.

Use the journalctl command to fetch the log files.
```
# journalctl -u ceph-81007864-7c60-11f0-8388-525400762293@smb.mysmb.0.0.mycephfs11.dyudcn.service
...
```
