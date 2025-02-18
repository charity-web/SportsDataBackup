**SportsDataBackup**

**Prerequisites**
Before running the scripts, ensure you have the following:

**1 Create Rapidapi Account**
Rapidapi.com account, will be needed to access highlight images and videos.

For this example we will be using NCAA (USA College Basketball) highlights since it's included for free in the basic plan. Sports Highlights API is the endpoint we will be using

**2 Verify prerequites are installed**
Docker should be pre-installed in most regions docker --version

AWS CloudShell has AWS CLI pre-installed aws --version

Python3 should be pre-installed also python3 --version

Install gettext package - envsubst is a command-line utility is used for environment variable substituition in shell scripts and text files. Install Steps

**3 Retrieve AWS Account ID**
Copy your AWS Account ID Once logged in to the AWS Management Console Click on your account name in the top right corner You will see your account ID Copy and save this somewhere safe because you will need to update codes in the labs later

**4 Retrieve Access Keys and Secret Access Keys**
You can check to see if you have an access key in the IAM dashboard Under Users, click on a user and then "Security Credentials" Scroll down until you see the Access Key section You will not be able to retrieve your secret access key so if you don't have that somewhere, you need to create an access key.

**Technical Diagram**


**START HERE**
**Step 1: Clone The Repo**
```
git clone https://github.com/alahl1/SportsDataBackup
cd src
```

**Step 2: Create and Configure the .env File**
Search & Replace the following values:

1. Your-AWS-Account-ID
```
aws sts get-caller-identity --query "Account" --output text
```
2. Your-RAPIDAPI-Key
3. Your-AWS-Access-Key
4. Your-AWS-Secret-Access-key
5. S3_BUCKET_NAME=your-alias
6. Your-MediaConvert-Endpoint
```
aws mediaconvert describe-endpoints
```
7. SUBNET_ID=subnet-
8. SECURITY_GROUP_ID=sg-

**Steps for SubnetID and Security Group ID:**

1. In the github repo, there is a resources folder and copy the entire contents
2. In the AWS Cloudshell or vs code terminal, create the file vpc_setup.sh and paste the script inside.
3. Run the script
```
bash vpc_setup.sh
```
4. You will see variables in the output, paste these variables into Subnet_ID and Security_Group_ID
   
**Step 3: Create DynamoDB Table**
1. In the CLI, run the following command to create an on demand table
```
aws dynamodb create-table \
    --table-name SportsHighlights \
    --attribute-definitions AttributeName=id,AttributeType=S \
    --key-schema AttributeName=id,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
```
**Step 4: Load Environment Variables**
```
set -a
source .env
set +a
```
Optional - Verify the variables are loaded
```
echo $AWS_LOGS_GROUP
echo $TASK_FAMILY
echo $AWS_ACCOUNT_ID
```
**Step 4: Generate Final JSON Files from Templates**
1. ECS Task Definition
```
envsubst < taskdef.template.json > taskdef.json
```
2. S3/DynamoDB Policy
```
envsubst < s3_dynamodb_policy.template.json > s3_dynamodb_policy.json
```
3. ECS Target
```
envsubst < ecsTarget.template.json > ecsTarget.json
```
4. ECS Events Role Policy
```
envsubst < ecseventsrole-policy.template.json > ecseventsrole-policy.json
```
*Optional - Open the gnerated files using cat or a text editor to confirm that all place holders have been correctly replaced

**Step 5: Build and Push Docker Image**
1. Create an ECR Repo
```
aws ecr create-repository --repository-name sports-backup
```
2. Log In To ECR
```
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```
3. Build the Docker Image
```
docker build -t sports-backup .
```
4.Tag the Image for ECR

```
docker tag sports-backup:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/sports-backup:latest
```
5. Push the Image
```
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/sports-backup:latest
```
**Step 6: Create AWS Resources**
1. Register the ECS Task Definition
```
aws ecs register-task-definition --cli-input-json file://taskdef.json --region ${AWS_REGION}
```
2. Create the CloudWatch Logs Group
```
aws logs create-log-group --log-group-name "${AWS_LOGS_GROUP}" --region ${AWS_REGION}
```
3. Attach the S3/DynamoDB Policy to the ECS Task Execution Role
```
aws iam put-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-name S3DynamoDBAccessPolicy \
  --policy-document file://s3_dynamodb_policy.json
```
4. Set up the ECS Events Role Create the Role with Trust Policy
```
aws iam create-role --role-name ecsEventsRole --assume-role-policy-document file://ecsEventsRole-trust.json
```
5. Attach the Events Role Policy
```
aws iam put-role-policy --role-name ecsEventsRole --policy-name ecsEventsPolicy --policy-document file://ecseventsrole-policy.json
```
**Step 7: Create an EventBridge Rule to Schedule the Task**
1. Create the Rule
```
aws events put-rule --name SportsBackupScheduleRule --schedule-expression "rate(1 day)" --region ${AWS_REGION}
```
2. Add the Target
```
aws events put-targets --rule SportsBackupScheduleRule --targets file://ecsTarget.json --region ${AWS_REGION}
```
**Step 8: Manually Test ECS Task**
```
aws ecs run-task \
  --cluster sports-backup-cluster \
  --launch-type FARGATE \
  --task-definition ${TASK_FAMILY} \
  --network-configuration "awsvpcConfiguration={subnets=[\"${SUBNET_ID}\"],securityGroups=[\"${SECURITY_GROUP_ID}\"],assignPublicIp=\"ENABLED\"}" \
  --region ${AWS_REGION}
```
**What We Learned**
1. Using templates to generate json files
2. Integrating DynamoDB to store data backup
3. Cloudwatcher for logging
   
**Future Enhancements**
1. Integrate exporting a table from DynamoDB to an S3 bucket
2. Configure an automated backup
3. Creating batch processing of the entire Json file (importing more than 10 videos)
