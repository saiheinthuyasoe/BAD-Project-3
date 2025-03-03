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
