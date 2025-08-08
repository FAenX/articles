# Infrastructure & Backend Guidelines: Enterprise Deployment Standards

## Executive Summary

This guide provides comprehensive infrastructure and backend implementation standards for enterprise applications, covering containerization with AWS Fargate, Aurora database management, CloudFormation infrastructure as code, and multi-environment deployment patterns. These guidelines ensure scalability, security, cost optimization, and portability across different deployment environments.

```mermaid
graph TD
    A[Infrastructure & Backend] --> B[Containerization]
    A --> C[AWS Infrastructure]
    A --> D[Database Management]
    A --> E[Deployment Strategy]

    B --> B1[Docker Compose]
    B --> B2[AWS Fargate]
    B --> B3[Container Orchestration]

    C --> C1[CloudFormation]
    C --> C2[VPC & Networking]
    C --> C3[Security & IAM]

    D --> D1[Aurora PostgreSQL]
    D --> D2[Local Development]
    D --> D3[Data Migration]

    E --> E1[Multi-Environment]
    E --> E2[CI/CD Pipeline]
    E --> E3[Monitoring & Logging]

    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style C fill:#e8f5e8
    style D fill:#fff3e0
    style E fill:#fce4ec
```

---

## 1. Repository Structure

### 1.1 Recommended Organization

```
enterprise-platform/
├── frontend/                          # UI Implementation
│   ├── src/
│   ├── public/
│   ├── package.json
│   └── README.md
│
├── backend/                           # Backend Services
│   ├── services/
│   │   ├── user-service/
│   │   │   ├── src/
│   │   │   ├── Dockerfile
│   │   │   ├── package.json
│   │   │   └── tests/
│   │   ├── product-service/
│   │   └── order-service/
│   ├── shared/
│   │   ├── middleware/
│   │   ├── utils/
│   │   └── types/
│   └── docker-compose.yml
│
├── infrastructure/                    # Infrastructure as Code
│   ├── cloudformation/
│   │   ├── templates/
│   │   │   ├── vpc.yml
│   │   │   ├── aurora.yml
│   │   │   ├── fargate.yml
│   │   │   ├── alb.yml
│   │   │   └── security.yml
│   │   ├── scripts/
│   │   └── environments/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── prod/
│   ├── terraform/                     # Alternative IaC
│   └── scripts/
│
├── deployment/                        # Deployment Configuration
│   ├── docker/
│   │   ├── Dockerfile.base
│   │   └── docker-compose.yml
│   ├── kubernetes/                    # K8s manifests (optional)
│   ├── scripts/
│   └── configs/
│
├── docs/                             # Documentation
│   ├── architecture/
│   ├── api/
│   ├── deployment/
│   └── troubleshooting/
│
├── scripts/                          # Utility Scripts
│   ├── setup.sh
│   ├── deploy.sh
│   └── backup.sh
│
└── README.md
```

---

## 2. Containerization Strategy

### 2.1 Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Database Services
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: enterprise_dev
      POSTGRES_USER: dev_user
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - enterprise-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - enterprise-network

  # Backend Services
  user-service:
    build:
      context: ./backend/services/user-service
      dockerfile: Dockerfile
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://dev_user:dev_password@postgres:5432/enterprise_dev
      REDIS_URL: redis://redis:6379
    ports:
      - "3001:3000"
    depends_on:
      - postgres
      - redis
    networks:
      - enterprise-network
    volumes:
      - ./backend/services/user-service:/app
      - /app/node_modules

  product-service:
    build:
      context: ./backend/services/product-service
      dockerfile: Dockerfile
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://dev_user:dev_password@postgres:5432/enterprise_dev
      REDIS_URL: redis://redis:6379
    ports:
      - "3002:3000"
    depends_on:
      - postgres
      - redis
    networks:
      - enterprise-network
    volumes:
      - ./backend/services/product-service:/app
      - /app/node_modules

  # Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    environment:
      REACT_APP_API_URL: http://localhost:3001
      REACT_APP_WS_URL: ws://localhost:3001
    ports:
      - "3000:3000"
    depends_on:
      - user-service
      - product-service
    networks:
      - enterprise-network
    volumes:
      - ./frontend:/app
      - /app/node_modules

  # Monitoring
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - enterprise-network

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - enterprise-network

