## Setup
```
alias grpc='grpcurl -cacert ~/resources/certs/ca/ca.crt -cert ~/resources/certs/mycephfs11/mycephfs11.crt -key ~/resources/certs/mycephfs11/mycephfs11.key --import-path ~/resources/ -proto control.proto'
```

## SambaControl/Info

Returns general information on the services.

Parameters: No parameters needed.

```
# grpc 192.168.145.11:54445 SambaControl/Info
{
  "sambaInfo": {
    "version": "Version 4.23.6",
    "clustered": true
  },
  "containerInfo": {
    "sambaccVersion": "0.2.dev1+g62c3f422a",
    "containerVersion": "(unknown)"
  }
}
```

## SambaControl/Status

Returns the timestamp returned by the smbstatus command. This indicates that the Samba server is alive.

Parameters: No parameters needed.

```
# grpc 192.168.145.11:54445 SambaControl/Status
{
  "serverTimestamp": "2026-03-27T12:18:12.196725+0000"
}
```

## SambaControl/CloseShare

Closes shares to clients.

Parameters:
  - share_name: (string) Share name
  - denied_users: (bool) deny users, 1 / 0

```
```



## SambaControl/KillClientConnection

Kill client connection.

Parameters:
  - ip_address: (string) IP address of the client.

```
```


## SambaControl/ConfigDump

ConfigDump streams a textual represenation of the configuration

Parameters:
- source: Must be set to valid configuration source.
	- CONFIG_FOR_SAMBA
	- CONFIG_FOR_CTDB
	- CONFIG_FOR_SAMBACC
- hash: Optional. if set, returns hash of the content returned.

```
# grpc -d '{"source": "CONFIG_FOR_SAMBA", "hash": 1}' 192.168.145.11:54445 SambaControl/ConfigDump
{
  "line": {
    "content": "[logging]\n"
  }
}
{
  "line": {
    "lineNumber": "1",
    "content": "log level = NOTICE\n"
  }
}
{
  "line": {
    "lineNumber": "2",
    "content": "\n"
  }
}
{
  "line": {
    "lineNumber": "3",
    "content": "[cluster]\n"
  }
}
{
  "line": {
    "lineNumber": "4",
    "content": "recovery lock = !/usr/bin/samba-container ctdb-rados-mutex rados://.smb/mysmb/cluster.meta.lock\n"
  }
}
{
  "line": {
    "lineNumber": "5",
    "content": "nodes list = !/usr/bin/samba-container ctdb-list-nodes\n"
  }
}
{
  "line": {
    "lineNumber": "6",
    "content": "\n"
  }
}
{
  "line": {
    "lineNumber": "7",
    "content": "[legacy]\n"
  }
}
{
  "line": {
    "lineNumber": "8",
    "content": "realtime scheduling = false\n"
  }
}
{
  "line": {
    "lineNumber": "9",
    "content": "script log level = ERROR\n"
  }
}
{
  "line": {
    "lineNumber": "10",
    "content": "\n"
  }
}
{
  "digest": {
    "hash": "HASH_ALG_SHA256",
    "configDigest": "ace4db00a8804cab1202d9c925a46b92cafab895a7054a37b3a4f2e43fcfdf17"
  }
}

```
**Tip**: For a user friendly output, pass it through the json processor

```
# grpc -d '{"source": "CONFIG_FOR_SAMBA"}' 192.168.145.11:54445 SambaControl/ConfigDump|jq -j '.[].content'
[global]
	load printers = No
	printing = bsd
	printcap name = /dev/null
	disable spoolss = Yes
	smbd profiling level = on
	server smb transports = 445
	netbios name = mysmb

[mysmb-root]
	path = /volumes/_nogroup/smbshares/a71a5222-1b48-47b8-b85b-3d0793a0f4b2/
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
	posix locking = no
	ceph_new:proxy = yes
```

## SambaControl/ConfigSummary

Returns the hash of the configuration. This can be used to check if the configuration has changed.

Parameters:
- source: Must be set to valid configuration source.
	- CONFIG_FOR_SAMBA
	- CONFIG_FOR_CTDB
	- CONFIG_FOR_SAMBACC
