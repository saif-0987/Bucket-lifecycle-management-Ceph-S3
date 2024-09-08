# bucket-lifecycle-management
Bucket Lifecycle Management in Ceph S3 ( RGW )

You can use a bucket lifecycle configuration to manage your objects so they are stored effectively throughout their lifetime. The S3 API in the Ceph Object Gateway supports a subset of the AWS bucket lifecycle actions.
We will perform below steps to fulfil this 
- object tagging
- Implement bucket lifecycle managemnet


## Object tagging in Ceph S3 using CLI
The Ceph S3 has feature to assign tags—custom key-value pairs—to various resources such as objects, buckets, and roles. This tagging feature allows users to organize resources according to their needs. 
Once tagged, attribute-based access control (ABAC) can control access to these resources.
### Pre-requisites:
Before you begin, ensure you have the following tools installed:
- `s3cmd`
- `awscli`

### Create Bucket:
To create a new bucket, use the following command:
```
root@client-01:~# s3cmd mb s3://my-test-bucket
```

### Upload Objects into the bucket
Upload your objects to the newly created bucket using:
```
root@client-01:~# s3cmd put file-01.txt s3://my-test-bucket
upload: 'file-01.txt' -> 's3://my-test-bucket/file-01.txt'  [1 of 1]
1048576 of 1048576   100% in    0s    24.81 MB/s  done
```
Substitute path/to/your/file with the path to your local file and `your-bucket-name` with the name of your bucket.

### Ensure no tag is assigned to the objects
```
root@client-01:~# aws s3api get-object-tagging --bucket my-test-bucket --key file-01.txt --endpoint-url http://192.168.99.132:8001
{
    "TagSet": []
}
```
Replace `your-bucket-name` with your bucket name and `your-object-key` with the key of the object.

### Assign tag to objects
To assign a tag to an object, use the following command:
```
root@client-01:~# aws s3api put-object-tagging  --bucket my-test-bucket  --key file-01.txt  --tagging '{"TagSet": [{ "Key": "designation", "Value": "confidential" }]}' --endpoint-url http://192.168.99.132:8001
```

### Verify that objects is now tagged 
To verify that the object has been tagged, use:
```
root@client-01:~# aws s3api get-object-tagging --bucket my-test-bucket --key file-01.txt --endpoint-url http://192.168.99.132:8001

{
    "TagSet": [
        {
            "Key": "designation",
            "Value": "confidential"
        }
    ]
}
```
This command will display the tags associated with the object.


## Configure bucket lifecycle management
The following command retrieves metadata for an object in a bucket 

```
root@client-01:~# aws s3api head-object --bucket my-test-bucket  --key file-01.txt  --endpoint-url http://192.168.99.132:8001 
{
    "AcceptRanges": "bytes",
    "LastModified": "2024-09-08T10:45:25+00:00",
    "ContentLength": 1048576,
    "ETag": "\"b6d81b360a5672d80c27430f39153e2c\"",
    "ContentType": "application/octet-stream",
    "Metadata": {
        "s3cmd-attrs": "atime:1725792239/ctime:1725792239/gid:0/gname:root/md5:b6d81b360a5672d80c27430f39153e2c/mode:33188/mtime:1725792239/uid:0/uname:root"
    },
    "StorageClass": "STANDARD"
}
```
### Implementing Lifecycle Management with Lifecycle Configuration File

To configure lifecycle management for an S3 bucket, use the following AWS CLI command:
```
aws s3api put-bucket-lifecycle-configuration --bucket my-test-bucket --lifecycle-configuration file://expiration_ingest_bucket.json --endpoint-url http://192.168.99.132:8001
```

### Checking Lifecycle Management Status
To verify if the lifecycle management has been scheduled, execute the following command:
```
radosgw-admin lc list
[
 {
        "bucket": ":my-test-bucket:9aab0753-5801-4158-95d9-88bd8de0ca71.44122.1",
        "started": "Thu, 01 Jan 1970 00:00:00 GMT",
        "status": "UNINITIAL"
    }
]
```
If the status shows `UNINITIAL`, this indicates that the lifecycle management has been scheduled:

## Setting rgw_lc_debug_interval
To avoid waiting for an extended period to validate the lifecycle management, adjust the debug interval to view results immediately:
```
ceph config set global rgw_lc_debug_interval 10
```


## Restarting the RGW Daemon
Restart the RGW (Rados Gateway) daemon to apply the new changes:
```
ceph orch restart rgw.client
```
## Verifying Lifecycle Management Execution
A `COMPLETE` status indicates that the lifecycle management process has finished:
```
radosgw-admin lc list
[
    {
        "bucket": ":my-test-bucket:9aab0753-5801-4158-95d9-88bd8de0ca71.44122.1",
        "started": "Sun, 08 Sep 2024 14:04:33 GMT",
        "status": "COMPLETE"
    }
]
```
You can then confirm that no objects remain in the bucket:
```
root@client-01:~# s3cmd ls s3://my-test-bucket
```
Also we are not able to see any metadata for that objects since that objects has now been deleted.
```
root@client-01:~# aws s3api head-object --bucket my-test-bucket  --key file-01.txt  --endpoint-url http://192.168.99.132:8001 

An error occurred (404) when calling the HeadObject operation: Not Found
```