volumes:
  postgres_data:
  grafana_data:

networks:
  enterprise-network:
    driver: bridge
```

### 2.2 Multi-Stage Dockerfile

```dockerfile
# backend/services/user-service/Dockerfile
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build the application
RUN npm run build

# Production image, copy all the files and run the app
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production

# Create a non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built application
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package*.json ./

# Change ownership
RUN chown -R nextjs:nodejs /app
USER nextjs

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "dist/index.js"]
```

---

## 3. AWS Infrastructure with CloudFormation

### 3.1 VPC Configuration

```yaml
# infrastructure/cloudformation/templates/vpc.yml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Enterprise Platform VPC with Public and Private Subnets'

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: Environment name

  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
    Description: CIDR block for VPC

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-enterprise-vpc'
        - Key: Environment
          Value: !Ref Environment

  # Public Subnets
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-1'

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-2'

  # Private Subnets
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.3.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-subnet-1'

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.4.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-subnet-2'

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-igw'

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  # NAT Gateway
  NatGatewayEIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-gateway'

  # Route Tables
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-routes'

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-routes'

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway

  # Route Table Associations
  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRouteTable

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${Environment}-vpc-id'

  PublicSubnets:
    Description: Public Subnets
    Value: !Join [',', [!Ref PublicSubnet1, !Ref PublicSubnet2]]
    Export:
      Name: !Sub '${Environment}-public-subnets'

  PrivateSubnets:
    Description: Private Subnets
    Value: !Join [',', [!Ref PrivateSubnet1, !Ref PrivateSubnet2]]
    Export:
      Name: !Sub '${Environment}-private-subnets'
```

### 3.2 Aurora Database Configuration

```yaml
# infrastructure/cloudformation/templates/aurora.yml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Aurora PostgreSQL Database Cluster'

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: Environment name

  DBName:
    Type: String
    Default: enterprise
    Description: Database name

  DBUsername:
    Type: String
    Default: admin
    Description: Database master username

  DBPassword:
    Type: String
    NoEcho: true
    Description: Database master password

  DBInstanceClass:
    Type: String
    Default: db.r6g.large
    Description: Database instance class

  DBClusterSize:
    Type: Number
    Default: 2
    MinValue: 1
    MaxValue: 15
    Description: Number of database instances in cluster

