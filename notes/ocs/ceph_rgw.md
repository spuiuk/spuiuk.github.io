# Ceph RGW

This document describes how to setup and do basic operations on the Object Gateway(RGW).

## Setup

We assume that you have a Ceph Cluster up and running.

Check health of the Ceph Cluster
```
# ceph health
HEALTH_OK
# ceph -s
  cluster:
    id:     60052ad2-b739-11f1-a197-525400e72318
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum mycephfs11 (age 31m) [leader: mycephfs11]
    mgr: mycephfs11.uqyaxi(active, since 2m), standbys: mycephfs11.wqwxgo
    mds: 1/1 daemons up, 1 standby
    osd: 2 osds: 2 up (since 29m), 2 in (since 30m)

  data:
    volumes: 1/1 healthy
    pools:   4 pools, 177 pgs
    objects: 25 objects, 490 KiB
    usage:   54 MiB used, 12 GiB / 12 GiB avail
    pgs:     177 active+clean

```

Check to see if RGW orchestrator module is enabled.
```
# ceph mgr module ls |grep rgw
rgw                   on
```
If not, enable the module.
```
# ceph mgr module enable rgw
# ceph mgr module ls |grep rgw
rgw                   on
```

## Deploy RGW service

```
# ceph orch apply rgw gw1 --placement="1 mycephfs11" --port=8080
Scheduled rgw.gw1 update...
```
Change the placement parameters to satisfy your cluster requirements.

Check the status of the rgw daemon
```
# ceph orch ps --daemon-type rgw
NAME                       HOST        PORTS   STATUS         REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID
rgw.gw1.mycephfs11.hqepau  mycephfs11  *:8080  running (28s)    25s ago  28s    53.5M        -  20.2.4   6facee348180  c3e5378cfc76
```
In this case, the rgw daemon is running at http://mycephfs11:8080. This is the **endpoint-url** for your cluster.

You can also check the http service using curl
```
# curl -I http://mycephfs11:8080
HTTP/1.1 200 OK
x-amz-request-id: tx00000527b0b424e55a3ec-006ab3b279-14295-default
Content-Type: application/xml
Server: Ceph Object Gateway (tentacle)
Content-Length: 0
Date: Wed, 23 Sep 2026 11:05:29 GMT
Connection: Keep-Alive
```

## Create user

Create your first user
```
# radosgw-admin user create --uid="demo-user" --display-name="Demo User"
{
    "user_id": "demo-user",
    "display_name": "Demo User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "demo-user",
            "access_key": "FKVYSX2HA00E0CLBNO07",
            "secret_key": "zlUqXaAsvXprpv5MHjn6CB6fHKjt1YWfsw6atmjs",
            "active": true,
            "create_date": "2026-09-23T11:07:28.415903Z"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "STANDARD",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": [],
    "account_id": "",
    "path": "/",
    "create_date": "2026-09-23T11:07:28.415479Z",
    "tags": [],
    "group_ids": []
}
```
We are interested in the access_key and the secret_key for the user_id "demo-user" and will use it to access the object store.

You can alternatively specify the access_key and secret_key when creating the user
```
# radosgw-admin user create --uid demo-user2 --display-name "Demo User 2" --access-key demo-user2 --secret mysecret
{
    "user_id": "demo-user2",
    "display_name": "Demo User 2",
..
```

If you would like to list all the users on the object store
```
# radosgw-admin user list
[
    "dashboard",
    "demo-user2",
    "demo-user"
]
```
or alternatively
```
# radosgw-admin metadata list
[
    "account",
    "bucket",
    "bucket.instance",
    "group",
    "oidc",
    "otp",
    "roles",
    "topic",
    "user"
]

# radosgw-admin metadata list user
[
    "dashboard",
    "demo-user2",
    "demo-user"
]
```

