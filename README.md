# AWS AI Engineering Project

This guide provides instructions for deploying a generative AI application on AWS. You will use Terraform to build the infrastructure, create a knowledge base in Amazon Bedrock, and build a Python application to query the knowledge base and generate responses.

## Prerequisites

Before you begin, ensure you have the following installed and configured:

- **AWS CLI:** Configured with your AWS credentials.
- **Terraform:** Version 1.0.0 or later.
- **Python:** Version 3.8 or later.
- **Git:** To clone the repository.

---

## Step 1: Initial Setup

Clone the project repository to your local machine.

```bash
git clone <your-repo-url>
cd <project-root-directory>
```

---

## Step 2: Infrastructure Deployment with Terraform

Due to some limitations on my local machine (e.g., low RAM, network restrictions), we will use Terraform in a Multi-Account AWS Setup to deploy the infrastructure. This approach runs Terraform in a stable, managed environment from a dedicated EC2 instance.

### 2:1 Prerequisites

1.  Two AWS Accounts (a "Management Account" and a "Target Account").
2.  Administrator access to both accounts to create IAM roles, policies, S3 buckets, and EC2 instances.
3.  Your Terraform project files ready.
4.  Replace `ACCOUNT_A_ID` and `ACCOUNT_B_ID` in the examples below with your actual AWS Account IDs.

---

### Step 2:2 Configure the Target Account (Account B)

In this step, we create an IAM role that Terraform will assume to get the permissions it needs to manage resources in this account.

### 1.1. Create an IAM Policy with Permissions

First, create a policy that grants permissions to create the specific resources in your Terraform project.

1.  Navigate to **IAM -> Policies -> Create policy**.
2.  Switch to the **JSON** tab and paste a policy. For this project, you'll need permissions for VPC, S3, and Aurora. **Note:** For a real-world project, always follow the principle of least privilege.
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "ec2:*",
            "s3:*",
            "rds:*",
            "iam:GetPolicy",
            "iam:GetPolicyVersion",
            "iam:GetRole"
          ],
          "Resource": "*"
        }
      ]
    }
    ```
3.  Click **Next**, give the policy a name (e.g., `TerraformProjectPermissions`), and create it.

### 1.2. Create the Cross-Account IAM Role

1.  Navigate to **IAM -> Roles -> Create role**.
2.  For "Trusted entity type", select **AWS account**.
3.  Choose **Another AWS account** and enter the ID of your **Management Account (Account A)**.
4.  Click **Next**.
5.  On the "Add permissions" screen, select the `TerraformProjectPermissions` policy you just created.
6.  Click **Next**.
7.  Give the role a name (e.g., `TerraformExecutionRole`) and create it.

### 1.3. (Verification) Check the Trust Relationship

1.  Go to the `TerraformExecutionRole` you just created.
2.  Click on the **Trust relationships** tab. It should have a policy that allows `sts:AssumeRole` from the root of `Account A`.

---

## Step 2: Configure the Management Account (Account A)

Now, we'll set up the EC2 instance and the permissions it needs to assume the role in Account B.

### 2.1. Create S3 Backend and DynamoDB Table for State Locking

It is critical to store your Terraform state remotely.

1.  **Create S3 Bucket:**
    - Navigate to **S3** and create a new bucket (e.g., `my-terraform-project-state-backend`).
    - Enable **Block all public access** (default).
    - Enable **Bucket Versioning** to protect against state file corruption or deletion.
2.  **Create DynamoDB Table:**
    - Navigate to **DynamoDB -> Tables -> Create table**.
    - Set the "Table name" to `my-terraform-lock-table`.
    - Set the "Partition key" to `LockID` (Type: String).
    - Select **On-demand** capacity and create the table.

### 2.2. Create the IAM Role for the EC2 Instance

1.  Navigate to **IAM -> Roles -> Create role**.
2.  For "Trusted entity type", select **AWS service**.
3.  For "Use case", choose **EC2** and click **Next**.
4.  On the "Add permissions" screen, we will create two custom policies. Click **Next** for now, give the role a name (e.g., `RoleForEC2InAccountA`), and create it.

### 2.3. Create and Attach Policies to the EC2 Role

1.  **Create AssumeRole Policy:**

    - Navigate to **IAM -> Policies -> Create policy**.
    - Switch to the **JSON** tab and paste the following, replacing `ACCOUNT_B_ID`:
      ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::ACCOUNT_B_ID:role/TerraformExecutionRole"
          }
        ]
      }
      ```
    - Name it `AllowAssumeRoleInAccountB` and create it.

