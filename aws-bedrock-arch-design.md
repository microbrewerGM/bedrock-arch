# AWS Bedrock Integration Architecture Document

## 1. Introduction & Goals

### 1.1 Purpose & Scope
This document outlines the architecture and design for providing secure, controlled, and auditable access to Large Language Models (LLMs) hosted on AWS Bedrock for internal developers through VS Code extensions within a highly regulated and secure corporate environment.

### 1.2 Context
This system operates within a highly regulated corporate network environment, demanding strict adherence to:
- Security best practices
- Enterprise compliance standards
- Comprehensive auditing requirements
- AWS organizational policies (including SCPs)

### 1.3 Core Goals
- Provide developers with efficient access to approved LLM capabilities to enhance productivity and innovation
- Ensure all access is authenticated and authorized via AWS Identity Center
- Maintain strict security controls (network isolation, least-privilege, encryption)
- Implement comprehensive auditing and usage reporting for compliance and cost management
- Leverage AWS managed services to minimize operational overhead
- Ensure the solution is fully documented and maintainable

## 2. Use Case Definition

### 2.1 Actors/Stakeholders

#### Primary Users
- **Application Developers**: Need secure, integrated access to LLMs via VS Code for AI-enhanced development workflows (code assistance, generation, explanation, etc.)

#### Systems
- **VS Code Extension**: Custom or configured extension capable of AWS authentication and signed API calls

#### Key Stakeholders
- **Security Team**: Requires assurance of robust access controls, data protection, comprehensive audit trails, and policy compliance
- **DevOps/Platform Team**: Responsible for deploying, managing, and monitoring the infrastructure using Infrastructure as Code (IaC)
- **Compliance Team**: Needs verification that the solution meets regulatory requirements (data residency, auditability) and internal governance standards
- **AWS Platform Administrator**: Configures and maintains AWS infrastructure, policies, and monitoring

### 2.2 Core Requirements
- Secure access to AWS Bedrock models via a private API Gateway endpoint
- Authentication and authorization through AWS Identity Center (IAM Identity Center)
- Fine-grained access control based on developer roles and permissions
- Complete audit trail of all API requests and model interactions
- Detailed usage metrics and reporting capabilities
- Compliance with regulatory requirements and internal policies
- Network isolation through VPC configuration and VPC endpoints

### 2.3 Workflow

![Authentication and Request Flow Diagram](diagrams/authentication-flow-diagram.md)
*Authentication and request flow diagram showing the path from developer to Bedrock models*

1. **Authentication**: Developer authenticates to AWS via AWS Identity Center, obtaining temporary AWS credentials mapped to a specific IAM role.

2. **API Request**: VS Code extension constructs a request and sends it to the private AWS API Gateway endpoint, signed using the temporary credentials (AWS Signature Version 4).

3. **Authorization & Validation**: 
   - API Gateway authenticates the request using IAM authorization
   - Verifies the signature and caller's identity
   - Authorizes based on IAM policies attached to the developer's assumed role
   - Validates the request payload against a predefined schema

4. **Backend Processing**: API Gateway forwards the validated request to the Lambda proxy function within the private VPC.

5. **Role Assumption & Processing**: The Lambda function:
   - Logs initial request metadata to CloudWatch
   - Assumes a dedicated IAM role (Bedrock-Access-Role) with least-privilege permissions
   - Performs input validation/sanitization
   - Constructs the appropriate API call for AWS Bedrock

6. **Bedrock Interaction**: Lambda sends the request to AWS Bedrock via a VPC Endpoint.

7. **LLM Processing**: AWS Bedrock routes the request to the selected foundation model, processes the prompt, and returns the response.

8. **Response Handling & Auditing**: Lambda receives the response and:
   - Performs output validation/scanning (optional)
   - Logs detailed audit information to CloudWatch Logs
   - Formats the LLM response for API Gateway and VS Code

9. **Client Response**: API Gateway forwards the response back to the VS Code extension, which displays the result to the developer.

### 2.4 Why Lambda Between API Gateway and Bedrock

![Lambda Intermediary Role Diagram](diagrams/lambda-intermediary-role-diagram.md)
*Lambda Intermediary Role Diagram illustrating the Lambda function's critical role as a security boundary and business logic processor*