Resources:
  # Security Group for Aurora
  AuroraSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for Aurora database
      VpcId:
        Fn::ImportValue: !Sub '${Environment}-vpc-id'
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref FargateSecurityGroup

  # Subnet Group
  AuroraSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Aurora subnet group
      SubnetIds:
        Fn::Split:
          - ','
          - Fn::ImportValue: !Sub '${Environment}-private-subnets'
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-aurora-subnet-group'

  # Parameter Group
  AuroraParameterGroup:
    Type: AWS::RDS::DBParameterGroup
    Properties:
      Description: Aurora PostgreSQL parameter group
      Family: aurora-postgresql15
      Parameters:
        shared_preload_libraries: 'pg_stat_statements'
        log_statement: 'all'
        log_min_duration_statement: '1000'

  # Cluster Parameter Group
  AuroraClusterParameterGroup:
    Type: AWS::RDS::DBClusterParameterGroup
    Properties:
      Description: Aurora PostgreSQL cluster parameter group
      Family: aurora-postgresql15
      Parameters:
        aurora_enable_replica_logging: '1'

  # Aurora Cluster
  AuroraCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      DBClusterIdentifier: !Sub '${Environment}-aurora-cluster'
      Engine: aurora-postgresql
      EngineVersion: '15.4'
      EngineMode: provisioned
      DatabaseName: !Ref DBName
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      DBSubnetGroupName: !Ref AuroraSubnetGroup
      VpcSecurityGroupIds:
        - !Ref AuroraSecurityGroup
      DBClusterParameterGroupName: !Ref AuroraClusterParameterGroup
      BackupRetentionPeriod: 7
      PreferredBackupWindow: '03:00-04:00'
      PreferredMaintenanceWindow: 'sun:04:00-sun:05:00'
      DeletionProtection: !If [IsProd, true, false]
      StorageEncrypted: true
      KmsKeyId: !Ref AuroraKMSKey
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-aurora-cluster'
        - Key: Environment
          Value: !Ref Environment

  # Aurora Instances
  AuroraPrimaryInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: !Sub '${Environment}-aurora-primary'
      DBClusterIdentifier: !Ref AuroraCluster
      Engine: aurora-postgresql
      DBInstanceClass: !Ref DBInstanceClass
      PubliclyAccessible: false
      AutoMinorVersionUpgrade: true
      MonitoringInterval: 60
      MonitoringRoleArn: !GetAtt AuroraMonitoringRole.Arn
      PerformanceInsightsEnabled: true
      PerformanceInsightsRetentionPeriod: 7
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-aurora-primary'

  AuroraReplicaInstance:
    Type: AWS::RDS::DBInstance
    Condition: HasReplica
    Properties:
      DBInstanceIdentifier: !Sub '${Environment}-aurora-replica'
      DBClusterIdentifier: !Ref AuroraCluster
      Engine: aurora-postgresql
      DBInstanceClass: !Ref DBInstanceClass
      PubliclyAccessible: false
      AutoMinorVersionUpgrade: true
      MonitoringInterval: 60
      MonitoringRoleArn: !GetAtt AuroraMonitoringRole.Arn
      PerformanceInsightsEnabled: true
      PerformanceInsightsRetentionPeriod: 7
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-aurora-replica'

  # KMS Key for Encryption
  AuroraKMSKey:
    Type: AWS::KMS::Key
    Properties:
      Description: KMS key for Aurora encryption
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'
          - Sid: Allow CloudWatch Logs
            Effect: Allow
            Principal:
              Service: logs.region.amazonaws.com
            Action:
              - kms:Encrypt*
              - kms:Decrypt*
              - kms:ReEncrypt*
              - kms:GenerateDataKey*
              - kms:Describe*
            Resource: '*'

  # IAM Role for Monitoring
  AuroraMonitoringRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: monitoring.rds.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole

Conditions:
  IsProd: !Equals [!Ref Environment, 'prod']
  HasReplica: !GreaterThan [!Ref DBClusterSize, 1]

Outputs:
  AuroraClusterEndpoint:
    Description: Aurora cluster endpoint
    Value: !GetAtt AuroraCluster.Endpoint.Address
    Export:
      Name: !Sub '${Environment}-aurora-endpoint'

  AuroraClusterPort:
    Description: Aurora cluster port
    Value: !GetAtt AuroraCluster.Endpoint.Port
    Export:
      Name: !Sub '${Environment}-aurora-port'
```

### 3.3 Fargate Service Configuration

```yaml
# infrastructure/cloudformation/templates/fargate.yml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'AWS Fargate Service Configuration'

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: Environment name

  ServiceName:
    Type: String
    Description: Name of the service

  ContainerPort:
    Type: Number
    Default: 3000
    Description: Container port

  ContainerCpu:
    Type: Number
    Default: 256
    AllowedValues: [256, 512, 1024, 2048, 4096]
    Description: Container CPU units

  ContainerMemory:
    Type: Number
    Default: 512
    AllowedValues: [512, 1024, 2048, 4096, 8192, 16384, 30720]
    Description: Container memory in MiB

  DesiredCount:
    Type: Number
    Default: 2
    MinValue: 1
    MaxValue: 10
    Description: Desired number of tasks

