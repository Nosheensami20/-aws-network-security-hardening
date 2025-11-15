# Network Security Hardening Project

## Project Overview

This project demonstrates the implementation of a comprehensive, multi-layered AWS network security architecture using AWS CLI. The solution establishes defense-in-depth through VPC segmentation, security group hardening, network ACL restrictions, and WAF protection.

## Architecture

![AWS Network Security Architecture](images/architecture-diagram.png)

## Prerequisites
- AWS CLI installed and configured
- Admin AWS account access for initial IAM user setup
- Your public IP address for SSH access restrictions
- Basic understanding of AWS VPC networking concepts

## Project Goals
- Build secure VPC with network segmentation
- Implement least-privilege security policies
- Deploy WAF for application-layer protection

## Project Structure: 
```
├── README.md                           # Project overview
├── images/
│   └── architecture-diagram.png        # Architecture diagram
├── docs/                               # Implementation guide
│   ├── phase-1-network-foundation.md
│   ├── phase-2-security-hardening.md
│   └── phase-3-waf-alb.md
│── screenshots/                        # Visual proof  
│── scripts                             # Configuration files
```
## Implementation Phases

### Phase I: Network Foundation
**What we built:** VPC, Subnets, IGW, NAT Gateway, Route Tables

- Created an isolated VPC (10.0.0.0/16) with public/private subnet segmentation
- Configured Internet Gateway for public subnet internet access
- Deployed NAT Gateway to enable secure outbound traffic from private subnets
- Established proper routing tables for controlled traffic flows

👉 [View detailed guide](docs/phase-1-network-foundation.md)

### Phase II: Security Hardening
**What we hardened:** Security Groups, NACLs

- Audited and hardened Security Groups following least-privilege principles
- Restricted NACLs to allow only necessary traffic (SSH from admin IP, HTTP/HTTPS)
- Eliminated overly permissive default security configurations
- Verified no resources remain under insecure default security groups

👉 [View detailed guide](docs/phase-2-security-hardening.md)

### Phase III: WAF & ALB Protection
**What we deployed:** ALB, WAF, CloudWatch logging

- Deployed Application Load Balancer with AWS WAF protection
- Configured managed rule groups to block common web exploits (SQL injection, anonymous IPs, known bad inputs)
- Successfully blocked real-world attack attempts with 403 responses
- Implemented CloudWatch logging for threat monitoring and audit trails
- Hardened security groups to restrict EC2 access exclusively through ALB

👉 [View detailed guide](docs/phase-3-waf-alb.md)

## Security Outcomes
Successfully blocked actual attacks:
- **SQL injection** attempts
- **Anonymous IP** traffic

**Example Attack Blocked:**
- Client IP: 146.103.3.7
- Attempted URI: /.env
- Terminating Rule: ExploitablePaths_URIPATH
- Action: BLOCK (403 Forbidden)

## Project Achievments:
- Defense-in-depth architecture
- Real-time threat detection and blocking
- Comprehensive audit logging
- Zero-trust network segmentation

## Cleanup:
To avoid unexpected AWS charges, remove all resources (VPC, subnets, NAT Gateway, ALB, EC2, RDS, CloudWatch logs) created for this project after testing.
"# -aws-network-security-hardening" 
