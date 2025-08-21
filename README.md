# cloudfomation-templates

Awesome, Atul — here’s a **ready-to-use CloudFormation template kit** you can drop into a repo for training/projects. I’ve kept each template self-contained, production-friendly (tags, parameters, outputs), and documented. You can deploy any stack with:

```bash
# Validate
aws cloudformation validate-template --template-body file://vpc.yml

# Create / Update
aws cloudformation deploy \
  --template-file vpc.yml \
  --stack-name demo-vpc \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides Env=dev Project=inventory
```

> Tip: Use consistent `Env` and `Project` tags across stacks so you can filter resources easily.

---

# 1) vpc.yml — VPC, 2 AZ public/private subnets, NAT, routes

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: VPC with 2AZ public/private subnets, IGW, NAT, routes

Parameters:
  Env:
    Type: String
    Default: dev
  Project:
    Type: String
    Default: demo
  VPCCidr:
    Type: String
    Default: 10.0.0.0/16
  PublicSubnet1Cidr:
    Type: String
    Default: 10.0.1.0/24
  PublicSubnet2Cidr:
    Type: String
    Default: 10.0.2.0/24
  PrivateSubnet1Cidr:
    Type: String
    Default: 10.0.11.0/24
  PrivateSubnet2Cidr:
    Type: String
    Default: 10.0.12.0/24

Mappings:
  TagMap:
    Defaults:
      Owner: cloudnautic

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VPCCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - { Key: Name, Value: !Sub "${Project}-${Env}-vpc" }
        - { Key: Project, Value: !Ref Project }
        - { Key: Env, Value: !Ref Env }

  IGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - { Key: Name, Value: !Sub "${Project}-${Env}-igw" }

  VPCIGWAtt:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW

  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - { Key: Name, Value: !Sub "${Project}-${Env}-public-rt" }

  PublicRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref IGW

  EIPNat:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc

  NAT:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt EIPNat.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - { Key: Name, Value: !Sub "${Project}-${Env}-nat" }

  PrivateRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - { Key: Name, Value: !Sub "${Project}-${Env}-private-rt" }

  PrivateDefaultRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRT
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NAT

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnet1Cidr
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [0, !GetAZs ""]
      Tags:
        - { Key: Name, Value: !Sub "${Project}-${Env}-public-a" }

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnet2Cidr
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [1, !GetAZs ""]
      Tags:
        - { Key: Name, Value: !Sub "${Project}-${Env}-public-b" }

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PrivateSubnet1Cidr
      MapPublicIpOnLaunch: false
      AvailabilityZone: !Select [0, !GetAZs ""]
      Tags:
        - { Key: Name, Value: !Sub "${Project}-${Env}-private-a" }

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PrivateSubnet2Cidr
      MapPublicIpOnLaunch: false
      AvailabilityZone: !Select [1, !GetAZs ""]
      Tags:
        - { Key: Name, Value: !Sub "${Project}-${Env}-private-b" }

  AssocPub1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRT

  AssocPub2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRT

  AssocPri1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRT

  AssocPri2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRT

Outputs:
  VpcId: { Value: !Ref VPC, Export: { Name: !Sub "${Project}-${Env}-VpcId" } }
  PublicSubnetIds: { Value: !Join [",", [!Ref PublicSubnet1, !Ref PublicSubnet2]] }
  PrivateSubnetIds: { Value: !Join [",", [!Ref PrivateSubnet1, !Ref PrivateSubnet2]] }
```

---

# 2) s3-bucket.yml — Secure S3 with encryption, versioning, lifecycle, block public access

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: S3 bucket with SSE-S3, versioning, lifecycle, public access blocked

Parameters:
  BucketName:
    Type: String
  ExpireDays:
    Type: Number
    Default: 90

Resources:
  Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: ExpireOld
            Status: Enabled
            ExpirationInDays: !Ref ExpireDays
            NoncurrentVersionExpirationInDays: 30
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256

Outputs:
  BucketArn: { Value: !GetAtt Bucket.Arn }
```

---

# 3) iam-roles.yml — EC2 role with SSM access + InstanceProfile

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 IAM Role with SSM and CloudWatchAgent access

