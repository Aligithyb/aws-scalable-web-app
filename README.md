Scalable & Highly Available Web Application Architecture on AWS
Project: Graduation Project — Scalable Web Application with Decoupled Frontend/Backend, ALB, and Auto Scaling
1. Solution Architecture Diagram
graph TD
    User([End User / Web Client]) -->|DNS Lookup| R53[Amazon Route 53]
    R53 -->|Alias Target| CF[Amazon CloudFront CDN]
    CF -->|HTTPS Traffic| WAF[AWS WAF Web ACL]
    WAF -->|Filtered Traffic| ALB[Application Load Balancer]

    subgraph VPC [AWS VPC - 10.0.0.0/16]
        subgraph IGW_NAT [Gateway Layer]
            IGW[Internet Gateway]
            NAT1[NAT Gateway - AZ 1a]
            NAT2[NAT Gateway - AZ 1b]
        end

        subgraph AZ1 [Availability Zone 1: us-east-1a]
            subgraph FrontendSub1 [Frontend Subnet - 10.0.1.0/24]
                FE_1[Frontend Web Instance 1]
            end
            subgraph BackendSub1 [Backend App Subnet - 10.0.10.0/24]
                BE_1[Backend API Instance 1]
            end
            subgraph PrivDBSub1 [Private DB Subnet - 10.0.20.0/24]
                RDS_Primary[(RDS MySQL Primary)]
            end
        end

        subgraph AZ2 [Availability Zone 2: us-east-1b]
            subgraph FrontendSub2 [Frontend Subnet - 10.0.2.0/24]
                FE_2[Frontend Web Instance 2]
            end
            subgraph BackendSub2 [Backend App Subnet - 10.0.11.0/24]
                BE_2[Backend API Instance 2]
            end
            subgraph PrivDBSub2 [Private DB Subnet - 10.0.21.0/24]
                RDS_Standby[(RDS MySQL Standby - Sync Replication)]
            end
        end

        ASG_FE[Frontend Auto Scaling Group] -.-> FE_1
        ASG_FE -.-> FE_2
        ASG_BE[Backend Auto Scaling Group] -.-> BE_1
        ASG_BE -.-> BE_2
    end

    ALB -->|HTTP 80| FE_1
    ALB -->|HTTP 80| FE_2
    
    FE_1 -->|API Proxy 3000| BE_1
    FE_2 -->|API Proxy 3000| BE_2
    
    BE_1 -->|MySQL 3306| RDS_Primary
    BE_2 -->|MySQL 3306| RDS_Primary
    RDS_Primary <-->|Synchronous Replication| RDS_Standby

    FE_1 -.->|Outbound Updates| NAT1
    BE_1 -.->|Outbound Updates| NAT1
    FE_2 -.->|Outbound Updates| NAT2
    BE_2 -.->|Outbound Updates| NAT2
    NAT1 --> IGW
    NAT2 --> IGW


<img width="868" height="428" alt="image" src="https://github.com/user-attachments/assets/62d8b61a-7029-4794-949f-5b3bec3cdd59" />


    SSM[AWS Systems Manager - Session Manager] -.->|Bastion-Free Access| FE_1
    SSM -.->|Bastion-Free Access| BE_1
    CW[AWS CloudWatch & SNS] <== Alarms & Monitoring ==> ASG_BE

Amazon CloudFront: Configured as the global Edge distribution with optimized caching rules for static frontend assets (/static/*, CSS, JS, images), reducing load on decoupled frontend web servers.
Amazon Route 53: Manages public DNS records using Alias routing directly to CloudFront/ALB.
Compute & Load Balancing Tier
Application Load Balancer (ALB): Public-facing across frontend subnets. Routes client traffic to the decoupled frontend tier and monitors target group health via HTTP /healthcheck status codes.
Auto Scaling Groups (ASG):
 * Frontend ASG: Min Size: 2 | Desired Capacity: 2 | Max Size: 6 (Runs web/UI layer across separate subnets).
 * Backend ASG: Min Size: 2 | Desired Capacity: 2 | Max Size: 6 (Runs API/business logic layer in isolated backend subnets).
 * Launch Templates: Amazon Linux 2023 AMI with User Data scripts initializing decoupled application layers and CloudWatch agents.
 * Scaling Policies: Dynamic Target Tracking targeting 55% Average CPU Utilization.
Database Tier
Amazon RDS for MySQL: Multi-AZ deployment configuration.
Failover Capabilities: Synchronous physical block-level replication to AZ-1b with automated DNS failover under 60 seconds during availability loss.
Operational Management & Security
AWS Systems Manager (SSM) Session Manager: Bastion-free server administration. Inbound SSH port 22 is permanently closed across all instances. Administrative access is audited via AWS CloudTrail.
Amazon CloudWatch & SNS: Real-time monitoring metrics (EC2 CPU, ALB target response time, RDS free storage space) tied to SNS topics for immediate administrator notifications.
5. Security Group Control Matrix
| Security Group ID | Inbound Traffic | Source Restriction | Outbound Rules | Purposes |
|---|---|---|---|---|
| sg-alb | HTTP (80), HTTPS (443) | 0.0.0.0/0 | HTTP (3000) -> sg-frontend | Public perimeter load balancing |
| sg-frontend | Custom TCP (3000) | sg-alb ONLY | Custom TCP (8080) -> sg-backend | Decoupled frontend presentation layer isolation |
| sg-backend | Custom TCP (8080) | sg-frontend ONLY | All Traffic via NAT | Decoupled backend API computation layer isolation |
| sg-rds | MySQL (3306) | sg-backend ONLY | Outbound Ingress Only | Strict database tier access rule |
6. Project Verification & Validation
High Availability Test: Verified database resiliency and decoupled tier communication by simulating a primary node reboot (force-failover). Secondary database node promoted within 38 seconds without service downtime.
Auto Scaling Verification: Executed synthetic CPU load spikes (stress-ng) on both frontend and backend instances. CloudWatch triggered ASG alarms, independently scaling instances across both AZs and distinct subnets.
Security Audit: Confirmed 0 open external SSH ports across all decoupled subnets. Successfully established administrative shell via SSM Session Manager.