The Lambda function serves several essential purposes as an intermediary between API Gateway and Bedrock:

1. **Role Assumption & Security Boundary**:
   - AWS API Gateway cannot directly assume different IAM roles
   - The Lambda creates a critical security boundary, limiting direct Bedrock access
   - Using the AssumeRole pattern isolates permissions and follows the principle of least privilege
   - Creates a clear audit trail showing exactly which entity accessed Bedrock

2. **Business Logic & Input/Output Processing**:
   - Performs custom validation beyond what API Gateway schema validation can provide
   - Handles model-specific request formatting (different LLMs require different payload structures)
   - Conducts content scanning and sanitization as needed
   - Unifies response formats across different model types

3. **Enhanced Logging & Auditing**:
   - Captures detailed metadata about each request
   - Produces structured logs for audit trails and metrics
   - Creates a standardized observability layer

4. **Error Handling & Resilience**:
   - Provides custom error handling and retry logic
   - Can implement rate limiting and throttling specific to models or users
   - Simplifies client-side code by handling Bedrock-specific error cases

Without this Lambda layer, the implementation would either need to give broader permissions to API Gateway/clients directly or require much more complex client-side code in the VS Code extension.

## 3. High-Level Architecture

### 3.1 Architectural Layers

![Architecture Layers Diagram](diagrams/architecture-layers-diagram.md)
*Architecture Layers Diagram showing the architectural layers and components from developers to Bedrock*

1. **Client Layer**: VS Code Extension on developer workstations
2. **Access/Control Layer**: Private AWS API Gateway (REST API)
3. **Authentication/Authorization Layer**: AWS Identity Center, IAM Roles & Policies, SCPs
4. **Orchestration Layer**: AWS Lambda (BedrockProxyLambda)
5. **AI Service Layer**: AWS Bedrock (accessing specific Foundation Models)
6. **Networking Layer**: AWS VPC, Private Subnets, Security Groups, VPC Endpoints
7. **Governance/Observability Layer**: AWS CloudTrail, CloudWatch

### 3.2 Core Components

#### 3.2.1 AWS Identity Center
- Centralized user directory integration with corporate identity provider
- Assigns users/groups to AWS accounts and specific Permission Sets
- Maps to IAM roles (Developer-LLM-Access-Role-Basic/Advanced)

#### 3.2.2 AWS API Gateway (Private REST API)
- Provides secure, private RESTful endpoint
- Accessible only from within the VPC or connected networks
- Uses IAM authorization and request validation
- Implements throttling and resource policies
- Routes requests to Lambda via proxy integration

#### 3.2.3 AWS Lambda (BedrockProxyLambda)
- Acts as secure proxy between API Gateway and Bedrock
- Enforces business logic, validation, and security controls
- Performs role assumption for Bedrock access
- Implements detailed logging and auditing
- Runs within private VPC subnet

#### 3.2.4 AWS Bedrock
- Managed service providing access to selected foundation models
- Access restricted via IAM policies and SCPs
- Accessed via VPC Endpoint for network security

#### 3.2.5 AWS IAM
- Implements roles with fine-grained permissions:
  - Developer-LLM-Access-Role (assumed via Identity Center)
  - BedrockProxyLambda-Execution-Role (for Lambda service)
  - Bedrock-Access-Role (assumed by Lambda function)
- Uses inline policies for precise permission boundaries
- Enforces SCPs at the organization level

#### 3.2.6 VPC & Networking
- Private subnets for Lambda function
- Security groups for traffic control
- VPC endpoints (PrivateLink) for AWS services
- Ensures traffic stays within AWS network backbone

#### 3.2.7 Observability & Governance
- CloudTrail for API call logging
- CloudWatch for logs, metrics, and alarms
- Custom reporting for usage tracking and cost allocation

## 4. Low-Level Design

### 4.1 Identity & Access Management

#### 4.1.1 AWS Identity Center Configuration

![Identity Center Configuration Diagram](diagrams/identity-center-configuration-diagram.md)
*Identity Center Configuration Diagram illustrating Identity Center integration with corporate identity provider and role mappings*

