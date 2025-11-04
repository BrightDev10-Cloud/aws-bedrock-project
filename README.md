# AWS AI Engineering Project

This readme guides you through deploying a generative AI application on AWS, covering infrastructure provisioning, security setup, data preparation, Bedrock knowledge base creation, Python integration, and final project submission. It is crafted for clarity and completeness, directly referencing the project rubric requirements.

---

## Project Overview

You will use Terraform to provision a VPC, Aurora Serverless database, and S3 bucket. You’ll then configure a Bedrock Knowledge Base, upload documents, and implement Python integrations to query the knowledge base and generate model responses.

---

## Prerequisites

Ensure the following tools are installed and configured before you begin:

- AWS CLI (configured with your AWS credentials)
- Terraform (version 1.0.0 or later)
- Python (version 3.10 or later)
- Git

---

## 1. Initial Setup

### Setting up a Python Virtual Environment

- Ensure you have Python 3.10 or later installed. Check your version with:

```bash
    python --version
```

- Install venv if it's not already installed:

- - On Ubuntu/Debian:

```bash
sudo apt-get install python3-venv
```

- - On macOS/Windows: It should be included with Python 3.3+
    Create a new directory for your project and navigate to it:

```bash
    mkdir aws-bedrock-project
    cd aws-bedrock-project
```

- Create a new virtual environment:

```bash
    python -m venv venv
```

- Activate the virtual environment:

- - On Windows:

```bash
    venv\Scripts\activate
```

- - On macOS/Linux:

```bash
    source venv/bin/activate
```

`Your prompt should change to indicate that the virtual environment is active.`

- Clone the repository and navigate to the root project directory:

```sh
    git clone <your-repo-url>
    cd <project-root-directory>
```

---

## 2. Infrastructure Deployment (Stack 1)

Deploy the foundational infrastructure: VPC, Aurora Serverless Database, S3 bucket.

1. Navigate to the Stack 1 directory from the project root directory:

   ```sh
   cd stack1
   ```

2. Initialize Terraform:

   ```sh
   terraform init
   ```

3. Review and adjust variables in `main.tf` as needed:

   - `region`: AWS region (e.g., "us-west-2")
   - `cidr`: VPC CIDR block
   - `cluster_identifier`: Aurora Serverless cluster name

4. Deploy the infrastructure:
   ```sh
   terraform apply
   ```
   - Review the plan and type "yes" to proceed.
   - After the Terraform deployment is complete, note the outputs, particularly the Aurora cluster endpoint.
   - Save a screenshot of your successful Terraform apply output in the `Screenshots/` folder.

---

## 3. Security Configuration

Store your Aurora database credentials using AWS Secrets Manager.

- Go to the AWS Secrets Manager console.
- Choose "Store a new secret" > "Credentials for RDS database."
- Enter your Aurora username and password.
- Select the RDS database created.
- Name and describe the secret.
- Save a screenshot of the secret manager interface showing your RDS secret.

---

## 4. Database Preparation

Prepare Aurora PostgreSQL for vector storage:

- Go to the Amazon RDS console > Query Editor.
- Connect to your Aurora database cluster.
- Execute the following SQL from `scripts/aurora_sql.sql`:
  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  ```
- Run:
  ```sql
  SELECT * FROM pg_extension;
  ```
- Save a screenshot showing the query results.

---

## 5. Bedrock Knowledge Base (Stack 2)

Provision Amazon Bedrock Knowledge Base using Terraform.

1. Navigate to Stack 2:

   ```sh
   cd ../stack2
   ```

2. Initialize Terraform:

   ```sh
   terraform init
   ```

3. Update variables in `main.tf` using outputs from Stack 1 (S3 bucket name, Aurora ARN).

4. Deploy the infrastructure:
   ```sh
   terraform apply
   ```
   - Review changes and confirm ("yes").
   - Save a screenshot of the deployed knowledge base interface.

---

## 6. S3 Data Upload & Sync

Upload PDF documents to S3 and sync with Bedrock KB.

1. Place your PDFs in `spec-sheets/`.

2. Update `scripts/upload_to_s3.py` with your actual bucket name.

3. Run the upload script:

   ```sh
   python scripts/upload_to_s3.py
   ```

4. Go to Amazon Bedrock console > knowledge base > select data source > click "Sync".
   - Save a screenshot of successful data sync from AWS Console.

---

## 7. Verification Queries

Verify vector storage and table creation in RDS Query Editor:

1. Check for pg_vector extension:

   ```sql
   SELECT * FROM pg_extension;
   ```

   - Save the results.

2. List Bedrock tables:
   ```sql
   SELECT table_schema || '.' || table_name AS show_tables
   FROM information_schema.tables
   WHERE table_type = 'BASE TABLE'
   AND table_schema = 'bedrock_integration';
   ```
   - Save the results.

---

## 8. Python Integration with Bedrock

Implement and validate Python functions in `bedrock_utils.py`:

- `query_knowledge_base(query)`: Use boto3 to interact with Bedrock KB
- `generate_response(prompt)`: Invoke language model via Bedrock
- `valid_prompt(prompt)`: Validate/categorize user prompts

Provide code snippets for each function and sample output for valid prompt filtering. For example:

- - ### 1. Setting up the Boto3 Client

Initialize the Boto3 client to interact with the Amazon Bedrock service at the top of your `bedrock_utils.py` file.  
Be sure to use the AWS region where your stacks were deployed.

```python
import boto3
import json

