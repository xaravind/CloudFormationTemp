# Fully Automated VPC and EC2 Setup with VPC Peering — Powered by CloudFormation

Provision **two isolated VPCs** (Internal & Client), launch **public and private EC2 instances** in each, and **enable secure cross-VPC communication** between private instances via **VPC Peering**, using only **CloudFormation templates**.


### Architecture Diagram

![VPC Architecture](/Workflow.gif)

---

## **What We'll Build — Fully Automated via CloudFormation**

With **Infrastructure as Code (IaC)** using **AWS CloudFormation**, we'll build a complete, production-like network setup **without any manual steps**:

* **Create Two VPCs** – Each with custom CIDR blocks for logical separation (e.g., `10.0.0.0/16` and `10.1.0.0/16`).
* **Deploy Public and Private Subnets** – In each VPC, spread across different Availability Zones.
* **Launch EC2 Instances** – One public and one private instance in each VPC.
* **Establish VPC Peering** – To enable communication between private subnets in both VPCs.
* **Configure Route Tables** – Modify route tables to allow secure, cross-VPC traffic flow.
* **Test file transfer** between private instances (via Bastion/Jump host).

---

## 📖 Theory at the Final Pages of PDF.

* What is a VPC?
* CIDR, Subnets, NAT, IGW
* What is VPC Peering & why we use it
* Security Groups vs NACLs
* CloudFormation benefits

---

## Let’s Build

### ✅ First Stage: Write CloudFormation Templates in VS Code (YAML Format)

CloudFormation supports both **JSON** and **YAML**, but YAML is much simpler to write and read. It is also easier compared to **Terraform**, which uses its own custom language (HCL).


Start by creating the following templates:

### 1. `vpc-CFT.yml`

> For this template, I used references from the official AWS documentation:

> 🔗 (https://docs.aws.amazon.com/codebuild/latest/userguide/cloudformation-vpc-template.html)


This template creates all required networking components for one VPC:

* VPC
* Public and Private Subnets
* Internet Gateway (IGW)
* NAT Gateway and Elastic IP
* Route Tables and associations
* Exports private route table ID for reuse

```bash
#01-vpc-CFT.yml
AWSTemplateFormatVersion: '2010-09-09'
Description: This template deploys a VPC, with a public and a private subnets spread
  across two Availability Zones. It deploys an internet gateway, with a default
  route on the public subnets. It deploys a  NAT gateways,
  and default routes for them in the private subnets.
# Resources section

#################################################################
#                       PARAMETERS                              #
#################################################################

Parameters:
  Name:
    Description: A name that is prefixed to resource names
    Type: String

  VpcCIDR:
    Description: Please enter the IP range (CIDR notation) for this VPC
    Type: String
    Default: 10.0.0.0/16

  PublicSubnetCIDR:
    Description: Please enter the IP range (CIDR notation) for the public subnet in the first Availability Zone
    Type: String
    Default: 10.0.0.0/24

  PrivateSubnetCIDR:
    Description: Please enter the IP range (CIDR notation) for the private subnet in the first Availability Zone
    Type: String
    Default: 10.0.16.0/24

#################################################################
#       Resources - (VPC, IGW , SUBNETS, EIP, NAT )             #
#################################################################

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${Name}-vpc"

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub "${Name}-igw"

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      CidrBlock: !Ref PublicSubnetCIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${Name}-public-subnet"

  PrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [ 1, !GetAZs  '' ]
      CidrBlock: !Ref PrivateSubnetCIDR
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub "${Name}-private-Subnet"

  NatGatewayEIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnet

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${Name}-pub-rtb"

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${Name}-pvt-rtb"

  DefaultPrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway

  PrivateSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      SubnetId: !Ref PrivateSubnet

#################################################################
#                       OUTPUTS                              #
#################################################################

Outputs:
  VPC:
    Description: A reference to the created VPC
    Value: !Ref VPC

  PublicSubnets:
    Description: A list of the public subnets
    Value: !Ref PublicSubnet

  PrivateSubnets:
    Description: A list of the private subnets
    Value: !Ref PrivateSubnet

  PrivateRouteTable:
    Description: The Private Route Table ID for the VPC
    Value: !Ref PrivateRouteTable
    Export:
      Name: !Sub "${Name}-PrivateRouteTableId"    
```
---

### ✅ Best Practices Followed:

* Used **Parameters** for reusability
* Exported private route table for importing in peering template
* Created NAT Gateway for private subnet internet access

---

### 2. `ec2-CFT.yml`

This template launches EC2 instances into a selected subnet (public or private) in a given VPC.

You can configure:

* VPC ID
* Subnet ID
* Instance type
* Key pair
* Public/private subnet option

```bash
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  This template launches an EC2 instance into a selected subnet within a given VPC.
  It uses region-based AMI mapping to dynamically select the appropriate Amazon Linux 2 image.

Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0c02fb55956c7d316
    us-west-2:
      AMI: ami-0892d3c7ee96c0bf7
    eu-central-1:
      AMI: ami-0233214e13e500f77
    ap-south-1:
      AMI: ami-0e306788ff2473ccb

Parameters:
  InstanceName:
    Description: Name tag for the EC2 instance
    Type: String

  VPCId:
    Description: The VPC ID where the instance will be launched
    Type: AWS::EC2::VPC::Id

  SubnetId:
    Description: The Subnet ID where the instance will be launched (public or private)
    Type: AWS::EC2::Subnet::Id

  InstanceType:
    Description: EC2 instance type
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t3.micro
      - t3.small

  KeyName:
    Description: The EC2 KeyPair name to enable SSH access
    Type: AWS::EC2::KeyPair::KeyName
  
  IsPublicSubnet:
    Type: String
    Default: "false"
    AllowedValues: ["true", "false"]
    Description: Whether the instance is in a public subnet (with public IP)

Conditions:
  IsPublic: !Equals [ !Ref IsPublicSubnet, "true" ]

Resources:
  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable SSH access
      VpcId: !Ref VPCId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: !Sub "${InstanceName}-sg"

  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", AMI]
      SubnetId: !Ref SubnetId
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      Tags:
        - Key: Name
          Value: !Ref InstanceName
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeSize: 8
            VolumeType: gp3
            DeleteOnTermination: true

Outputs:
  InstanceId:
    Description: EC2 Instance ID
    Value: !Ref EC2Instance

  InstancePublicIp:
    Condition: IsPublic
    Description: Public IP address (only if in public subnet)
    Value: !GetAtt EC2Instance.PublicIp

  InstancePrivateIp:
    Description: Private IP address
    Value: !GetAtt EC2Instance.PrivateIp
```
---

### ✅ Best Practices Followed:

* Used **Parameters** for reusability
* **Mapping** for region-based AMI selection
* **Conditions** for assigning public IP if in public subnet
* Custom **Security Group** for SSH access
* Defined **Block Device Mappings** for storage
* Outputs include instance ID, private IP, and optionally public IP

---


### 3. `vpc-peering.yml`

Automates VPC Peering setup and updates route tables for cross-VPC communication.

```bash
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC Peering between two VPCs in the same account and region

Parameters:
  VpcAId:
    Type: AWS::EC2::VPC::Id
    Description: The ID of the first VPC 

  VpcBId:
    Type: AWS::EC2::VPC::Id
    Description: The ID of the second VPC 

  VpcACidr:
    Type: String
    Description: CIDR block of VPC A

  VpcBCidr:
    Type: String
    Description: CIDR block of VPC B

  VpcARouteTableExportName:
    Type: String
    Description: Export name of VPC A's private route table 

  VpcBRouteTableExportName:
    Type: String
    Description: Export name of VPC B's private route table 

Resources:
  VPCPeeringConnection:
    Type: AWS::EC2::VPCPeeringConnection
    Properties:
      VpcId: !Ref VpcAId
      PeerVpcId: !Ref VpcBId
      Tags:
        - Key: Name
          Value: !Sub "${VpcAId}-${VpcBId}-peering"

  RouteFromVpcAToVpcB:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId:
        Fn::ImportValue: !Ref VpcARouteTableExportName
      DestinationCidrBlock: !Ref VpcBCidr
      VpcPeeringConnectionId: !Ref VPCPeeringConnection

  RouteFromVpcBToVpcA:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId:
        Fn::ImportValue: !Ref VpcBRouteTableExportName
      DestinationCidrBlock: !Ref VpcACidr
      VpcPeeringConnectionId: !Ref VPCPeeringConnection

Outputs:
  PeeringConnectionId:
    Description: The ID of the VPC Peering Connection
    Value: !Ref VPCPeeringConnection


```
---

### ✅ Best Practices Followed:

* Used **Parameters** for VPC IDs and CIDRs
* Used `!ImportValue` for route table IDs exported in VPC stack
* Automatically updates routes in both VPCs
* Outputs peering connection ID

---

### Push Files to GitHub and S3 if needed.

Use Git to push templates to a repository:

```bash
aravi@Aravind MINGW64 ~/CloudFormationTemp (main)
$ git add . ; git commit -m "ll" ; git push
[main 674cc30] ll
 3 files changed, 74 insertions(+), 4 deletions(-)
 create mode 100644 vpc-peering,yml
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 18 threads
Compressing objects: 100% (5/5), done.
Writing objects: 100% (5/5), 1.07 KiB | 548.00 KiB/s, done.
Total 5 (delta 2), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/xaravind/CloudFormationTemp.git
   b0dac60..674cc30  main -> main
```

This ensures your infrastructure code is stored, trackable, and reusable.

## ✅ Second Stage: Create VPCs with CloudFormation

### Create Internal VPC

1. Go to **AWS Console > CloudFormation > Create Stack**
2. Upload `vpc-CFT.yml`
3. Provide stack name: `Internal-vpc-stack`
4. Fill parameters:

   * Name: `Internal`
   * PrivateSubnetCIDR: `10.0.16.0/24`
   * PublicSubnetCIDR: `10.0.0.0/24`
   * VpcCIDR: `10.0.0.0/16`
5. Leave defaults, Review, and Submit
6. Wait for `CREATE_COMPLETE`
7. Check Resources and Outputs

### Create Client VPC (Repeat Same Steps with Updated Parameters)

* Stack Name: `Client-vpc-stack`
* Name: `Client-vpc`
* PrivateSubnetCIDR: `172.0.16.0/24`
* PublicSubnetCIDR: `172.168.0.0/24`
* VpcCIDR: `172.168.0.0/16`


## ✅ Third Stage: Launch EC2 Instances with CloudFormation

### Create Key Pair

1. Go to **EC2 > Key Pairs > Create Key Pair**
2. Name: `vpcPeer`
3. Key type: `RSA`, format: `.pem`


### Launch EC2 Instances

Repeat stack creation steps using `ec2-CFT.yml`, changing the parameters:

#### Internal VPC:

* Public Instance:

  * Stack: `Internal-ec2Pub-stack`
  * InstanceName: `Internal-PubApp-instance`
  * Subnet: Internal vpc public subnet
  * IsPublicSubnet: `true`
  * KeyName: vpcpeer
  * VPCId: Internal vpc ID

* Private Instance:

  * Stack: `Internal-ec2Pvt-stack`
  * InstanceName: `Internal-PvtDb-instance`
  * Subnet: Internal vpc private subnet
  * IsPublicSubnet: `false`
  * KeyName: vpcpeer  
  * VPCId: Internal vpc ID

#### Client VPC:

* Public Instance:

  * Stack: `Client-ec2Pub-stack`
  * InstanceName: `Client-PubApp-instance`
  * Subnet: Client vpc public subnet
  * IsPublicSubnet: `true`
  * KeyName: vpcpeer
  * VPCId: Client vpc ID

* Private Instance:

  * Stack: `Client-ec2Pvt-stack`
  * InstanceName: `Client-PvtDb-instance`
  * Subnet: Client vpc private subnet
  * IsPublicSubnet: `false`
  * KeyName: vpcpeer
  * VPCId: Client vpc ID

Go to **EC2 > Instances** to verify all 4 instances are running.


##  Fourth Stage: SSH Access & Testing

### Copy `.pem` Key to Public Instances

open terminal/windows power shell 

```bash
scp -i vpcPeer.pem vpcPeer.pem ec2-user@<public-ip>:~
```

```bash
PS C:\Users\aravi\Downloads> scp -i .\vpcPeer.pem .\vpcPeer.pem ec2-user@34.224.27.251:~
Warning: Permanently added '34.224.27.251' (ED25519) to the list of known hosts.
vpcPeer.pem                                                                           100% 1678     6.8KB/s   00:00
PS C:\Users\aravi\Downloads> scp -i .\vpcPeer.pem .\vpcPeer.pem ec2-user@:54.242.134.227:~
ED25519 key fingerprint is SHA256:+wXZNf0dPby5hrckChr4Gva9naVWagX0rb2+LXoGQMY.
vpcPeer.pem                                                                           100% 1678     7.4KB/s   00:00
PS C:\Users\aravi\Downloads>
```

### SSH into Public EC2

```bash
ssh -i vpcPeer.pem ec2-user@<public-dns>
```

Change key permissions if needed:

```bash
chmod 400 vpcPeer.pem
```

### SCP Key to Private EC2 from Public EC2

```bash
scp -i vpcPeer.pem vpcPeer.pem ec2-user@<private-ip>:~
```

### Jump into Private EC2 and Test Internet

```bash
ssh -i vpcPeer.pem ec2-user@<private-ip>
sudo yum update -y
```

### Try Copying File Between Private EC2 (Expected to Fail)

```bash
scp -i vpcPeer.pem client-pvt.txt ec2-user@<other-private-ip>:~
# Fails due to no peering connection yet
```

```bash
[ec2-user@ip-172-168-16-210 ~]$ scp -i vpcPeer.pem client-pvt.txt ec2-user@10.0.16.56:~
ssh: connect to host 10.0.16.56 port 22: Connection timed out
lost connection
[ec2-user@ip-172-168-16-210 ~]$
```

## 🌐 Fifth Stage: Enable VPC Peering

### Create Peering with CloudFormation

1. Go to **CloudFormation > Create Stack**
2. Upload `vpc-peering.yml`
3. Provide stack name: `vpc-peering-stack`
4. Fill Parameters:

   * VpcAId: `Internal VPC ID`
   * VpcACidr: `10.0.0.0/16`
   * VpcARouteTableExportName: `Internal-vpc-PrivateRouteTableId`
   * VpcBId: `Client VPC ID`
   * VpcBCidr: `172.168.0.0/16`
   * VpcBRouteTableExportName: `Client-vpc-PrivateRouteTableId`
5. Review and Submit
6. Wait for `CREATE_COMPLETE`


### Verify Route Table Updates

Check route tables of both VPCs — you should see routes pointing to the peer connection.


### Retest File Transfer Between Private Instances

```bash
scp -i vpcPeer.pem file.txt ec2-user@<peer-private-ip>:~
# Should now succeed
```


---

We successfully built a secure, fully automated, production-grade environment with:

* Isolated VPCs
* Public and Private Subnets
* Bastion Hosts
* VPC Peering
* CloudFormation automation




### What is a VPC and Why Do We Need It?

A **VPC (Virtual Private Cloud)** is an isolated section of the AWS cloud where you can launch AWS resources—like EC2 instances—within a logically separated network. It gives you full control over:

* ✅ **IP address ranges** using CIDR (Classless Inter-Domain Routing)
* ✅ **Subnets**, **routing**, and **internet access**
* ✅ **Security** through Security Groups and Network ACLs

In simple terms, a VPC lets you design your own virtual data center in the cloud. You control how resources communicate and who can access them.

---

In **real-time environments**, every organization uses VPCs to host internal applications securely. Typically, employees and administrators connect to VPCs using **VPN**, **AWS Direct Connect**, or **bastion hosts (jump servers)** to manage private/internal resources.

VPCs are critical for:

* Separating production, staging, and development environments
* Ensuring secure and private communication between AWS services
* Supporting hybrid cloud models with on-premises integration

---

### Key VPC Components:

#### 🔹 Subnets

* **Public Subnet:** Instances here can connect to the internet directly using an Internet Gateway.
* **Private Subnet:** Instances here **don’t have direct internet access**. Instead, they use a NAT Gateway to access the internet.

#### 🔹 Route Tables

* **Public Route Table:** Routes public subnet traffic through the Internet Gateway.
* **Private Route Table:** Routes private subnet traffic through the NAT Gateway to allow secure internet access.

#### 🔹 Gateways

* **Internet Gateway (IGW):** Lets resources in public subnets access the internet.
* **NAT Gateway (NGW):** Allows private subnet resources to access the internet **without exposing them** to incoming traffic.

#### 🔹 Security Layers

* **Security Groups:** Instance-level virtual firewalls that control inbound and outbound rules.
* **Network ACLs (NACLs):** Subnet-level firewalls that are stateless and used for additional filtering.

---

### What is VPC Peering and Why Is It Used?

**VPC Peering** is a network connection between two VPCs that lets them route traffic using **private IP addresses**. Peering allows:

* Secure communication between separate VPC environments (e.g., dev <--> prod)
* File transfers between applications hosted in different VPCs
* Private interaction between services **without using the internet**

VPC Peering is **one-to-one**, meaning both VPCs can initiate and respond to traffic once properly routed.

---

### 🔹 AWS Services Used in This Setup

* **CloudFormation:** Automates infrastructure provisioning using templates (Infrastructure as Code).
* **VPC:** Creates isolated networks for applications.
* **EC2:** Hosts public and private instances to test cross-VPC communication and secure file transfer.

---