```
Directory Integration:
  - Connected to corporate identity provider (e.g., Azure AD/Okta)

User Groups:
  - Corp-LLM-Users-Basic
  - Corp-LLM-Users-Advanced (for privileged access)

Permission Sets:
  - LLMAccessBasicPS → Developer-LLM-Access-Role-Basic
  - LLMAccessAdvancedPS → Developer-LLM-Access-Role-Advanced

Account Assignments:
  - User groups mapped to appropriate AWS account and Permission Sets
```

#### 4.1.2 IAM Roles & Policies Structure
```
IAM Roles:
├── Developer-LLM-Access-Role-<Tier> (Assumed via Identity Center)
│   └── Inline Policy: Allow execute-api:Invoke on specific API GW ARN
├── BedrockProxyLambda-Execution-Role (For Lambda Service)
│   ├── Trust Policy: lambda.amazonaws.com
│   └── Inline Policies:
│       ├── CloudWatchLogs: CreateLogGroup, CreateLogStream, PutLogEvents
│       ├── VPCNetworking: Create/Describe/DeleteNetworkInterface
│       ├── AssumeRole: Allow sts:AssumeRole on Bedrock-Access-Role ARN
└── Bedrock-Access-Role (Assumed by Lambda)
    ├── Trust Policy: Allow sts:AssumeRole ONLY from BedrockProxyLambda-Execution-Role ARN
    └── Inline Policy:
        └── BedrockModelAccess: Allow bedrock:InvokeModel on specific approved Model ARNs ONLY
```

#### 4.1.3 Service Control Policies (SCPs)
```
Region Restriction SCP:
  - Deny bedrock:* actions outside approved regions

Model Restriction SCP:
  - Deny bedrock:InvokeModel if resource ARN doesn't match approved models

API Configuration SCP:
  - Deny creation of public API Gateway endpoints

Resource Management SCP:
  - Require specific tags on resources (CostCenter, Project, SecurityLevel)
  - Ensure CloudTrail is enabled and cannot be disabled

Feature Control SCP:
  - Optionally deny specific Bedrock features if not approved
```

### 4.2 API Gateway Configuration

#### 4.2.1 Base Configuration
```
API Name: InternalBedrockProxyAPI
Endpoint Type: PRIVATE
```

#### 4.2.2 Resource Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:<region>:<account-id>:<api-id>/*/*/*",
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "<vpce-id-for-apigateway>"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:<region>:<account-id>:<api-id>/*/*/*",
      "Condition": {
        "StringEquals": {
          "aws:SourceVpce": "<vpce-id-for-apigateway>"
        }
      }
    }
  ]
}
```

#### 4.2.3 Resource/Method Definition
```
Resource: /invoke/{modelId}
  - Path Parameter: modelId (String, Required)

Method: POST (on /{modelId} resource)
  - Authorization: AWS_IAM
  - Request Validator: Validate body AND path parameters (modelId)
  - Request Models: JSON schema for request body validation
  - Integration: AWS_PROXY (Lambda Proxy)
  - Lambda Function: BedrockProxyLambda

Method: OPTIONS (on /{modelId} resource)
  - Purpose: Handle CORS preflight requests (if needed)
  - Authorization: NONE
  - Method Response: 200 OK
  - Integration: Mock (Return appropriate CORS headers)
```

#### 4.2.4 Logging & Deployment
```
Stage: v1
Logging:
  - CloudWatch Logs enabled
  - Log Level: INFO
  - Log full request/response data: Enabled (Review sensitivity)
Throttling: (Example Stage Settings)
  - Rate limit: 50 requests/sec
  - Burst limit: 100 requests
```

### 4.3 Lambda Function Implementation

#### 4.3.1 Runtime Configuration
```
Runtime: Python 3.11
Memory: 1024 MB (adjust based on performance testing)
Timeout: 60 seconds (allow buffer for model latency)
```

#### 4.3.2 Environment Variables
```
BEDROCK_ROLE_ARN: arn:aws:iam::<account-id>:role/Bedrock-Access-Role
ALLOWED_MODEL_IDS: amazon.titan-text-express-v1,anthropic.claude-v2
LOG_LEVEL: INFO
BEDROCK_REGION: us-east-1
POWERTOOLS_SERVICE_NAME: bedrock-proxy
```

#### 4.3.3 VPC Configuration
```
VPC: Corporate-VPC
Subnets: Private subnets across multiple AZs
Security Group: Lambda-SG
  - Outbound: Allow HTTPS (443) ONLY to VPC Endpoints