To obtain the access_key and secret_key after creation,
```
# radosgw-admin user info --uid=demo-user
{
    "user_id": "demo-user",
    "display_name": "Demo User",
..
```
or alternatively
```
# radosgw-admin metadata get user:demo-user
{
    "key": "user:demo-user",
    "ver": {
        "tag": "_aak2EGMhZX6e0Y_9n-eOioEs",
        "ver": 1
    },
    "mtime": "2026-09-23T11:07:28.415950Z",
    "data": {
        "user_id": "demo-user",
        "display_name": "Demo User",
        "email": "",
..
```

## Client setup

Install aws client
(for RHEL 10)
```
# dnf install awscli2
```

Configure aws client
```
# aws configure --profile demo-user
AWS Access Key ID [None]: FKVYSX2HA00E0CLBNO07
AWS Secret Access Key [None]: zlUqXaAsvXprpv5MHjn6CB6fHKjt1YWfsw6atmjs
Default region name [None]:
Default output format [None]: json
```
or alternatively for user demo-user2
```
# aws configure set aws_access_key_id demo-user2 --profile=demo-user2
# aws configure set aws_access_key_id mysecret --profile=demo-user2
# aws configure set region default --profile=demo-user2
# aws configure set output json --profile=demo-user2
```
The profile information is setup in the ~/.aws/config and ~/.aws/credentials files.

Test client connection
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3 ls
#
```
This should return no errors. It doesn't output anything yet.

## Create objects

Create a bucket
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3 mb s3://my-bucket
make_bucket failed: s3://my-bucket An error occurred (IllegalLocationConstraintException) when calling the CreateBucket operation: The aws-global location constraint is incompatible for the region specific endpoint this request was sent to.
```
This specific error was generated because we did not specify the correct zone.

To first find the zone set,
```
# radosgw-admin zone get
{
    "id": "a56e44ec-937b-46e9-8a45-f61cdb08d0fa",
    "name": "default",
..
```
We use the zone **default** and set this in the profile for the demo-user
```
# aws configure set region default --profile=demo-user
```
We can now create the bucket successfully
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3 mb s3://my-bucket
make_bucket: my-bucket
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3 ls
2026-09-23 11:43:37 my-bucket
```
Alternatively, we can use the s3api which provides more options to create the bucket
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3api create-bucket --bucket my-2nd-bucket
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3api list-buckets
{
    "Buckets": [
        {
            "Name": "my-2nd-bucket",
            "CreationDate": "2026-09-23T11:45:47.127000+00:00"
        },
        {
            "Name": "my-bucket",
            "CreationDate": "2026-09-23T11:43:37.873000+00:00"
        }
    ],
    "Owner": {
        "DisplayName": "Demo User",
        "ID": "demo-user"
    },
    "Prefix": null
}
```

You can also list the buckets created on the cluster using the radowgw-admin command
```
# radosgw-admin buckets list
[
    "my-bucket",
    "my-2nd-bucket"
]
```

Upload a file to this bucket and fetch it again
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3 cp /etc/passwd s3://my-bucket/p1
upload: ../etc/passwd to s3://my-bucket/p1
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3 ls s3://my-bucket/
2026-09-23 11:50:05       1311 p1
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3 cp s3://my-bucket/p1 /tmp/p1
download: s3://my-bucket/p1 to ../tmp/p1
```
Alternatively with the s3api commands
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3api put-object --bucket my-bucket --key p2 --body /etc/passwd
{
    "ETag": "\"9820d84c95aed471841314471323fcc3\"",
    "ChecksumCRC64NVME": "HbCWNJEc0wk=",
    "ChecksumType": "FULL_OBJECT"
}
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3api list-objects --bucket my-bucket
{
    "Contents": [
        {
            "Key": "p1",
            "LastModified": "2026-09-23T11:50:05+00:00",
            "ETag": "\"9820d84c95aed471841314471323fcc3\"",
            "Size": 1311,
            "StorageClass": "STANDARD",
            "Owner": {
                "DisplayName": "Demo User",
                "ID": "demo-user"
            }
        },
        {
            "Key": "p2",
            "LastModified": "2026-09-23T11:51:33+00:00",
            "ETag": "\"9820d84c95aed471841314471323fcc3\"",
            "Size": 1311,
            "StorageClass": "STANDARD",
            "Owner": {
                "DisplayName": "Demo User",
                "ID": "demo-user"
            }
        }
    ],
    "RequestCharged": null,
    "Prefix": ""
}
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3api get-object --bucket my-bucket --key p2 /tmp/p2
{
    "AcceptRanges": "bytes",
    "LastModified": "2026-09-23T11:51:33+00:00",
    "ContentLength": 1311,
    "ETag": "\"9820d84c95aed471841314471323fcc3\"",
    "ChecksumCRC64NVME": "HbCWNJEc0wk=",
    "ChecksumType": "FULL_OBJECT",
    "ContentType": "binary/octet-stream",
    "Metadata": {}
}
```
However another user cannot access these objects from my-bucket
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user2 s3 ls s3://my-bucket

