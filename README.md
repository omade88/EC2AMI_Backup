## DEPLOY LAMBDA FUNCTION TO CREATE WEEKLY EC2 AMI BACKUP


## Goal
The goals of this project are to create weekly EC2 AMI backups of all EC2 instances running in the us-east-1 region and to delete AMIs older than 30 days.

## Pre-Requisites
01. Deploy Lambda Function as per the architecture shown above with required IAM roles.
02. Schedule Lambda Function to run weekly once Sunday 5 AM EST using cloudwatch event as Lambda trigger.
03. Create an SNS topic and subscribe email to receive notifications.

## Automation
01. Write Python code as per the below high level requirement create AMI and delete old AMIs.
- Describe list of all EC2 instances in the us-east-1 region.
- Get the list of EC2 Instance IDs as list data type
- Get the EC2 Tag having key “Name”
- Loop the list of Instance IDs and create AMI
- Add tags to the AMI – Assign the EC2 Name tag to the AMI to identify which AMI belongs to which EC2 instance.
- Add description to the AMI to understand which AMI belongs to which server.
- Add name to the AMI – Name includes server name (Get from Instance Tag) append with the date when the AMI was created.
- Delete the old unused AMIs whichever is older than 30 days from the date of AMI creation.
- Ensure that only unused AMIs are deleted, Skip the AMIs that are in use.
- Print the AMIs that are deleted.
- Print exceptions if any issues in creating or deleting AMI
- Notify the SNS topic if there are any exceptions in creating or deleting AMI.

02. Deploy Python code to Lambda Function.

## Validation
Run Lambda Function and verify AMI copy created for all EC2 instances with tags and deleted unused AMIs older than 30 days.


## LET'S DO IT
To achieve the goal of creating weekly EC2 AMI backups and deleting AMIs older than 30 days, follow these steps:

## Architecture Overview
01. AWS Lambda: The core of the automation, running Python code to handle AMI creation and deletion.
02. AWS IAM Roles: Provide the necessary permissions for the Lambda function to interact with EC2, CloudWatch, and SNS.
03. AWS CloudWatch Events: Trigger the Lambda function weekly.
04. AWS SNS: Send notifications about the Lambda function’s execution status.


## Pre-Requisites
01.	IAM Role for Lambda: Create an IAM role with the following permissions:
- ec2:DescribeInstances
- ec2:CreateImage
- ec2:DescribeImages
- ec2:DeregisterImage
- sns:Publish
02.	CloudWatch Event Rule: Schedule the Lambda function to run every Sunday at 5 AM EST.
03.	EC2 Instances: Ensure you have at least 5 EC2 instances  with the tags
04.	SNS Topic: Create an SNS topic and subscribe to an email address for notifications.

## Step-by-Step Guide to Create and Deploy a Valid Lambda Function ZIP File
01. Write the Python Code
First, create a Python file named lambda_function.py with the following content. Lambda Function Code. Below is the Python code to be deployed in the Lambda function:

```bash
import boto3
from datetime import datetime, timedelta
import logging

# Initialize logger
logger = logging.getLogger()
logger.setLevel(logging.INFO)

ec2_client = boto3.client('ec2')
sns_client = boto3.client('sns')

def lambda_handler(event, context):
    try:
        # Describe EC2 instances
        instances = ec2_client.describe_instances(Filters=[{'Name': 'tag:Name', 'Values': ['dpt-web-server']}])
        instance_ids = [instance['InstanceId'] for reservation in instances['Reservations'] for instance in reservation['Instances']]

        # Create AMIs
        for instance_id in instance_ids:
            instance = ec2_client.describe_instances(InstanceIds=[instance_id])['Reservations'][0]['Instances'][0]
            instance_name = [tag['Value'] for tag in instance['Tags'] if tag['Key'] == 'Name'][0]
            ami_name = f"{instance_name}-{datetime.now().strftime('%Y-%m-%d')}"

            response = ec2_client.create_image(
                InstanceId=instance_id,
                Name=ami_name,
                Description=f"AMI of {instance_name} created on {datetime.now().strftime('%Y-%m-%d')}",
                NoReboot=True
            )

            ami_id = response['ImageId']
            ec2_client.create_tags(Resources=[ami_id], Tags=instance['Tags'])
            logger.info(f"Created AMI {ami_id} for instance {instance_id}")

        # Delete old AMIs
        delete_date = datetime.now() - timedelta(days=30)
        images = ec2_client.describe_images(Owners=['self'])['Images']

        for image in images:
            creation_date = datetime.strptime(image['CreationDate'], '%Y-%m-%dT%H:%M:%S.%fZ')
            if creation_date < delete_date:
                ec2_client.deregister_image(ImageId=image['ImageId'])
                logger.info(f"Deregistered AMI {image['ImageId']} created on {creation_date}")

    except Exception as e:
        logger.error(f"Error occurred: {str(e)}")
        sns_client.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:YourSNSTopic',
            Subject='Error in AMI Backup Lambda Function',
            Message=str(e)
        )

    return {
        'statusCode': 200,
        'body': 'AMI backup and cleanup process completed successfully.'
    }
```
02. Create a ZIP File
   1. Create a Directory:
- Create a new directory to store your Lambda function code.
- Place your lambda_function.py file in this directory.
   2. Zip the Directory:
- Open a terminal or command prompt and navigate to the directory containing your lambda_function.py.
- Run the following command to create a ZIP file:
```bash
zip -r function.zip lambda_function.py
```


