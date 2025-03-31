# Architecture Layers Diagram

## Layer Documentation

| Layer | Components | Purpose | Key Security Aspects |
|-------|------------|---------|---------------------|
| Client Layer | VS Code Extension | Interface for developers | Secure credential handling |
| Access/Control Layer | Private API Gateway | Secure entry point | IAM auth, request validation |
| Authentication Layer | AWS Identity Center, IAM | User identity and permissions | SSO, least privilege access |
| Orchestration Layer | Lambda Function | Request processing | Role assumption, validation |
| AI Service Layer | AWS Bedrock | LLM processing | Controlled model access |
| Networking Layer | VPC, Endpoints, Security Groups | Network isolation | Private connectivity |
| Governance Layer | CloudTrail, CloudWatch | Auditing and monitoring | Immutable logs, alerting |

## Mermaid Diagram

```mermaid
flowchart TD
    subgraph "Corporate Network"
        Developer["Developer\nWorkstation"]
        VSCode["VS Code + Extension"]
    end
    
    subgraph "AWS Environment"
        subgraph "Client Layer"
            VSCodeExt["VS Code Extension\n(AWS SDK)"]
        end
        
        subgraph "Authentication Layer"
            IdCenter["AWS Identity Center\n(SSO Authentication)"]
            IAMRoles["IAM Roles\n(Permissions)"]
            SCPs["Service Control\nPolicies"]
        end
        
        subgraph "Access Layer"
            APIGateway["Private API Gateway\n(IAM Auth + Validation)"]
        end
        
        subgraph "Orchestration Layer"
            Lambda["Lambda Function\n(Proxy + Logic)"]
        end
        
        subgraph "AI Service Layer"
            Bedrock["AWS Bedrock\n(Foundation Models)"]
        end
        
        subgraph "Networking Layer"
            VPC["Virtual Private Cloud"]
            PrivateSubnets["Private Subnets"]
            SecurityGroups["Security Groups"]
            VPCEndpoints["VPC Endpoints\n(PrivateLink)"]
        end
        
        subgraph "Governance Layer"
            CloudTrail["CloudTrail\n(API Auditing)"]
            CloudWatch["CloudWatch\n(Logs + Metrics)"]
            Reporting["Custom Reporting\n(Usage + Compliance)"]
        end
    end
    
    Developer --> VSCode
    VSCode --> VSCodeExt
    
    VSCodeExt -->|"1. Authenticate"| IdCenter
    IdCenter -->|"2. Map to IAM Role"| IAMRoles
    IAMRoles -->|"3. Temporary\nCredentials"| VSCodeExt
    
    VSCodeExt -->|"4. Signed\nRequest"| APIGateway
    APIGateway -->|"5. Proxy\nRequest"| Lambda
    
    Lambda -->|"6. Assume Role\n+ Call Bedrock"| Bedrock
    Bedrock -->|"7. Response"| Lambda
    Lambda -->|"8. Formatted\nResponse"| APIGateway
    APIGateway -->|"9. Return\nResponse"| VSCodeExt
    
    Lambda -.->|"Run in"| PrivateSubnets
    PrivateSubnets -.->|"Part of"| VPC
    PrivateSubnets -.->|"Protected by"| SecurityGroups
    VPCEndpoints -.->|"Secure access to\nAWS Services"| VPC
    
    Lambda -->|"Log"| CloudWatch
    APIGateway -->|"Log API Calls"| CloudTrail
    CloudWatch -->|"Feed"| Reporting
    
    SCPs -.->|"Guardrails"| IAMRoles
    SCPs -.->|"Restrict"| Bedrock
    
    classDef corpNet fill:#e4f7fb,stroke:#333,stroke-width:1px;
    classDef clientLayer fill:#f9d5e5,stroke:#333,stroke-width:1px;
    classDef authLayer fill:#eeeeee,stroke:#333,stroke-width:1px;
    classDef accessLayer fill:#dddddd,stroke:#333,stroke-width:1px;
    classDef orchestrationLayer fill:#d3b5e5,stroke:#333,stroke-width:1px;
    classDef aiLayer fill:#b5e5d3,stroke:#333,stroke-width:1px;
    classDef networkLayer fill:#d0e8f2,stroke:#333,stroke-width:1px;
    classDef govLayer fill:#f2e8d0,stroke:#333,stroke-width:1px;
    
    class Developer,VSCode corpNet;
    class VSCodeExt clientLayer;
    class IdCenter,IAMRoles,SCPs authLayer;
    class APIGateway accessLayer;
    class Lambda orchestrationLayer;
    class Bedrock aiLayer;
    class VPC,PrivateSubnets,SecurityGroups,VPCEndpoints networkLayer;
    class CloudTrail,CloudWatch,Reporting govLayer;
```

## Rendered Diagram Image

![Architecture Layers Diagram](images/arch-layers.png)
*Architecture Layers Diagram showing the architectural layers and components from developers to Bedrock*

## Layer Descriptions

### Client Layer
The Client Layer consists of the VS Code extension on the developer's workstation, which provides an interface for developers to interact with the AWS Bedrock LLMs. It handles authentication via AWS Identity Center and formats API requests.

### Authentication Layer
The Authentication Layer manages identity and access control through AWS Identity Center, IAM roles and permissions, and Service Control Policies (SCPs). It ensures only authorized developers can access the API and enforces least-privilege access.

### Access/Control Layer
The Access Layer provides a secure entry point through a private API Gateway. It enforces IAM authentication, validates requests against schemas, and implements throttling and resource policies.

### Orchestration Layer
The Orchestration Layer, implemented as a Lambda function, handles request processing, input validation, role assumption for Bedrock access, and logging. It acts as a critical security boundary and business logic processor.

### AI Service Layer
The AI Service Layer consists of AWS Bedrock, which provides access to foundation models. Access is restricted via IAM policies and SCPs, ensuring only approved models can be used.

### Networking Layer
The Networking Layer provides network isolation through a VPC, private subnets, security groups, and VPC endpoints (PrivateLink), ensuring traffic stays within the AWS network backbone and doesn't traverse the public internet.

### Governance Layer
The Governance Layer manages auditing and monitoring through CloudTrail (for API auditing), CloudWatch (for logs and metrics), and custom reporting solutions for usage tracking and compliance.
