# Lambda Intermediary Role Diagram

## Role Documentation

| Function | Description | Security Aspect | Business Logic Aspect |
|----------|-------------|-----------------|----------------------|
| Security Boundary | Creates isolation between API Gateway and Bedrock | Prevents direct access to Bedrock | N/A |
| Role Assumption | Uses STS to assume dedicated Bedrock role | Follows least-privilege principle | N/A |
| Audit Trail | Logs detailed information about requests and responses | Creates clear evidence of who accessed what | Provides usage metrics and analytics |
| Input Validation | Validates inputs beyond schema validation | Prevents injection attacks | Ensures request quality |
| Model-Specific Formatting | Transforms requests to model-specific formats | N/A | Handles different model interfaces |
| Error Handling | Processes and standardizes errors | Hides internal errors from users | Provides meaningful error messages |
| Content Scanning | Optional scanning for sensitive data | Prevents PII leakage | Enforces content policies |

## Mermaid Diagram

```mermaid
flowchart TD
    subgraph UserAccess ["User Access Layer"]
        APIGateway["API Gateway\n(Private Endpoint)"]
    end
    
    subgraph SecurityLayer ["Security & Control Layer"]
        Lambda["Lambda Proxy Function"]
        CloudWatch["CloudWatch Logs"]
        Validation["Input/Output\nValidation"]
        ErrorHandling["Error Handling"]
        ModelFormatting["Model-Specific\nFormatting"]
    end
    
    subgraph IAMLayer ["IAM Security Layer"]
        STS["AWS STS"]
        AssumeRole["AssumeRole Operation\n(Bedrock-Access-Role)"]
    end
    
    subgraph ServiceLayer ["AI Service Layer"]
        Bedrock["AWS Bedrock\n(Foundation Models)"]
    end
    
    APIGateway -->|"Request\n(IAM Auth)"| Lambda
    Lambda -->|"1. Log Request"| CloudWatch
    Lambda -->|"2. Validate\nInput"| Validation
    Validation -->|"Valid"| Lambda
    
    Lambda -->|"3. Assume\nRole"| STS
    STS --> AssumeRole
    AssumeRole -->|"Temp Credentials"| Lambda
    
    Lambda -->|"4. Format for\nTarget Model"| ModelFormatting
    ModelFormatting --> Lambda
    
    Lambda -->|"5. Invoke\n(with assumed role)"| Bedrock
    Bedrock -->|"Response"| Lambda
    
    Lambda -->|"6. Handle Errors\n(if any)"| ErrorHandling
    ErrorHandling --> Lambda
    
    Lambda -->|"7. Log Response"| CloudWatch
    Lambda -->|"8. Return\nFormatted Response"| APIGateway
    
    classDef securityColor fill:#f96,stroke:#333,stroke-width:2px;
    classDef businessColor fill:#9cf,stroke:#333,stroke-width:2px;
    classDef serviceColor fill:#c9f,stroke:#333,stroke-width:2px;
    
    class Lambda,STS,AssumeRole,CloudWatch securityColor;
    class Validation,ModelFormatting,ErrorHandling businessColor;
    class Bedrock serviceColor;
```

## Rendered Diagram Image

![Lambda Intermediary Role Diagram](images/lambda-role.png)
*Lambda Intermediary Role Diagram illustrating the Lambda function's critical role as a security boundary and business logic processor*

## Benefits of Lambda Intermediary

1. **Security Isolation**
   * API Gateway cannot directly assume different IAM roles
   * Lambda creates a critical security boundary
   * Enables least-privilege permissions through role assumption
   * Creates clear audit trail of which entity accessed Bedrock

2. **Request/Response Processing**
   * Customized validation beyond API Gateway schema validation
   * Model-specific payload formatting for different LLM providers
   * Content filtering and PII detection (optional)
   * Standardized response formatting

3. **Operational Excellence**
   * Detailed structured logging for audit trails
   * Custom error handling and retry logic
   * Monitoring and alerting capabilities
   * Rate limiting and throttling options

4. **Client Simplification**
   * VS Code extension code remains simpler
   * Complex error handling centralized in Lambda
   * Consistent interface across multiple models
