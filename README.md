# Replicating AWS Labs on Your Windows 11 Device

I'll help you replicate these labs on your Windows 11 device using your AWS account. I'll go through each lab systematically, adapting the instructions for your environment while skipping any parts that would incur unnecessary costs.

## General Setup

Before we start with the individual labs, let's set up your AWS CLI configuration on your Windows 11 device:

```bash
aws configure
```

When prompted, enter:
- AWS Access Key ID: [Your access key]
- AWS Secret Access Key: [Your secret key]
- Default region name: us-east-1
- Default output format: json

Now let's proceed with each lab:

## Lab 8.1: Migrating a Web Application to Docker Containers

### Task 1: Preparing the development environment

1. First, make sure Docker Desktop is installed on your Windows 11 machine. If not, download and install it from [Docker's website](https://www.docker.com/products/docker-desktop/).

2. Create a working directory for the lab:

```bash
mkdir -p c:\Users\saihe\Projects\Final BAD\lab 8.1\working
cd c:\Users\saihe\Projects\Final BAD\lab 8.1\working
```

3. Extract the code files if you haven't already:

```bash
cd c:\Users\saihe\Projects\Final BAD\lab 8.1
```

### Task 2: Analyzing the existing application infrastructure

Since we don't have the actual EC2 instances running, we'll work with the code you've extracted.

1. Navigate to the codebase directory:

```bash
cd c:\Users\saihe\Projects\Final BAD\lab 8.1\resources\codebase_partner
```

2. Review the application code to understand its structure.

### Task 3: Migrating the application to a Docker container

1. Create a working directory for the node application:

```bash
mkdir -p c:\Users\saihe\Projects\Final BAD\lab 8.1\containers\node_app
cd c:\Users\saihe\Projects\Final BAD\lab 8.1\containers\node_app
```

2. Copy the code base to the new directory:

```bash
xcopy c:\Users\saihe\Projects\Final BAD\lab 8.1\resources\codebase_partner c:\Users\saihe\Projects\Final BAD\lab 8.1\containers\node_app\codebase_partner /E /I
```

3. Create a Dockerfile:

```bash
cd c:\Users\saihe\Projects\Final BAD\lab 8.1\containers\node_app\codebase_partner
```

Create a file named `Dockerfile` with the following content:

```dockerfile:c:\Users\saihe\Projects\Final BAD\lab 8.1\containers\node_app\codebase_partner\Dockerfile
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "run", "start"]
```

4. Build the Docker image:

```bash
docker build --tag node_app .
```

5. Run the Docker container:

```bash
docker run -d --name node_app_1 -p 3000:3000 node_app
```

6. Verify the container is running:

```bash
docker container ls
```

7. Test the application:
   - Open a browser and navigate to http://localhost:3000

### Task 4: Creating an Amazon ECR repository and pushing the image

1. Create an ECR repository:

```bash
aws ecr create-repository --repository-name cafe/node-web-app --region us-east-1
```

2. Get the login command for ECR:

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 831926586385.dkr.ecr.us-east-1.amazonaws.com
```

3. Tag the Docker image:

```bash
docker tag node_app:latest 831926586385.dkr.ecr.us-east-1.amazonaws.com/cafe/node-web-app:latest
```

4. Push the image to ECR:

```bash
docker push 831926586385.dkr.ecr.us-east-1.amazonaws.com/cafe/node-web-app:latest
```

## Lab 8.2: Running Containers on a Managed Service

### Task 1: Preparing the development environment

1. Create a working directory:

```bash
mkdir -p c:\Users\saihe\Projects\Final BAD\lab 8.2\working
cd c:\Users\saihe\Projects\Final BAD\lab 8.2\working
```

### Task 2: Configuring the subnets for Amazon RDS and Elastic Beanstalk

1. Create a VPC for your resources:

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=IDE_VPC}]"
```

2. Create two subnets in different availability zones:

```bash
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=IDE_Subnet}]"

aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=extraSubnetForRds}]"
```

3. Create an internet gateway and attach it to the VPC:

```bash
aws ec2 create-internet-gateway --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=IDE_IGW}]"

aws ec2 attach-internet-gateway --internet-gateway-id <igw-id> --vpc-id <vpc-id>
```

4. Create a route table and add a route to the internet:

```bash
aws ec2 create-route-table --vpc-id <vpc-id> --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=IDE_RT}]"

aws ec2 create-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <igw-id>
```

5. Associate the route table with both subnets:

```bash
aws ec2 associate-route-table --route-table-id <rtb-id> --subnet-id <subnet1-id>
aws ec2 associate-route-table --route-table-id <rtb-id> --subnet-id <subnet2-id>
```

### Task 3: Setting up an Aurora Serverless database

This part would incur costs, so we'll skip the actual creation but outline the steps:

1. Create a security group for the database:

```bash
aws ec2 create-security-group --group-name Aurora-SG --description "Security group for Aurora" --vpc-id <vpc-id>
```

2. Add inbound rules to the security group:

```bash
aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol tcp --port 3306 --source-group <sg-id>
```

3. For a real implementation, you would create an Aurora Serverless cluster with:

```bash
aws rds create-db-cluster --db-cluster-identifier supplierdb --engine aurora-mysql --engine-version 5.7 --master-username admin --master-user-password coffee_beans_for_all --db-subnet-group-name <subnet-group-name> --vpc-security-group-ids <sg-id> --engine-mode serverless --scaling-configuration MinCapacity=2,MaxCapacity=16,AutoPause=true,SecondsUntilAutoPause=300,TimeoutAction=ForceApplyCapacityChange
```

### Task 4: Reviewing the container image

1. Pull the Docker image from ECR:

```bash
docker pull 831926586385.dkr.ecr.us-east-1.amazonaws.com/cafe/node-web-app:latest
```

2. Run the container locally to test:

```bash
docker run -d -p 3000:3000 831926586385.dkr.ecr.us-east-1.amazonaws.com/cafe/node-web-app:latest
```

### Task 5: Creating an Elastic Beanstalk application

This would also incur costs, so we'll outline the steps without implementing:

1. Create an Elastic Beanstalk application:

```bash
aws elasticbeanstalk create-application --application-name coffee-suppliers
```

2. Create an application version:

```bash
aws elasticbeanstalk create-application-version --application-name coffee-suppliers --version-label v1 --source-bundle S3Bucket=<bucket-name>,S3Key=<key>
```

3. Create an environment:

```bash
aws elasticbeanstalk create-environment --application-name coffee-suppliers --environment-name MyEnv --solution-stack-name "64bit Amazon Linux 2 v3.4.9 running Docker" --version-label v1
```

## Lab 9.1: Caching Application Data with ElastiCache

### Task 1: Preparing the development environment

1. Create a working directory:

```bash
mkdir -p c:\Users\saihe\Projects\Final BAD\lab 9.1\working
cd c:\Users\saihe\Projects\Final BAD\lab 9.1\working
```

### Task 2: Configuring the subnets for ElastiCache to use

1. Create a security group for ElastiCache:

```bash
aws ec2 create-security-group --group-name ElastiCache-SG --description "Security group for ElastiCache" --vpc-id <vpc-id>
```

2. Add inbound rules to the security group:

```bash
aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol tcp --port 11211 --source-group <aurora-sg-id>
```

3. Create a subnet group for ElastiCache:

```bash
aws elasticache create-cache-subnet-group --cache-subnet-group-name ElastiCacheSubnetGroup --cache-subnet-group-description "Subnet Group for ElastiCache" --subnet-ids <subnet1-id> <subnet2-id>
```

### Task 3: Creating the ElastiCache cluster

This would incur costs, so we'll outline the steps:

```bash
aws elasticache create-cache-cluster --cache-cluster-id MemcachedCache --engine memcached --cache-node-type cache.r6g.large --num-cache-nodes 3 --cache-subnet-group-name ElastiCacheSubnetGroup --security-group-ids <sg-id> --engine-version 1.6.6
```

### Task 4-6: Working with the cache

For these tasks, we'll need to create Python scripts to interact with the cache. Here's an example of a script to find all records:

```python:c:\Users\saihe\Projects\Final BAD\lab 9.1\python_3\find_all.py
import pymysql
import json
from pymemcache import base
import time

# Define the connection to the Aurora database
mydb = pymysql.connect(
    host="<your-aurora-endpoint>",
    user="admin",
    password="coffee_beans_for_all",
    database="suppliers"
)

# Define the connection to the Memcached cache
memcached_client = base.Client(('<your-memcached-endpoint>', 11211))

# Define the TTL for cached items (3 minutes)
TTL_INT = 180

# Try to get data from the cache
data = memcached_client.get('all_beans')

if data:
    # If data is found in the cache, decode and print it
    print("Data from cache:")
    print(data.decode('utf-8'))
else:
    # If no data is found in the cache, query the database
    print("No data in cache. Querying database...")
    db_query = "SELECT * FROM beans"
    mycursor = mydb.cursor()
    mycursor.execute(db_query)
    
    # Format the results as JSON
    rows = mycursor.fetchall()
    field_names = [i[0] for i in mycursor.description]
    output = []
    
    for row in rows:
        output.append(dict(zip(field_names, row)))
    
    output_json = json.dumps(output)
    
    # Store the results in the cache
    memcached_call = memcached_client.set('all_beans', output_json, TTL_INT)
    
    print("Data from database:")
    print(output_json)
```

## Lab 9.2: Implementing CloudFront for Caching and Application Security

### Task 1: Preparing the development environment

1. Create a working directory:

```bash
mkdir -p c:\Users\saihe\Projects\Final BAD\lab 9.2\working
cd c:\Users\saihe\Projects\Final BAD\lab 9.2\working
```

### Task 2: Configuring a distribution for static website content

1. Create an S3 bucket for your website:

```bash
aws s3 mb s3://bad6520051final --region us-east-1
```

2. Upload website files to the bucket:

```bash
aws s3 cp c:\Users\saihe\Projects\Final BAD\lab 9.2\resources\website s3://bad6520051final/ --recursive --cache-control "max-age=0"
```

3. Create a CloudFront distribution:

```bash
aws cloudfront create-distribution --origin-domain-name bad6520051final.s3.us-east-1.amazonaws.com --default-root-object index.html
```

4. Update the bucket policy to only allow access from CloudFront:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::bad6520051final/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<your-distribution-id>"
        }
      }
    }
  ]
}
```

### Task 3: Securing network access to the distribution using AWS WAF

1. Create an IP set:

```bash
aws wafv2 create-ip-set --name office --scope CLOUDFRONT --ip-address-version IPV4 --addresses "<your-ip>/32" --region us-east-1
```

2. Create a web ACL:

```bash
aws wafv2 create-web-acl --name CafeWebsiteAcl --scope CLOUDFRONT --default-action Block={} --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=CafeWebsiteAcl --region us-east-1
```

3. Add a rule to the web ACL:

```bash
aws wafv2 update-web-acl --name CafeWebsiteAcl --scope CLOUDFRONT --id <web-acl-id> --default-action Block={} --rules '[{"Name":"AllowOffice","Priority":0,"Statement":{"IPSetReferenceStatement":{"ARN":"<ip-set-arn>"}},"Action":{"Allow":{}},"VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"AllowOffice"}}]' --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=CafeWebsiteAcl --region us-east-1
```

I'll continue with the implementation of Lab 10.1 and complete the remaining steps.

## Lab 10.1: Implementing a Messaging System Using Amazon SNS and Amazon SQS (continued)

### Task 2: Configuring the Amazon SQS dead-letter queue

1. Create a dead-letter queue:

```bash
cd c:\Users\saihe\Projects\Final BAD\lab 10.1\resources\sqs-sns
```

Create a file named `create-dlq.json`:

```json:c:\Users\saihe\Projects\Final BAD\lab 10.1\resources\sqs-sns\create-dlq.json
{
    "FifoQueue": "true",
    "VisibilityTimeout": "20",
    "ReceiveMessageWaitTimeSeconds": "0",
    "ContentBasedDeduplication": "false",
    "DeduplicationScope": "queue"
}
```

Now create the queue:

```bash
aws sqs create-queue --queue-name DeadLetterQueue.fifo --attributes file://create-dlq.json
```

2. Create a policy for the dead-letter queue:

```json:c:\Users\saihe\Projects\Final BAD\lab 10.1\resources\sqs-sns\dlq-policy.json
{"Policy": "{\"Version\": \"2008-10-17\",\"Id\": \"DlqSqsPolicy\",\"Statement\": [{\"Sid\": \"dead-letter-sqs\",\"Effect\": \"Allow\",\"Principal\": {\"AWS\": \"arn:aws:iam::831926586385:root\"},\"Action\": [\"SQS:*\"],\"Resource\": \"arn:aws:sqs:us-east-1:831926586385:DeadLetterQueue.fifo\"}]}"}
```

Apply the policy:

```bash
aws sqs set-queue-attributes --queue-url "https://sqs.us-east-1.amazonaws.com/831926586385/DeadLetterQueue.fifo" --attributes file://dlq-policy.json
```

### Task 3: Configuring the SQS queue

1. Create the main queue configuration file:

```json:c:\Users\saihe\Projects\Final BAD\lab 10.1\resources\sqs-sns\create-beans-queue.json
{
    "FifoQueue": "true",
    "VisibilityTimeout": "30",
    "ReceiveMessageWaitTimeSeconds": "20",
    "ContentBasedDeduplication": "true",
    "DeduplicationScope": "queue",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:831926586385:DeadLetterQueue.fifo\",\"maxReceiveCount\":\"5\"}"
}
```

2. Create the main queue:

```bash
aws sqs create-queue --queue-name updated_beans.fifo --attributes file://create-beans-queue.json
```

3. Create a policy for the main queue:

```json:c:\Users\saihe\Projects\Final BAD\lab 10.1\resources\sqs-sns\beans-policy.json
{"Policy": "{\"Version\": \"2008-10-17\",\"Id\": \"BeansSqsPolicy\",\"Statement\": [{\"Sid\": \"beans-sqs\",\"Effect\": \"Allow\",\"Principal\": {\"AWS\": \"arn:aws:iam::831926586385:root\"},\"Action\": \"SQS:*\",\"Resource\": \"arn:aws:sqs:us-east-1:831926586385:updated_beans.fifo\"},{\"Sid\": \"topic-subscription\",\"Effect\": \"Allow\",\"Principal\": {\"AWS\": \"arn:aws:iam::831926586385:root\",\"Service\": \"sns.amazonaws.com\"},\"Action\": \"SQS:SendMessage\",\"Resource\": \"arn:aws:sqs:us-east-1:831926586385:updated_beans.fifo\",\"Condition\": {\"ArnLike\": {\"aws:SourceArn\": \"arn:aws:sns:us-east-1:831926586385:updated_beans_sns.fifo\"}}}]}"}
```

4. Apply the policy:

```bash
aws sqs set-queue-attributes --queue-url "https://sqs.us-east-1.amazonaws.com/831926586385/updated_beans.fifo" --attributes file://beans-policy.json
```

### Task 4: Configuring the SNS topic

1. Create the SNS topic:

```bash
aws sns create-topic --name updated_beans_sns.fifo --attributes FifoTopic=true,ContentBasedDeduplication=true
```

2. Create a subscription to connect the SNS topic to the SQS queue:

```bash
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:831926586385:updated_beans_sns.fifo --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:831926586385:updated_beans.fifo --attributes RawMessageDelivery=true
```

### Task 5: Creating a publisher to send messages to the SNS topic

1. Create a Python script to publish messages to the SNS topic:

```python:c:\Users\saihe\Projects\Final BAD\lab 10.1\resources\sqs-sns\publish_beans.py
import boto3
import json
import uuid

