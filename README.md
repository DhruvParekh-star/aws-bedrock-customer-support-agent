# AWS Bedrock Customer Support Agent

An AWS-based customer support agent that uses **Amazon Bedrock** to classify customer requests, route them through specialized workflows, answer FAQ questions, and create structured bug reports using an integrated tool workflow.

The project demonstrates an end-to-end generative AI customer-support architecture using **Amazon Bedrock Flows, Amazon Bedrock AgentCore, AWS Lambda, Amazon DynamoDB, AWS CloudFormation, Python, and an evaluation harness**.

---

## Overview

Customer-support applications often need to handle different types of requests through different workflows.

For example:

- A customer asking about a return policy needs an informational response.
- A customer reporting a broken checkout button needs a bug-report workflow.
- A request outside the application's supported domain should not receive an invented answer.

This project addresses that problem by introducing a **classification and routing layer** before the customer request reaches its final processing path.

The high-level workflow is:

```text
Customer Message
       |
       v
Request Classifier
       |
       v
Request Type Condition
       |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
      BUG                 FAQ             Other Request
       |                   |                   |
       v                   v                   v
Bug Report Workflow    FAQ Response       Other Response
       |
       v
Bug Report Tool
       |
       v
Amazon DynamoDB
       |
       v
Customer Response
```

## Project Highlights
- **Amazon Bedrock-powered customer-support agent**
- **Customer request classification**
- **Conditional request routing using Amazon Bedrock Flows**
- **Dedicated BUG workflow**
- **Dedicated FAQ workflow**
- **Handling of unsupported/other requests**
- **Multi-step bug-report information collection**
- **Automated bug-ticket creation**
- **Bug-ticket persistence in Amazon DynamoDB**
- **Amazon Bedrock AgentCore integration**
- **AWS Lambda-based tool integration**
- **AWS CloudFormation infrastructure**
- **Evaluation dataset and evaluation harness**
- **1.00 correctness score across 6 evaluated prompts**

---

## Architecture

The project uses an Amazon Bedrock Flow to classify and route incoming customer requests.
The main flow contains the following components:

```text
                         +------------------+
                         | Customer Message |
                         +--------+---------+
                                  |
                                  v
                       +---------------------+
                       | RequestClassifier   |
                       |   Bedrock Prompt    |
                       +----------+----------+
                                  |
                                  v
                       +---------------------+
                       | RequestTypeCondition|
                       | Conditional Router  |
                       +----------+----------+
                                  |
              +-------------------+-------------------+
              |                   |                   |
              v                   v                   v
        +-----------+       +-----------+       +-------------+
        |    BUG    |       |    FAQ    |       |    OTHER    |
        +-----+-----+       +-----+-----+       +------+------+
              |                   |                    |
              v                   v                    v
   +-------------------+   +-------------+   +----------------+
   | FormatBugResponse |   |  FAQAnswer  |   | OtherRequest   |
   +---------+---------+   +------+------+   +-------+--------+
             |                    |                    |
             v                    v                    v
   +-------------------+     FAQ Response      Other Response
   |  BugReportAgent   |
   +---------+---------+
             |
             v
   +-------------------+
   | Bug Report Tool   |
   +---------+---------+
             |
             v
   +-------------------+
   | Amazon DynamoDB   |
   +-------------------+
             |
             v
       Ticket Created
```

---

## Classification and Routing

Classification is the central routing mechanism of the application.
The `RequestClassifier` prompt analyzes the incoming customer message and determines the appropriate request category.
The output is then passed to the `RequestTypeCondition` node.

The routing logic is:
- `BUG` -> `FormatBugResponse`
- `FAQ` -> `FAQAnswer`
- `OTHER` -> `OtherRequest`

The condition node uses expressions equivalent to:
```text
condition == "BUG"
```
and:
```text
condition == "FAQ"
```
If neither condition is satisfied, the request is routed to:
```text
OtherRequest
```
This architecture separates classification from request processing, allowing individual workflows to be developed and maintained independently.

---

## Supported Request Types

### 1. BUG
The `BUG` category is used when a customer reports that something is not working as expected.
- **Example:** *"The checkout button is not working."*
- The request is routed to the bug-report workflow.

### 2. FAQ
The `FAQ` category is used for questions that can be answered using the online-shop information provided to the agent.
- **Examples:**
  - *"What is your return policy?"*
  - *"How do I track my order?"*
- The agent uses the available FAQ information to generate the response.

### 3. Other Request
Requests that do not belong to the supported `BUG` or `FAQ` categories are routed through the `OtherRequest` path.
- **Example:** *"Can you recommend a good movie to watch tonight?"*
- Rather than treating this as an online-shop question, the system recognizes that it falls outside the supported customer-support categories.

---

## Bug Report Workflow