- hash: Optional. if set, selects hash algorithm to use.
	- SHA256
```
# grpc -d '{"source": "CONFIG_FOR_SAMBA"}' 192.168.145.11:54445 SambaControl/ConfigSummary
{
  "source": "CONFIG_FOR_SAMBA",
  "digest": {
    "hash": "HASH_ALG_SHA256",
    "config_digest": "e47940b291ca5cdded4e68a38dc9b94538f976a7fc75de151e78192c2654fa54"
  }
}
```

## SambaControl/ConfigSharesList

Streams a list of shares. Can be used to list the shares exported outside the smb protocol.

Parameters:
- source: Must be set to valid configuration source.
	- CONFIG_FOR_SAMBA

```
# grpc -d '{"source": "CONFIG_FOR_SAMBA"}' 192.168.145.11:54445 SambaControl/ConfigSharesList
{
  "name": "mysmb-root"
}

```

## SambaControl/SetDebugLevel

Attempts to apply the given debug level to the daemon.

Parameters:
- process: The process which makes up the SMB service.
  - SMB_PROCESS_SMB
  - SMB_PROCESS_WINBIND
  - SMB_PROCESS_CTDB
- debug_level: This has to be provided as a string.
  - "0"-"10"; as defined for the service

```
# grpc -d '{"process": "SMB_PROCESS_SMB","debug_level": "5"}' 192.168.145.11:54445 SambaControl/SetDebugLevel
{
  "process": "SMB_PROCESS_SMB",
  "debug_level": "5"
}# grpc -d '{"process": "SMB_PROCESS_SMB","debug_level": "5"}' 192.168.145.11:54445 SambaControl/SetDebugLevel
{
  "process": "SMB_PROCESS_SMB",
  "debugLevel": "5"
}
```

## SambaControl/GetDebugLevel

Attempts to get the debug level for the smb process

Parameters:
- process: The process which makes up the SMB service.
  - SMB_PROCESS_SMB
  - SMB_PROCESS_WINBIND
  - SMB_PROCESS_CTDB
```
# grpc -d '{"process": "SMB_PROCESS_SMB"}' 192.168.145.11:54445 SambaControl/GetDebugLevel
{
  "process": "SMB_PROCESS_SMB",
  "debug_level": "5"
}
```

## SambaControl/CTDBStatus

Returns the current CTDB status.

Parameters: No parameters needed.

```
# grpc 192.168.145.11:54445 SambaControl/CTDBStatus
{
  "node_status": {
    "node_count": 3,
    "nodes": [
      {
        "address": "192.168.145.12",
        "flags_raw": 3,
        "flags": {
          "banned": false,
          "deleted": false,
          "disabled": false,
          "disconnected": true,
          "inactive": true,
          "stopped": false,
          "unhealthy": true,
          "unknown": false
        }
      },
      {
        "pnn": 1,
        "address": "192.168.145.13",
        "flags_raw": 3,
        "flags": {
          "banned": false,
          "deleted": false,
          "disabled": false,
          "disconnected": true,
          "inactive": true,
          "stopped": false,
          "unhealthy": true,
          "unknown": false
        }
      },
      {
        "pnn": 2,
        "address": "192.168.121.176",
        "flags_raw": 2,
        "flags": {
          "banned": false,
          "deleted": false,
          "disabled": false,
          "disconnected": false,
          "inactive": false,
          "stopped": false,
          "unhealthy": true,
          "unknown": false
        },
        "this_node": true
      }
    ]
  },
  "vnn_status": {
    "size": 3,
    "vnn_map": [
      {},
      {
        "hash": 1,
        "lmaster": 1
      },
      {
        "hash": 2,
        "lmaster": 2
      }
    ]
  },
  "recovery_mode": "RECOVERY",
  "recovery_mode_raw": "1",
  "leader": "4294967295"
}

```
## SambaControl/CTDBMoveIP

Move the public ip address to a different node.

Parameters:
  - ip: (string) ip address to move
  - node: (string) node to move to.

```
```
