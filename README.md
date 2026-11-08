# Scalable & Highly Available Web Application Architecture on AWS

**Author:** Ayman Aly Mahmoud  
**Email:** ayman@manara.tech  
**LinkedIn:** [Ayman Mahmoud](https://www.linkedin.com/in/ayman-mahmoud)  
**Project:** Graduation Project — Scalable Web Application with ALB and Auto Scaling  

---

## 1. Solution Architecture Diagram

```mermaid
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
            subgraph PubSub1 [Public Subnet - 10.0.1.0/24]
                ALB_Node1[ALB Node AZ-1a]
            end
            subgraph PrivAppSub1 [Private App Subnet - 10.0.10.0/24]
                EC2_1[EC2 App Instance 1]
            end
            subgraph PrivDBSub1 [Private DB Subnet - 10.0.20.0/24]
                RDS_Primary[(RDS MySQL Primary)]
            end
        end

        subgraph AZ2 [Availability Zone 2: us-east-1b]
            subgraph PubSub2 [Public Subnet - 10.0.2.0/24]
                ALB_Node2[ALB Node AZ-1b]
            end
            subgraph PrivAppSub2 [Private App Subnet - 10.0.11.0/24]
                EC2_2[EC2 App Instance 2]
            end
            subgraph PrivDBSub2 [Private DB Subnet - 10.0.21.0/24]
                RDS_Standby[(RDS MySQL Standby - Sync Replication)]
            end
        end

        ASG[Auto Scaling Group] -.-> EC2_1
        ASG -.-> EC2_2
    end

    ALB -->|HTTP 8080| EC2_1
    ALB -->|HTTP 8080| EC2_2
    
    EC2_1 -->|MySQL 3306| RDS_Primary
    EC2_2 -->|MySQL 3306| RDS_Primary
    RDS_Primary <-->|Synchronous Replication| RDS_Standby

    EC2_1 -.->|Outbound Updates| NAT1
    EC2_2 -.->|Outbound Updates| NAT2
    NAT1 --> IGW
    NAT2 --> IGW

    SSM[AWS Systems Manager - Session Manager] -.->|Bastion-Free Access| EC2_1
    SSM -.->|Bastion-Free Access| EC2_2
    CW[AWS CloudWatch & SNS] <== Alarms & Monitoring ==> ASG
