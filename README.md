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




As you can see the template has many variables. For more information on how to create your own template visit; What is AWS CloudFormation? — AWS CloudFormation (amazon.com). The sample templates and template snippets are essential in this process. The possibilities are truly endless. In my template I have Parameters and Resources that will be created. The “!Ref” references other areas of the code making it more flexible to adapt to your own needs.






Back in the AWS console, we will begin creating our Stack. Click on Template is ready and Upload a Template file. Choose the YAML template. Then hit Next.





![image](https://user-images.githubusercontent.com/115881685/217728780-c340e2d3-790f-4dfc-bb36-0c6ff681de89.png)




On the next screen you will specify the Stack details. This is where the Parameters come into play. You can name your Stack, choose your Key Pair and choose your VPC. The VPC will need a CIDR of 10.0.0.0/16. Then click Next.





![image](https://user-images.githubusercontent.com/115881685/217728927-e09643e5-3464-4942-b27e-de7ed3837acb.png)



On the next page leave everything at default and click Next. You will be brought to the Review page. Here you can look it over and click Create Stack.




![image](https://user-images.githubusercontent.com/115881685/217729369-f9efbd30-098f-4209-8d16-4adf90f61d8e.png)





You will be brought to a screen that should look like the above. Stating the Stack Create_In_Progress. If you navigate to the Resources tab you can watch as each resource is created and see where things are created successfully and where they may fail. I had to do a lot of troubleshooting from this stage. After a few minutes your screen will hopefully look like this!





![image](https://user-images.githubusercontent.com/115881685/217729533-5072ffd4-3cc0-4fd6-add0-ac769aaf53dc.png)



### Step 2
You can now navigate to Instances. You will see that you have 3 new instances up and running, spread across two availability zones.



![image](https://user-images.githubusercontent.com/115881685/217729715-29103ab9-e55b-40f1-9d8b-09faaa323347.png)




Grab the public IPv4 address and paste it into your browser to check that Apache is up and running as well.



![image](https://user-images.githubusercontent.com/115881685/217730051-961fb0e6-5bdc-4692-a88c-cfa616cccd11.png)




The test page shows that it was a success!


### Step 3

Now to test if the Auto scaling group is doing it’s job I will terminate 2 of the instances. Here goes.



![image](https://user-images.githubusercontent.com/115881685/217730237-ca19435a-43a0-4d4e-9e4a-f74e2fc358fc.png)




The 2 instances were terminated. In their place 2 more instances have populated.




![image](https://user-images.githubusercontent.com/115881685/217730383-9d4b83e2-4fc1-4e9f-917d-a49a0669bfd5.png)




This shows that our ASG successfully scaled out!




Be sure to delete your CloudFormation Stack so as not to be charged. CloudFormation is free to use but not all the resources it provisions are. This was a challenging project and I hope you feel as accomplished as I do after completing it!