# Create an SNS client
sns = boto3.client('sns')

# Define the SNS topic ARN
topic_arn = 'arn:aws:sns:us-east-1:831926586385:updated_beans_sns.fifo'

# Define the message data
messages = [
    {
        "id": 1,
        "name": "AnyCompany coffee suppliers",
        "address": "123 Any Street",
        "city": "Any Town",
        "state": "WA",
        "email": "info@example.com",
        "phone": "555-555-0100",
        "beans": [
            {
                "id": 1,
                "name": "Arabica",
                "quantity": 100
            },
            {
                "id": 2,
                "name": "Robusta",
                "quantity": 50
            }
        ]
    },
    {
        "id": 2,
        "name": "Central Example Corp. coffee",
        "address": "100 Main Street",
        "city": "Nowhere",
        "state": "CO",
        "email": "info@example.net",
        "phone": "555-555-0101",
        "beans": [
            {
                "id": 1,
                "name": "Arabica",
                "quantity": 75
            },
            {
                "id": 3,
                "name": "Liberica",
                "quantity": 25
            }
        ]
    },
    {
        "id": 3,
        "name": "North East AnyCompany coffee suppliers",
        "address": "1001 Main Street",
        "city": "Any Town",
        "state": "NY",
        "email": "info@example.co",
        "phone": "555-555-0102",
        "beans": [
            {
                "id": 2,
                "name": "Robusta",
                "quantity": 120
            }
        ]
    }
]

