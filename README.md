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

### 1. Initialize Project

```bash
mkdir agentcore-vpc && cd agentcore-vpc
cdk init app --language typescript
npm install
```

### 2. Create Stack File

Create `lib/agentcore-vpc-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import { Construct } from 'constructs';

export class AgentcoreVpcStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Create VPC with NO NAT Gateway (private only, using VPC endpoints)
    const vpc = new ec2.Vpc(this, 'AgentCoreVpc', {
      ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),
      maxAzs: 2,
      natGateways: 0,
      subnetConfiguration: [
        {
          name: 'Isolated',
          subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
          cidrMask: 24,
        },
      ],
      enableDnsHostnames: true,
      enableDnsSupport: true,
    });

    // Get isolated subnets
    const isolatedSubnets = vpc.selectSubnets({
      subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
    }).subnets;

    // Security group for agent runtime
    const agentRuntimeSecurityGroup = new ec2.SecurityGroup(
      this,
      'AgentRuntimeSG',
      {
        vpc: vpc,
        description: 'Security group for agent runtime',
        allowAllOutbound: true,
      }
    );

    // Add port 8080 ingress for health checks
    agentRuntimeSecurityGroup.addIngressRule(
      ec2.Peer.ipv4(vpc.vpcCidrBlock),
      ec2.Port.tcp(8080),
      'Allow port 8080 for health checks'
    );

    // Security group for VPC endpoints
    const vpcEndpointSecurityGroup = new ec2.SecurityGroup(
      this,
      'VpcEndpointSG',
      {
        vpc: vpc,
        description: 'Security group for VPC endpoints',
        allowAllOutbound: true,
      }
    );

    // Allow HTTPS from agent runtime to VPC endpoints
    vpcEndpointSecurityGroup.addIngressRule(
      agentRuntimeSecurityGroup,
      ec2.Port.tcp(443),
      'Allow HTTPS from agent runtime'
    );

    // ECR API endpoint
    vpc.addInterfaceEndpoint('EcrApiEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.ECR,
      securityGroups: [vpcEndpointSecurityGroup],
      privateDnsEnabled: true,
    });

    // ECR Docker endpoint
    vpc.addInterfaceEndpoint('EcrDockerEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.ECR_DOCKER,
      securityGroups: [vpcEndpointSecurityGroup],
      privateDnsEnabled: true,
    });

    // CloudWatch Logs endpoint
    vpc.addInterfaceEndpoint('CloudWatchLogsEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.CLOUDWATCH_LOGS,
      securityGroups: [vpcEndpointSecurityGroup],
      privateDnsEnabled: true,
    });

    // Bedrock Runtime endpoint
    vpc.addInterfaceEndpoint('BedrockRuntimeEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.BEDROCK_RUNTIME,
      securityGroups: [vpcEndpointSecurityGroup],
      privateDnsEnabled: true,
    });

    // S3 Gateway endpoint
    vpc.addGatewayEndpoint('S3Endpoint', {
      service: ec2.GatewayVpcEndpointAwsService.S3,
    });

    // Outputs
    new cdk.CfnOutput(this, 'VpcId', {
      value: vpc.vpcId,
      description: 'VPC ID',
      exportName: 'AgentCoreVpcId',
    });

    new cdk.CfnOutput(this, 'AgentRuntimeSecurityGroupId', {
      value: agentRuntimeSecurityGroup.securityGroupId,
      description: 'Agent Runtime Security Group ID',
      exportName: 'AgentCoreSecurityGroupId',
    });

    new cdk.CfnOutput(this, 'VpcEndpointSecurityGroupId', {
      value: vpcEndpointSecurityGroup.securityGroupId,
      description: 'VPC Endpoint Security Group ID',
    });

    if (isolatedSubnets.length > 0) {
      new cdk.CfnOutput(this, 'PrivateSubnet1', {
        value: isolatedSubnets[0].subnetId,
        description: 'Private Subnet 1 ID',
        exportName: 'AgentCoreSubnet1',
      });
    }

    if (isolatedSubnets.length > 1) {
      new cdk.CfnOutput(this, 'PrivateSubnet2', {
        value: isolatedSubnets[1].subnetId,
        description: 'Private Subnet 2 ID',
        exportName: 'AgentCoreSubnet2',
      });
    }

    new cdk.CfnOutput(this, 'PrivateSubnetIds', {
      value: isolatedSubnets.map(s => s.subnetId).join(','),
      description: 'All Private Subnet IDs (comma-separated)',
    });
  }
}
```

### 3. Update App Entry Point

Update `bin/agentcore-vpc.ts`:

```typescript
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { AgentcoreVpcStack } from '../lib/agentcore-vpc-stack';

const app = new cdk.App();

new AgentcoreVpcStack(app, 'AgentcoreVpcStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: 'us-west-2',
  },
});
```

### 4. Deploy

```bash
# Bootstrap CDK (first time only)
cdk bootstrap

# Deploy
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