# Replace 'us-west-2' with your stack's AWS region.
bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-west-2')
```

---

- - ### 2. Querying the Knowledge Base

The `query_knowledge_base` function searches your Bedrock Knowledge Base for relevant information and generates a response.

```python
def query_knowledge_base(query, kb_id):
    """
    Queries the specified Bedrock Knowledge Base and returns the response.
    """
    try:
        response = bedrock_runtime.retrieve_and_generate(
            input={'text': query},
            retrievalConfiguration={
                'vectorSearchConfiguration': {
                    'numberOfResults': 3  # Number of source documents to retrieve
                }
            },
            knowledgeBaseId=kb_id
        )
        return response
    except Exception as e:
        print(f"Error querying knowledge base: {e}")
        return None
```

**Usage:**

- `kb_id`: Obtain your Knowledge Base ID from the AWS console or Terraform output.
- `query`: The user's question.
- The function returns a dictionary with the generated answer and its source documents.

**Example:**

```python
kb_id = "YOUR_KNOWLEDGE_BASE_ID"
user_query = "What are the key features of the BD850 bulldozer?"
kb_response = query_knowledge_base(user_query, kb_id)
if kb_response:
    answer = kb_response['output']['text']
    citations = kb_response['citations']
    print("Answer:", answer)
    for i, citation in enumerate(citations):
        retrieved_text = citation['retrievedReference']['text']
        print(f"Source {i+1}:", retrieved_text)
```

---

- - ### 3. Generating Model Responses

The `generate_response` function directly invokes a language model in Bedrock to generate a response to any given prompt.

```python
def generate_response(prompt):
    """
    Invokes a language model in Bedrock to generate a response.
    """
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

**Arguments:**

- `prompt`: Any natural language string.

**Notes:**

- `model_id` selects the language model.
- Adjust `model_id`, `temperature`, and `top_p` in the body as needed.

---

- - ### 4. Validating Prompts

The `valid_prompt` function filters user prompts to determine if they are specific to your knowledge base, improving response relevance.

```python
def valid_prompt(prompt):
    """
    Validates and categorizes a user prompt based on keywords.
    Returns True if the prompt is considered relevant, False otherwise.
    """
    # Keywords related to the heavy machinery spec sheets
    keywords = [
        "bulldozer", "bd850",
        "dump truck", "dt1000",
        "excavator", "x950",
        "forklift", "fl250",
        "crane", "mc750",
        "spec sheet", "capacity", "engine", "weight", "dimensions"
    ]

    lower_prompt = prompt.lower()
    for keyword in keywords:
        if keyword in lower_prompt:
            return True
    return False
```

**Sample Usage:**

```python
prompts = [
    "What is the operating weight of the x950 excavator?",
    "Tell me a joke.",
    "Can you provide the spec sheet for the forklift fl250?",
    "What is the weather like today?",
    "compare the engine specs of the bd850 and the dt1000"
]

for p in prompts:
    is_valid = valid_prompt(p)
    print(f"Prompt: '{p}'")
    print(f"Is valid: {is_valid}")
    if is_valid:
        print("Action: Querying knowledge base...")
    else:
        print("Action: Responding with a general-purpose answer...")
    print("-" * 20)
```

**Expected Output:**

```
Prompt: 'What is the operating weight of the x950 excavator?'
Is valid: True
Action: Querying knowledge base...
--------------------
Prompt: 'Tell me a joke.'
Is valid: False
Action: Responding with a general-purpose answer...
--------------------
Prompt: 'Can you provide the spec sheet for the forklift fl250?'
Is valid: True
Action: Querying knowledge base...
--------------------
Prompt: 'What is the weather like today?'
Is valid: False
Action: Responding with a general-purpose answer...
--------------------
Prompt: 'compare the engine specs of the bd850 and the dt1000'
Is valid: True
Action: Querying knowledge base...
--------------------
```

