# Reducing the cost of SSE\-KMS with Amazon S3 Bucket Keys<a name="bucket-key"></a>

Amazon S3 Bucket Keys reduce the cost of Amazon S3 server\-side encryption using AWS Key Management Service \(SSE\-KMS\)\. This new bucket\-level key for SSE can reduce AWS KMS request costs by up to 99 percent by decreasing the request traffic from Amazon S3 to AWS KMS\. With a few clicks in the AWS Management Console, and without any changes to your client applications, you can configure your bucket to use an S3 Bucket Key for AWS KMS\-based encryption on new objects\.

## S3 Bucket Keys for SSE\-KMS<a name="bucket-key-overview"></a>

Workloads that access millions or billions of objects encrypted with SSE\-KMS can generate large volumes of requests to AWS KMS\. When you use SSE\-KMS to protect your data without an S3 Bucket Key, Amazon S3 uses an individual AWS KMS [data key](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#data-keys) for every object\. It makes a call to AWS KMS every time a request is made against a KMS\-encrypted object\. For information about how SSE\-KMS works, see [Using server\-side encryption with AWS Key Management Service \(SSE\-KMS\)](UsingKMSEncryption.md)\. 

When you configure your bucket to use an S3 Bucket Key for SSE\-KMS, AWS generates a short\-lived bucket\-level key from AWS KMS then temporarily keeps it in S3\. This bucket\-level key will create data keys for new objects during its lifecycle\. S3 Bucket Keys are used for a limited time period within Amazon S3, reducing the need for S3 to make requests to AWS KMS to complete encryption operations\. This reduces traffic from S3 to AWS KMS, allowing you to access AWS KMS\-encrypted objects in Amazon S3 at a fraction of the previous cost\.

A unique bucket\-level key is generated for each requester to ensure an AWS KMS CloudTrail event captures the requester\. If two different IAM roles each put objects into the same bucket during the same time period, this generates two bucket\-level keys since each AWS IAM role is counted as a different requester\. Additionally, the same role, from a *different* IAM session, will still be recognized as a different requester\. For example, an Amazon EMR cluster composed of ten Amazon EC2 instances, will have ten different IAM sessions even if each session assumes the same role\. In this scenario, the Amazon EMR job will use at least ten bucket\-level keys at any given time\. AWS KMS request savings reflect the number of requesters, request patterns, and relative age of the objects requested\. For example, a fewer number of requesters, requesting multiple objects in a limited time window, and encrypted with the same bucket\-level key, will still result in greater savings\.

When you configure an S3 Bucket Key, objects that are already in the bucket do not use the S3 Bucket Key\. To configure an S3 Bucket Key for existing objects, you can use a COPY operation\. For more information, see [Configuring an S3 Bucket Key at the object level using Batch Operations, REST API, AWS SDKs, or AWS CLI](configuring-bucket-key-object.md)\.

Amazon S3 will only share an S3 Bucket Key for objects encrypted by the same AWS KMS key\.

![\[Diagram showing AWS KMS generating a bucket key that creates data keys for objects in a bucket in S3.\]](http://docs.aws.amazon.com/AmazonS3/latest/userguide/images/S3-Bucket-Keys.png)

## Configuring S3 Bucket Keys<a name="configure-bucket-key"></a>

You can configure your bucket to use an S3 Bucket Key for SSE\-KMS on new objects through the Amazon S3 console, AWS SDKs, AWS CLI, or REST API\. With S3 Bucket Keys enabled on your bucket, objects uploaded with a different specified SSE\-KMS key will use their own S3 Bucket Keys\. Regardless of your S3 Bucket Key setting, you can include the `x-amz-server-side-encryption-bucket-key-enabled` header with a `true` or `false` value in your request, to override the bucket setting\.

Before you configure your bucket to use an S3 Bucket Key, review  [Changes to note before enabling an S3 Bucket Key](#bucket-key-changes)\. 

### Configuring an S3 Bucket Key using the Amazon S3 console<a name="configure-bucket-key-console"></a>

When you create a new bucket, you can configure your bucket to use an S3 Bucket Key for SSE\-KMS on new objects\. You can also configure an existing bucket to use an S3 Bucket Key for SSE\-KMS on new objects by updating your bucket properties\. 

For more information, see [Configuring your bucket to use an S3 Bucket Key with SSE\-KMS for new objects](configuring-bucket-key.md)\.

### REST API, AWS CLI, and AWS SDK support for S3 Bucket Keys<a name="configure-bucket-key-programmatic"></a>

You can use the REST API, AWS CLI, or AWS SDK to configure your bucket to use an S3 Bucket Key for SSE\-KMS on new objects\. You can also enable an S3 Bucket Key at the object level\.

For more information, see the following: 
+ [Configuring an S3 Bucket Key at the object level using Batch Operations, REST API, AWS SDKs, or AWS CLI](configuring-bucket-key-object.md)
+ [Configuring your bucket to use an S3 Bucket Key with SSE\-KMS for new objects](configuring-bucket-key.md)

The following APIs support S3 Bucket Keys for SSE\-KMS:
+ [PutBucketEncryption](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketEncryption.html)
  + `ServerSideEncryptionRule` accepts the `BucketKeyEnabled` parameter for enabling and disabling an S3 Bucket Key\.
+ [GetBucketEncryption](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketEncryption.html)
  + `ServerSideEncryptionRule` returns the settings for `BucketKeyEnabled`\.
+ [PutObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html), [CopyObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_CopyObject.html), [CreateMultipartUpload](https://docs.aws.amazon.com/AmazonS3/latest/API/API_CreateMultipartUpload.html), and [PostObject](https://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectPOST.html)
  + `x-amz-server-side-encryption-bucket-key-enabled` request header enables or disables an S3 Bucket Key at the object level\.
+ [HeadObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_HeadObject.html), [GetObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html), [UploadPartCopy](https://docs.aws.amazon.com/AmazonS3/latest/API/API_UploadPartCopy.html), [UploadPart](https://docs.aws.amazon.com/AmazonS3/latest/API/API_UploadPart.html), and [CompleteMultipartUpload](https://docs.aws.amazon.com/AmazonS3/latest/API/API_CompleteMultipartUpload.html)
  + `x-amz-server-side-encryption-bucket-key-enabled` response header indicates if an S3 Bucket Key is enabled or disabled for an object\.

### Working with AWS CloudFormation<a name="configure-bucket-key-cfn"></a>

In AWS CloudFormation, the `AWS::S3::Bucket` resource includes an encryption property called `BucketKeyEnabled` that you can use to enable or disable an S3 Bucket Key\. 

For more information, see [Using AWS CloudFormation](configuring-bucket-key.md#enable-bucket-key-cloudformation)\.

## Changes to note before enabling an S3 Bucket Key<a name="bucket-key-changes"></a>

Before you enable an S3 Bucket Key, please note the following related changes:

### IAM or KMS key policies<a name="bucket-key-policies"></a>

If your existing IAM policies or AWS KMS key policies use your object Amazon Resource Name \(ARN\) as the encryption context to refine or limit access to your KMS key, these policies won’t work with an S3 Bucket Key\. S3 Bucket Keys use the bucket ARN as encryption context\. Before you enable an S3 Bucket Key, update your IAM policies or AWS KMS key policies to use your bucket ARN as encryption context\.

For more information about encryption context and S3 Bucket Keys, see [Encryption context](UsingKMSEncryption.md#encryption-context)\.

### AWS KMS CloudTrail events<a name="bucket-key-cloudtrail"></a>

After you enable an S3 Bucket Key, your AWS KMS CloudTrail events log your bucket ARN instead of your object ARN\. Additionally, you see fewer KMS CloudTrail events for SSE\-KMS objects in your logs\. Because key material is time\-limited in Amazon S3, fewer requests are made to AWS KMS\.

## Using an S3 Bucket Key with replication<a name="bucket-key-replication"></a>

You can use S3 Bucket Keys with Same\-Region Replication \(SRR\) and Cross\-Region Replication \(CRR\)\.

When Amazon S3 replicates an encrypted object, it generally preserves the encryption settings of the replica object in the destination bucket\. However, if the source object is not encrypted and your destination bucket uses default encryption or an S3 Bucket Key, Amazon S3 encrypts the object with the destination bucket’s configuration\. 

The following examples illustrate how an S3 Bucket Key works with replication\. For more information, see [Replicating objects created with server\-side encryption \(SSE\-C, SSE\-S3, SSE\-KMS\)](replication-config-for-kms-objects.md)\. 

**Example 1 – Source object uses S3 Bucket Keys, destination bucket uses default encryption**  
If your source object uses an S3 Bucket Key but your destination bucket uses default encryption with SSE\-KMS, the replica object maintains its S3 Bucket Key encryption settings in the destination bucket\. The destination bucket still uses default encryption with SSE\-KMS\.   


**Example 2 – Source object is not encrypted, destination bucket uses an S3 Bucket Key with SSE\-KMS**  
If your source object is not encrypted and the destination bucket uses an S3 Bucket Key with SSE\-KMS, the source object is encrypted with an S3 Bucket Key using SSE\-KMS in the destination bucket\. This results in the `ETag` of the source object being different from the `ETag` of the replica object\. You must update applications that use the `ETag` to accommodate for this difference\.

## Working with S3 Bucket Keys<a name="using-bucket-key"></a>

For more information about enabling and working with S3 Bucket Keys, see the following sections:
+ [Configuring your bucket to use an S3 Bucket Key with SSE\-KMS for new objects](configuring-bucket-key.md)
+ [Configuring an S3 Bucket Key at the object level using Batch Operations, REST API, AWS SDKs, or AWS CLI](configuring-bucket-key-object.md)
+ [Viewing settings for an S3 Bucket Key ](viewing-bucket-key-settings.md)