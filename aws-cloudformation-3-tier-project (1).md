# AWS CloudFormation – Real-World 3-Tier Infrastructure Project

## Overview

This project teaches AWS CloudFormation by building a production-style, multi-tier AWS environment using Infrastructure as Code (IaC).

Instead of creating resources manually in the AWS Console, CloudFormation templates will create and manage the complete infrastructure.

### Final Architecture

```text
                         INTERNET
                            |
                            v
                    +---------------+
                    |      ALB      |
                    | Load Balancer |
                    +-------+-------+
                            |
               +------------+------------+
               |                         |
               v                         v
        +-------------+           +-------------+
        |   EC2-A     |           |   EC2-B     |
        | Application |           | Application |
        +------+------+           +------+------+
               |                         |
               +------------+------------+
                            |
                            v
                     +-------------+
                     |  RDS MySQL  |
                     |   Private   |
                     +-------------+
```

The infrastructure will contain:

- VPC
- 2 Availability Zones
- 2 Public Subnets
- 2 Private Application Subnets
- Internet Gateway
- NAT Gateways
- Route Tables
- Security Groups
- Application Load Balancer
- Target Group
- Launch Template
- Auto Scaling Group
- EC2 instances
- RDS MySQL
- CloudFormation Parameters
- CloudFormation Outputs
- CloudFormation Conditions
- CloudFormation Mappings
- Nested Stacks

---

# 1. Learning Objectives

By completing this project, you will understand:

- Infrastructure as Code
- CloudFormation templates
- YAML syntax
- CloudFormation stacks
- Parameters
- Resources
- Outputs
- `!Ref`
- `!GetAtt`
- `!Sub`
- `!FindInMap`
- Conditions
- Mappings
- Dependencies
- Nested Stacks
- VPC networking
- Public and private subnets
- Internet Gateway
- NAT Gateway
- Route Tables
- Security Groups
- EC2
- Launch Templates
- Auto Scaling
- Application Load Balancer
- Target Groups
- RDS MySQL
- CloudFormation stack updates
- Change Sets
- Rollback
- Infrastructure troubleshooting

---

# 2. Project Structure

For a real-world project, do not put everything into one huge YAML file.

Use separate templates:

```text
cloudformation-project/
│
├── master.yaml
│
├── 01-network.yaml
├── 02-security.yaml
├── 03-database.yaml
├── 04-compute.yaml
└── 05-loadbalancer.yaml
```

### Purpose of each file

| File | Purpose |
|---|---|
| `master.yaml` | Main orchestration template |
| `01-network.yaml` | VPC and networking |
| `02-security.yaml` | Security Groups |
| `03-database.yaml` | RDS MySQL |
| `04-compute.yaml` | EC2 / Launch Template / Auto Scaling |
| `05-loadbalancer.yaml` | ALB and Target Group |

---

# 3. Deployment Flow

The complete infrastructure will follow this dependency flow:

```text
Master Stack
     |
     +--------------------+
     |                    |
     v                    v
 Network Stack       Security Stack
     |                    |
     +----------+---------+
                |
                v
        Database Stack
                |
                v
         Compute Stack
                |
                v
       Load Balancer Stack
```

The network must exist before resources that depend on the network.

---

# 4. Stage 1 – Create the VPC

The first stage creates:

```text
VPC
 |
 +-- Availability Zone A
 |      |
 |      +-- Public Subnet A
 |      +-- Private Subnet A
 |
 +-- Availability Zone B
        |
        +-- Public Subnet B
        +-- Private Subnet B
```

## Suggested CIDR design

```text
VPC
10.0.0.0/16

Public Subnet A
10.0.1.0/24

Public Subnet B
10.0.2.0/24

Private Subnet A
10.0.11.0/24

Private Subnet B
10.0.12.0/24
```

---

# 5. Stage 2 – Internet Gateway

The Internet Gateway provides Internet connectivity to the public subnets.