The bug-report workflow is designed as a multi-step conversational process.
The agent does not immediately create a ticket when a customer reports a problem.
Instead, it collects the information required to create a useful bug report.

The workflow can collect:
- Bug description
- Steps to reproduce
- Browser
- Operating system
- Device
- Environment information

The process can be represented as:

```text
Customer reports problem
          |
          v
Collect bug description
          |
          v
Collect reproduction steps
          |
          v
Collect environment information
          |
          v
Call bug-report tool
          |
          v
Create ticket
          |
          v
Store ticket in DynamoDB
          |
          v
Return ticket ID
```

### Example Bug Report Conversation

A sample interaction is:

> **you>** The checkout button is not working.  
> **bot>** Please provide the steps to reproduce the issue with the checkout button.  
> **you>** I added an item to my cart, went to checkout, and clicked Place Order, but nothing happened.  
> **bot>** Please provide the browser, operating system, and device you are using.  
> **you>** I'm using Windows 11 with Chrome on my laptop.  
> **bot>** Your bug report was successfully created.  

The agent then returns the generated ticket ID to the customer:
> *Your ticket has been created successfully. The engineering team will investigate the issue.*

---

## Bug Report Tool

The project includes a dedicated Python implementation for creating bug reports:
`src/tools/create_bug_report.py`

The tool is responsible for creating the structured bug-report record after the required information has been collected.
The resulting ticket contains information such as:
- `ticketId`
- `createdAt`
- `description`
- `environment`
- `status`
- `stepsToReproduce`

New tickets are created with an open status: `status = OPEN`.

---

## Amazon DynamoDB

Bug reports are persisted in Amazon DynamoDB.
The DynamoDB table provides persistent storage for tickets created through the customer-support workflow.

A stored ticket contains fields including:

| Field | Description |
|---|---|
| `ticketId` | Unique ticket identifier |
| `createdAt` | Ticket creation timestamp |
| `description` | Description of the reported problem |
| `environment` | Customer environment information |
| `status` | Current ticket status |
| `stepsToReproduce` | Steps required to reproduce the issue |

Example stored tickets can be seen in the project's DynamoDB screenshot.

---

## FAQ Knowledge

The project includes an FAQ knowledge file:
`src/agent/online_shop_faq.md`

This file contains information about the fictional online shop that the agent can use when responding to supported FAQ requests.

Example questions include:
- What is your return policy?
- How do I track my order?
- Can I change the delivery driver assigned to my package?

The agent is expected to use the available information rather than inventing unsupported policies or details.

---

## Amazon Bedrock Flow

The main customer-support routing flow is implemented using Amazon Bedrock Flows.
The Flow contains:

- **RequestClassifier**: Classifies the customer's request.
- **RequestTypeCondition**: Evaluates the classifier result and determines which path should be executed.
- **FormatBugResponse**: Prepares the bug-report path for the next processing stage.
- **BugReportAgent**: Handles the bug-report workflow and tool interaction.
- **FAQAnswer**: Generates responses for supported FAQ questions.
- **OtherRequest**: Handles requests that do not match the BUG or FAQ categories.
- **Flow Output Nodes**: The different paths terminate in their corresponding Flow output nodes.

---

## AgentCore Integration

The project also contains Amazon Bedrock AgentCore configuration and supporting scripts.

Relevant files:
```text
src/agentcore/
├── agentcore_config.json
└── cleanup_agentcore.py
```
- `agentcore_config.json`: Contains the configuration used by the AgentCore-related setup.
- `cleanup_agentcore.py`: Provides cleanup functionality for resources created during development.

---

## AWS Lambda and Tools

The project uses Python-based tools to support agent functionality.

Tool-related files include:
```text
src/tools/
├── create_bug_report.py
├── create_harness.py
└── setup_gateway.py
```
- `create_bug_report.py`: Creates the structured bug report and persists the ticket information.
- `create_harness.py`: Supports the evaluation harness setup.
- `setup_gateway.py`: Provides supporting setup functionality for the tool/gateway integration.

---

## Infrastructure as Code

AWS infrastructure configuration is organized under:
```text
infrastructure/
└── cloudformation/
```
The project includes:
- `cloudformation-tool.yaml`
- `cloudformation-testing.yaml`

These CloudFormation templates provide infrastructure configuration for the project's tool and testing components. Using CloudFormation makes the infrastructure configuration reproducible and easier to manage.

---

## Evaluation

Evaluation is an important part of this project.
The evaluation workflow is organized under:
```text
evaluation/
├── datasets/
└── harness/
```
The project contains both evaluation dataset generation and harness configuration.

### Evaluation Dataset
The dataset-generation script is: `evaluation/datasets/generate-eval-dataset.py`  
The generated evaluation dataset is stored as: `evaluation/datasets/output_eval_dataset.jsonl`

The dataset contains representative customer-support prompts and expected behavior.

