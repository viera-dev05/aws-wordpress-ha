# Security Configuration

## 1. Overview

This document describes the security configuration implemented in the AWS WordPress infrastructure, including network isolation, access restrictions, data encryption, credential management, and known security limitations.

The infrastructure was deployed using AWS Academy Learner Lab. Some security features are not available or were intentionally omitted due to the academic scope and environment restrictions. Network ACLs (NACLs) were not explicitly configured as part of this project. The default Network ACL configuration was retained.

## 2. Network Security

The infrastructure uses a Virtual Private Cloud (VPC) with public and private subnets distributed across two Availability Zones.

Network isolation is implemented through subnet separation and Security Groups.

### 2.1 Network Isolation

The architecture separates resources according to their function:

- **Public subnets:** Host the Application Load Balancer and the temporary EC2 instance (`aws-wordpress-temp`). This instance was not included in the architecture because it is not part of the normal function of the application.
- **Private application subnets:** Host the WordPress EC2 instances managed by the Auto Scaling Group.
- **Private database subnets:** Host the Amazon RDS MySQL database.

The database is not publicly accessible. External traffic reaches the application through the Application Load Balancer.

### 2.2 Security Groups

The infrastructure uses four main Security Groups to control communication between components.

| Security Group | Inbound Rules | Purpose |
|---|---|---|
| `wordpress-alb-sg` | TCP 80 from `0.0.0.0/0` | Allows incoming HTTP traffic to the Application Load Balancer. |
| `wordpress-ec2-sg` | TCP 8080 from `wordpress-alb-sg` | Allows application traffic from the Load Balancer to WordPress instances. |
| `wordpress-efs-sg` | TCP 2049 from `wordpress-ec2-sg` | Allows NFS communication between EC2 instances and EFS. |
| `wordpress-rds-sg` | TCP 3306 from `wordpress-ec2-sg` | Allows MySQL connections from the application instances. |

Outbound rules allow all traffic to `0.0.0.0/0`.

This configuration restricts inbound communication between infrastructure components to the required application and storage ports.

### 2.3 Administrative Access

A temporary EC2 instance named `aws-wordpress-temp` is deployed in a public subnet for administrative and testing purposes.

Its Security Group includes an SSH rule (TCP port 22) restricted to the administrator's public IP address (`My IP`).

This rule remains enabled because the temporary instance is required for infrastructure testing and administration.

The application instances managed by the Auto Scaling Group remain in private subnets.

### 2.4 Host-based Firewall

The `firewalld` package is not installed on the Amazon Linux 2023 instances. No additional host-based firewall rules were configured as part of this project due to its academic scope.

Network traffic restrictions are primarily managed through AWS Security Groups.

For production deployments, host-based firewall rules should be considered as an additional layer of defense in depth.

## 3. Identity and Access Management

No IAM Roles or Instance Profiles are associated with the EC2 instances.

IAM-based access management for application instances was not implemented because of the restrictions of the AWS Academy Learner Lab environment.

Consequently, the instances do not use IAM Roles to obtain temporary AWS credentials for accessing other AWS services.

This represents a limitation compared with a production deployment, where dedicated IAM Roles with least-privilege permissions should be assigned to application instances.

## 4. Data Protection and Encryption

Encryption is implemented for the persistent storage services used by the application.

### 4.1 Amazon RDS

The Amazon RDS MySQL database has encryption at rest enabled.

- Encryption is managed through AWS Key Management Service (AWS KMS).
- The AWS-managed KMS key `aws/rds` is used.
- Public accessibility is disabled.
- IAM database authentication is disabled.
- Database connections are restricted through `wordpress-rds-sg`, allowing MySQL traffic only from the application instances.

### 4.2 Amazon EFS

Amazon Elastic File System (EFS) uses encryption at rest and encryption in transit.

- Encryption at rest is enabled.
- Encryption in transit is enabled through TLS.
- NFS traffic uses TCP port 2049.
- Access is restricted through `wordpress-efs-sg`.

