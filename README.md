# AWS AI Engineering Project

This guide provides instructions for deploying a generative AI application on AWS. You will use Terraform Cloud to build the infrastructure, create a knowledge base in Amazon Bedrock, and build a Python application to query the knowledge base and generate responses.

## Prerequisites

Before you begin, ensure you have the following installed and configured:

*   **AWS CLI:** Configured with your AWS credentials for your target account.
*   **Terraform:** Version 1.0.0 or later.
*   **Python:** Version 3.8 or later.
*   **Git & GitHub Account:** To store your code and connect to Terraform Cloud.

---

## Step 1: Initial Setup

Clone the project repository to your local machine.

```bash
git clone <your-repo-url>
cd <project-root-directory>
```

---

## Step 2: Infrastructure Deployment with Terraform Cloud

We will use the VCS-driven (Version Control System) workflow in Terraform Cloud. This is the standard practice and bypasses local machine issues by running Terraform in a stable, managed environment.

### 2.1: Create Terraform Cloud Workspaces

You need two separate workspaces to manage the two stacks of your project.

1.  **Sign up:** Create a free [Terraform Cloud](https://app.terraform.io/signup/account) account and create an organization.
2.  **Connect to VCS:** Connect your Terraform Cloud organization to your Git provider (e.g., GitHub).

3.  **Create Workspace for Stack 1:**
    *   Create a new workspace named `aws-bedrock-project` (or a similar name).
    *   Link it to your project's GitHub repository.
    *   In the workspace settings (**Settings -> General**), set the **Terraform Working Directory** to: `stack1`

4.  **Create Workspace for Stack 2:**
    *   Create another new workspace named `aws-bedrock-project-stack2`.
    *   Link it to the **same** GitHub repository.
    *   In this workspace's settings, set the **Terraform Working Directory** to: `stack2`

### 2.2: Configure Workspace Variables

In **both** workspaces (`aws-bedrock-project` and `aws-bedrock-project-stack2`), you must add your AWS credentials.

1.  Navigate to your workspace's **Variables** tab.
2.  Add the following under the **Environment Variables** section. Mark the secret values as **"Sensitive"**.

| Key | Value |
| :--- | :--- |
| `AWS_ACCESS_KEY_ID` | Your AWS Access Key ID |
| `AWS_SECRET_ACCESS_KEY` | Your AWS Secret Access Key |
| `AWS_SESSION_TOKEN` | Your AWS Session Token (if using temporary credentials) |

### 2.3: Deploy the Infrastructure

The deployment is a two-step process managed entirely through the Terraform Cloud UI.

1.  **Push Your Code:** Commit and push your latest code changes to your Git repository. This automatically triggers new plans in both of your workspaces.

2.  **Deploy Stack 1:**
    *   Go to your `aws-bedrock-project` (stack1) workspace in Terraform Cloud.
    *   Wait for the plan to finish. Review the proposed changes.
    *   Click **"Confirm & Apply"** and wait for the apply to complete successfully.

3.  **Deploy Stack 2:**
    *   **After Stack 1 is complete**, go to your `aws-bedrock-project-stack2` workspace.
    *   A plan should have already run. It will show that it is reading data from the `stack1` workspace.
    *   Review the plan and click **"Confirm & Apply"**.

*Save screenshots of your successful Terraform apply logs from both workspaces.*

---

## Step 3: Security Configuration

Store your Aurora database credentials using AWS Secrets Manager.

1.  Go to the **AWS Secrets Manager** console.
2.  Choose **"Store a new secret"** > **"Credentials for RDS database."**
3.  Enter your Aurora username and password.
4.  Select the RDS database created in Stack 1.
5.  Name and describe the secret.

*Save a screenshot of the secret manager interface showing your created RDS secret.*

---

## Step 4: Database Preparation

Prepare Aurora PostgreSQL for vector storage.

1.  Go to the **Amazon RDS console > Query Editor**.
2.  Connect to your Aurora database cluster.
3.  Execute the following SQL from `scripts/aurora_sql.sql`:
    ```sql
    CREATE EXTENSION IF NOT EXISTS vector;
    ```
4.  Run `SELECT * FROM pg_extension;` to verify.

*Save a screenshot showing the query results.*

---

## Step 5: S3 Data Upload & Sync

Upload PDF documents to S3 and sync with your Bedrock Knowledge Base.

1.  Place your PDF files in `spec-sheets/`.
2.  Update `scripts/upload_s3.py` with your actual S3 bucket name (you can find this in the outputs of your `stack1` Terraform run).
3.  Run the upload script:
    ```bash
    python scripts/upload_to_s3.py
    ```
4.  In the **Amazon Bedrock console**, navigate to your knowledge base, select the data source, and click **"Sync"**.

*Save a screenshot of the successful data sync from the AWS Console.*

---

## Step 6: Python Integration and Application

Implement the functions in `bedrock_utils.py` and run the Streamlit application.

### 6.1: Implement Utility Functions

Complete the functions in `bedrock_utils.py` as described in the file's comments. This includes `query_knowledge_base`, `generate_response`, and `valid_prompt`.

### 6.2: Run the Streamlit App

1.  Ensure you have Streamlit installed:
    ```bash
    pip install streamlit
    ```
2.  Navigate to the directory containing `app.py` and run:
    ```bash
    streamlit run app.py
    ```
3.  Your browser will open to the chat application. Interact with it to test your full solution.

---

## Step 7: Model Parameters Explanation

Create a document named `temperature_top_p_explanation.pdf` or `.docx`. In 1–2 paragraphs, explain how `temperature` and `top_p` affect the creativity, randomness, and diversity of large language model responses.

---

## Step 8: Final Checklist & Submission

*   Create a `Screenshots/` folder and add all required screenshots.
*   Ensure all code files are present and complete.
*   Include the `temperature_top_p_explanation` document.
*   Zip the entire project directory as `LastName_FirstName_ProjectSubmission.zip`.