```

#### 4.3.4 Logic Outline (Pseudocode)
```python
# Import necessary libraries
import boto3, json, logging, time, os
from botocore.exceptions import ClientError

# Initialize logger and clients
logger = setup_structured_logger()
sts_client = boto3.client('sts')

# Environment variables
BEDROCK_ROLE_ARN = os.environ['BEDROCK_ROLE_ARN']
ALLOWED_MODEL_IDS = os.environ['ALLOWED_MODEL_IDS'].split(',')
BEDROCK_REGION = os.environ['BEDROCK_REGION']

def lambda_handler(event, context):
    start_time = time.time()
    request_id = context.aws_request_id
    
    try:
        # Extract request details
        caller_arn = event['requestContext']['identity']['userArn']
        model_id = event.get('pathParameters', {}).get('modelId')
        body = json.loads(event['body'])
        prompt = body.get('prompt')

        # Input Validation
        if not model_id or model_id not in ALLOWED_MODEL_IDS:
            logger.warning("Invalid or disallowed model requested", extra={"model_id": model_id})
            return error_response(403, "Model not allowed or invalid")
        if not prompt:
            logger.error("Missing prompt in request body")
            return error_response(400, "Missing prompt")
        
        # Log request metadata
        logger.info("Processing request", extra={"caller": caller_arn, "model_id": model_id})
        
        # Assume Bedrock access role
        assumed_role = sts_client.assume_role(
            RoleArn=BEDROCK_ROLE_ARN,
            RoleSessionName=f"BedrockProxySession-{request_id[:8]}"
        )
        credentials = assumed_role['Credentials']
        
        # Initialize Bedrock client with assumed credentials
        bedrock_client = boto3.client(
            'bedrock-runtime',
            region_name=BEDROCK_REGION,
            aws_access_key_id=credentials['AccessKeyId'],
            aws_secret_access_key=credentials['SecretAccessKey'],
            aws_session_token=credentials['SessionToken']
        )
        
        # Prepare model-specific payload
        payload = construct_bedrock_payload(model_id, body)
        
        # Invoke Bedrock model
        response = bedrock_client.invoke_model(
            modelId=model_id,
            contentType='application/json',
            accept='application/json',
            body=json.dumps(payload)
        )
        response_body = json.loads(response['body'].read())
        
        # Parse model-specific response
        parsed_response = parse_bedrock_response(model_id, response_body)
        
        # Calculate metrics and log audit record
        latency = time.time() - start_time
        log_audit_record(logger, caller_arn, model_id, request_id, latency, parsed_response['usage'])
        
        # Return formatted response
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json'
            },
            'body': json.dumps({
                'model': model_id,
                'generated_text': parsed_response['generated_text'],
                'usage': {
                    'input_tokens': parsed_response['usage'].get('input_tokens', 0),
                    'output_tokens': parsed_response['usage'].get('output_tokens', 0),
                    'latency_seconds': latency
                }
            })
        }
    
    # Handle specific error types
    except ClientError as e:
        error_code = e.response.get("Error", {}).get("Code")
        logger.exception(f"AWS ClientError: {error_code}")
        if error_code == 'ThrottlingException':
            return error_response(429, "Too many requests")
        return error_response(500, f"AWS Error: {error_code}")
    except Exception as e:
        logger.exception("Unhandled exception")
        return error_response(500, "Internal Server Error")

# Helper functions for payload construction, response parsing, etc.
def construct_bedrock_payload(model_id, body):
    # Model-specific payload formatting
    if model_id.startswith('anthropic.claude'):
        return {
            "prompt": f"Human: {body.get('prompt')}\n\nAssistant:",
            "temperature": body.get('temperature', 0.7),
            "max_tokens_to_sample": body.get('max_tokens', 1000)
        }
    else:  # Generic approach (customize per model family)
        return {
            "inputText": body.get('prompt'),
            "textGenerationConfig": {
                "temperature": body.get('temperature', 0.7),
                "maxTokenCount": body.get('max_tokens', 1000)
            }
        }

