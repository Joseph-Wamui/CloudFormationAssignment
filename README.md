# PROJECT OVERVIEW

This project entails creation of a  CloudFormation template that provisions the following resources:
- A Virtual Private Cloud (VPC) with two public subnets
- Auto Scaling group of Windows EC2 instances
- S3 bucket

## PREREQUISITES

-  AWS account is require with the appropriate permissions to create the resources defined.
- AWS CLI installed and configured.

## Deployment Steps

Create a yaml file, to configure the resources:
1. Configure the VPC with the following properties:
  - Cidr block: 10.0.0.0/16
  - EnableDnsHostnames: true
  - EnableDnsSupport: true
  - Tags: Assign a key named Name with the value 'Assignment-VPC' to label the resource.
  ```yaml
 VPC:
    Type: 'AWS::EC2::VPC'
      Properties:
        CidrBlock: 10.0.0.0/16
        EnableDnsHostnames: true
        EnableDnsSupport: true
        Tags:
          - Key: Name
            Value: Assignment-VPC 
```
2. Configure the Internet Gateway and attach it to the VPC:
```yaml
  InternetGateway:
    Type: 'AWS::EC2::InternetGateway'
    Properties:
      Tags:
        - Key: Name
          Value: Assignment-IGW

  AttachGateway:    
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
```
3. Configure two public subnets with the following properties:
- Reference to the VPC created earlier.
- Cidr blocks: 1.0.1.0/24 - for the first Public Subnet and 1.0.2.0/24 for the second Public Subnet
- Availability zones: Assign the first public subnet to the first Availability Zone and the second public subnet to the second Availability Zone.
- Enable the "Map Public IP on Launch" setting to true for both subnets, to automatically assign public IP addresses to instances.
- Tags for labelling the resources
```yaml
  PublicSubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.1.0/24'
      AvailabilityZone: !Select [ 0, !GetAZs ''] #selects first AZ
      MapPublicIpOnLaunch: true      
      Tags:
        - Key: Name
          Value: Assignment-Public-Subnet-1

  PublicSubnet2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.2.0/24'
      AvailabilityZone: !Select [ 1, !GetAZs ''] #selects second AZ
      MapPublicIpOnLaunch: true      
      Tags:
        - Key: Name
          Value: Assignment-Public-Subnet-2
```
4. Configure the Route Table:
- Create the a Public route table, attached to the earleir created VPC
- Create a Public Route with a destination CIDR block of 0.0.0.0/0, which directs all outbound traffic to the Internet Gateway. This enables internet access for instances within the public subnets.
- Associate the Route Table with both Public Subnet 1 and Public Subnet 2.
```yaml
  PublicRouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: Assignment-Public-Route-Table

  PublicRoute:
    Type: 'AWS::EC2::Route'
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable
      

  PublicSubnet2RouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable
```
5. Configure a security group to allow RDP access:
- Reference the earlier created VPC.
- Set the Security Group to allow inbound traffic on port 3389 (TCP Protocol) from any IP address (0.0.0.0/0).
-  Add a Tag to label the resource.
```yaml
  RDPSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: 'Allow RDP Access'
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3389
          ToPort: 3389
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: Assignment-RDP-SG
```
6. Configure Launch template for the creating EC2 Instances with the following properties:
- The name of the launch template is set to Assignment-Launch-Template
- Launch Template Data:
  - Image ID of ami-0c232c9952bd4be59: Microsoft Windows Server 2022 Base.
  - Instance type of t3.micro.
  - Reference to the earlier created Security Group- RDPSecurityGroup.
  - A keyPair- JosephKey'.
- A Tag for labelling the resource.
```yaml
  # Launch Template for EC2
  LaunchTemplate:
    Type: 'AWS::EC2::LaunchTemplate'
    Properties:
      LaunchTemplateName: Assignment-Launch-Template
      LaunchTemplateData:
        ImageId: ami-0c232c9952bd4be59 # Microsoft Windows Server 2022 Base
        InstanceType: t3.micro
        SecurityGroupIds:
          - !Ref RDPSecurityGroup
        KeyName: JosephKey
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: Assignment-Win-Server-Instance
```

7. Set the Auto scaling Group with the following properties:
- The Auto Scaling Group is named Assignment-AutoScaling-Group.
- Reference the earlier created Launch Template, and set '!GetAtt LaunchTemplate.LatestVersionNumber', to ensure that the Auto Scaling Group launches instances according to the latest configuration in the launch template.
- Scaling configuration: 1 minimum number of instances, 3 maximum number of instances, and 2 number of instances to be maintained by the Auto Scaling Group.
- VPCZoneIdentifier: The Auto Scaling Group is configured to launch instances in the two public subnets, PublicSubnet1 and Public Subnet2
- Configure Tags to name the Auto Scaling Group  and set PropagateAtLaunch: true.
```yaml
   AutoScalingGroup:
    Type: 'AWS::AutoScaling::AutoScalingGroup'
    Properties:
      AutoScalingGroupName: Assignment-AutoScaling-Group
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: '1'
      MaxSize: '3'
      DesiredCapacity: '2'
      VPCZoneIdentifier:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      Tags:
        - Key: Name
          Value: Windows EC2 Instance
          PropagateAtLaunch: true
```
8. Configure S3 bucket with the following properties:
- BucketName: Skillsync-Joseph-assignment
- AccessControl set to Private to ensure the data stored in the bucket is secure and not publicly accessible.
- VersioningConfiguration set to Enabled 
- A tag to label the resource
```yaml
 # S3 Bucket to be used the instances
  S3Bucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: SkillsyncJosephAssignment
      AccessControl: Private
      VersioningConfiguration:
        Status: Enabled
      Tags:
        - Key: Name
          Value: Assignment-S3-Bucket
```
## Uploading the YAML File to AWS CloudFormation Console
- Navigate to the CloudFormation dashboard within the AWS Management Console, choose "Create stack" and select "Choose an existing template." Choose "Upload a template file," and then click "Choose file" to select your YAML file from its saved location. 
- After uploading, enter a stack name, configure any additional settings as needed, and review the details. Finally, click "Create stack" to start the creation process
- Monitor the progress until the stack status changes from "CREATE_IN_PROGRESS" to "CREATE_COMPLETE," indicating that all resources have been successfully created as in the image below.
![CREATE_COMPLETEScreenshot](/Assignment/Screenshot%201.png)





          