Resources:
  # ECS Cluster
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub '${Environment}-enterprise-cluster'
      CapacityProviders:
        - FARGATE
        - FARGATE_SPOT
      DefaultCapacityProviderStrategy:
        - CapacityProvider: FARGATE
          Weight: 1
      Settings:
        - Name: containerInsights
          Value: enabled
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-enterprise-cluster'

  # Task Definition
  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub '${Environment}-${ServiceName}'
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      Cpu: !Ref ContainerCpu
      Memory: !Ref ContainerMemory
      ExecutionRoleArn: !GetAtt ECSExecutionRole.Arn
      TaskRoleArn: !GetAtt ECSTaskRole.Arn
      ContainerDefinitions:
        - Name: !Ref ServiceName
          Image: !Sub '${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${ServiceName}:latest'
          PortMappings:
            - ContainerPort: !Ref ContainerPort
              Protocol: tcp
          Environment:
            - Name: NODE_ENV
              Value: !Ref Environment
            - Name: AWS_REGION
              Value: !Ref AWS::Region
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref CloudWatchLogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: ecs
          HealthCheck:
            Command:
              - CMD-SHELL
              - curl -f http://localhost:!Ref ContainerPort/health || exit 1
            Interval: 30
            Timeout: 5
            Retries: 3
            StartPeriod: 60

  # ECS Service
  ECSService:
    Type: AWS::ECS::Service
    Properties:
      ServiceName: !Sub '${Environment}-${ServiceName}'
      Cluster: !Ref ECSCluster
      TaskDefinition: !Ref TaskDefinition
      DesiredCount: !Ref DesiredCount
      LaunchType: FARGATE
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: DISABLED
          SecurityGroups:
            - !Ref FargateSecurityGroup
          Subnets:
            Fn::Split:
              - ','
              - Fn::ImportValue: !Sub '${Environment}-private-subnets'
      LoadBalancers:
        - ContainerName: !Ref ServiceName
          ContainerPort: !Ref ContainerPort
          TargetGroupArn: !Ref TargetGroup
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 100
        DeploymentCircuitBreaker:
          Enable: true
          Rollback: true

  # Application Load Balancer
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub '${Environment}-${ServiceName}-alb'
      Scheme: internet-facing
      Type: application
      IpAddressType: ipv4
      Subnets:
        Fn::Split:
          - ','
          - Fn::ImportValue: !Sub '${Environment}-public-subnets'
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-${ServiceName}-alb'

  # Target Group
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub '${Environment}-${ServiceName}-tg'
      Port: !Ref ContainerPort
      Protocol: HTTP
      TargetType: ip
      VpcId:
        Fn::ImportValue: !Sub '${Environment}-vpc-id'
      HealthCheckPath: /health
      HealthCheckIntervalSeconds: 30
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-${ServiceName}-tg'

  # Listener
  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP

  # Security Groups
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for ALB
      VpcId:
        Fn::ImportValue: !Sub '${Environment}-vpc-id'
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  FargateSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for Fargate tasks
      VpcId:
        Fn::ImportValue: !Sub '${Environment}-vpc-id'
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: !Ref ContainerPort
          ToPort: !Ref ContainerPort
          SourceSecurityGroupId: !Ref ALBSecurityGroup

  # IAM Roles
  ECSExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
      Policies:
        - PolicyName: ECSExecutionPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ecr:GetAuthorizationToken
                  - ecr:BatchCheckLayerAvailability
                  - ecr:GetDownloadUrlForLayer
                  - ecr:BatchGetImage
                Resource: '*'

  ECSTaskRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
        - arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

  # CloudWatch Log Group
  CloudWatchLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/ecs/${Environment}-${ServiceName}'
      RetentionInDays: 30

Outputs:
  ServiceURL:
    Description: Service URL
    Value: !Sub 'http://${ApplicationLoadBalancer.DNSName}'
    Export:
      Name: !Sub '${Environment}-${ServiceName}-url'

  ECSClusterName:
    Description: ECS Cluster Name
    Value: !Ref ECSCluster
    Export:
      Name: !Sub '${Environment}-ecs-cluster'