def parse_bedrock_response(model_id, response_body):
    # Extract model-specific response content
    if model_id.startswith('anthropic.claude'):
        generated_text = response_body.get('completion', '')
        input_tokens = response_body.get('usage', {}).get('input_tokens', 0)
        output_tokens = response_body.get('usage', {}).get('output_tokens', 0)
    else:  # Generic approach (customize per model family)
        generated_text = response_body.get('results', [{}])[0].get('outputText', '')
        input_tokens = -1  # Not available in this example
        output_tokens = -1  # Not available in this example
    
    return {
        'generated_text': generated_text,
        'usage': {
            'input_tokens': input_tokens,
            'output_tokens': output_tokens
        }
    }

def log_audit_record(logger, caller_arn, model_id, request_id, latency, usage):
    logger.info({
        'event': 'bedrock_invoked',
        'caller_arn': caller_arn,
        'model_id': model_id,
        'request_id': request_id,
        'status': 'success',
        'latency_seconds': latency,
        'input_tokens': usage.get('input_tokens', 0),
        'output_tokens': usage.get('output_tokens', 0)
    })

def error_response(status_code, message):
    return {
        'statusCode': status_code,
        'headers': {
            'Content-Type': 'application/json'
        },
        'body': json.dumps({
            'error': message
        })
    }
```

### 4.4 Networking Configuration

#### 4.4.1 VPC Endpoints
```
VPC Endpoints (Interface/PrivateLink):
  - com.amazonaws.<region>.execute-api
  - com.amazonaws.<region>.bedrock-runtime
  - com.amazonaws.<region>.sts
  - com.amazonaws.<region>.logs

Security Groups:
  - Lambda-SG:
    - Outbound: HTTPS (443) to VPC Endpoint security groups
  
  - VPCEndpoint-SG:
    - Inbound: HTTPS (443) from Lambda-SG
```

#### 4.4.2 DNS Configuration
```
VPC Settings:
  - DNS Resolution: Enabled
  - DNS Hostnames: Enabled

VPC Endpoint Settings:
  - Private DNS: Enabled for all endpoints
```

### 4.5 Auditing & Monitoring

#### 4.5.1 CloudTrail Configuration
```
Trail Name: Organization-Security-Trail
Settings:
  - Multi-region: Yes
  - Management events: All (Read/Write)
  - Log File Validation: Enabled
  - KMS Encryption: Enabled
  - S3 Bucket: security-audit-logs-<account-id>
    - Retention policy aligned with compliance requirements
```

#### 4.5.2 CloudWatch Logs
```
Log Groups:
  - /aws/apigateway/InternalBedrockProxyAPI
  - /aws/lambda/BedrockProxyLambda

Retention Periods:
  - Align with compliance requirements (e.g., 90 days)

Log Format:
  - Structured JSON logging from Lambda
  - Full API Gateway request/response logging
```

#### 4.5.3 CloudWatch Metrics & Alarms
```
Metrics:
  - API Gateway: Count, Latency, 4XXError, 5XXError
  - Lambda: Invocations, Errors, Duration, Throttles
  - Custom Metrics: 
    - BedrockInvocationCount (by model)
    - BedrockLatency
    - InputTokenCount
    - OutputTokenCount
    - PerUserModelInvocations

Alarms:
  - High API Error Rate (>1%)
  - High Lambda Error Rate (>1%)
  - High API Latency (>p95 10s)
  - High Lambda Duration (>80% timeout)
  - Anomalous Invocation Count
```

#### 4.5.4 Custom Reporting
```
Implementation: 
  - CloudWatch Logs Subscription Filter → Lambda
  - Lambda processes and aggregates audit logs
  - Store aggregated data in DynamoDB/S3

Aggregation Fields:
  - Date (YYYY-MM-DD)
  - Hour
  - User ARN
  - Department (derived from user groups/tags)
  - Model ID
  - Success/Error Count
  - Total Input/Output Tokens
  - Average Latency

Visualization:
  - Amazon QuickSight dashboards
  - Cost allocation reports by department
  - Usage patterns by model and time period