# Publish each message to the SNS topic
for message in messages:
    # Convert the message to JSON
    message_json = json.dumps(message)
    
    # Generate a unique message group ID and deduplication ID
    message_group_id = 'beans-update'
    message_deduplication_id = str(uuid.uuid4())
    
    # Publish the message to the SNS topic
    response = sns.publish(
        TopicArn=topic_arn,
        Message=message_json,
        MessageGroupId=message_group_id,
        MessageDeduplicationId=message_deduplication_id
    )
    
    print(response)
```

2. Run the publisher script:

```bash
cd c:\Users\saihe\Projects\Final BAD\lab 10.1\resources\sqs-sns
python publish_beans.py
```

### Task 6: Testing the queue

1. Check if messages are in the queue:

```bash
aws sqs receive-message --queue-url https://sqs.us-east-1.amazonaws.com/831926586385/updated_beans.fifo --max-number-of-messages 10 --wait-time-seconds 20
```

### Task 7: Configuring the application to poll the queue

1. Create a Node.js consumer script:

```javascript:c:\Users\saihe\Projects\Final BAD\lab 10.1\resources\sqs-sns\consumer.js
const AWS = require('aws-sdk');
const mysql = require('mysql');
const { promisify } = require('util');

// Configure AWS SDK
AWS.config.update({ region: 'us-east-1' });