argument of type 'NoneType' is not iterable
```
Note that we user demo-user2 by setting the profile used by the aws client.

demo-user can also setup a ACL to allow demo-user2 to read the bucket my-bucket. This allows the user demo-user2 to list out the bucket but not download any objects.
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user s3api put-bucket-acl --bucket my-bucket --grant-read 'id=demo-user2'
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user2 s3 ls s3://my-bucket
2026-09-23 11:50:05       1311 p1
2026-09-23 11:51:33       1311 p2
# aws --endpoint-url "http://mycephfs11:8080" --profile=demo-user2 s3 cp s3://my-bucket/p1 /tmp/p1-2
fatal error: An error occurred (403) when calling the HeadObject operation: Forbidden
```
For better control, use a bucket policy which we will cover when looking at multitenancy.

## Multitenancy

A tenant is an organisation which shares the cluster resources with other tenants. Resources created by one tenant are limited to users belonging to that tenant. This also allows for the same bucket name to be created separately by each tenant. Ceph-RGW supports multiple tenants on the same cluster. This is known as multitenancy.

Consider a new tenant testx. To create users user1 and user2 within the tenant testx,

```
# radosgw-admin user create --tenant testx --uid=user1 --display-name="TestX - User 1"
{
    "user_id": "testx$user1",
    "display_name": "TestX - User 1",
..
            "user": "testx$user1",
            "access_key": "13URYV2NCYWNXZ3L1FRR",
            "secret_key": "tKY2AxXNiU7kenBZv8YM9RMtElL7r16uzaA2b2PN",
..

# radosgw-admin user create --tenant testx --uid=user2 --display-name="TestX - User 2"
{
    "user_id": "testx$user2",
    "display_name": "TestX - User 2",
..
            "user": "testx$user2",
            "access_key": "CVWDOTQNCMXSX0M69WII",
            "secret_key": "9BskrHQ1oW3p3fNCwRf2MJocKR3ArLFfSHkSsjUG",
..
```
Notice the user_id created is of the form tenant$username. ie. testx$user1 and testx$user2.

Add the configuration profile  to aws
```
# aws configure --profile user1
AWS Access Key ID [None]: 13URYV2NCYWNXZ3L1FRR
AWS Secret Access Key [None]: tKY2AxXNiU7kenBZv8YM9RMtElL7r16uzaA2b2PN
Default region name [None]: default
Default output format [None]: json
# aws configure --profile user2
AWS Access Key ID [None]: CVWDOTQNCMXSX0M69WII
AWS Secret Access Key [None]: 9BskrHQ1oW3p3fNCwRf2MJocKR3ArLFfSHkSsjUG
Default region name [None]: default
Default output format [None]: json
```

