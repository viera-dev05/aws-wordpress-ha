# Design Decisions

## 1. Overview

This document describes the main design decisions made during the development of the AWS WordPress infrastructure. It explains the reasoning behind the selected services, configurations, and implementation approaches, including relevant trade-offs and limitations.

The architecture was designed to demonstrate high availability, network segmentation, scalability, and shared storage while considering the resource and service restrictions of the AWS Academy Learner Lab environment.

## 2. Application Selection

### 2.1 WordPress

WordPress was selected as the application platform because it provides a functional website without requiring application development from scratch.

This allowed the project to focus primarily on designing, deploying, and managing the AWS infrastructure rather than developing a custom web application.

## 3. Application Deployment

### 3.1 Docker and Custom AMI

Docker was selected as the application deployment mechanism for the following reasons:

- **Local testing:** The application environment could be tested locally before deployment to AWS.
- **Portability:** Docker provides a consistent application environment across different systems.
- **Isolation:** Application components run in containers, reducing dependencies on the host environment.
- **Maintainability:** Docker Compose simplifies container configuration and management.

A custom Amazon Machine Image (AMI) was also created from a previously configured EC2 instance.

The AMI includes the required host configuration, Docker installation, and WordPress deployment files. It is used by the Launch Template to provision application instances through the Auto Scaling Group.

## 4. Compute

### 4.1 EC2 Instance Type

The `t3.medium` instance type was selected for the WordPress application instances.

It provides 2 vCPUs and 4 GiB of memory, offering a reasonable starting point for the resource requirements of this academic project.

This instance type was selected as an initial configuration and could be adjusted according to application performance and resource utilization.

### 4.2 Auto Scaling Group

An Auto Scaling Group was selected to manage the lifecycle and availability of the WordPress application instances.

It allows EC2 instances to be provisioned automatically using the custom AMI and Launch Template.

The initial configuration was planned with the following capacity:

- Minimum capacity: 2 instances.
- Desired capacity: 2 instances.
- Maximum capacity: 4 instances.

However, the configuration was reduced to control resource consumption and costs within the AWS Academy Learner Lab environment:

- Minimum capacity: 1 instance.
- Desired capacity: 1 instance.
- Maximum capacity: 2 instances.

No dynamic scaling policies were configured. The Auto Scaling Group currently operates within its defined capacity limits.

This configuration demonstrates automated instance provisioning, although the reduced minimum capacity does not provide continuous application-level redundancy.

## 5. Networking

### 5.1 VPC and Subnet Segmentation

A dedicated VPC was created to isolate the project infrastructure from other AWS networks.

The VPC uses the CIDR block `10.0.0.0/16` and contains six subnets distributed across two Availability Zones:

- Two public subnets.
- Two private application subnets.
- Two private database subnets.

This segmentation separates resources according to their responsibilities and network access requirements.

The public subnets host the Application Load Balancer and the temporary administrative EC2 instance. The private application subnets host the WordPress instances, while the private database subnets are dedicated to Amazon RDS.

This design limits direct public exposure of application and database resources.

### 5.2 Availability Zones

Two Availability Zones were selected to distribute infrastructure resources and improve availability.

The application and database layers are distributed across separate Availability Zones to reduce dependence on a single location within the AWS Region.

This architecture provides a foundation for redundancy and fault tolerance, although some components, such as the single NAT Gateway, introduce availability limitations.

### 5.3 Application Load Balancer

An internet-facing Application Load Balancer (ALB) was selected as the public entry point for the WordPress application.

It distributes incoming HTTP requests across the application instances managed by the Auto Scaling Group.

The ALB allows the application instances to remain in private subnets without requiring direct public exposure.

An HTTP listener on TCP port 80 was configured. HTTPS termination was not implemented due to the academic scope of the project.

### 5.4 NAT Gateway Configuration

A NAT Gateway was deployed in a public subnet to provide outbound internet connectivity for resources in the private subnets.

The initial design considered deploying one NAT Gateway per Availability Zone. However, a single shared NAT Gateway was selected to reduce infrastructure costs.

This configuration allows private application instances to access external resources without requiring public IP addresses.

The trade-off is reduced availability: if the Availability Zone hosting the NAT Gateway becomes unavailable, private resources in the other Availability Zone may lose outbound internet connectivity.

Using a shared NAT Gateway may also introduce cross-Availability Zone traffic and associated costs.

For a production environment requiring greater availability, deploying a NAT Gateway in each Availability Zone should be considered.

## 6. Database

### 6.1 MySQL vs. MariaDB

MySQL was selected as the database engine for the WordPress application.

Both MySQL and MariaDB are supported database options for WordPress. No specific performance advantage was required for the scope of this project.

MySQL was chosen to maintain a conventional WordPress database configuration.

### 6.2 Amazon RDS Multi-AZ

Amazon RDS for MySQL was selected as the managed database service.

A Multi-AZ deployment was configured to improve database availability and provide automatic failover capabilities.

The database is deployed in private subnets and does not allow public accessibility.

This approach reduces database administration requirements while providing a more resilient database architecture than a single-instance deployment.

## 7. Shared Storage

### 7.1 Amazon EFS vs. Amazon S3

Amazon Elastic File System (EFS) was selected to provide shared storage for WordPress content across application instances.

EFS allows multiple EC2 instances to access the same file system, which is important when application instances are managed by an Auto Scaling Group.

A Regional EFS file system with mount targets in both Availability Zones was selected to support access from application instances in either zone.

Amazon S3 was considered as an alternative. However, integrating S3 with WordPress for media storage would require additional configuration or plugins.

EFS was therefore selected because it provides direct shared file system access with less application-level modification.

## 8. Security-Related Decisions

### 8.1 Credential Management

Database credentials are stored in a `.env` file used by Docker Compose to configure the WordPress container.

The file is included in the custom AMI used by the Auto Scaling Group so that newly launched instances can obtain the database configuration automatically.

This approach was selected as a practical simplification for the academic project.

However, including credentials in an AMI is not recommended for production environments because users with sufficient access to the image or instance filesystem may retrieve them.

AWS Secrets Manager should be considered for production deployments to securely store and retrieve application credentials at runtime.

### 8.2 Network ACL Configuration

No custom Network ACL rules were configured as part of this project. The default Network ACL configuration was retained.

Traffic restrictions between infrastructure components are primarily managed through Security Groups.

Custom Network ACL rules could be introduced in a production environment if additional subnet-level traffic controls are required.

### 8.3 Host-based Firewall Configuration

No host-based firewall was configured on the Amazon Linux 2023 EC2 instances. The `firewalld` package was not installed.

Network traffic restrictions are primarily managed through AWS Security Groups.

This configuration was retained due to the academic scope of the project. For production deployments, host-based firewall rules could provide an additional layer of defense in depth.

## 9. Conclusion

The design decisions prioritize network segmentation, application portability, shared storage, and database availability while maintaining reasonable infrastructure costs within the AWS Academy Learner Lab environment.

Some configurations were simplified to accommodate the academic scope and resource limitations. These include the single NAT Gateway, reduced Auto Scaling Group capacity, HTTP-only access, and simplified credential management.

The resulting architecture demonstrates fundamental AWS infrastructure design principles while identifying the trade-offs and improvements required for a more resilient and production-ready deployment.
