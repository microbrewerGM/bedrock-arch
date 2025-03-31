# VS Code to Amazon Bedrock: Minimal Secure Architecture

## 1. Introduction & Goals

### 1.1 Purpose & Scope
This document outlines the minimum infrastructure and configuration required to connect VS Code tooling to Amazon Bedrock LLMs via a secure API Gateway. The design prioritizes simplicity while maintaining security through private network connections, IAM-based authorization, and proper access controls while eliminating unnecessary complexity.

### 1.2 Context
This architecture operates within a secure network environment, ensuring that:
- Connections to LLMs are authenticated and authorized
- Network traffic is secure and private
- Access to AI services follows least-privilege principles
- Implementation is efficient and maintainable

### 1.3 Core Goals
- Provide VS Code with direct, secure access to Amazon Bedrock capabilities
- Maintain end-to-end security through private network connections
- Minimize infrastructure complexity while ensuring proper security controls
- Enable developer productivity through streamlined LLM access

## 2. Minimal Architecture Overview

![Minimal Architecture Diagram](diagrams/minimal-architecture-diagram.md)

```
┌─────────────┐     ┌──────────────┐     ┌────────────────┐     ┌────────────────┐
│             │     │ AWS Private  │     │                │     │                │
│   VS Code   │────▶│ API Gateway  │────▶│ Amazon Bedrock │────▶│    LLM Models  │
│ Extensions  │     │ Endpoint     │     │    Service     │     │                │
└─────────────┘     └──────────────┘     └────────────────┘     └────────────────┘
       │                   │                     │
       │                   │                     │
       ▼                   ▼                     ▼
┌─────────────┐     ┌──────────────┐     ┌────────────────┐
│             │     │              │     │                │
│ AWS Toolkit │     │ IAM/Identity │     │ IAM Role for   │
│ Auth Plugin │     │    Center    │     │ Bedrock Access │
└─────────────┘     └──────────────┘     └────────────────┘
```

## 3. Component Details

### 3.1 VS Code Integration

#### 3.1.1 Required Extensions
- **AWS Toolkit for VS Code**: Provides authentication with AWS services
- **Custom LLM Extension or Integration**: Communicates with API Gateway endpoint

#### 3.1.2 Authentication Flow
1. Developer authenticates via AWS Toolkit extension using AWS Identity Center
2. AWS Toolkit securely stores temporary credentials
3. LLM extension utilizes these credentials to sign API requests to API Gateway

### 3.2 AWS API Gateway

#### 3.2.1 Core Configuration
- **Type**: REST API with PRIVATE endpoint type
- **Authorization**: AWS_IAM (uses Signature Version 4 signing)
- **Network Access**: Restricted to VPC Endpoint with private DNS enabled

#### 3.2.2 API Structure
```
Resource: /invoke/{modelId}
Method: POST
  - Request Validation: Validate body and path parameters
  - Integration: Direct integration with Amazon Bedrock
  - IAM Authorization: Yes
```

### 3.3 Network Security

#### 3.3.1 VPC Endpoints
```
Required VPC Endpoints:
- com.amazonaws.<region>.execute-api
- com.amazonaws.<region>.bedrock-runtime
```

#### 3.3.2 Security Groups
```
VPCEndpoint-SG:
- Inbound: HTTPS (443) from authorized sources only
- Outbound: HTTPS (443) to AWS services only
```

### 3.4 Identity & Access Management

#### 3.4.1 AWS Identity Center Configuration
```
Directory Integration:
- Connect to corporate identity provider (optional for minimal setup)

User Groups:
- LLM-Users

Permission Sets:
- LLMAccessPS → maps to Bedrock-Access-Role
```

#### 3.4.2 IAM Roles & Policies
```
IAM Role: Bedrock-Access-Role
  - Trust relationship with Identity Center (or direct IAM users)
  - Permissions:
      - execute-api:Invoke on API Gateway ARN
      - bedrock:InvokeModel on approved model ARNs
```

#### 3.4.3 Resource Policy (API Gateway)
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

## 4. API Gateway Integration with Bedrock

### 4.1 Request Mapping

API Gateway uses request/response mapping templates to transform the incoming VS Code extension request into the format expected by Amazon Bedrock.

#### 4.1.1 Request Template Example
```velocity
#set($inputRoot = $input.path('$'))
{
  "modelId": "$input.params('modelId')",
  "contentType": "application/json",
  "accept": "application/json",
  "body": {
    #if($input.params('modelId').startsWith("anthropic.claude"))
      "prompt": "Human: $inputRoot.prompt\n\nAssistant:",
      "temperature": $inputRoot.temperature,
      "max_tokens_to_sample": $inputRoot.max_tokens
    #else
      "inputText": "$inputRoot.prompt",
      "textGenerationConfig": {
        "temperature": $inputRoot.temperature,
        "maxTokenCount": $inputRoot.max_tokens
      }
    #end
  }
}
```

### 4.2 Response Mapping

#### 4.2.1 Response Template Example
```velocity
#set($inputRoot = $input.path('$'))
#if($input.params('modelId').startsWith("anthropic.claude"))
{
  "model": "$input.params('modelId')",
  "generated_text": "$inputRoot.completion",
  "usage": {
    "input_tokens": $inputRoot.usage.input_tokens,
    "output_tokens": $inputRoot.usage.output_tokens,
    "latency_seconds": 0 ## Placeholder, API Gateway doesn't provide this
  }
}
#else
{
  "model": "$input.params('modelId')",
  "generated_text": "$inputRoot.results[0].outputText",
  "usage": {
    "input_tokens": 0, ## Placeholder as some models don't provide this
    "output_tokens": 0, ## Placeholder as some models don't provide this
    "latency_seconds": 0 ## Placeholder, API Gateway doesn't provide this
  }
}
#end
```

## 6. Security Best Practices

### 6.1 Implement Transport Encryption
- Enforce TLS 1.2+ for all connections
- Use secure cipher suites for VPC Endpoints

### 6.2 Use Least Privilege Access
- Restrict IAM roles to specific model ARNs only
- Set resource policies to limit access to API Gateway

### 6.3 Enable CloudTrail Logging
- Track all API calls to Amazon Bedrock
- Log API Gateway usage for audit purposes

### 6.4 Implement Request Throttling
- Configure usage plans in API Gateway
- Prevent abuse and control costs

## 7. Implementation Checklist

1. **Set Up AWS Identity Center** (or IAM for simpler deployments)
   - Create necessary groups and permission sets
   - Configure identity provider integration if needed

2. **Configure VPC and Endpoints**
   - Set up VPC Endpoints for API Gateway and Bedrock
   - Configure security groups with restrictive rules

3. **Deploy API Gateway**
   - Create private REST API
   - Configure resource policy limiting access to VPC Endpoint
   - Set up API resource, method, and AWS service integration
   - Configure request/response mapping templates

4. **Configure IAM Permissions**
   - Create or update role for Bedrock access
   - Attach policies with least privilege permissions

5. **Test Connectivity**
   - Verify API Gateway can invoke Bedrock models
   - Test end-to-end connection from VS Code

6. **Implement VS Code Extension**
   - Integrate with AWS Toolkit for authentication
   - Implement API Gateway request handling
