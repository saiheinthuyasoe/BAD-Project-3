**Yes! Here's a highly detailed, step-by-step breakdown of the entire process using VSCode:**

---

### **1. Prepare VSCode Environment**
**a. Install AWS Toolkit in VSCode**  
- Open VSCode → Extensions (`Ctrl+Shift+X`).  
- Search for **AWS Toolkit** → Install.  
- Configure AWS credentials:  
  - Open Command Palette (`Ctrl+Shift+P`) → **AWS: Connect to AWS**.  
  - Choose **AWS: Create Credentials Profile** → Enter your AWS access key ID and secret key.

**b. Download Lab Files**  
- In the VSCode terminal:  
  ```bash
  # Download code.zip
  wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCDEV-2-91558/10-lab-sqs/code.zip -P /home/ec2-user/environment

  # Extract files
  unzip code.zip -d ~/environment
  ```

**c. Run Setup Script**  
- Navigate to the `resources` directory:  
  ```bash
  cd ~/environment/resources
  ```
- Make the script executable and run it:  
  ```bash
  chmod +x ./setup.sh && ./setup.sh
  ```
  - When prompted, enter your **public IPv4 address** (get it from [whatismyipaddress.com](https://whatismyipaddress.com)).

---

### **2. Create Dead-Letter Queue (DLQ)**
**a. Create DLQ**  
- Navigate to the `sqs-sns` folder:  
  ```bash
  cd ~/environment/resources/sqs-sns
  ```
- Create the DLQ:  
  ```bash
  aws sqs create-queue --queue-name DeadLetterQueue.fifo --attributes file://create-dlq.json
  ```
  - **Output Example:**  
    ```json
    {
      "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789012/DeadLetterQueue.fifo"
    }
    ```
  - Save the `QueueUrl` for later.

**b. Update DLQ Policy**  
- Open `dlq-policy.json` in VSCode.  
- Replace all `<FMI_1>` placeholders with your **AWS account ID** (find it via `aws sts get-caller-identity`).  
- Apply the policy:  
  ```bash
  aws sqs set-queue-attributes --queue-url "<DLQ_QUEUE_URL>" --attributes file://dlq-policy.json
  ```

---

### **3. Create Main SQS Queue (`updated_beans.fifo`)**
**a. Create the Queue**  
- Edit `create-beans-queue.json` in VSCode:  
  - Replace `<FMI_1>` with your AWS account ID in the `RedrivePolicy`.  
- Run:  
  ```bash
  aws sqs create-queue --queue-name updated_beans.fifo --attributes file://create-beans-queue.json
  ```
  - Save the `QueueUrl` from the output.

**b. Update Queue Policy**  
- Open `beans-queue-policy.json` in VSCode.  
- Replace all `<FMI_1>` with your AWS account ID.  
- Apply the policy:  
  ```bash
  aws sqs set-queue-attributes --queue-url "<MAIN_QUEUE_URL>" --attributes file://beans-queue-policy.json
  ```

---

### **4. Create SNS Topic (`updated_beans_sns.fifo`)**
**a. Create the Topic**  
- Run:  
  ```bash
  aws sns create-topic --name updated_beans_sns.fifo --attributes DisplayName="updated beans sns",ContentBasedDeduplication="true",FifoTopic="true"
  ```
  - **Output Example:**  
    ```json
    {
      "TopicArn": "arn:aws:sns:us-east-1:123456789012:updated_beans_sns.fifo"
    }
    ```
  - Save the `TopicArn`.

**b. Update Topic Policy**  
- Open `beans-topic-policy.json` in VSCode.  
- Replace all `<FMI_2>` with your AWS account ID.  
- Apply the policy:  
  ```bash
  aws sns set-topic-attributes --cli-input-json file://beans-topic-policy.json
  ```

---

### **5. Link SNS to SQS**
- Subscribe the SQS queue to the SNS topic:  
  ```bash
  aws sns subscribe --topic-arn "<SNS_TOPIC_ARN>" --protocol sqs --notification-endpoint "<SQS_QUEUE_ARN>"
  ```
  - Replace:  
    - `<SNS_TOPIC_ARN>`: ARN from Step 4a.  
    - `<SQS_QUEUE_ARN>`: ARN of `updated_beans.fifo` (get it via `aws sqs get-queue-attributes --queue-url "<QUEUE_URL>" --attribute-names QueueArn`).

---

### **6. Test the Messaging System**
**a. Modify Python Publisher**  
- Open `send_beans_update.py` in VSCode.  
- Replace `<FMI_1>` with your AWS account ID in the `sns_topic` variable.  
  ```python
  sns_topic = 'arn:aws:sns:us-east-1:123456789012:updated_beans_sns.fifo'
  ```

**b. Send Test Messages**  
- Run:  
  ```bash
  cd ~/environment/resources/sqs-sns
  python3 send_beans_update.py beans_update_1.txt
  ```
- Check messages in the AWS SQS Console:  
  - Navigate to [SQS Console](https://console.aws.amazon.com/sqs) → Open `updated_beans.fifo` → **Poll for messages**.

---

### **7. Configure the Coffee Suppliers App**
**a. Add `about.html`**  
- Navigate to the app’s public folder:  
  ```bash
  cd ~/environment/resources/codebase_partner/app/public
  ```
- Create `about.html`:  
  ```bash
  touch about.html
  ```
- Add content (example):  
  ```html
  <!DOCTYPE html>
  <html>
  <head>
      <title>About Our Team</title>
  </head>
  <body>
      <h1>Team Members</h1>
      <ul>
          <li>John Doe - Lead Developer</li>
          <li>Jane Smith - DevOps Engineer</li>
      </ul>
  </body>
  </html>
  ```

**b. Update Elastic Beanstalk Environment**  
- In the [Elastic Beanstalk Console](https://console.aws.amazon.com/elasticbeanstalk):  
  1. Select your environment → **Configuration** → **Software** → **Edit**.  
  2. Add environment variables:  
     - **SQS_ENDPOINT**: `https://sqs.us-east-1.amazonaws.com/<ACCOUNT_ID>/updated_beans.fifo`  
     - **SQS_REGION**: `us-east-1`  
  3. **Apply** changes (deploys the app automatically).

---

### **8. Grant Access to `mchayapol@gmail.com`**
**a. Create IAM Policy**  
- Create `vpc-read-only-policy.json`:  
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*"
    }]
  }
  ```
- Create the policy:  
  ```bash
  aws iam create-policy --policy-name VPCReadOnly --policy-document file://vpc-read-only-policy.json
  ```

**b. Create IAM User**  
- Create the user:  
  ```bash
  aws iam create-user --user-name mchayapol
  ```
- Attach the policy:  
  ```bash
  aws iam attach-user-policy --user-name mchayapol --policy-arn "arn:aws:iam::<ACCOUNT_ID>:policy/VPCReadOnly"
  ```
- Generate access keys:  
  ```bash
  aws iam create-access-key --user-name mchayapol
  ```
- Share the **Access Key ID** and **Secret Access Key** with `mchayapol@gmail.com`.

---

### **9. Verify Everything**
**a. Test the App**  
- Visit the Elastic Beanstalk URL (e.g., `https://myenv.eba-xyz.us-east-1.elasticbeanstalk.com/beans`).  
- Check `/about.html` to confirm the team page loads.

**b. Check Dead-Letter Queue**  
- After sending invalid messages (e.g., `beans_update_2.txt`), confirm they appear in `DeadLetterQueue.fifo`.

**c. Validate IAM Access**  
- Ask `mchayapol@gmail.com` to run:  
  ```bash
  aws ec2 describe-vpcs --region us-east-1
  ```
  (Should return VPC details without errors.)

---

### **Key Files & Commands Recap**
| **Action**                | **File/Command**                                                                 |
|---------------------------|----------------------------------------------------------------------------------|
| DLQ Policy                | `dlq-policy.json` (replace account ID)                                           |
| SQS Main Queue Policy     | `beans-queue-policy.json` (replace account ID)                                   |
| SNS Topic Policy          | `beans-topic-policy.json` (replace account ID)                                   |
| Python Publisher          | `send_beans_update.py` (update SNS topic ARN)                                   |
| Deploy App                | `eb deploy` (from `codebase_partner` directory)                                  |

**Troubleshooting Tips**  
- Use `aws logs tail /aws/elasticbeanstalk/MyEnv/var/log/web.stdout.log` to view app logs.  
- Always validate JSON policies with [IAM Policy Validator](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_policy-validator.html).