### Evaluation Harness
The evaluation harness contains:
```text
evaluation/harness/
├── harness-tests-template.json
└── harness-tests.json
```
These files are used to configure and execute evaluation tests for the agent.

### Evaluation Results
The completed evaluation achieved a **Correctness Score: 1.00**.

| Metric | Result |
|---|---|
| Correctness | 1.00 |
| Average Score | 1.000 |
| Prompts Evaluated | 6 |

All six evaluated prompts received a score of 1. This indicates that the agent produced responses matching the expected behavior for the tested scenarios.

### Evaluated Scenarios
The evaluation includes different types of customer-support interactions.

1. **Return Policy**
   - *Prompt:* What is your return policy?
   - *Expected behavior:* Answer the question using the FAQ.

2. **Order Tracking**
   - *Prompt:* How do I track my order?
   - *Expected behavior:* Answer using the available FAQ information.

3. **Incomplete Bug Report**
   - *Prompt:* The checkout button is not working. I click it but nothing happens.
   - *Expected behavior:* Treat the request as a bug report and collect the required information before creating the ticket.

4. **Detailed Bug Report**
   - *Prompt:* The checkout button does not work. To reproduce it, I add an item to my cart, go to checkout, and click the Place Order button.
   - *Expected behavior:* Continue the bug-report workflow and collect any remaining required information.

5. **Unsupported Request**
   - *Prompt:* Can you recommend a good movie to watch tonight?
   - *Expected behavior:* Treat the request as an unsupported request rather than answering outside the application's supported domain.

6. **Delivery Driver Question**
   - *Prompt:* Can I change the delivery driver assigned to my package?
   - *Expected behavior:* Use the FAQ when the required information is available and do not invent information that is not provided.

---

## Project Structure

```text
aws-bedrock-customer-support-agent/
│
├── docs/
│   │
│   ├── architecture/
│   │   └── architecture.md
│   │
│   └── screenshots/
│       ├── chatbot_conversation.png
│       ├── classifier-node.png
│       ├── condition-node.png
│       ├── conditions.png
│       ├── correctness.png
│       ├── customer-chatbot-support.png
│       ├── dynamodb_table.png
│       ├── metrics_run.png
│       ├── prompts1.png
│       └── prompts2.png
│
├── evaluation/
│   ├── datasets/
│   │   ├── generate-eval-dataset.py
│   │   └── output_eval_dataset.jsonl
│   │
│   └── harness/
│       ├── harness-tests-template.json
│       └── harness-tests.json
│
├── infrastructure/
│   └── cloudformation/
│       ├── cloudformation-testing.yaml
│       └── cloudformation-tool.yaml
│
├── src/
│   ├── agent/
│   │   ├── chat.py
│   │   ├── online_shop_faq.md
│   │   └── system_prompt.txt
│   │
│   ├── agentcore/
│   │   ├── agentcore_config.json
│   │   └── cleanup_agentcore.py
│   │
│   └── tools/
│       ├── create_bug_report.py
│       ├── create_harness.py
│       └── setup_gateway.py
│
├── .gitignore
├── README.md
└── requirements.txt
```

> **Note:** Update the screenshot filenames in this section if the actual filenames in your repository differ.

---

## AWS Services and Technologies

| Service / Technology | Purpose |
|---|---|
| **Amazon Bedrock** | Generative AI capabilities |
| **Amazon Bedrock Flows** | Request classification and conditional routing |
| **Amazon Bedrock AgentCore** | Agent and tool integration |
| **AWS Lambda** | Serverless tool execution |
| **Amazon DynamoDB** | Persistent bug-report storage |
| **AWS CloudFormation** | Infrastructure as code |
| **Python** | Application and tool implementation |
| **JSON / JSONL** | Configuration and evaluation data |

---

## Installation

### Prerequisites
Before running the project, make sure you have:
- Python installed
- AWS CLI installed
- An AWS account
- Appropriate AWS permissions
- Access to the required Amazon Bedrock resources
- Required AWS infrastructure configured
- Required dependencies installed

### Clone the Repository
```bash
git clone https://github.com/DhruvParekh-star/aws-bedrock-customer-support-agent.git
cd aws-bedrock-customer-support-agent
```

### Create a Virtual Environment

**Windows:**
```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

**macOS / Linux:**
```bash
python3 -m venv .venv
source .venv/bin/activate
```

### Install Dependencies
```bash
pip install -r requirements.txt
```

### AWS Configuration
Configure AWS credentials using a secure AWS authentication method.
Verify the AWS CLI configuration:
```bash
aws sts get-caller-identity
```
The command should return information about the currently authenticated AWS identity.

> **Important:** Never commit AWS credentials, secret keys, session tokens, API keys, or other sensitive information to this repository.

---

## Running the Application

The main conversational application is located at: `src/agent/chat.py`

Run the application from the agent directory:
```bash
cd src/agent
python chat.py
```

The application starts an interactive customer-support session.

### Example Interactions

#### FAQ Request
```text
you> What is your return policy?