EFS provides shared storage for WordPress content across application instances.

### 4.3 Application Load Balancer

The Application Load Balancer is internet-facing and accepts HTTP traffic on TCP port 80.

HTTPS termination and SSL/TLS certificates through AWS Certificate Manager (ACM) have not been configured.

Therefore, communication between external clients and the Load Balancer is not encrypted at the application transport layer.

This is a known security limitation and should be addressed before deploying the architecture in a production environment.

## 5. Credential Management

WordPress database credentials are stored in an environment file named `.env`.

Docker Compose uses these environment variables to configure the WordPress container, while `wp-config.php` defines the database connection constants:

- `DB_NAME`
- `DB_USER`
- `DB_PASSWORD`
- `DB_HOST`

The `.env` file is included in the Amazon Machine Image (AMI) used by the Auto Scaling Group. This allows newly launched instances to obtain the database configuration without requiring manual configuration.

This approach was selected as a practical simplification for the academic project.

However, storing database credentials directly in an AMI is not considered a recommended production practice. Anyone with sufficient access to the AMI or the instance filesystem may potentially retrieve those credentials.

For a production deployment, AWS Secrets Manager or another dedicated secrets management solution should be used to store and retrieve database credentials securely.

## 6. Security Monitoring and Additional Services

The following AWS security services were not implemented in this project:

| Service | Status | Description |
|---|---|---|
| AWS WAF | Not implemented | No web application firewall is configured to filter malicious HTTP requests. |
| AWS CloudTrail | Not implemented | API activity auditing was not configured as part of this project. |
| Amazon GuardDuty | Not implemented | No managed threat detection is enabled. |
| AWS Config | Not implemented | Resource configuration monitoring and compliance tracking were not configured. |
| AWS Secrets Manager | Not implemented | Application credentials are managed through the `.env` file. |

CloudWatch metrics are available for infrastructure monitoring, but no additional security monitoring or threat detection configuration was implemented.

## 7. Security Limitations and Production Recommendations

The following improvements should be considered before using this architecture in a production environment:

| Area | Current Limitation | Production Recommendation |
|---|---|---|
| IAM | EC2 instances have no associated IAM Roles. | Assign dedicated IAM Roles with least-privilege permissions. |
| HTTPS | The Application Load Balancer accepts HTTP traffic only. | Configure an HTTPS listener and an ACM certificate. |
| Credentials | Database credentials are stored in `.env` and included in the AMI. | Use AWS Secrets Manager and retrieve credentials securely at runtime. |
| SSH Access | SSH access remains enabled for the temporary public EC2 instance. | Use a controlled administrative access solution and restrict SSH exposure. |
| Host-based Firewall | No host-based firewall was configured on the EC2 instances. | Implement host-based firewall rules as an additional layer of defense. |
| Web Protection | AWS WAF is not configured. | Implement AWS WAF to filter malicious web traffic. |
| Audit Logging | CloudTrail is not configured as part of this project. | Enable CloudTrail for API activity auditing. |
| Threat Detection | GuardDuty is not enabled. | Enable Amazon GuardDuty for threat detection. |
| Configuration Compliance | AWS Config is not enabled. | Use AWS Config to monitor resource configurations and compliance. |
| Network ACLs | No custom Network ACL rules were configured. | Define and implement appropriate Network ACL rules if additional subnet-level traffic controls are required. |

These limitations reflect the academic scope and constraints of the AWS Academy environment rather than a complete production security implementation.

## 8. Conclusion

The project implements basic infrastructure security through network segmentation, Security Groups, private application and database subnets, restricted database access, and encryption for RDS and EFS.

However, certain security capabilities, including IAM-based instance permissions, HTTPS termination, custom Network ACL rules, host-based firewall configuration, centralized secrets management, and advanced threat detection, remain outside the current implementation.

The architecture demonstrates fundamental AWS security practices while identifying the additional controls required for a production-ready deployment.
