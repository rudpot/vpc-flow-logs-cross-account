# How to configure cross-account VPC flow logs

The included cloudformation templates will create a bucket in the "remote" or "destination" account, then create a bucket in the "source" account and set up two flow logs from a single subnet to both the "source" and "remote" buckets.

Create as follows:

```bash
# aws cli profile names defined in ~/.aws/config
PROFILE_SRC=ct-acct1
PROFILE_DST=ct-sstest

# Pick a subnet ID in the source account for which you want to set up a flow log
SRC_SUBNET_ID=subnet-075bac08182c7ffd0

# Get SOURCE account ID
SRC_ACCOUNT_ID=$(aws --profile ${PROFILE_SRC} sts get-caller-identity --query 'Account' --output text)

# Create the TARGET configurations
aws --profile ${PROFILE_DST} cloudformation create-stack \
  --stack-name flow-log-dst \
  --template-body file://template-remote-bucket.yaml  \
  --parameters ParameterKey=SourceAccountId,ParameterValue=${SRC_ACCOUNT_ID}

# Get TARGET bucekt ARN
DST_BUCKET_ARN=$(aws --profile ${PROFILE_DST} cloudformation describe-stacks   --stack-name flow-log-dst  --query 'Stacks[0].Outputs[?OutputKey == `RemoteBucketArn`].OutputValue' --output text)

# Create the SOURCE configurations
aws --profile ${PROFILE_SRC} cloudformation create-stack \
  --stack-name flow-log-src \
  --template-body file://template-flow-logs-and-bucket.yaml  \
  --parameters \
    ParameterKey=FlowLogSubnet,ParameterValue=${SRC_SUBNET_ID} \
    ParameterKey=RemoteBucketArn,ParameterValue=${DST_BUCKET_ARN}
```

After a couple minutes you should see flow logs in both buckets.

Clean up by deleteing the cloudformation stacks. This will require manually deleting the bucket content.