```text
Internet
   |
   v
Internet Gateway
   |
   v
VPC
   |
   +-- Public Subnet A
   +-- Public Subnet B
```

Public subnet route:

```text
0.0.0.0/0
     |
     v
Internet Gateway
```

---

# 6. Stage 3 – NAT Gateway

Private application instances should not be directly reachable from the Internet.

However, they may need outbound Internet access for:

- Windows updates
- Linux package installation
- Application downloads
- API access

NAT Gateway provides outbound Internet access.

```text
Private EC2
    |
    v
NAT Gateway
    |
    v
Internet Gateway
    |
    v
Internet
```

The Internet cannot directly initiate a connection to the private EC2 through the NAT Gateway.

---

# 7. Stage 4 – Security Groups

Use separate Security Groups for each layer.

```text
Internet
   |
   v
ALB Security Group
   |
   v
Application Security Group
   |
   v
Database Security Group
```

## ALB Security Group

Allow:

```text
HTTP  80
HTTPS 443
```

from the Internet if required.

## Application Security Group

Allow application traffic only from the ALB Security Group.

Example:

```text
ALB-SG
   |
   | TCP 80
   v
APP-SG
```

## Database Security Group

Allow MySQL:

```text
TCP 3306
```

only from the Application Security Group.

```text
APP-SG
   |
   | TCP 3306
   v
DB-SG
```

Do not expose RDS port 3306 directly to the Internet.

---

# 8. Stage 5 – EC2 Launch Template

Instead of creating EC2 instances individually, create a Launch Template.

```text
Launch Template
       |
       v
Auto Scaling Group
       |
       +---- EC2-A
       |
       +---- EC2-B
```

The Launch Template can define:

- AMI
- Instance Type
- Security Group
- IAM Instance Profile
- EBS volume
- User Data
- Tags

---

# 9. Stage 6 – Auto Scaling Group

The Auto Scaling Group maintains the desired number of EC2 instances.

Example:

```text
Desired Capacity = 2
Minimum Capacity = 2
Maximum Capacity = 4
```

Architecture:

```text
             Auto Scaling Group
                     |
             +-------+-------+
             |               |
             v               v
           EC2-A           EC2-B
```

If an instance fails, Auto Scaling can replace it.

---

# 10. Stage 7 – Application Load Balancer

The Application Load Balancer distributes traffic between application servers.

```text
User
 |
 v
ALB
 |
 +----> EC2-A
 |
 +----> EC2-B
```

The ALB uses a Target Group.

```text
ALB
 |
 v
Target Group
 |
 +---- EC2-A
 |
 +---- EC2-B
```

---

# 11. Stage 8 – RDS MySQL

The database should be placed in private subnets.

```text
EC2-A
   |
   |
EC2-B
   |
   +----------+
              |
              v
          RDS MySQL
```

Use a DB Subnet Group containing the private subnets.

Example:

```text
Private Subnet A
Private Subnet B
       |
       v
RDS DB Subnet Group
       |
       v
RDS MySQL
```

For production environments, consider Multi-AZ and appropriate backup settings.

---

# 12. CloudFormation Parameters

Parameters make templates reusable.

Example:

```yaml
Parameters:

  InstanceType:
    Type: String
    Default: t3.micro

  EnvironmentName:
    Type: String
    Default: dev
```

When creating the stack, values can be changed without editing the template.

---

# 13. CloudFormation Resources

Resources define AWS infrastructure.

Example:

```yaml
Resources:

  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
```

This tells CloudFormation to create a VPC.

---

# 14. CloudFormation Outputs

Outputs expose useful values after deployment.

Example:

```yaml
Outputs:

  VPCId:
    Description: VPC ID
    Value: !Ref MyVPC
```

Other useful outputs:

```text
VPC ID
Subnet IDs
Security Group IDs
Load Balancer DNS
RDS Endpoint
EC2 Instance IDs
```

---

# 15. Important CloudFormation Functions

## `!Ref`