// Create SQS service object
const sqs = new AWS.SQS({ apiVersion: '2012-11-05' });

// Queue URL
const queueURL = "https://sqs.us-east-1.amazonaws.com/831926586385/updated_beans.fifo";

// Database connection
const connection = mysql.createConnection({
  host: 'your-aurora-endpoint',
  user: 'admin',
  password: 'coffee_beans_for_all',
  database: 'suppliers'
});

// Promisify the query method
const query = promisify(connection.query).bind(connection);

// Connect to the database
connection.connect();

// Parameters for receiving messages
const params = {
  QueueUrl: queueURL,
  MaxNumberOfMessages: 10,
  VisibilityTimeout: 30,
  WaitTimeSeconds: 20
};

// Function to process a message
async function processMessage(message) {
  try {
    // Parse the message body
    const body = JSON.parse(message.Body);
    const messageContent = JSON.parse(body.Message);
    
    console.log('Processing message:', messageContent);
    
    // Update supplier beans in the database
    const supplierId = messageContent.id;
    const beans = messageContent.beans;
    
    for (const bean of beans) {
      // Check if the bean exists for this supplier
      const checkSql = `SELECT * FROM supplier_beans WHERE supplier_id = ? AND bean_id = ?`;
      const checkResults = await query(checkSql, [supplierId, bean.id]);
      
      if (checkResults.length > 0) {
        // Update existing bean
        const updateSql = `UPDATE supplier_beans SET quantity = ? WHERE supplier_id = ? AND bean_id = ?`;
        await query(updateSql, [bean.quantity, supplierId, bean.id]);
        console.log(`Updated bean ${bean.name} for supplier ${supplierId}`);
      } else {
        // Insert new bean
        const insertSql = `INSERT INTO supplier_beans (supplier_id, bean_id, quantity) VALUES (?, ?, ?)`;
        await query(insertSql, [supplierId, bean.id, bean.quantity]);
        console.log(`Added bean ${bean.name} for supplier ${supplierId}`);
      }
    }
    
    // Delete the message from the queue
    const deleteParams = {
      QueueUrl: queueURL,
      ReceiptHandle: message.ReceiptHandle
    };
    
    await sqs.deleteMessage(deleteParams).promise();
    console.log('Message processed and deleted from queue');
    
    return true;
  } catch (error) {
    console.error('Error processing message:', error);
    return false;
  }
}