```

### 4.6 VS Code Extension Integration

![VS Code Extension Integration Diagram](diagrams/vscode-extension-integration-diagram.md)
*VS Code Extension Integration Diagram showing VS Code extension integration with authentication flow and request handling*

#### 4.6.1 Authentication Flow
```
1. Developer launches VS Code
2. AWS Toolkit extension initiates SSO login flow
3. Browser opens AWS Identity Center login page
4. Developer authenticates with corporate credentials
5. AWS Toolkit stores temporary credentials
6. LLM extension accesses these credentials to sign API requests
```

#### 4.6.2 API Interaction
```
Request Format:
  - Endpoint: https://<api-id>.execute-api.<region>.amazonaws.com/v1/invoke/{modelId}
  - Method: POST
  - Headers: 
    - Content-Type: application/json
    - Authorization: AWS Signature V4 (handled by SDK)
  - Body:
    {
      "prompt": "Explain this code: function hello() { return 'world'; }",
      "temperature": 0.7,
      "max_tokens": 1000
    }

Response Format:
  - Status: 200 OK
  - Body:
    {
      "model": "anthropic.claude-v2",
      "generated_text": "This code defines a JavaScript function...",
      "usage": {
        "input_tokens": 12,
        "output_tokens": 42,
        "latency_seconds": 0.876
      }
    }
```

## 5. Security Controls

### 5.1 Authentication & Authorization
- All access requires authentication via AWS Identity Center mapped to least-privilege IAM roles
- API Gateway uses IAM Authorization
- Lambda uses role assumption (STS) for Bedrock access, limiting blast radius
- SCPs provide organizational guardrails
- Regular access reviews for Identity Center group memberships and permission sets

### 5.2 Network Security
- Private API Gateway endpoint accessible only via VPC Endpoint
- Lambda function operates within private VPC subnets
- Communication uses VPC Endpoints, keeping traffic off the public internet
- Security Groups enforce strict ingress/egress rules
- Network ACLs provide stateless subnet boundary protection (optional)

### 5.3 Data Security
- **Encryption in Transit**: TLS 1.3 enforced for all connections
- **Encryption at Rest**:
  - CloudTrail/CloudWatch Logs encrypted using KMS
  - Lambda environment variables encrypted
  - S3 for custom reporting using SSE-KMS
- **Data Handling**:
  - Input/Output validation and sanitization
  - Option for PII detection/redaction
  - Careful logging practices to avoid sensitive data exposure

### 5.4 Code Security
- Lambda function code subject to static analysis security testing (SAST)
- Dependency scanning for vulnerabilities
- Secure coding practices followed

### 5.5 Auditing & Compliance
- Comprehensive logging via CloudTrail and CloudWatch
- Alerting configured for security events, errors, and anomalies
- Custom reporting for detailed usage analysis

## 6. Optional Feature Sets

### 6.1 X-Ray Tracing & Advanced Observability

This section describes an optional enhanced observability layer using AWS X-Ray for distributed tracing.

#### 6.1.1 Value Proposition
- End-to-end request visibility from API Gateway through Lambda to Bedrock
- Performance bottleneck identification
- Error analysis across service boundaries
- Service dependency mapping

#### 6.1.2 Required Components
```
Additional IAM Permissions:
  - BedrockProxyLambda-Execution-Role:
    - xray:PutTraceSegments
    - xray:PutTelemetryRecords

Additional VPC Endpoints:
  - com.amazonaws.<region>.xray

Additional Configuration:
  - Enable Tracing on API Gateway Stage
  - Enable Active Tracing on Lambda Function
```

#### 6.1.3 Lambda Implementation Additions
```python
# Additional imports for X-Ray
from aws_xray_sdk.core import patch_all
from aws_lambda_powertools import Tracer

# Initialize X-Ray
patch_all()
tracer = Tracer()

@tracer.capture_lambda_handler
def lambda_handler(event, context):
    # Existing code...
    
    # Add X-Ray annotations and metadata
    tracer.put_annotation("bedrock_model_id", model_id)
    
    # Capture subsegments for key operations
    with tracer.capture_subsegment('assume_role'):
        assumed_role = sts_client.assume_role(...)
    
    with tracer.capture_subsegment('invoke_bedrock_model'):
        response = bedrock_client.invoke_model(...)
        
    # Existing code...