Now create a new bucket as user1.
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=user1 s3api create-bucket --bucket testx-bucket
# aws --endpoint-url "http://mycephfs11:8080" --profile=user1 s3api list-buckets
{
    "Buckets": [
        {
            "Name": "testx-bucket",
            "CreationDate": "2026-09-23T20:27:11.486000+00:00"
        }
    ],
    "Owner": {
        "DisplayName": "TestX - User 1",
        "ID": "testx$user1"
    },
    "Prefix": null
}
```
List the buckets using the radosgw-admin command
```
# radosgw-admin buckets list
[
    "my-bucket",
    "my-2nd-bucket",
    "testx/testx-bucket"
]
```
Notice that the new bucket is named with the pattern tenant/testx-bucket ie. testx/testx-bucket. This allows the cluster to differentiate between buckets created by each tenant.

Now copy over a new object to the newly created bucket
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=user1 s3api put-object --bucket testx-bucket --key=p1 --body /etc/passwd
{
    "ETag": "\"9820d84c95aed471841314471323fcc3\"",
    "ChecksumCRC64NVME": "HbCWNJEc0wk=",
    "ChecksumType": "FULL_OBJECT"
}
# aws --endpoint-url "http://mycephfs11:8080" --profile=user1 s3api list-objects --bucket testx-bucket
{
    "Contents": [
        {
            "Key": "p1",
            "LastModified": "2026-09-23T20:30:41+00:00",
            "ETag": "\"9820d84c95aed471841314471323fcc3\"",
            "Size": 1311,
            "StorageClass": "STANDARD",
            "Owner": {
                "DisplayName": "TestX - User 1",
                "ID": "testx$user1"
            }
        }
    ],
    "RequestCharged": null,
    "Prefix": ""
}

```

We now would like to access this bucket as user2.
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=user2 s3api list-buckets
{
    "Buckets": [],
    "Owner": {
        "DisplayName": "TestX - User 2",
        "ID": "testx$user2"
    },
    "Prefix": null
}
# aws --endpoint-url "http://mycephfs11:8080" --profile=user2 s3api list-objects --bucket testx-bucket

argument of type 'NoneType' is not iterable
```
The user2 does not have access to the bucket nor can they list the bucket. To get around this, we need to set a bucket policy.

First create a file policy.json with the following content
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "User2ReadOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::testx:user/user2"
      },
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::testx-bucket",
        "arn:aws:s3:::testx-bucket/*"
      ]
    }
  ]
}
```
The AWS Resource Name(ARN) points to the tenant username and the bucket/objects.

Next apply the policy
```
# aws --endpoint-url "http://mycephfs11:8080" --profile=user1 s3api put-bucket-policy --bucket testx-bucket --policy file://policy.json

# aws --endpoint-url "http://mycephfs11:8080" --profile=user1 s3api get-bucket-policy --bucket testx-bucket --output text
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "User2ReadOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::testx:user/user2"
...

```

Now retry the operations to list and fetch objects from the bucket testx-bucket as user2.
```
#  aws --endpoint-url "http://mycephfs11:8080" --profile=user2 s3api list-objects --bucket testx-bucket
{
    "Contents": [
        {
            "Key": "p1",
            "LastModified": "2026-09-23T20:30:41+00:00",
            "ETag": "\"9820d84c95aed471841314471323fcc3\"",
            "Size": 1311,
            "StorageClass": "STANDARD",
            "Owner": {
                "DisplayName": "TestX - User 1",
                "ID": "testx$user1"
            }
        }
    ],
    "RequestCharged": null,
    "Prefix": ""
}
#  aws --endpoint-url "http://mycephfs11:8080" --profile=user2 s3api get-object --bucket testx-bucket --key=p1 /tmp/p1-user2
{
    "AcceptRanges": "bytes",
    "LastModified": "2026-09-23T20:30:41+00:00",
    "ContentLength": 1311,
    "ETag": "\"9820d84c95aed471841314471323fcc3\"",
    "ChecksumCRC64NVME": "HbCWNJEc0wk=",
    "ChecksumType": "FULL_OBJECT",
    "ContentType": "binary/octet-stream",
    "Metadata": {}
}
```
However user2 has readonly access on the bucket and cannot put in new objects.
```
#  aws --endpoint-url "http://mycephfs11:8080" --profile=user2 s3api put-object --bucket testx-bucket --key=p2 --body /etc/group

argument of type 'NoneType' is not iterable
```