// Main polling function
async function pollQueue() {
  try {
    // Receive messages
    const data = await sqs.receiveMessage(params).promise();
    
    if (data.Messages) {
      console.log(`Received ${data.Messages.length} messages`);
      
      // Process each message
      for (const message of data.Messages) {
        await processMessage(message);
      }
    } else {
      console.log('No messages to process');
    }
  } catch (error) {
    console.error('Error polling queue:', error);
  }
  
  // Continue polling
  setTimeout(pollQueue, 5000);
}

// Start polling
console.log('Starting to poll for messages...');
pollQueue();

// Handle application shutdown
process.on('SIGINT', () => {
  console.log('Closing database connection...');
  connection.end();
  process.exit();
});
```

2. Install the required Node.js packages:

```bash
cd c:\Users\saihe\Projects\Final BAD\lab 10.1\resources\sqs-sns
npm init -y
npm install aws-sdk mysql
```

3. Run the consumer script:

```bash
node consumer.js
```

## Summary

You've now replicated the key components of all the labs:

1. **Lab 8.1**: Created Docker containers for the web application
2. **Lab 8.2**: Set up the infrastructure for running containers on managed services
3. **Lab 9.1**: Implemented caching with ElastiCache
4. **Lab 9.2**: Set up CloudFront for content delivery and security
5. **Lab 10.1**: Implemented a messaging system with SNS and SQS

The website should be accessible at: https://bad6520051final.s3.us-east-1.amazonaws.com

Note that some parts involving actual resource creation that would incur costs were outlined but not executed. You can implement these as needed based on your budget and requirements.