## 9. Model Parameters Explanation

Create `temperature_top_p_explanation.pdf` or `.docx`, explaining in 1–2 paragraphs:

- How `temperature` and `top_p` affect the creativity, randomness, and overall output of large language models.

---

## 10. Final Checklist & Submission

- Create a `Screenshots/` folder.
- Add all required screenshots.
- Ensure all code files are present.
- Include the temperature/top_p written explanation file.
- Zip the entire project directory as `LastName_FirstName_ProjectSubmission.zip`.

---

## Enhancing Response Transparency and Document Coverage in Bedrock Apps

### 1. Displaying the Source of Information

When you use the Bedrock `retrieve_and_generate` API, the response contains a `citations` list. Each citation includes a `retrievedReference` object providing the exact document excerpt and its source (usually an S3 URI).

#### Example: Citation Data Structure

```json
{
  "text": "The maximum travel speed of the BD850 bulldozer is 11 km/h.",
  "location": {
    "type": "S3",
    "s3Location": {
      "uri": "s3://your-bucket-name/spec-sheets/bulldozer-bd850-spec-sheet.pdf"
    }
  }
}
```

- **text**: Excerpt used for the answer.
- **location.s3Location.uri**: S3 path to the source document.

#### How to Display These Sources

After calling `query_knowledge_base`, parse the response to show the answer and its sources.  
Here’s an example in Python (`app.py`):

```python
# from bedrock_utils import query_knowledge_base, valid_prompt

if valid_prompt(user_query):
    kb_response = query_knowledge_base(user_query, kb_id)

    if kb_response:
        answer = kb_response['output']['text']
        citations = kb_response['citations']

        print("\nAnswer:")
        print(answer)
        print("\nSources:")

        if not citations:
            print("No sources found for this answer.")
        else:
            unique_sources = set()
            for citation in citations:
                retrieved_reference = citation.get('retrievedReference', {})
                location = retrieved_reference.get('location', {})
                s3_location = location.get('s3Location', {})
                uri = s3_location.get('uri')
                if uri:
                    unique_sources.add(uri)
            for i, source_uri in enumerate(unique_sources):
                file_name = source_uri.split('/')[-1]
                print(f"[{i+1}] {file_name}")
else:
    print("I can only answer questions about heavy machinery. Please ask a relevant question.")
```

- First prints the answer.
- Next, lists the unique file names of the documents used to support the answer.

---

Here’s your updated `app.py` file, enhanced to display the sources of your Bedrock Knowledge Base responses directly in the Streamlit chat UI. After the assistant gives the answer, it lists out the source document filenames (from S3) referenced in the answer. This makes your app more transparent and helps users verify responses.

---

# Updated `app.py` for Source Display

```python
import streamlit as st
import boto3
from botocore.exceptions import ClientError
import json
from bedrock_utils import query_knowledge_base, generate_response, valid_prompt

# Streamlit UI
st.title("Bedrock Chat Application")

# Sidebar for configurations
st.sidebar.header("Configuration")
model_id = st.sidebar.selectbox("Select LLM Model", ["anthropic.claude-3-haiku-20240307-v1:0", "anthropic.claude-3-5-sonnet-20240620-v1:0"])
kb_id = st.sidebar.text_input("Knowledge Base ID", "your-knowledge-base-id")
temperature = st.sidebar.select_slider("Temperature", [i/10 for i in range(0,11)],1)
top_p = st.sidebar.select_slider("Top_P", [i/1000 for i in range(0,1001)], 1)

# Initialize chat history
if "messages" not in st.session_state:
    st.session_state.messages = []

# Display chat messages
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# Chat input
if prompt := st.chat_input("What would you like to know?"):
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)

    sources_output = ""
    if valid_prompt(prompt, model_id):
        kb_response = query_knowledge_base(prompt, kb_id)
        if kb_response:
            answer = kb_response['output']['text']
            citations = kb_response.get('citations', [])

            sources_output += "\n\n**Sources:**\n"
            if not citations:
                sources_output += "No sources found for this answer."
            else:
                unique_sources = set()
                for citation in citations:
                    retrieved_reference = citation.get('retrievedReference', {})
                    location = retrieved_reference.get('location', {})
                    s3_location = location.get('s3Location', {})
                    uri = s3_location.get('uri')
                    if uri:
                        unique_sources.add(uri)
                for i, source_uri in enumerate(unique_sources):
                    file_name = source_uri.split('/')[-1]
                    sources_output += f"[{i+1}] {file_name}\n"

            response = f"{answer}{sources_output}"
        else:
            response = "No response from knowledge base."
    else:
        response = "I can only answer questions about heavy machinery. Please ask a relevant question."

    with st.chat_message("assistant"):
        st.markdown(response)
    st.session_state.messages.append({"role": "assistant", "content": response})
```

