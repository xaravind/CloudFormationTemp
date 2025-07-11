**"Fully Automated VPC and EC2 Setup with VPC Peering — Powered by CloudFormation"**

Provision **two isolated VPCs** (Internal & Client), launch **public and private EC2 instances** in each, and **enable secure cross-VPC communication** between private instances via **VPC Peering**, using only **CloudFormation templates**.

### ** Architecture Diagram:**

===
GIF image
===

---

### **What We'll Build — Fully Automated via CloudFormation**

With **Infrastructure as Code (IaC)** using **AWS CloudFormation**, we'll build a complete, production-like network setup **without any manual steps**:

1. **Create Two VPCs** – Each with custom CIDR blocks for logical separation (e.g., `10.0.0.0/16` and `10.1.0.0/16`).
2. **Deploy Public and Private Subnets** – In each VPC, spread across different Availability Zones.
3. **Launch EC2 Instances** – One public and one private instance in each VPC.
4. **Establish VPC Peering** – To enable communication between private subnets in both VPCs.
5. **Configure Route Tables** – Modify route tables to allow secure, cross-VPC traffic flow.

Final
 **Test file transfer** between private instances (via Bastion/Jump host)


##  Theory at the Final Pages of PDF.
* What is a VPC?
* CIDR, Subnets, NAT, IGW
* What is VPC Peering & why we use it
* Security Groups vs NACLs
* CloudFormation benefits

---

Your notes are clear and straightforward! I've cleaned up the grammar, improved sentence flow slightly, and kept the tone simple and easy to understand — just like your original style. Here's the refined version:

---

### **Let’s Build**

First Stage :- Write CloudFormation templates in your local **VS Code** using **YAML format**.

CloudFormation supports both **JSON** and **YAML**, but YAML is much simpler to write and read. Unlike other languages, YAML has zero complexity in formatting. It is also easier compared to **Terraform**, which uses its own custom language (HashiCorp Configuration Language - HCL).

> For this template, I used references from the official AWS documentation:

> 🔗 (https://docs.aws.amazon.com/codebuild/latest/userguide/cloudformation-vpc-template.html)

---

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
```
---

###  **Best Practices I Followed While Writing This Template**

* **Used Parameters**: So we can reuse the template multiple times just by changing values — no need to touch the actual code.
* The template creates **all the required networking components**, including:

  * VPC
  * Public and Private Subnets
  * Internet Gateway (IGW) and attachment to VPC
  * Route Tables for both public and private
  * Subnet-to-Route Table associations
  * NAT Gateway with Elastic IP for internet access from the private subnet

---

> **I followed the same best practices from `vpc-CFT.yml` to create both `ec2-CFT.yml` and `vpc-peering.yml` templates.**

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

To use the above template, all you need beforehand is:

* A valid **VPC**
* An existing **EC2 key pair**

Once provided, this template will automatically launch an EC2 instance into your chosen **public** or **private** subnet.

---

### **Best Practices I Followed While Writing This Template**

* **Used Parameters** to make the template reusable across environments. You only need to provide values like VPC ID, subnet ID, instance type, and key name during launch.

* **Used Mappings** to dynamically select the correct AMI based on the region. No need to hardcode AMI IDs.

* **Included Conditions** to toggle public IP assignment, allowing flexibility between public and private subnets.

* **Defined a Security Group** to allow SSH (port 22) access. You can customize this for stricter rules.

* **Used Block Device Mappings** to configure an 8GB `gp3` EBS volume that gets deleted on instance termination.

* **Clean Outputs Section** that returns the Instance ID, Private IP (always), and Public IP (only for public subnets). Useful for validation or automation.
---

Once the Templates are finished push to your Git-hub repo

```bash
aravi@Aravind MINGW64 ~/11_04/CloudFormationTemp (main)
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

In this stage, we write all the required template files and push them to GitHub for version control and reuse.

Second Stage: Launch Infra with Cloud Formation








###  What is a VPC and Why Do We Need It?

A **VPC (Virtual Private Cloud)** is an isolated section of the AWS cloud where you can launch resources—like EC2 instances—within a logically separated network. It provides full control over:

* ✅ **IP address ranges** using CIDR (Classless Inter-Domain Routing)
* ✅ **Subnets**, **routing**, and **internet access**
* ✅ **Security** through Security Groups and Network ACLs

In simple terms, a VPC allows you to design your own virtual data center in the cloud—with complete control over who can access what and how.

---

In **real-time scenarios**, every organization uses VPCs to host internal applications and services securely. Typically, employees or administrators **access VPCs through VPN connections**, **Direct Connect**, or **bastion hosts (jump servers)** for internal/private resource management.

VPCs play a vital role in:

* Segregating production, staging, and development environments
* Ensuring secure and private communication across AWS services
* Supporting hybrid cloud models with on-premises integration

---

###  Key VPC Components:

#### 🔹 Subnets:

* **Public Subnet:** A subnet whose instances can directly communicate with the internet via an Internet Gateway.
* **Private Subnet:** A subnet that has no direct internet access. Resources inside can only access the internet via a NAT Gateway..

#### 🔹 Route Tables:

* **Public Route Table:** Contains routes that direct traffic from public subnets to the Internet Gateway.
* **Private Route Table:** Contains routes for private subnets, typically directing internet-bound traffic through a NAT Gateway.

#### 🔹 Gateways:

* **Internet Gateway (IGW):** A gateway that allows instances in a public subnet to connect to the internet.
* **NAT Gateway (NGW):** Allows instances in a private subnet to initiate outbound traffic to the internet while preventing inbound traffic from the internet.

#### 🔹 Security Layers:

* **Security Groups:** Virtual firewalls at the instance level.
* **NACLs (Network ACLs):** Optional stateless firewalls at the subnet level.

---

###  What is VPC Peering and Why Is It Used?

**VPC Peering** is a networking connection between two VPCs that enables you to route traffic between them using private IPs. It’s commonly used to allow:

* Communication between different environments (e.g., dev and prod)
* File sharing between isolated networks
* Secure cross-VPC service interaction without going over the internet

---

### 🔹 AWS Services Used in This Setup:

* **CloudFormation:** Automates the provisioning of infrastructure via templates (IaC).
* **VPC:** Defines isolated network boundaries and routing.
* **EC2:** Hosts public and private instances for secure file transfers and communication across VPCs.

---




