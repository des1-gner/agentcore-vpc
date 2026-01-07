# VPC Configuration for Amazon Bedrock AgentCore Runtime

This CDK project creates a secure VPC configuration for running Amazon Bedrock AgentCore Runtime in private subnets without internet access.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16)                        │
│                                                             │
│  ┌──────────────────┐         ┌──────────────────┐        │
│  │  Private Subnet  │         │  Private Subnet  │        │
│  │  10.0.0.0/24     │         │  10.0.1.0/24     │        │
│  │  (AZ-A)          │         │  (AZ-B)          │        │
│  └────────┬─────────┘         └────────┬─────────┘        │
│           │                            │                   │
│           │    ┌──────────────┐       │                   │
│           └────┤ AgentCore    ├───────┘                   │
│                │ Runtime       │                           │
│                │ (port 8080)   │                           │
│                └───────┬───────┘                           │
│                        │ HTTPS (443)                       │
│                        ↓                                   │
│           ┌────────────────────────┐                       │
│           │   VPC Endpoints        │                       │
│           │  - ECR API             │                       │
│           │  - ECR Docker          │                       │
│           │  - CloudWatch Logs     │                       │
│           │  - Bedrock Runtime     │                       │
│           │  - S3 (Gateway)        │                       │
│           └────────────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

## What This Creates

- **VPC** with private isolated subnets (no internet access)
- **VPC Endpoints** for ECR, CloudWatch Logs, Bedrock Runtime, and S3
- **Security Groups** with proper ingress/egress rules
- **2 Private Subnets** across different Availability Zones

## Prerequisites

- AWS CDK installed (`npm install -g aws-cdk`)
- Node.js 18+
- AWS CLI configured with credentials

## Deployment

```bash
# 1. Initialize project
mkdir agentcore-vpc && cd agentcore-vpc
cdk init app --language typescript

# 2. Copy the stack code to lib/agentcore-vpc-stack.ts

# 3. Install dependencies
npm install

# 4. Bootstrap CDK (first time only)
cdk bootstrap

# 5. Deploy
cdk deploy
```

## Outputs

After deployment, note these values for configuring your AgentCore runtime:

- `VpcId` - VPC ID
- `AgentRuntimeSecurityGroupId` - Security Group ID
- `PrivateSubnet1` - Subnet 1 ID
- `PrivateSubnet2` - Subnet 2 ID

## Using with AgentCore

Update your AgentCore runtime to use the created VPC, subnets, and security group from the CDK outputs.

## Cleanup

To remove all resources:

```bash
cdk destroy
```

## Additional Resources

- [AgentCore VPC Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-vpc.html)
- [VPC Endpoints Documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [AWS CDK Documentation](https://docs.aws.amazon.com/cdk/v2/guide/home.html)
