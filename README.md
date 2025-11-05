# Guide: Running Terraform in a Multi-Account AWS Setup

This guide provides instructions for setting up a secure, multi-account environment to manage AWS resources using Terraform from a dedicated EC2 instance.

The architecture involves:
*   **Management Account (Account A):** An AWS account that hosts the EC2 instance where Terraform commands will be executed.
*   **Target Account (Account B):** An AWS account (e.g., a federated educational account) where the AWS resources will be created and managed.

This setup uses IAM Roles for cross-account access, which is the most secure and recommended method.

## Prerequisites

1.  Two AWS Accounts (a "Management Account" and a "Target Account").
2.  Administrator access to both accounts to create IAM roles, policies, S3 buckets, and EC2 instances.
3.  Your Terraform project files ready.
4.  Replace `ACCOUNT_A_ID` and `ACCOUNT_B_ID` in the examples below with your actual AWS Account IDs.

---

## Step 1: Configure the Target Account (Account B)

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
    *   Navigate to **S3** and create a new bucket (e.g., `my-terraform-project-state-backend`).
    *   Enable **Block all public access** (default).
    *   Enable **Bucket Versioning** to protect against state file corruption or deletion.
2.  **Create DynamoDB Table:**
    *   Navigate to **DynamoDB -> Tables -> Create table**.
    *   Set the "Table name" to `my-terraform-lock-table`.
    *   Set the "Partition key" to `LockID` (Type: String).
    *   Select **On-demand** capacity and create the table.

### 2.2. Create the IAM Role for the EC2 Instance

1.  Navigate to **IAM -> Roles -> Create role**.
2.  For "Trusted entity type", select **AWS service**.
3.  For "Use case", choose **EC2** and click **Next**.
4.  On the "Add permissions" screen, we will create two custom policies. Click **Next** for now, give the role a name (e.g., `RoleForEC2InAccountA`), and create it.

### 2.3. Create and Attach Policies to the EC2 Role

1.  **Create AssumeRole Policy:**
    *   Navigate to **IAM -> Policies -> Create policy**.
    *   Switch to the **JSON** tab and paste the following, replacing `ACCOUNT_B_ID`:
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
    *   Name it `AllowAssumeRoleInAccountB` and create it.

2.  **Create State Backend Policy:**
    *   Create another policy named `TerraformStateBackendAccess` with the following JSON, replacing the bucket and table names with your own:
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
                    "Resource": ["arn:aws:s3:::my-terraform-project-state-backend/terraform.tfstate"]
                },
                {
                    "Effect": "Allow",
                    "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:DeleteItem"],
                    "Resource": ["arn:aws:dynamodb:*:*:table/my-terraform-lock-table"]
                }
            ]
        }
        ```

3.  **Attach Policies:**
    *   Go to the `RoleForEC2InAccountA` role you created.
    *   Under "Permissions", click **Add permissions -> Attach policies**.
    *   Search for and attach `AllowAssumeRoleInAccountB` and `TerraformStateBackendAccess`.

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

```terraform
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

```terraform
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

You have now successfully configured a multi-account Terraform workflow.