---

## How to Preview the Streamlit UI in Your Browser

1. **Ensure your Python environment has Streamlit installed:**

   ```sh
   pip install streamlit
   ```

2. **Navigate to your project directory (where `app.py` is located):**

   ```sh
   cd /path/to/your/project
   ```

3. **Run the Streamlit app from the terminal:**

   ```sh
   streamlit run app.py
   ```

4. **Your browser will open automatically (or visit the link output in your terminal, typically http://localhost:8501) to preview and interact with the chat app UI.**

---

### 2. Increasing Knowledge Base Coverage

The quality and usefulness of your responses improve with more relevant and authoritative documents in the knowledge base.

#### How to Add Documents

##### Step 1: Find More Source PDFs

- Use search engines for high-quality documents, such as:
  - "Caterpillar D9 bulldozer spec sheet pdf"
  - "Komatsu PC210 excavator manual pdf"
  - "Liebherr mobile crane LTM 11200 specs pdf"

##### Step 2: Add PDFs to Your Project

- Place downloaded PDF files in the `spec-sheets/` folder.

##### Step 3: Upload PDFs to S3

- Run the S3 upload script (ensure `bucket_name` is correct):

  ```sh
  python scripts/upload_s3.py
  ```

##### Step 4: Sync Your Bedrock Knowledge Base

- Go to the Amazon Bedrock console.
- Navigate to **Knowledge bases**.
- Select your KB and its S3 data source.
- Click **Sync**.

After sync completes, the new documents will be available for retrieval in responses, making your application more comprehensive and useful.

### TroubleShooting

- If you get an error like this

```bash
   1 [ERROR] vertex "provider[\"registry.terraform.io/hashicorp/aws\"]" error: timeout while waiting for
     plugin to start
   2 ...
   3 [ERROR] provider: plugin process exited: ... error="signal: killed"
```

This is what it means:

The log confirms that the AWS provider plugin is not just timing out; it's being forcefully "killed" by your
operating system. This is a classic sign of a security feature on your Mac stopping the program because it's
an executable file that was downloaded from the internet.

Even though you've disabled your VPN and may not have a third-party antivirus, macOS has its own built-in
security system (called Gatekeeper) that is likely causing this.

The Solution: Manually Approve the Plugin

We need to manually tell your Mac that you trust this specific plugin. Here’s how to do it:

Step 1: Open the Plugin Folder

Go to your terminal, make sure you are in the stack1 directory, and run this command to open the correct
folder in Finder:

1 open .terraform/providers/registry.terraform.io/hashicorp/aws/6.18.0/darwin_amd64/

Step 2: Manually Run the Plugin to Approve It

```bash
1.  In the Finder window that opens, you will see an executable file named terraform-provider-aws_v6.18.0_x5.
2.  Control-click (or right-click) on that file and select Open from the context menu.
3.  You will see a warning dialog from macOS saying it can't verify the developer. This is expected.
4.  Click the Open button on the dialog.
```

This action registers a security exception for the plugin. The plugin might open a terminal window and then
immediately close or show an error, which is perfectly fine. The important part is that you've now told
macOS to trust it.

Step 3: Run Terraform Apply Again

Go back to your terminal and run terraform apply one more time.

```bash
1 terraform apply
```

This should now work as expected.
If the above steps do not resolve the issue, you can try the following alternative method using the `xattr` command:

- Step 1: Navigate into the provider directory

  Copy and paste this cd (change directory) command into your terminal:

```bash
   1 cd .terraform/providers/registry.terraform.io/hashicorp/aws/6.18.0/darwin_amd64/
```

Step 2: Verify the file is there

Run the ls -l command to list the files. You should see terraform-provider-aws_v6.18.0_x5 in the output.

```bash
   1 ls -l
```

Step 3: Run the `xattr` command

Now that you are in the correct directory, the command is much simpler. Run this:

```bash
   1 xattr -d com.apple.quarantine terraform-provider-aws_v6.18.0_x5
```

Step 4: Return to the `stack1` directory

Navigate back to the stack1 directory where your main.tf file is located. The cd ../../../../../../ command
will take you back.

```bash
   1 cd ../../../../../../

```

Step 5: Run Terraform Apply

Now, try the apply command one more time.

```bash
   1 terraform apply
```
