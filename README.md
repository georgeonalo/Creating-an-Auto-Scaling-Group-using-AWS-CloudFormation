# Creating an Auto Scaling Group using AWS CloudFormation


![image](https://user-images.githubusercontent.com/115881685/217727104-d676c639-aabf-488c-a322-fe964d262cc3.png)



AWS CloudFormation is a service that allows you to configure, provision, and manage your AWS infrastructure as code. It allows you to create Stacks of resources from templates.

CloudFormation can be used to deploy almost any AWS service. In this project, I will use a template to create an autoscaling group using t2.micro EC2 instances across 2 public subnets in a VPC. It includes an internet gateway to allow internet access to the EC2 instances. Along with a web server security group to allow inbound traffic and ssh access. The instances will have an Apache web server installed upon launch. The autoscaling group will provide a minimum of 2 instances, the desired capacity of 3, and a maximum of 5 instances. To test that the ASG is scaling out I will demonstrate terminating 2 instances.




### Prerequisites
* An AWS account.

* A code editor, Visual Studio Code.


### Step 1

In the AWS console navigate to CloudFormation and click Create stack.

![image](https://user-images.githubusercontent.com/115881685/217727944-7884e6cf-eb42-4ffb-aa7b-67c1ff6b6e6c.png)



Copy my template below and save it as a .yml file.



```
AWSTemplateFormatVersion: "2010-09-09"

Description: This template creates an autoscaling group with EC2 instances in a VPC with 2 public subnets. The instances have an apache web server installed.

Parameters:
  VpcId:
    Type: 'AWS::EC2::VPC::Id'
    Description: VpcId of your existing Virtual Private Cloud (VPC)
    ConstraintDescription: must be the VPC Id of an existing Virtual Private Cloud.

  LaunchTemplateVersionNumber:
    Default: 1
    Type: String

  KeyName:
    Description: The EC2 Key Pair to allow SSH access to the instances
    Type: 'AWS::EC2::KeyPair::KeyName'
    ConstraintDescription: must be the name of an existing EC2 KeyPair.

  SSHLocation:
    AllowedPattern: '(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})'
    ConstraintDescription: must be a valid IP CIDR range of the form x.x.x.x/x.
    Default: 0.0.0.0/0
    Description: The IP address range that can be used to access the web server using SSH.
    MaxLength: '18'
    MinLength: '9'
    Type: String

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: True
      EnableDnsSupport: True
      InstanceTenancy: default

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: LuitGW

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      CidrBlock: 10.0.0.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: Public Subnet 1
      VpcId: !Ref VPC

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Select [ 1, !GetAZs '' ]
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: Public Subnet 2
      VpcId: !Ref VPC

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      Tags:
        - Key: Name
          Value: PublicRouteTable
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2

  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Open HTTP (port 80) and SSH (port 22)
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: !Ref SSHLocation

  LaunchTemplate:
    Type: 'AWS::EC2::LaunchTemplate'
    Properties:
      LaunchTemplateName: !Sub '${AWS::StackName}-launch-template-for-auto-scaling'
      LaunchTemplateData:
        NetworkInterfaces:
          - DeviceIndex: 0
            AssociatePublicIpAddress: true
            Groups:
              - !Ref InstanceSecurityGroup
            DeleteOnTermination: true
        Placement:
          Tenancy: default
        ImageId: ami-0022f774911c1d690
        KeyName: !Ref KeyName
        InstanceType: t2.micro
        UserData:
          Fn::Base64:
            !Sub |
              #!/bin/bash
              yum update -y
              yum install httpd -y
              systemctl start httpd
              systemctl enable httpd

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      MinSize: '2'
      MaxSize: '5'
      DesiredCapacity: '3'
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !Ref LaunchTemplateVersionNumber
      VPCZoneIdentifier:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
```
