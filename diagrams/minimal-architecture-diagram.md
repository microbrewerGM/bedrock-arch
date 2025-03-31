# VS Code to Bedrock Minimal Architecture

```mermaid
graph LR
    VS["VS Code\nExtensions"] --> AT["AWS Toolkit\nAuthentication"]
    AT --> AG["Private API Gateway"]
    AG --> BR["Amazon Bedrock"]
    BR --> LLM["LLM Models"]
    
    VS -.-> IAM["IAM/Identity Center"]
    AG -.-> IAM
    BR -.-> BR_IAM["IAM Role for\nBedrock Access"]
    
    classDef main fill:#2374ab,stroke:#013a63,color:#ffffff;
    classDef auth fill:#80a4ed,stroke:#3d5a80,color:#ffffff;
    classDef aws fill:#ff9900,stroke:#ec7211,color:#ffffff;
    
    class VS,AG main;
    class AT,IAM,BR_IAM auth;
    class BR,LLM aws;
```

This diagram illustrates the minimal secure architecture connecting VS Code to Amazon Bedrock:

1. **VS Code Extensions** use the AWS Toolkit for authentication and connect to a Private API Gateway
2. **API Gateway** provides a secure endpoint that connects directly to Amazon Bedrock
3. **IAM/Identity Center** manages authentication and permissions throughout the flow
4. **Amazon Bedrock** processes LLM requests and returns responses to VS Code

The solid lines show the request flow, while dotted lines indicate authentication/permission relationships.

This design eliminates unnecessary components (like intermediate Lambda functions) while maintaining security through private connections, proper IAM controls, and API Gateway's direct service integration capabilities.
