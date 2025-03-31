# CodeGate Integration for AWS Bedrock

This repository contains the architecture design and implementation files for integrating CodeGate with AWS Bedrock to provide enhanced security, workspace management, and model routing capabilities.

## Overview

CodeGate is a security-focused gateway for AI applications and coding assistants that provides:

- **Security Analysis**: Scans AI-generated code for security issues
- **Workspace Management**: Isolated development spaces with separate histories and configurations
- **Model Muxing**: Routes requests to different AI models based on workspaces or file types
- **Secrets/PII Redaction**: Prevents sensitive information from being sent to AI models
- **Dependency Risk Awareness**: Identifies outdated or vulnerable packages in suggestions
- **Request Audit Trail**: Logs all interactions for security compliance

This project provides two implementation patterns:

1. **Local Developer Setup**: Docker-based deployment for individual developer workstations
2. **AWS Production Implementation**: Scalable multi-user implementation integrated with the existing AWS Bedrock architecture

## Local Developer Setup

The local setup uses Docker and NGINX to create a secure, isolated environment for developers to use CodeGate on their local workstations.

### Prerequisites

- Docker and Docker Compose
- OpenSSL (for generating self-signed certificates)
- AWS credentials with Bedrock access

### Setup Instructions

1. Navigate to the `codegate-local` directory:
   ```
   cd codegate-local
   ```

2. Make the setup script executable:
   ```
   chmod +x setup.sh
   ```

3. Run the setup script:
   ```
   ./setup.sh
   ```

4. Follow the prompts to configure AWS credentials and API Gateway endpoint.

5. Once setup is complete, access the CodeGate dashboard at http://localhost:9090.

6. Configure your AI coding assistant (like Cline, Continue, Aider, or GitHub Copilot) to use https://localhost:8443 as the API endpoint.

### Configuration

The local setup includes:

- `docker-compose.yml`: Defines the CodeGate and NGINX containers
- `nginx.conf`: Configures NGINX as a reverse proxy with SSL/TLS termination
- `config.yaml`: Sets up CodeGate workspaces, model configurations, and security settings
- `setup.sh`: Automates the configuration and deployment process

## AWS Multi-User Implementation

The AWS implementation provides a scalable, multi-user environment integrated with the existing AWS Bedrock architecture.

### Architecture

The AWS deployment uses:

- **Application Load Balancer** for TLS termination and user-specific routing
- **Amazon ECS with Fargate** for running CodeGate containers
- **Amazon RDS PostgreSQL** for a centralized database
- **Amazon EFS** for persistent workspace storage
- **AWS Identity Center** for authentication and authorization
- **AWS WAF** for additional security protection

### Deployment Instructions

1. Navigate to the `codegate-aws` directory:
   ```
   cd codegate-aws
   ```

2. Review and customize the CloudFormation template parameters:
   ```
   nano cloudformation-template.yaml
   ```

3. Deploy the CloudFormation stack:
   ```
   aws cloudformation create-stack \
     --stack-name codegate \
     --template-body file://cloudformation-template.yaml \
     --capabilities CAPABILITY_IAM \
     --parameters ParameterKey=Environment,ParameterValue=dev
   ```

4. After deployment, the CodeGate API and Dashboard URLs will be available in the CloudFormation stack outputs.

### User Isolation

The AWS implementation supports user isolation through:

1. **Dedicated CodeGate ECS Tasks** for users or groups
2. **Partitioned Database Access** with row-level security
3. **Isolated EFS Directories** with IAM-based access controls
4. **Load Balancer Routing** based on user identity

## Integration with Existing Bedrock Architecture

CodeGate integrates with the existing AWS Bedrock architecture by:

1. Using the existing API Gateway as the entry point to Bedrock models
2. Leveraging the Lambda intermediary for role assumption and security
3. Maintaining the security boundary between users and foundation models
4. Providing additional security features on top of the existing architecture

## Documentation

- `codegate-integration-design.md`: Detailed architecture and design document
- `diagrams/`: Architecture and flow diagrams

## Security Considerations

- All external connections use TLS 1.2+ encryption
- Secrets and sensitive data are stored in AWS Secrets Manager
- All data at rest is encrypted (EFS, RDS)
- Proper IAM roles with least privilege principles
- Network isolation using VPC, security groups, and private subnets

## Support

For questions or issues, please contact the architecture team.
