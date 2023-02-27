# AWS Free-Tier/Three-Tiered Network
This repo contains AWS CloudFormation Templates and dependencies to create three-tier, secure VPC in two availability zones.

This project serves more to build experience than to solve any practical issues associated with hosting a secure app on AWS. However, some of the cost-saving measures I implemented (such as running a NAT Instance rather than a NAT gateway, see section 4) may be of interest to individuals or small dev/test teams.

## Table of Contents
1. Installation Instructions
2. Architectural overview
2. Template details

## Installation Instructions
Installation is easy. All you need is the "01_Network.json" CloudFormation template.
1. Decide which region you would like to create these resources in. 
2. Find the ImageID for the Amazon Linux 2 free-tier eligible AMI in your region.
3. Download and open "01_Network.json" and replace the "ImageID" (line 298) with the ID appropriate for your region. The template comes pre-configured to work for us-east-1, current as of 2023-02-25.
4. Upload the modified "01_Network.json" file to CloudFormation and create a new stack. Be sure to use an IAM user or assign a CloudFormation role that can create VPC resources and EC2 instances.
5. CloudFormation will prompt for more information as necessary.

## Architectural Overview
This network is built on three-tier architecture. The design of the three tiers (presentation tier, app tier, and data tier) is modelled after the [WordPress: Best Practices on AWS](https://aws.amazon.com/blogs/architecture/wordpress-best-practices-on-aws/) whitepaper. 

The three tiers are populated as follows:
 |Layer|Function|
 |---|---|
 | 1. Presentation tier | Dual purpose NAT Instance/Bastion server
 |2. Application tier | App server 
 |3. Data Tier|RDS Database

Each layer is given its own subnet in a single availability zone (AZ). A second set of subnets is defined in a second AZ, which satisfies the subnet group requirements of RDS and allows for easy scaling in the future.

The VPC connects to the internet through an **Internet Gateway**. Only the presentation layer is a publicly accessible (i.e. has a route to the Internet Gateway defined). 

Note that I opt to create a NAT Instance out of an EC2 instance, rather than use AWS's built-in NAT Gateways. This is a cost-saving measure. See template details for more information

<Details><Summary>See the architectural diagram</summary>
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="/assets/WordPress Architecture DarkMode.jpeg" | width=750>
  <source media="(prefers-color-scheme: light)" srcset="/assets/WordPress Architecture DarkMode.jpeg" | width=750>
  <img alt="A diagram of the architecture that is created with these CloudFormation Templates." src="/assets/WordPress Architecture DarkMode.jpeg" | width=750>
</picture>
</Details>

## Template Details
#### TEMPLATE: 01_Network.json
This template lays down the necessary networking infrastructure for the app and database servers to come. The architecture is initialized in the first two AZs (alphabetically) in the region from which the template is run in CloudFormation. 

Instead of a NAT Gateway (which can cost nearly $40/month), an EC2 instance is spun up and configured to act as a NAT Instance. To be clear, NAT Gateways are better than this NAT Instance in nearly every environment. They are fully managed, scalable, and highly available. I only went with a NAT Instance as a cost-saving measure.

To configure the NAT Instance, I disable `SourceDestCheck` and run the following bootstrap script:
```
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
This design of this NAT instance was inspired by a [blog post](https://www.kabisa.nl/tech/cost-saving-with-nat-instances/) by Luk van den Borne from 2019. 

**NOTE: THE AMI USED FOR THIS INSTANCE IS ONLY AVAILABLE IN us-east-1.** To create a stack in a different region, change the AMI ID to one belonging to an Amazon Linux 2 image in your region.

<Details><Summary>See parameters</summary>

 |Parameter|Function|
 |---|---|
 |EnvironmentName | An environment name that is prefixed to resource names |
 | VpcCIDR | Please enter the IP range (CIDR notation) for this VPC |
 | PublicSubnet1CIDR | Please enter the IP range (CIDR notation) for the public subnet in the first Availability Zone |
 | PublicSubnet2CIDR | Please enter the IP range (CIDR notation) for the public subnet in the second Availability Zone |        
 | PrivateAppSubnet1CIDR | Please enter the IP range (CIDR notation) for the (private) app subnet in the first Availability Zone |
 | PrivateAppSubnet2CIDR | Please enter the IP range (CIDR notation) for the (private) app subnet in the second Availability Zone |
 | PrivateDBSubnet1CIDR | Please enter the IP range (CIDR notation) for the (private) database subnet in the first Availability Zone |
 | PrivateDBSubnet2CIDR | Please enter the IP range (CIDR notation) for the (private) database subnet in the second Availability Zone |
 | SSHLocation | The IP address range that can be used to SSH to the EC2 instances |
 | KeyName | Name of an existing EC2 KeyPair to enable SSH access to the NAT instance. Note that the Securit Group associated with the NAT instance does not allow SSH traffic. However, the same instance will later be configured as a Bastion host which *does* allow SSH. |
 </Details>

<Details><Summary>See resources</summary>

 |Resource|Description|
 |---|---|
 VPC|A virtual private cloud with the CIDR block specified in the parameters
 InternetGateway|Default Internet Gateway
 InternetGatewayAttachment|Connect Internet Gateway to VPC
 PublicSubnet1 and PublicSubnet2|Makes a call to `"Fn::GetAZs"` to get a list of AZs in the region you are running the template in. Initializes a subnet in the first (Subnet1) or second (Subnet2) AZ in the region (alphabetically). `MapPublicIpOnLaunch` is set to `true`.
PrivateAppSubnet1/2 and PrivateDBSubnet1/2|Makes a call to `"Fn::GetAZs"` to get a list of AZs in the region you are running the template in. Initializes a subnet in the first (Subnet1) or second (Subnet2) AZ in the region (alphabetically).
NATSecurityGroup|Security group to be used by the NAT Instance. Enables HTTP/HTTPS communication to and from any IP.
NATInstance1|Initializes an t2.micro (free-tier) EC2 instance inside PublicSubnet1. The bootstrap script and `SourceDestCheck=false` attribute together enable IP forwarding (i.e. NAT funcationality). An SSH key pair is associated with the instance for later use. **NOTE: THE AMI USED FOR THIS INSTANCE IS ONLY AVAILABLE IN us-east-1.**
PublicRouteTable and PublicInternetRoute|Creates a route to the internet through the Internet Gateway.
PrivateRouteTable and PrivateInternetRoute|Creates a route to the internet through the NAT Instance.
___RouteTableAssociation|Associates each of the subnets with either PublicRouteTable (public subnets) or PrivateRouteTable (private subnets).
</details>