Resources:
  EC2Role:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "ec2-ssm-role-${AWS::StackName}"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: { Service: ec2.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

  EC2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles: [!Ref EC2Role]
      InstanceProfileName: !Sub "ec2-ssm-profile-${AWS::StackName}"

Outputs:
  InstanceProfileName: { Value: !Ref EC2InstanceProfile }
```

---

# 4) ec2.yml — Public EC2 with SG + UserData (Apache demo)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Public EC2 with Apache demo page

Parameters:
  VpcId: { Type: String }
  SubnetId: { Type: String }
  KeyName: { Type: AWS::EC2::KeyPair::KeyName }

Resources:
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP/SSH
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - { IpProtocol: tcp, FromPort: 22, ToPort: 22, CidrIp: 0.0.0.0/0 }
        - { IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0 }

  WebInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmi
      InstanceType: t3.micro
      KeyName: !Ref KeyName
      IamInstanceProfile: !Ref InstanceProfileName
      NetworkInterfaces:
        - DeviceIndex: "0"
          AssociatePublicIpAddress: true
          SubnetId: !Ref SubnetId
          GroupSet: [!Ref WebSG]
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          amazon-linux-extras install epel -y || true
          yum install -y httpd
          echo "<h1>Hello from CloudFormation EC2</h1>" > /var/www/html/index.html
          systemctl enable --now httpd
    Metadata:
      AWS::CloudFormation::Init: {}

  # SSM-friendly Latest Amazon Linux 2023 AMI
  LatestAmi:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

  InstanceProfileName:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /demo/iam/ec2InstanceProfileName # or pass as Parameter

Outputs:
  PublicIP: { Value: !GetAtt WebInstance.PublicIp }
```

> If you used the **iam-roles.yml** stack, store the InstanceProfile name in SSM at `/demo/iam/ec2InstanceProfileName`.

---

# 5) alb-asg.yml — ALB + TargetGroup + LaunchTemplate + ASG

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: ALB in public subnets with ASG in private subnets

Parameters:
  VpcId: { Type: String }
  PublicSubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
  PrivateSubnetIds:
    Type: List<AWS::EC2::Subnet::Id>

Resources:
  AlbSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VpcId
      GroupDescription: Allow HTTP from anywhere
      SecurityGroupIngress:
        - { IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0 }

  AppSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VpcId
      GroupDescription: Allow HTTP from ALB
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref AlbSG

  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets: !Ref PublicSubnetIds
      SecurityGroups: [!Ref AlbSG]
      LoadBalancerAttributes:
        - Key: deletion_protection.enabled
          Value: false

  TG:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId: !Ref VpcId
      Protocol: HTTP
      Port: 80
      TargetType: instance
      HealthCheckPath: /

  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TG

  LT:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: !Ref AmiId
        InstanceType: t3.micro
        SecurityGroupIds: [!Ref AppSG]
        UserData: !Base64 |
          #!/bin/bash
          yum install -y httpd
          echo "Hello from ASG" > /var/www/html/index.html
          systemctl enable --now httpd
  AmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

  ASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier: !Ref PrivateSubnetIds
      MinSize: "2"
      MaxSize: "4"
      DesiredCapacity: "2"
      TargetGroupARNs: [!Ref TG]
      LaunchTemplate:
        LaunchTemplateId: !Ref LT
        Version: !GetAtt LT.LatestVersionNumber

Outputs:
  AlbDNS: { Value: !GetAtt ALB.DNSName }
```

---

# 6) rds-mysql.yml — Private RDS MySQL with NoEcho creds

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: RDS MySQL in private subnets (no public access)

Parameters:
  VpcId: { Type: String }
  DBSubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
  DBUsername:
    Type: String
    NoEcho: true
  DBPassword:
    Type: String
    NoEcho: true
  DBName:
    Type: String
    Default: appdb

Resources:
  RDSSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VpcId
      GroupDescription: RDS access from private subnets
      # In practice, allow only from app SG; here we keep closed by default.

  SubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Private DB subnets
      SubnetIds: !Ref DBSubnetIds

  DB:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: mysql
      EngineVersion: "8.0"
      DBInstanceClass: db.t3.micro
      AllocatedStorage: "20"
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      DBName: !Ref DBName
      VPCSecurityGroups: [!Ref RDSSG]
      DBSubnetGroupName: !Ref SubnetGroup
      PubliclyAccessible: false
      BackupRetentionPeriod: 7
      MultiAZ: false
      DeletionProtection: false

Outputs:
  DBEndpoint: { Value: !GetAtt DB.Endpoint.Address }
```

---

# 7) dynamodb.yml — On-demand table with TTL + GSI

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: DynamoDB PAY_PER_REQUEST with TTL and a GSI

Parameters:
  TableName: { Type: String, Default: Inventory }
Resources:
  Table:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Ref TableName
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - { AttributeName: PK, AttributeType: S }
        - { AttributeName: SK, AttributeType: S }
        - { AttributeName: Category, AttributeType: S }
      KeySchema:
        - { AttributeName: PK, KeyType: HASH }
        - { AttributeName: SK, KeyType: RANGE }
      GlobalSecondaryIndexes:
        - IndexName: CategoryIndex
          KeySchema:
            - { AttributeName: Category, KeyType: HASH }
          Projection: { ProjectionType: ALL }
      TimeToLiveSpecification:
        AttributeName: ttl
        Enabled: true
Outputs:
  TableArn: { Value: !GetAtt Table.Arn }
```

---

# 8) lambda-apigw\.yml — Hello Lambda + REST API Gateway

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Lambda + API Gateway (Hello World)

Resources:
  Role:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: { Service: lambda.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  Fn:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt Role.Arn
      Code:
        ZipFile: |
          def handler(event, context):
              return {"statusCode": 200, "body": "Hello from Lambda via API Gateway"}
      Timeout: 5

  Api:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: hello-api

  RootRes:
    Type: AWS::ApiGateway::Resource
    Properties:
      ParentId: !GetAtt Api.RootResourceId
      PathPart: hello
      RestApiId: !Ref Api

  MethodGet:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref Api
      ResourceId: !Ref RootRes
      HttpMethod: GET
      AuthorizationType: NONE
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub
          arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${Fn.Arn}/invocations

  Deployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn: MethodGet
    Properties:
      RestApiId: !Ref Api
      StageName: prod

  Perm:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref Fn
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${Api}/*/GET/hello

Outputs:
  InvokeURL: { Value: !Sub "https://${Api}.execute-api.${AWS::Region}.amazonaws.com/prod/hello" }
```

---

# 9) cloudwatch-alarms.yml — CPU alarm + SNS topic

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: CloudWatch alarm for EC2 CPU > 80% + SNS email

Parameters:
  Email:
    Type: String
  InstanceId:
    Type: String

Resources:
  Topic:
    Type: AWS::SNS::Topic
    Properties:
      Subscription:
        - Protocol: email
          Endpoint: !Ref Email

  AlarmHighCPU:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: CPU > 80% for 5 minutes
      Namespace: AWS/EC2
      MetricName: CPUUtilization
      Dimensions:
        - Name: InstanceId
          Value: !Ref InstanceId
      Statistic: Average
      Period: 60
      EvaluationPeriods: 5
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold
      AlarmActions: [!Ref Topic]
      OKActions: [!Ref Topic]

Outputs:
  SnsTopicArn: { Value: !Ref Topic }
```

---

# 10) cloudfront-s3-static-site.yml — OAC-secured S3 origin

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: CloudFront + S3 static site using Origin Access Control (OAC)

Parameters:
  BucketName: { Type: String }

Resources:
  SiteBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      VersioningConfiguration: { Status: Enabled }
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  OAC:
    Type: AWS::CloudFront::OriginAccessControl
    Properties:
      Name: !Sub "${BucketName}-oac"
      OriginAccessControlConfig:
        Name: !Sub "${BucketName}-oac"
        OriginAccessControlOriginType: s3
        SigningBehavior: always
        SigningProtocol: sigv4

  Distribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Enabled: true
        DefaultRootObject: index.html
        Origins:
          - Id: s3Origin
            DomainName: !GetAtt SiteBucket.RegionalDomainName
            S3OriginConfig: {}
            OriginAccessControlId: !Ref OAC
        DefaultCacheBehavior:
          TargetOriginId: s3Origin
          ViewerProtocolPolicy: redirect-to-https
          AllowedMethods: [GET, HEAD]
          CachedMethods: [GET, HEAD]
        HttpVersion: http2

  BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref SiteBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: AllowCloudFrontAccess
            Effect: Allow
            Principal: { Service: cloudfront.amazonaws.com }
            Action: s3:GetObject
            Resource: !Sub "${SiteBucket.Arn}/*"
            Condition:
              StringEquals:
                AWS:SourceArn: !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${Distribution}"

Outputs:
  CDNDomain: { Value: !GetAtt Distribution.DomainName }
```

---

# 11) eks-cluster.yml — EKS Cluster + Managed NodeGroup (minimal)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Minimal EKS cluster + managed node group

Parameters:
  ClusterName: { Type: String, Default: demo-eks }
  VpcId: { Type: String }
  PrivateSubnetIds: { Type: List<AWS::EC2::Subnet::Id> }

Resources:
  ClusterRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: { Service: eks.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

  NodeRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: { Service: ec2.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

  ClusterSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: EKS cluster communication
      VpcId: !Ref VpcId

  Cluster:
    Type: AWS::EKS::Cluster
    Properties:
      Name: !Ref ClusterName
      RoleArn: !GetAtt ClusterRole.Arn
      Version: "1.30"
      ResourcesVpcConfig:
        SecurityGroupIds: [!Ref ClusterSG]
        SubnetIds: !Ref PrivateSubnetIds

  NodeGroup:
    Type: AWS::EKS::Nodegroup
    Properties:
      ClusterName: !Ref Cluster
      NodeRole: !GetAtt NodeRole.Arn
      ScalingConfig: { DesiredSize: 2, MinSize: 2, MaxSize: 4 }
      Subnets: !Ref PrivateSubnetIds
      AmiType: AL2_x86_64
      InstanceTypes: [t3.small]
      DiskSize: 20

Outputs:
  ClusterNameOut: { Value: !Ref Cluster }
```

---

# 12) nested-master.yml — Example master stack calling child stacks (S3 TemplateURL)

> Upload child templates (e.g., `vpc.yml`, `alb-asg.yml`) to an S3 bucket first, then reference their **S3 URLs**.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Nested master stack wiring VPC + ALB/ASG

Parameters:
  VpcTemplateUrl: { Type: String }
  AlbAsgTemplateUrl: { Type: String }
  Env: { Type: String, Default: dev }
  Project: { Type: String, Default: demo }

Resources:
  VpcStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Ref VpcTemplateUrl
      Parameters:
        Env: !Ref Env
        Project: !Ref Project

  AlbAsgStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Ref AlbAsgTemplateUrl
      Parameters:
        VpcId: !GetAtt VpcStack.Outputs.VpcId
        PublicSubnetIds: !Split [",", !GetAtt VpcStack.Outputs.PublicSubnetIds]
        PrivateSubnetIds: !Split [",", !GetAtt VpcStack.Outputs.PrivateSubnetIds]
```

---

## Common deploy commands (copy-paste ready)

```bash
# 0) Set env
export STACK=vpc-dev
export REGION=us-east-1

# 1) VPC
aws cloudformation deploy \
  --region $REGION \
  --template-file vpc.yml \
  --stack-name $STACK \
  --parameter-overrides Env=dev Project=inventory

# 2) EC2 in a public subnet (replace SubnetId with output)
PUB_SUBNET=$(aws cloudformation describe-stacks --stack-name $STACK \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnetIds'].OutputValue" \
  --output text | cut -d',' -f1)
VPC_ID=$(aws cloudformation describe-stacks --stack-name $STACK \
  --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)

aws cloudformation deploy \
  --template-file ec2.yml \
  --stack-name ec2-demo \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides VpcId=$VPC_ID SubnetId=$PUB_SUBNET KeyName=your-keypair

# 3) ALB+ASG using VPC outputs
PUBS=$(aws cloudformation describe-stacks --stack-name $STACK \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnetIds'].OutputValue" --output text)
PRIVS=$(aws cloudformation describe-stacks --stack-name $STACK \
  --query "Stacks[0].Outputs[?OutputKey=='PrivateSubnetIds'].OutputValue" --output text)

aws cloudformation deploy \
  --template-file alb-asg.yml \
  --stack-name alb-asg-demo \
  --parameter-overrides VpcId=$VPC_ID PublicSubnetIds="$PUBS" PrivateSubnetIds="$PRIVS"

# 4) S3 secure bucket
aws cloudformation deploy \
  --template-file s3-bucket.yml \
  --stack-name s3-demo \
  --parameter-overrides BucketName=your-unique-bucket-name ExpireDays=60

# 5) Lambda + API
aws cloudformation deploy \
  --template-file lambda-apigw.yml \
  --stack-name lambda-api-demo --capabilities CAPABILITY_NAMED_IAM
```

---

## Want this as a repo?

If you want, I’ll package these into a **clean GitHub repo layout**:

```
cloudformation-kit/
├─ 01-vpc.yml
├─ 02-iam-roles.yml
├─ 03-ec2.yml
├─ 04-alb-asg.yml
├─ 05-rds-mysql.yml
├─ 06-dynamodb.yml
├─ 07-lambda-apigw.yml
├─ 08-cloudwatch-alarms.yml
├─ 09-cloudfront-s3-static-site.yml
├─ 10-eks-cluster.yml
└─ 99-nested-master.yml
```

I’ve “updated the same” by turning your high-level list into complete, deployable templates + commands. If you want any **service added (ECS, ElastiCache, OpenSearch, SQS/SNS fanout, EventBridge)**, say the word and I’ll drop in those templates too.
