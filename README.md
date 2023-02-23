# AWS Free-Tier/Three-Tiered WordPress App
This repo contains AWS CloudFormation Templates and dependencies to create a WordPress blog in AWS. The app will be loaded into a three-tier, secure VPC. Nevertheless, these templates will ONLY initiate free-tier resources.

This project serves more to build experience than to solve any practical issues associated with hosting a wordpress app on AWS. However, some of the cost-saving measures I implemented (such as running a NAT Instance rather than a NAT gateway, see section 4) may be of interest to individuals or small dev/test teams.

## Table of Contents
1. Architectural overview
2. Initialize the VPC
3. Prepare Security Groups
4. Start RDS MySQL Server and EC2 Instances
6. Configure WordPress

## Architectural Overview
This WordPress app is built on three-tier architecture. Within a single availability zone (AZ), 

<Details><Summary>See the architectural diagram</summary>
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="/assets/WordPress Architecture Dark Mode.jpeg" | width=750>
  <source media="(prefers-color-scheme: light)" srcset="/assets/WordPress Architecture.jpeg" | width=750>
  <img alt="A diagram of the architecture that is created with these CloudFormation Templates." src="/assets/WordPress Architecture.jpeg" | width=750>
</picture>
</Details>

