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