```

#### 6.1.4 Observability Benefits
- Service Maps: Visual representation of request flow and dependencies
- Trace Analysis: Detailed timing information for each request segment
- Annotation Search: Find traces by model ID, error type, etc.
- Performance Insights: Identify slow components or poor-performing models

### 6.2 Enhanced Content Scanning & Sanitization

This section outlines optional integrations for more robust content analysis.

#### 6.2.1 Value Proposition
- PII detection and redaction in prompts/responses
- Content policy enforcement
- Regulatory compliance for sensitive information

#### 6.2.2 Required Components
```
Additional Services:
  - Amazon Comprehend (for PII detection)
  - Optional custom content scanning Lambda

Additional VPC Endpoints:
  - com.amazonaws.<region>.comprehend

Additional IAM Permissions:
  - BedrockProxyLambda-Execution-Role:
    - comprehend:DetectPiiEntities
```

#### 6.2.3 Implementation Considerations
- Performance impact (latency increase of ~100-500ms)
- Cost implications (Comprehend charges per API call)
- Accuracy trade-offs (false positives/negatives)
- Integration points in Lambda function for both input and output scanning

## 7. Implementation Roadmap

### 7.1 Phase 1: Core Infrastructure
1. Configure AWS Identity Center integration
2. Implement IAM roles and policies
3. Configure VPC and VPC endpoints
4. Deploy API Gateway with resource policy
5. Implement basic Lambda proxy function
6. Configure CloudTrail and CloudWatch logging

### 7.2 Phase 2: Enhanced Security & Monitoring
1. Implement SCPs at organization level
2. Enhance CloudWatch metrics and alarms
3. Implement structured logging with AWS Lambda Powertools
4. Develop initial audit reporting capabilities

### 7.3 Phase 3: Advanced Features (Optional)
1. Implement X-Ray tracing if needed
2. Add input/output validation and sanitization
3. Add support for additional Bedrock models
4. Enhance the VS Code extension experience
5. Develop comprehensive dashboards and cost allocation reports

## 8. Operational Considerations

### 8.1 Monitoring & Alerting
- Implement comprehensive CloudWatch dashboards
- Configure alerts for security, performance, and usage anomalies
- Establish incident response procedures

### 8.2 Cost Management
- Use tagging for cost allocation
- Monitor usage patterns and implement budgets
- Analyze token usage by department/team

### 8.3 Maintenance
- Regular updates for Lambda runtime and dependencies
- Periodic security reviews and penetration testing
- Training for administrators and users

## Appendix A: Persona Profiles

### A.1 Developer Persona
![Developer Persona Profiles](diagrams/developer-persona-diagram.md)
*Developer Persona Profiles showing visual representations of the developer, security, and DevOps personas*

- **Name**: Alex Chen
- **Role**: Senior Software Engineer, Product Development Team
- **Technical Background**: 8+ years in software development, proficient in Python, TypeScript, Java
- **Primary Goals**:
  - Accelerate code development using AI assistance
  - Generate and refine code documentation
  - Debug complex issues with AI guidance
- **Pain Points**:
  - Corporate security policies restrict general internet LLM usage
  - Needs seamless integration into existing development workflow
  - Requires high performance and reliable access to LLMs

### A.2 Security Team Persona

- **Name**: Jordan Rivera
- **Role**: Senior Information Security Officer
- **Technical Background**: 12+ years in security, CISSP certified, expertise in cloud security and compliance
- **Primary Goals**:
  - Ensure all AI tools meet regulatory requirements
  - Prevent data leakage and protect sensitive information
  - Maintain comprehensive audit trails
- **Pain Points**:
  - Needs visibility into all AI interactions
  - Requires assurance of proper access controls
  - Must address regulatory compliance requirements

### A.3 DevOps Team Persona

- **Name**: Taylor Kim
- **Role**: Platform Engineer, Cloud Infrastructure Team
- **Technical Background**: 6+ years in DevOps, AWS certified, IaC expertise
- **Primary Goals**:
  - Build scalable, maintainable infrastructure
  - Automate deployment and monitoring
  - Ensure high availability and performance
- **Pain Points**:
  - Managing complex AWS permissions and networking
  - Balancing security requirements with developer experience
  - Monitoring and troubleshooting distributed services