2.  **Create State Backend Policy:**

    - Create another policy named `TerraformStateBackendAccess` with the following JSON, replacing the bucket and table names with your own:
      ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": ["s3:ListBucket"],
            "Resource": ["arn:aws:s3:::my-terraform-project-state-backend"]
          },
          {
            "Effect": "Allow",
            "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
            "Resource": [
              "arn:aws:s3:::my-terraform-project-state-backend/terraform.tfstate"
            ]
          },
          {
            "Effect": "Allow",
            "Action": [
              "dynamodb:GetItem",
              "dynamodb:PutItem",
              "dynamodb:DeleteItem"
            ],
            "Resource": ["arn:aws:dynamodb:*:*:table/my-terraform-lock-table"]
          }
        ]
      }
      ```

3.  **Attach Policies:**
    - Go to the `RoleForEC2InAccountA` role you created.
    - Under "Permissions", click **Add permissions -> Attach policies**.
    - Search for and attach `AllowAssumeRoleInAccountB` and `TerraformStateBackendAccess`.

### 2.4. Launch and Configure the EC2 Instance

1.  Navigate to **EC2 -> Instances -> Launch instances**.
2.  Choose an Amazon Machine Image (AMI), such as **Amazon Linux 2**.
3.  Choose an instance type (e.g., `t2.micro`).
4.  **Network Settings:** Ensure the instance has outbound internet access (e.g., by enabling "Auto-assign public IP" in a default VPC).
5.  **IAM Instance Profile:** In "Advanced details", select the `RoleForEC2InAccountA` as the IAM instance profile.
6.  Launch the instance.
7.  After launching, install Terraform on the EC2 instance.

---

## Step 3: Configure the Terraform Project

Update your Terraform code to use the remote state backend and the assume role configuration.

### 3.1. Add the Backend Configuration

In your main Terraform file (e.g., `main.tf`), add the following block:

```bash
    terraform
        terraform {
        backend "s3" {
            bucket         = "my-terraform-project-state-backend"
            key            = "terraform.tfstate"
            region         = "us-east-1" # The region of your S3 bucket
            dynamodb_table = "my-terraform-lock-table"
        }
    }
```

### 3.2. Add the Provider Configuration

Update your AWS provider block to assume the role in Account B.

```bash
    terraform
        provider "aws" {
        region = "us-west-1" // Your TARGET region for resources

        assume_role {
            role_arn = "arn:aws:iam::ACCOUNT_B_ID:role/TerraformExecutionRole"
            }
        }
```

---

## Step 4: Execute Terraform

1.  Connect to your EC2 instance in Account A using SSH.
2.  Upload your Terraform project files to the instance or clone them from a Git repository.
3.  Navigate into your project directory.
4.  Run `terraform init`. Terraform will configure the S3 backend and download the necessary providers.
5.  Run `terraform plan`. Terraform will assume the role in Account B and show you the changes to be made.
6.  Run `terraform apply` to create the resources in Account B.

`# You have now successfully configured a multi-account Terraform workflow.`

_Save a screenshot of your successful Terraform apply output for both stacks._

---

## Step 3: Security Configuration

Create a secret in AWS Secrets Manager to store your RDS database credentials.

1.  Go to the **AWS Secrets Manager** console.
2.  Click **"Store a new secret"**.
3.  Select **"Credentials for RDS database"**.
4.  Enter the username and password for your Aurora database.
5.  Select the RDS database you created in Stack 1.
6.  Give the secret a name and description, then store it.

_Save a screenshot of the secret manager interface showing your created RDS secret._

---

## Step 4: Database Preparation

Connect to your Aurora PostgreSQL database to enable the `vector` extension.