```

---

## 4. Multi-Environment Deployment

### 4.1 Environment Configuration

```yaml
# infrastructure/cloudformation/environments/dev/parameters.json
{
  "Parameters": {
    "Environment": "dev",
    "DBInstanceClass": "db.r6g.large",
    "DBClusterSize": "1",
    "ContainerCpu": "256",
    "ContainerMemory": "512",
    "DesiredCount": "1"
  }
}

# infrastructure/cloudformation/environments/staging/parameters.json
{
  "Parameters": {
    "Environment": "staging",
    "DBInstanceClass": "db.r6g.large",
    "DBClusterSize": "2",
    "ContainerCpu": "512",
    "ContainerMemory": "1024",
    "DesiredCount": "2"
  }
}

# infrastructure/cloudformation/environments/prod/parameters.json
{
  "Parameters": {
    "Environment": "prod",
    "DBInstanceClass": "db.r6g.xlarge",
    "DBClusterSize": "3",
    "ContainerCpu": "1024",
    "ContainerMemory": "2048",
    "DesiredCount": "3"
  }
}
```

### 4.2 Deployment Scripts

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

ENVIRONMENT=$1
SERVICE=$2
ACTION=$3

if [ -z "$ENVIRONMENT" ] || [ -z "$SERVICE" ] || [ -z "$ACTION" ]; then
    echo "Usage: $0 <environment> <service> <action>"
    echo "Environments: dev, staging, prod"
    echo "Services: user-service, product-service, order-service"
    echo "Actions: deploy, update, destroy"
    exit 1
fi

# Validate environment
if [[ ! "$ENVIRONMENT" =~ ^(dev|staging|prod)$ ]]; then
    echo "Invalid environment: $ENVIRONMENT"
    exit 1
fi

# Validate service
if [[ ! "$SERVICE" =~ ^(user-service|product-service|order-service)$ ]]; then
    echo "Invalid service: $SERVICE"
    exit 1
fi

# Set AWS region
export AWS_REGION=${AWS_REGION:-us-east-1}

# Deploy infrastructure
deploy_infrastructure() {
    echo "Deploying infrastructure for $ENVIRONMENT..."
    
    # Deploy VPC
    aws cloudformation deploy \
        --template-file infrastructure/cloudformation/templates/vpc.yml \
        --stack-name $ENVIRONMENT-enterprise-vpc \
        --parameter-overrides Environment=$ENVIRONMENT \
        --capabilities CAPABILITY_IAM \
        --region $AWS_REGION

    # Deploy Aurora
    aws cloudformation deploy \
        --template-file infrastructure/cloudformation/templates/aurora.yml \
        --stack-name $ENVIRONMENT-enterprise-aurora \
        --parameter-overrides \
            Environment=$ENVIRONMENT \
            DBPassword=$(aws secretsmanager get-secret-value --secret-id $ENVIRONMENT/db/password --query SecretString --output text) \
        --capabilities CAPABILITY_IAM \
        --region $AWS_REGION

    # Deploy Fargate service
    aws cloudformation deploy \
        --template-file infrastructure/cloudformation/templates/fargate.yml \
        --stack-name $ENVIRONMENT-enterprise-$SERVICE \
        --parameter-overrides \
            Environment=$ENVIRONMENT \
            ServiceName=$SERVICE \
        --capabilities CAPABILITY_IAM \
        --region $AWS_REGION
}

# Build and push Docker image
build_and_push_image() {
    echo "Building and pushing Docker image for $SERVICE..."
    
    # Get ECR repository URI
    REPO_URI=$(aws ecr describe-repositories --repository-names $SERVICE --query 'repositories[0].repositoryUri' --output text 2>/dev/null || \
                aws ecr create-repository --repository-name $SERVICE --query 'repository.repositoryUri' --output text)
    
    # Login to ECR
    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $REPO_URI
    
    # Build image
    docker build -t $SERVICE ./backend/services/$SERVICE
    
    # Tag image
    docker tag $SERVICE:latest $REPO_URI:latest
    docker tag $SERVICE:latest $REPO_URI:$ENVIRONMENT-$(date +%Y%m%d-%H%M%S)
    
    # Push image
    docker push $REPO_URI:latest
    docker push $REPO_URI:$ENVIRONMENT-$(date +%Y%m%d-%H%M%S)
}

# Update ECS service
update_service() {
    echo "Updating ECS service for $SERVICE..."
    
    # Force new deployment
    aws ecs update-service \
        --cluster $ENVIRONMENT-enterprise-cluster \
        --service $ENVIRONMENT-$SERVICE \
        --force-new-deployment \
        --region $AWS_REGION
}

# Main deployment logic
case $ACTION in
    "deploy")
        deploy_infrastructure
        build_and_push_image
        update_service
        ;;
    "update")
        build_and_push_image
        update_service
        ;;
    "destroy")
        echo "Destroying $ENVIRONMENT-$SERVICE..."
        aws cloudformation delete-stack \
            --stack-name $ENVIRONMENT-enterprise-$SERVICE \
            --region $AWS_REGION
        ;;
    *)
        echo "Invalid action: $ACTION"
        exit 1
        ;;
esac

echo "Deployment completed successfully!"
```