bot> ...
```
*The request is classified as an FAQ-related request and routed to the FAQ workflow.*

#### Bug Report
```text
you> The checkout button is not working.

bot> Please provide the steps to reproduce the issue with the checkout button.
```
*The agent continues collecting the required information. After the information is complete:*
```text
bot> Your ticket has been created successfully.
```

#### Unsupported Request
```text
you> Can you recommend a good movie to watch tonight?

bot> ...
```
*The request is classified as an unsupported/other request rather than being incorrectly treated as an online-shop FAQ.*

---

## Screenshots

The repository contains screenshots demonstrating the implementation and evaluation results:

- **Customer Support Conversation:** `docs/screenshots/chatbot_conversation.png`
- **Request Classifier:** `docs/screenshots/classifier-node.png` — Bedrock Flow uses a prompt node to classify incoming customer requests.
- **Conditional Routing:** `docs/screenshots/condition-node.png` — Condition node evaluates the classifier output and routes the request.
- **Routing Conditions:** `docs/screenshots/conditions.png` — Distinguishes between BUG and FAQ requests, with an additional path for unsupported requests.
- **Customer Support Flow:** `docs/screenshots/customer-chatbot-support.png` — Complete Bedrock Flow layout.
- **DynamoDB Bug Reports:** `docs/screenshots/dynamodb_table.png` — Created bug reports stored in the DynamoDB table.
- **Evaluation Correctness:** `docs/screenshots/correctness.png` — Evaluation correctness score (1.00).
- **Evaluation Metrics:** `docs/screenshots/metrics_run.png` — Evaluation run showing an average correctness score of 1.000 across 6 prompts.
- **Prompt Evaluation Results:** `docs/screenshots/prompts1.png`, `docs/screenshots/prompts2.png` — Prompts, outputs, ground truths, and scores.

---

## Security Considerations

This repository is intended to contain application source code, configuration, infrastructure definitions, evaluation data, and documentation.

**Do not commit:**
- AWS access keys / secret access keys / session tokens
- API keys / Passwords / Private keys
- `.env` files containing secrets
- Other sensitive credentials

Before publishing the repository, review all configuration and JSON files for sensitive information. The `.gitignore` file should be used to prevent accidental commits of local environments, credentials, logs, and temporary files.

---

## Limitations

This project is a demonstration of an AWS-based generative AI customer-support architecture.
A production implementation could require additional capabilities such as:
- User authentication and authorization
- Production-grade ticket management
- Human-agent escalation
- Persistent customer profiles
- Expanded customer-support categories
- More extensive evaluation datasets
- Automated regression testing
- Monitoring and observability
- Advanced error handling and retries
- Cost monitoring and controls
- Data retention and privacy policies
- Production CI/CD

---

## Future Improvements

- Add more customer-support request categories.
- Add additional specialized tools.
- Integrate with a production ticketing platform.
- Add human-agent escalation.
- Expand the evaluation dataset.
- Add automated regression testing.
- Improve classification robustness.
- Add monitoring, logging, and tracing.
- Add CI/CD deployment automation.
- Add stronger error handling and retry mechanisms.
- Add persistent customer/order information.

---

## Skills Demonstrated

This project demonstrates practical experience with:
- Generative AI application development
- Amazon Bedrock & Amazon Bedrock Flows
- Prompt engineering & AI classification
- Conditional routing
- Agent and tool orchestration
- Amazon Bedrock AgentCore
- AWS Lambda
- Amazon DynamoDB
- AWS CloudFormation
- Python
- Serverless architecture
- Evaluation of AI applications, datasets, and harnesses
- AWS resource management

---

## Project Outcome

The project successfully demonstrates an end-to-end customer-support workflow:

```text
Customer Request
       |
       v
Classification
       |
       v
Conditional Routing
       |
       +------------+------------+
       |            |            |
       v            v            v
      BUG          FAQ         OTHER
       |            |            |
       v            v            v
Bug Workflow    FAQ Answer   Other Response
       |
       v
Bug Report Tool
       |
       v
DynamoDB Ticket
       |
       v
Customer Response
```

The completed evaluation achieved:
- **Correctness:** 1.00
- **Average Score:** 1.000
- **Prompts Evaluated:** 6

This demonstrates the project's ability to classify and route customer requests and produce the expected behavior across the evaluated scenarios.

---

## Repository Information

- **Project:** `aws-bedrock-customer-support-agent`
- **Primary Technologies:** Amazon Bedrock, Bedrock Flows, AgentCore, AWS Lambda, Amazon DynamoDB, AWS CloudFormation, Python