Returns a resource's reference value.

Example:

```yaml
VpcId: !Ref MyVPC
```

## `!GetAtt`

Gets an attribute from a resource.

Example:

```yaml
Value: !GetAtt MyLoadBalancer.DNSName
```

## `!Sub`

Builds a string using variables.

Example:

```yaml
Value: !Sub '${MyLoadBalancer.DNSName}:80'
```

## `!FindInMap`

Gets a value from a Mapping.

Example:

```yaml
ImageId: !FindInMap
  - RegionMap
  - !Ref AWS::Region
  - AMI
```

---

# 16. CloudFormation Mappings

Mappings allow region-specific or environment-specific values.

Example:

```yaml
Mappings:

  EnvironmentMap:

    dev:
      InstanceType: t3.micro

    test:
      InstanceType: t3.small

    production:
      InstanceType: t3.medium
```

Then:

```yaml
InstanceType: !FindInMap
  - EnvironmentMap
  - !Ref EnvironmentName
  - InstanceType
```

---

# 17. CloudFormation Conditions

Conditions allow resources to be created only when a condition is true.

Example:

```yaml
Conditions:

  IsProduction:
    !Equals
      - !Ref EnvironmentName
      - production
```

A resource can then use:

```yaml
Condition: IsProduction
```

---

# 18. Nested Stacks

For large projects, use nested stacks.

Example:

```text
master.yaml
     |
     +-- 01-network.yaml
     |
     +-- 02-security.yaml
     |
     +-- 03-database.yaml
     |
     +-- 04-compute.yaml
     |
     +-- 05-loadbalancer.yaml
```

The master template orchestrates the other templates.

This makes the project easier to:

- Maintain
- Test
- Update
- Troubleshoot
- Reuse

---

# 19. Deployment Order

Deploy in this order:

```text
1. Network
       |
       v
2. Security
       |
       v
3. Database
       |
       v
4. Compute
       |
       v
5. Load Balancer
       |
       v
6. Master Stack
```

For the final project, the master stack can orchestrate the complete deployment.

---

# 20. How to Create a CloudFormation Stack

Go to:

```text
AWS Console
   |
CloudFormation
   |
Stacks
   |
Create stack
   |
With new resources
```

Choose:

```text
Template is ready
```

Then:

```text
Upload a template file
```

Upload the YAML file.

Enter a stack name.

Example:

```text
production-infrastructure
```

Click:

```text
Next
Next
Next
Submit
```

---

# 21. CloudFormation Stack States

Important statuses:

```text
CREATE_IN_PROGRESS
CREATE_COMPLETE
CREATE_FAILED

UPDATE_IN_PROGRESS
UPDATE_COMPLETE
UPDATE_FAILED

ROLLBACK_IN_PROGRESS
ROLLBACK_COMPLETE

DELETE_IN_PROGRESS
DELETE_COMPLETE
```

During troubleshooting, always check the **Events** tab.

---

# 22. How to Troubleshoot CREATE_FAILED

Go to:

```text
CloudFormation
   |
Stacks
   |
Your Stack
   |
Events
```

Find:

```text
CREATE_FAILED
```

Read:

```text
Status reason
```

Example:

```text
The following resource failed to create:
WindowsSecurityGroup
```

Then look at the specific resource event for the detailed error.

---

# 23. Common Errors

### Key Pair Error

```text
Parameter validation failed:
parameter value for parameter name KeyName does not exist
```

Check:

- Key pair exists
- Correct AWS Region
- Correct KeyName

Example:

```text
aws-key
```

---

### Default VPC Error

```text
No default VPC for this user
```

Solution:

Create a VPC explicitly in CloudFormation.

Example:

```yaml
VpcId: !Ref MyVPC
```

---

### Availability Zone Error

Make sure the Availability Zones exist in the selected region.

You can use:

```yaml
AvailabilityZones:
  - !Select [0, !GetAZs '']
  - !Select [1, !GetAZs '']
```