---

## 5. Cost Optimization Strategies

### 5.1 Fargate Spot Instances

```yaml
# infrastructure/cloudformation/templates/fargate-spot.yml
Resources:
  # Spot Task Definition
  SpotTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub '${Environment}-${ServiceName}-spot'
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      Cpu: !Ref ContainerCpu
      Memory: !Ref ContainerMemory
      ExecutionRoleArn: !GetAtt ECSExecutionRole.Arn
      TaskRoleArn: !GetAtt ECSTaskRole.Arn
      CapacityProviderStrategy:
        - CapacityProvider: FARGATE_SPOT
          Weight: 1
        - CapacityProvider: FARGATE
          Weight: 0
      ContainerDefinitions:
        - Name: !Ref ServiceName
          Image: !Sub '${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${ServiceName}:latest'
          PortMappings:
            - ContainerPort: !Ref ContainerPort
              Protocol: tcp
          Environment:
            - Name: NODE_ENV
              Value: !Ref Environment
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref CloudWatchLogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: ecs
```

### 5.2 Aurora Serverless v2

```yaml
# infrastructure/cloudformation/templates/aurora-serverless.yml
Resources:
  AuroraServerlessCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      DBClusterIdentifier: !Sub '${Environment}-aurora-serverless'
      Engine: aurora-postgresql
      EngineVersion: '15.4'
      EngineMode: provisioned
      ServerlessV2ScalingConfiguration:
        MinCapacity: 0.5
        MaxCapacity: 16
      DatabaseName: !Ref DBName
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      DBSubnetGroupName: !Ref AuroraSubnetGroup
      VpcSecurityGroupIds:
        - !Ref AuroraSecurityGroup
      BackupRetentionPeriod: 7
      StorageEncrypted: true
      DeletionProtection: !If [IsProd, true, false]
```

---

## 6. Security Best Practices

### 6.1 IAM Policies

```yaml
# infrastructure/cloudformation/templates/security.yml
Resources:
  # ECS Task Role with minimal permissions
  ECSTaskRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: ECSTaskPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/ecs/${Environment}-*'
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                Resource: !Sub 'arn:aws:s3:::${Environment}-enterprise-bucket/*'
```

### 6.2 Security Groups

```yaml
# infrastructure/cloudformation/templates/security-groups.yml
Resources:
  # Restrictive security group for Fargate
  FargateSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for Fargate tasks
      VpcId:
        Fn::ImportValue: !Sub '${Environment}-vpc-id'
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: !Ref ContainerPort
          ToPort: !Ref ContainerPort
          SourceSecurityGroupId: !Ref ALBSecurityGroup
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref AuroraSecurityGroup
```

