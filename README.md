# Terraform Project: AWS VPC with Public & Private Subnets and EC2 Deployment
### Terraform · AWS VPC · EC2 · Internet Gateway · Route Tables · Security Groups

[![AWS](https://img.shields.io/badge/AWS-Infrastructure-orange?logo=amazon-aws)](https://aws.amazon.com/)
[![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform)](https://www.terraform.io/)
[![VPC](https://img.shields.io/badge/Amazon-VPC-FF9900?logo=amazon-aws)](https://aws.amazon.com/vpc/)
[![EC2](https://img.shields.io/badge/Amazon-EC2-yellow?logo=amazon-aws)](https://aws.amazon.com/ec2/)

---

## Overview

This project demonstrates how to provision a complete AWS networking environment using Terraform. The infrastructure includes a VPC, public and private subnets, an Internet Gateway, route tables, security groups, and an EC2 instance deployed inside the public subnet.

The goal is to simulate a basic production-style infrastructure deployment using Infrastructure as Code, demonstrating how AWS networking primitives are assembled and managed through a repeatable, version-controlled workflow.

---

## Architecture

![Architecture Diagram](architecture/00-terraform-vpc-ec2-architecture.png)

```
Internet
      │
      ▼
Internet Gateway  ── Provides internet access to the VPC
      │
      ▼
Public Route Table  ── Routes outbound traffic to the Internet Gateway
      │
      ▼
Public Subnet (10.0.1.0/24)  ── Hosts internet-facing resources
      │
      ▼
EC2 Instance (t3.micro)  ── Application server with public IP

Private Subnet (10.0.2.0/24)  ── Reserved for private services (databases, etc.)

Security Group  ── Controls inbound SSH access on port 22
```

---

## Services Used

| Service | Role |
|---|---|
| **Terraform** | Provisions and manages all AWS infrastructure as code |
| **Amazon VPC** | Isolated virtual network with CIDR block 10.0.0.0/16 |
| **Public Subnet** | Subnet for internet-facing resources (10.0.1.0/24) |
| **Private Subnet** | Subnet reserved for private infrastructure (10.0.2.0/24) |
| **Internet Gateway** | Allows communication between the VPC and the public internet |
| **Route Table** | Routes outbound traffic from the public subnet to the Internet Gateway |
| **Security Group** | Firewall rules controlling inbound and outbound traffic |
| **Amazon EC2** | t3.micro compute instance deployed in the public subnet |

---

## Infrastructure Flow

1. Terraform initialises the working directory and downloads the AWS provider plugin
2. `terraform plan` previews all resources to be created before any changes are made
3. `terraform apply` provisions the VPC, subnets, Internet Gateway, route tables, security group, and EC2 instance in sequence
4. The EC2 instance is launched in the public subnet with a public IP assigned automatically
5. The Internet Gateway is attached to the VPC and the public route table directs 0.0.0.0/0 traffic to it
6. The security group permits inbound SSH on port 22, enabling remote access to the instance
7. `terraform destroy` tears down all provisioned resources cleanly, leaving no orphaned infrastructure

---

## Project Structure

```
terraform-aws-vpc-ec2/
│
├── architecture/
│   └── 00-terraform-vpc-ec2-architecture.png    # End-to-end architecture overview
│
├── terraform/
│   └── main.tf                                  # All AWS resource definitions
│
├── screenshots/
│   ├── 01-terraform-init.png                    # Terraform initialisation output
│   ├── 02-terraform-plan-preview.png            # Plan resource preview
│   ├── 03-terraform-plan-summary.png            # Plan summary with resource count
│   ├── 04-terraform-apply-success.png           # Apply completion output
│   ├── 05-ec2-instance-running.png              # EC2 instance in running state
│   ├── 06-vpc-resource-map.png                  # VPC resource map in AWS console
│   └── 07-terraform-destroy-success.png         # Destroy completion output
│
├── README.md
├── LICENSE
└── .gitignore
```

---

## Screenshots

### 1. Terraform Init
*Terraform initialised successfully with the AWS provider downloaded and backend configured.*

![Terraform Init](screenshots/01-terraform-init.png)

---

### 2. Terraform Plan Preview
*Plan output previewing all resources to be created before any changes are applied.*

![Terraform Plan Preview](screenshots/02-terraform-plan-preview.png)

---

### 3. Terraform Plan Summary
*Plan summary confirming the total number of resources to be added, changed, and destroyed.*

![Terraform Plan Summary](screenshots/03-terraform-plan-summary.png)

---

### 4. Terraform Apply Success
*Apply completed successfully with all resources provisioned and outputs displayed.*

![Terraform Apply](screenshots/04-terraform-apply-success.png)

---

### 5. EC2 Instance Running
*EC2 instance in a running state with a public IP assigned, deployed in the public subnet.*

![EC2 Running](screenshots/05-ec2-instance-running.png)

---

### 6. VPC Resource Map
*AWS console resource map showing the VPC, subnets, Internet Gateway, and route table associations.*

![VPC Resource Map](screenshots/06-vpc-resource-map.png)

---

### 7. Terraform Destroy
*All resources successfully destroyed, confirming clean infrastructure teardown.*

![Terraform Destroy](screenshots/07-terraform-destroy-success.png)

---

## Troubleshooting

### 1. EC2 Instance Not Reachable After Deployment

After `terraform apply` completed successfully and the EC2 instance moved to a running state, SSH connections to the public IP were timing out with no response. The instance appeared healthy in the AWS console and the public IP was assigned correctly.

The root cause was a missing outbound rule and an overly restrictive inbound rule in the security group. The security group had been configured to allow inbound SSH on port 22, but the source CIDR was set to a specific IP range that didn't include the machine being used for testing. Additionally, Terraform's default security group behaviour does not automatically add an outbound rule, so return traffic was also being dropped.

**Fix:** Updated the security group resource in `main.tf` to set the inbound SSH source CIDR to `0.0.0.0/0` for testing purposes, and added an explicit egress block allowing all outbound traffic. Ran `terraform apply` to update the security group in place, after which SSH connections succeeded immediately.

**Lesson:** Security groups are stateful, but both inbound and egress rules must be explicitly defined in Terraform — unlike the AWS console, Terraform does not add a default allow-all egress rule automatically. Always verify the source CIDR on inbound rules matches your actual IP, and confirm egress rules are present before concluding the instance itself is misconfigured.

---

### 2. Terraform Apply Failing with "VPC Limit Exceeded"

Running `terraform apply` in a fresh AWS account returned an error immediately: `Error: creating VPC: VpcLimitExceeded`. No resources were created and the apply exited with a non-zero status.

The cause was hitting the default AWS limit of five VPCs per region. Previous project work and console experiments had consumed all available slots without explicit cleanup. Because Terraform had not provisioned those VPCs, it had no state for them and could not destroy them — they had to be removed manually.

**Fix:** Navigated to the VPC console in the target region and deleted unused VPCs that were left over from earlier experiments. Confirmed the VPC count dropped below the limit, then re-ran `terraform apply`, which completed successfully.

**Lesson:** AWS service limits apply to all resources in a region regardless of how they were created. Terraform cannot manage or destroy resources it did not provision. Running regular cleanup of manually created resources — or using AWS Config to track unmanaged infrastructure — prevents quota exhaustion from blocking IaC deployments. If limits are a persistent issue, a Service Quotas increase request can be submitted directly from the AWS console.

---

### 3. Public Subnet Instance Not Getting a Public IP

After deployment, the EC2 instance in the public subnet had no public IP address despite being placed in the subnet intended for internet-facing resources. The instance was reachable only from within the VPC and the VPC resource map showed no public IP association.

The issue was that the subnet resource in `main.tf` was missing the `map_public_ip_on_launch = true` attribute. Without this setting, instances launched into the subnet receive only a private IP by default, even if the subnet is associated with a route to the Internet Gateway. The routing was correct but the instance had no public address to route to.

**Fix:** Added `map_public_ip_on_launch = true` to the `aws_subnet` resource for the public subnet in `main.tf`. Because changing this attribute on an existing subnet does not retroactively assign IPs to running instances, the EC2 instance also needed to be replaced. Running `terraform apply` updated the subnet and triggered an instance replacement, after which the new instance launched with a public IP assigned.

**Lesson:** A subnet being associated with a public route table does not automatically make instances in it publicly accessible — the subnet must also be configured to assign public IPs at launch. These are two separate configuration points that must both be correct. When an instance is missing a public IP, check `map_public_ip_on_launch` on the subnet before investigating routing or Internet Gateway configuration.

---

## What I Learned

This project demonstrated that Infrastructure as Code is not just a convenience — it is a discipline that changes how infrastructure is designed, documented, and operated.

**Terraform's plan step is a forcing function for intentional infrastructure.** Before any resource is created, `terraform plan` surfaces every addition, modification, and destruction in a readable diff. This makes it impossible to accidentally provision resources or misconfigure settings without first seeing the intended change. Reviewing the plan carefully before applying — especially checking resource counts and attribute values — catches mistakes that would otherwise only appear after deployment.

**Subnet and routing configuration requires precision at multiple layers.** A working public subnet depends on at least three independent settings being correct simultaneously: the subnet must have `map_public_ip_on_launch` enabled, the route table must contain a route to the Internet Gateway, and the Internet Gateway must be attached to the VPC. Any one of these missing produces the same symptom — no internet connectivity — but each has a different fix. Building the habit of verifying all three before debugging deeper layers saves significant time.

**State is both Terraform's greatest strength and its most important constraint.** Terraform can only manage resources it provisioned. Resources created outside of Terraform — through the console, CLI, or other tools — are invisible to it and cannot be modified or destroyed through `terraform destroy`. Keeping all infrastructure changes within the Terraform workflow from the start avoids state drift and ensures the codebase remains the single source of truth for what is deployed.

---

## Author

**Sergiu Gota**
AWS Cloud Engineer

[![GitHub](https://img.shields.io/badge/GitHub-sergiugotacloud-181717?logo=github)](https://github.com/sergiugotacloud)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-sergiu--gota--cloud-0A66C2?logo=linkedin)](https://linkedin.com/in/sergiu-gota-cloud)

> Built as part of a cloud portfolio to demonstrate production-style infrastructure provisioning with Terraform on AWS.
> Feel free to fork, adapt, or reach out with questions.