1.  Go to the **Amazon RDS console -> Query Editor**.
2.  Connect to your Aurora database cluster.
3.  Execute the following SQL query:
    ```sql
    CREATE EXTENSION IF NOT EXISTS vector;
    ```
4.  Verify the extension is enabled:
    `sql
SELECT * FROM pg_extension;
`
    _Save a screenshot showing the successful result of the verification query._

---

## Step 5: S3 Data Upload & Sync

Upload your documents to the S3 bucket and sync them with your Bedrock Knowledge Base.

1.  Place your PDF files in the `spec-sheets/` directory.
2.  Open the `scripts/upload_s3.py` file and update the `bucket_name` variable with the name of the S3 bucket you created in Stack 1.
3.  From the root of the project directory, run the upload script:
    ```bash
    python scripts/upload_s3.py
    ```
4.  **Sync the Data Source:**
    - Go to the **Amazon Bedrock** console and navigate to your knowledge base.
    - Select your data source and click **"Sync"**.

_Save a screenshot of the successful data sync confirmation from the AWS Console._

---

## Step 6: Python Integration (`bedrock_utils.py`)

Complete the Python functions in `bedrock_utils.py` to interact with the Bedrock service.

### 1. Setting up the Boto3 Client

Initialize the Boto3 client at the top of your `bedrock_utils.py` file.

```python
import boto3
import json

# Replace 'us-west-2' with the AWS region where you deployed your stacks.
bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-west-2')
```

### 2. Querying the Knowledge Base

This function searches your Bedrock Knowledge Base.

```python
def query_knowledge_base(query, kb_id):
    try:
        response = bedrock_runtime.retrieve_and_generate(
            input={'text': query},
            retrievalConfiguration={
                'vectorSearchConfiguration': {
                    'numberOfResults': 3
                }
            },
            knowledgeBaseId=kb_id
        )
        return response
    except Exception as e:
        print(f"Error querying knowledge base: {e}")
        return None
```

### 3. Generating Model Responses

This function directly invokes a language model in Bedrock.

```python
def generate_response(prompt):
    # Example uses Anthropic's Claude 3 Sonnet model
    model_id = "anthropic.claude-3-sonnet-20240229-v1:0"

    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [
            {
                "role": "user",
                "content": [{"type": "text", "text": prompt}],
            }
        ],
        "temperature": 0.7,
        "top_p": 0.9,
    })

    try:
        response = bedrock_runtime.invoke_model(
            body=body,
            modelId=model_id,
            accept='application/json',
            contentType='application/json'
        )
        response_body = json.loads(response.get('body').read())
        return response_body['content'][0]['text']
    except Exception as e:
        print(f"Error generating response: {e}")
        return None
```

### 4. Validating Prompts

This function filters user prompts to determine if they are relevant to the knowledge base.

```python
def valid_prompt(prompt):
    keywords = [
        "bulldozer", "bd850", "dump truck", "dt1000", "excavator", "x950",
        "forklift", "fl250", "crane", "mc750", "spec sheet", "capacity",
        "engine", "weight", "dimensions"
    ]
    lower_prompt = prompt.lower()
    for keyword in keywords:
        if keyword in lower_prompt:
            return True
    return False
```

---

## Step 7: Running the Streamlit Application

Your `app.py` file has been updated to provide a chat interface and display sources for the answers.

1.  **Install Streamlit:**
    ```bash
    pip install streamlit
    ```
2.  **Run the app:**
    Navigate to the directory containing `app.py` and run:
    ```bash
    streamlit run app.py
    ```
3.  Your browser will open to the application (typically `http://localhost:8501`). Interact with the chat UI to test your application.

---

## Step 8: Model Parameters Explanation

Create a document named `temperature_top_p_explanation.pdf` or `.docx`. In 1-2 paragraphs, explain how the `temperature` and `top_p` parameters affect the creativity, randomness, and diversity of a large language model's output.

---

## Step 9: Final Checklist & Submission

1.  Create a `Screenshots/` folder and add all required screenshots.
2.  Ensure all code files (`.tf`, `.py`, etc.) are complete.
3.  Include your written explanation of `temperature` and `top_p`.
4.  Zip the entire project directory as `LastName_FirstName_ProjectSubmission.zip` for submission.