---

## 7. Monitoring and Logging

### 7.1 CloudWatch Configuration

```yaml
# infrastructure/cloudformation/templates/monitoring.yml
Resources:
  # CloudWatch Dashboard
  CloudWatchDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: !Sub '${Environment}-enterprise-dashboard'
      DashboardBody: !Sub |
        {
          "widgets": [
            {
              "type": "metric",
              "x": 0,
              "y": 0,
              "width": 12,
              "height": 6,
              "properties": {
                "metrics": [
                  ["AWS/ECS", "CPUUtilization", "ServiceName", "${Environment}-user-service"],
                  ["AWS/ECS", "MemoryUtilization", "ServiceName", "${Environment}-user-service"]
                ],
                "view": "timeSeries",
                "stacked": false,
                "region": "${AWS::Region}",
                "title": "ECS Service Metrics"
              }
            }
          ]
        }

  # CloudWatch Alarms
  HighCPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${Environment}-high-cpu-alarm'
      AlarmDescription: High CPU utilization
      MetricName: CPUUtilization
      Namespace: AWS/ECS
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: ServiceName
          Value: !Sub '${Environment}-user-service'
        - Name: ClusterName
          Value: !Sub '${Environment}-enterprise-cluster'
```

---

## 8. Implementation Checklist

### Infrastructure Setup
- [ ] Set up VPC with public and private subnets
- [ ] Configure Aurora PostgreSQL cluster
- [ ] Set up ECS Fargate cluster
- [ ] Configure Application Load Balancer
- [ ] Set up CloudWatch monitoring

### Security Configuration
- [ ] Implement least privilege IAM policies
- [ ] Configure security groups
- [ ] Enable encryption at rest and in transit
- [ ] Set up Secrets Manager for sensitive data
- [ ] Configure VPC endpoints for AWS services

### Deployment Pipeline
- [ ] Set up CI/CD pipeline
- [ ] Configure multi-environment deployment
- [ ] Implement blue-green deployment
- [ ] Set up automated testing
- [ ] Configure rollback procedures

### Cost Optimization
- [ ] Implement Fargate Spot instances
- [ ] Configure Aurora Serverless v2
- [ ] Set up auto-scaling policies
- [ ] Monitor and optimize resource usage
- [ ] Implement cost alerts

### Monitoring and Logging
- [ ] Set up CloudWatch dashboards
- [ ] Configure log aggregation
- [ ] Set up alerting and notifications
- [ ] Implement distributed tracing
- [ ] Configure performance monitoring

---

## Conclusion

This infrastructure and backend guidelines provide a comprehensive foundation for building scalable, secure, and cost-effective enterprise applications using AWS services. The container-first approach with Fargate ensures portability across different deployment environments, while the CloudFormation templates provide infrastructure as code for consistent deployments.

### Key Takeaways

1. **Containerization**: Use Docker for consistent deployments across environments
2. **AWS Fargate**: Serverless container orchestration for cost optimization
3. **Aurora PostgreSQL**: Managed database service for scalability and reliability
4. **CloudFormation**: Infrastructure as code for consistent deployments
5. **Security**: Implement least privilege and encryption at all layers
6. **Cost Optimization**: Use Spot instances and Serverless v2 for cost savings
7. **Monitoring**: Comprehensive logging and monitoring for operational excellence

### Success Metrics

- **Deployment Time**: < 10 minutes for full environment deployment
- **Cost Efficiency**: 40-60% cost savings with Spot instances and Serverless
- **Security**: Zero security vulnerabilities in infrastructure
- **Availability**: 99.9% uptime with multi-AZ deployment
- **Scalability**: Auto-scaling based on demand

By following these guidelines, you can build enterprise-grade infrastructure that is scalable, secure, and cost-effective while maintaining the flexibility to deploy on-premises if needed.

---

**Last Updated**: January 2025  
**Version**: 1.0  
**Status**: Production Ready 