---

### Security Group Error

Check:

- VPC ID
- CIDR
- Port
- Security Group references

---

# 24. Change Sets

Before updating a production stack, use a Change Set.

Concept:

```text
Old Template
      |
      v
New Template
      |
      v
Change Set
      |
      v
Review Changes
      |
      v
Execute
```

This lets you review what CloudFormation plans to change.

---

# 25. Stack Rollback

CloudFormation can automatically roll back failed deployments.

Example:

```text
CREATE
  |
  +-- VPC       SUCCESS
  +-- Subnet    SUCCESS
  +-- Security  FAILED
  |
  v
ROLLBACK
```

CloudFormation may remove resources created during the failed operation.

---

# 26. CloudFormation + Git

Store templates in Git.

Example:

```text
GitHub
   |
   +-- master.yaml
   +-- 01-network.yaml
   +-- 02-security.yaml
   +-- 03-database.yaml
   +-- 04-compute.yaml
   +-- 05-loadbalancer.yaml
```

Benefits:

- Version control
- Change history
- Team collaboration
- Code review
- CI/CD integration

---

# 27. CloudFormation + Jenkins

A DevOps pipeline can look like:

```text
Developer
    |
    v
GitHub
    |
    v
Jenkins
    |
    v
Validate CloudFormation
    |
    v
Create Change Set
    |
    v
Review
    |
    v
Deploy
    |
    v
AWS
```

Possible validation commands/tools include:

```text
cfn-lint
CloudFormation validate-template
Change Sets
```

---

# 28. Final Architecture

The final environment should look approximately like:

```text
                         INTERNET
                            |
                            v
                   +----------------+
                   | Internet       |
                   | Gateway        |
                   +-------+--------+
                           |
                           v
              +--------------------------+
              |          VPC             |
              |      10.0.0.0/16         |
              |                          |
              |  PUBLIC SUBNETS          |
              |  +--------------------+  |
              |  | ALB                |  |
              |  +---------+----------+  |
              |            |             |
              |  +---------+----------+  |
              |  |                    |  |
              |  v                    v  |
              | EC2-A               EC2-B|
              |  |                    |  |
              |  +---------+----------+  |
              |            |             |
              |  PRIVATE SUBNETS         |
              |            |             |
              |            v             |
              |       +----------+       |
              |       | RDS      |       |
              |       | MySQL    |       |
              |       +----------+       |
              |                          |
              +--------------------------+
```

---

# 29. Recommended Project Progression

Complete the project in this order:

## Phase 1

```text
VPC
Subnets
Internet Gateway
Route Tables
```

## Phase 2

```text
NAT Gateway
Private Routing
```

## Phase 3

```text
Security Groups
```

## Phase 4

```text
EC2
Launch Template
Auto Scaling Group
```

## Phase 5

```text
Application Load Balancer
Target Group
Health Checks
```

## Phase 6

```text
RDS MySQL
DB Subnet Group
Database Security
```

## Phase 7

```text
Nested Stacks
Parameters
Mappings
Conditions
Outputs
```

## Phase 8

```text
GitHub
Jenkins
CloudFormation CI/CD
```

---

# 30. Final Goal

The final goal is to be able to run:

```text
master.yaml
```

and have CloudFormation create the complete environment:

```text
VPC
 |
 +-- 2 Availability Zones
 |
 +-- Public Subnets
 |
 +-- Private Subnets
 |
 +-- Internet Gateway
 |
 +-- NAT Gateway
 |
 +-- Security Groups
 |
 +-- ALB
 |
 +-- Target Group
 |
 +-- Launch Template
 |
 +-- Auto Scaling Group
 |      |
 |      +-- EC2
 |      +-- EC2
 |
 +-- RDS MySQL
```

This project gives you practical experience with **AWS networking + EC2 + Load Balancing + Auto Scaling + RDS + Infrastructure as Code + CloudFormation + DevOps CI/CD**.
