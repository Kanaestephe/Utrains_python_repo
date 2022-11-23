## Part of our external audit showed that some of our S3 buckets are exposed and do not have encryption enabled and no bucket policy applied.

### Write a script that checks all the S3 buckets and identifies which are not encrypted and no bucket policy applied. After finding the ones that are on-compliant, fix them accordingly.

``` python
import boto3
import json
from botocore.exceptions import ClientError

AWS_REGION="<SET YOUR AWS REGION>"
s3_client = boto3.client('s3', region_name=AWS_REGION)

buckets_list = s3_client.list_buckets()

unencrypted_buckets=[]

for bucket in buckets_list['Buckets']:
    try:
        enc = s3_client.get_bucket_encryption(Bucket=bucket['Name'])
        rules = enc['ServerSideEncryptionConfiguration']['Rules']
        print(f"Bucket: {bucket['Name']}, Encryption: {rules}")
    except ClientError as e:
        # get the list of unencrypted buckets 
        if e.response['Error']['Code'] == 'ServerSideEncryptionConfigurationNotFoundError':
            print(f"Bucket: {bucket['Name']}, no server-side encryption")
            # put the encryption on bucket
            response= s3_client.put_bucket_encryption(
                Bucket= bucket,
                ServerSideEncryptionConfiguration={
                    'Rules': [
                        {'ApplyServerSideEncryptionByDefault': {'SSEAlgorithm': 'AES256'}},
                    ]
                }
            )
            unencrypted_buckets.append(bucket['Name'])
        else:
            print(f"Bucket: {bucket['Name']}, unexpected error for encryption check: {e}")

_no_policy_buckets=[]
for bucket in buckets_list:
    try:
        res = s3_client.get_bucket_policy_status(Bucket=bucket['Name'])
        policy_status= res['PolicyStatus']['IsPublic']
        if policy_status:
            print(f"Bucket: {bucket['Name']}, Status: {policy_status}")
        else:
            print(f"Bucket: {bucket['Name']}, no status")
            # set public policy on bucket
            # define a bucket policy of the bucket
            ### change BUCKET_ARN and IAM_ROLE_ARN in the bucket_policy
            bucket_policy = {
                "Id":
                "sid1",
                "Version":
                "2012-10-17",
                "Statement": [{
                    "Sid": "Sid1",
                    "Action": ["s3:GetObject"],
                    "Effect": "Allow",
                    "Resource": ["BUCKET_ARN/*"],
                    "Principal": {
                        "AWS": ["IAM_ROLE_ARN"]
                    }
                }]
            }
            response = s3_client.put_bucket_policy(Bucket=bucket['Name'],
                                        Policy=json.dumps(bucket_policy))
            _no_policy_buckets.append(bucket['Name'])

    except ClientError as e:
        print(f"Bucket: {bucket['Name']}, unexpected error for no policy check: {e.response}")

buck_enc_pol_changed= list(set(unencrypted_buckets+_no_policy_buckets))

if buck_enc_pol_changed:
    print("############# Bucket name: ########### \n")
    for bucket_name in buck_enc_pol_changed:
        print(bucket_name)

```
## Test version to change the one that's up

```python

import boto3
import json
from botocore.exceptions import ClientError

AWS_REGION='ap-south-1'
s3_client = boto3.client('s3', region_name=AWS_REGION)

buckets_list = s3_client.list_buckets()

buckets = [bucket['Name'] for bucket in buckets_list['Buckets']]

def ListEncryptionFixed():
    """ This function find all s3 buckets without Encryption and fixed them, 
    then return the list of fixed buckets """
    unencrypted_buckets_fixed=[]
    for bucket in buckets:
        # print(bucket['Name'])
        bucket_region=s3_client.head_bucket(Bucket=bucket)['ResponseMetadata']['HTTPHeaders']['x-amz-bucket-region']
        if  bucket_region== AWS_REGION:
        #if bucket == 'first-utrains-bucket':
            try:
                _bucket_enc_response = s3_client.get_bucket_encryption(Bucket=bucket)
                rules = _bucket_enc_response['ServerSideEncryptionConfiguration']['Rules']
                print(f"Bucket: {bucket}, Encryption: {rules}")
            except ClientError as e:
                # get the list of unencrypted buckets 
                if e.response['Error']['Code'] == 'ServerSideEncryptionConfigurationNotFoundError':
                    print(f"Bucket: {bucket}, no server-side encryption")
                    # put the encryption on bucket
                    response= s3_client.put_bucket_encryption(
                        Bucket= bucket,
                        ServerSideEncryptionConfiguration={
                            'Rules': [
                                {'ApplyServerSideEncryptionByDefault': {'SSEAlgorithm': 'AES256'}},
                            ]
                        }
                    )
                    print(response)
                    unencrypted_buckets_fixed.append(bucket)
                else:
                    print(f"Bucket: {bucket}, unexpected error for encryption check: {e}")
    return unencrypted_buckets_fixed


def set_bucket_policy(bucket_name):
    """ This function is used to attach a policy to a bucket"""
    # set public policy on bucket
    # define a bucket policy of the bucket
    ### change BUCKET_ARN and IAM_ROLE_ARN in the bucket_policy
    bucket_policy={
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": "s3:*",
                "Resource": f"arn:aws:s3:::{bucket_name}/*",
                "Principal": {
                    "AWS": [
                        "arn:aws:iam::076892551558:user/hermann90",
                    ]
                },
            }
        ]
    }
    bucket_policy=json.dumps(bucket_policy)
    response = s3_client.put_bucket_policy(Bucket=bucket_name, Policy=bucket_policy)
    return response

def ListPolicyFixed():
    """ This function find all s3 buckets with no policy and attach policy to them, 
    then return the list of fixed buckets """
    _no_policy_buckets_fixed=[]
    for bucket in buckets:
        #print(bucket)
        bucket_region=s3_client.head_bucket(Bucket=bucket)['ResponseMetadata']['HTTPHeaders']['x-amz-bucket-region']
        if  bucket_region== AWS_REGION:
            try:
                policy_status=s3_client.get_bucket_policy_status(Bucket=bucket)
                print(f"Bucket: {bucket}, policy: {policy_status}")
            except ClientError as e:
                if e.response['Error']['Code'] == 'NoSuchBucketPolicy':
                    print(f"Bucket: {bucket}, has no policy attached")
                    try:
                        attached_policy= set_bucket_policy(bucket)
                        print(attached_policy)
                        _no_policy_buckets_fixed.append(bucket)
                        print(f"Policy attached to {bucket}")  
                    except ClientError as e:
                        print(f"no policy attached, {e}")
                else:
                    print(f"Bucket: {bucket}, unexpected error for policy check: {e}")
    return _no_policy_buckets_fixed
#buck_enc_pol_changed= list(set(unencrypted_buckets+_no_policy_buckets))

encrypted_buckets= ListEncryptionFixed()
fixed_policy_buckets= ListPolicyFixed()

print("\n ############# List of buckets with no encryption fixed: ########### \n")
if encrypted_buckets:
    for bucket_name in encrypted_buckets:
        print(f"\t \t Bucket: {bucket_name} \n")
else:
    print(f"\t \t All Buckets are  encrypted \n")

print("\n ############# List of buckets with no encryption fixed: ########### \n")

if fixed_policy_buckets:
    for bucket_name in fixed_policy_buckets:
        print(f"\t \t Bucket: {bucket_name} \n")
else:
    print(f"\t \t All Buckets have policy \n")
    
``
