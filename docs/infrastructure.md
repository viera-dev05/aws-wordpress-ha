# Infrastructure Configuration

## 1. AWS Environment

| Parameter | Configuration |
|---|---|
| Cloud Provider | Amazon Web Services (AWS) |
| Environment | AWS Academy Learner Lab |
| Region | US East (N. Virginia) — `us-east-1` |
| Availability Zones | `us-east-1a`, `us-east-1b` |

## 2. Network Infrastructure

### VPC

| Parameter | Value |
|---|---|
| Name | `aws-wordpress-vpc` |
| IPv4 CIDR | `10.0.0.0/16` |

### Subnets

Six subnets are distributed across two Availability Zones.

| Subnet Name | Availability Zone | CIDR | Type |
|---|---|---|---|
| `subnet-public1-us-east-1a` | us-east-1a | 10.0.1.0/24 | Public |
| `subnet-private-app-us-east-1a` | us-east-1a | 10.0.11.0/24 | Private Application |
| `subnet-private-db-us-east-1a` | us-east-1a | 10.0.21.0/24 | Private Database |
| `subnet-public2-us-east-1b` | us-east-1b | 10.0.2.0/24 | Public |
| `subnet-private-app-us-east-1b` | us-east-1b | 10.0.12.0/24 | Private Application |
| `subnet-private-db-us-east-1b` | us-east-1b | 10.0.22.0/24 | Private Database |

### Internet Connectivity

| Resource | Name | Configuration |
|---|---|---|
| Internet Gateway | `igw-wordpress` | Attached to the VPC |
| NAT Gateway | `wordpress-nat-gateway` | Single NAT Gateway shared by both private application subnets |

The Internet Gateway provides internet connectivity to public subnets. The NAT Gateway allows resources in private application subnets to initiate outbound internet connections.

Database subnets have no default route to the internet.

### Route Tables

**Public Route Table — `rtb-public`**

Associated subnets:

- `subnet-public1-us-east-1a`
- `subnet-public2-us-east-1b`

| Destination | Target |
|---|---|
| 10.0.0.0/16 | Local |
| 0.0.0.0/0 | `igw-wordpress` |

**Private Application Route Table — `rtb-private1-us-east-1a`**

Associated subnet: `subnet-private-app-us-east-1a`

| Destination | Target |
|---|---|
| 10.0.0.0/16 | Local |
| 0.0.0.0/0 | `wordpress-nat-gateway` |

**Private Application Route Table — `rtb-private2-us-east-1b`**

Associated subnet: `subnet-private-app-us-east-1b`

| Destination | Target |
|---|---|
| 10.0.0.0/16 | Local |
| 0.0.0.0/0 | `wordpress-nat-gateway` |

**Private Database Route Table — `rtb-private-db`**

Associated subnets:

- `subnet-private-db-us-east-1a`
- `subnet-private-db-us-east-1b`

| Destination | Target |
|---|---|
| 10.0.0.0/16 | Local |

## 3. Application Load Balancer

| Parameter | Configuration |
|---|---|
| Name | `aws-wordpress-alb` |
| Scheme | Internet-facing |
| Availability Zones | `us-east-1a`, `us-east-1b` |
| Subnets | `subnet-public1-us-east-1a`, `subnet-public2-us-east-1b` |
| Security Group | `wordpress-alb-sg` |

### Listener Configuration

| Parameter | Value |
|---|---|
| Protocol | HTTP |
| Port | 80 |
| Default Action | Forward |
| Target Group | `aws-wordpress-private-tg` |
| Forward Weight | 100% |

HTTPS is not currently configured.

### Target Group

| Parameter | Configuration |
|---|---|
| Name | `aws-wordpress-private-tg` |
| Target Type | Instance |
| Protocol | HTTP |
| Protocol Version | HTTP/1.1 |
| Port | 8080 |

### Health Checks

| Parameter | Value |
|---|---|
| Protocol | HTTP |
| Path | `/` |
| Healthy Threshold | 5 consecutive successful checks |
| Unhealthy Threshold | 2 consecutive failed checks |
| Timeout | 5 seconds |
| Interval | 30 seconds |
| Success Codes | 200–399 |

## 4. EC2 and Auto Scaling

### Temporary Administrative Instance

A temporary EC2 instance named `aws-wordpress-temp` was initially deployed in a public subnet and used as the base environment for configuring the WordPress application.

After completing the application configuration, an Amazon Machine Image (AMI) was created from this instance. This AMI serves as the launch template image used by the Auto Scaling Group to provision WordPress application instances.

The instance is also retained for administrative tasks, infrastructure configuration, and testing purposes.

Unlike the application instances managed by the Auto Scaling Group, this instance is not part of the production application deployment.

Its configuration includes:

- **Subnet:** Public subnet.
- **Network access:** Internet connectivity through the Internet Gateway.
- **Administrative access:** SSH (TCP port 22), restricted to the administrator's public IP address.
- **Purpose:** AMI creation, infrastructure administration, and testing.

### Launch Template

| Parameter | Configuration |
|---|---|
| Name | `aws-wordpress-launch-template` |
| Instance Type | `t3.medium` |
| AMI | `aws-wordpress-ami` |
| Operating System | Amazon Linux 2023 |
| Security Group | `wordpress-ec2-sg` |
| Root Volume | 8 GiB |
| Storage Type | Default |

The custom AMI was created from a preconfigured EC2 instance and includes:

- Docker preinstalled.
- Docker Compose configuration for the WordPress application.
- WordPress container based on the `wordpress:php8.3-apache` image.
- Local `.env` file containing database connection variables.
- `amazon-efs-utils` for Amazon EFS integration.
- Persistent EFS mount configuration.

The AMI provides a consistent application environment for instances launched by the Auto Scaling Group.

### Docker Deployment

WordPress runs inside a Docker container managed through Docker Compose on each EC2 application instance.

The deployment uses the following configuration:

| Parameter | Configuration |
|---|---|
| Compose Project Name | `aws-wordpress` |
| Service Name | `wordpress` |
| Container Image | `wordpress:php8.3-apache` |
| Container Port | 80 |
| EC2 Host Port | 8080 |
| Port Mapping | `8080:80` |
| Restart Policy | `unless-stopped` |
| Environment Configuration | Local `.env` file |

#### Persistent Storage

The WordPress container uses a bind mount to connect the EFS-mounted directory on the EC2 host to the container filesystem.

| Host Path | Container Path | Storage |
|---|---|---|
| `/mnt/wordpress/wp-content` | `/var/www/html/wp-content` | Amazon EFS |

This configuration ensures that uploaded media, plugins, and themes are shared across all application instances, while the WordPress container runs independently on each EC2 instance.

#### Database Connectivity

Docker Compose loads database connection variables from the local `.env` file, which provides the configuration required for WordPress to connect to the centralized Amazon RDS MySQL database.

### Auto Scaling Group

| Parameter | Configuration |
|---|---|
| Name | `aws-wordpress-asg` |
| Launch Template | `aws-wordpress-launch-template` |
| Subnets | `subnet-private-app-us-east-1a`, `subnet-private-app-us-east-1b` |
| Health Check Type | ELB |
| Health Check Grace Period | 300 seconds |
| Minimum Capacity (Target) | 2 |
| Desired Capacity (Target) | 2 |
| Maximum Capacity | 4 |
| Dynamic Scaling Policies | None configured |

**Current capacity:** Minimum and desired capacity are temporarily set to 1 instance, and maximum capacity to 2 instances to reduce costs. The intended deployment configuration is 2 minimum, 2 desired, and 4 maximum instances.

The Auto Scaling Group distributes application instances across both Availability Zones.

## 5. Database — Amazon RDS

### DB Instance

| Parameter | Configuration |
|---|---|
| Identifier | `aws-wordpress-db` |
| Engine | MySQL Community |
| Engine Version | 8.4.11 |
| Instance Class | `db.t3.micro` |
| Port | 3306 |
| Multi-AZ Deployment | Enabled |
| Primary Availability Zone | `us-east-1b` |
| Secondary Availability Zone | `us-east-1a` |
| Security Group | `wordpress-rds-sg` |
| Public Accessibility | Disabled |
| IAM Database Authentication | Disabled |

### Storage

| Parameter | Configuration |
|---|---|
| Storage Type | General Purpose SSD (gp3) |
| Allocated Storage | 20 GiB |
| Provisioned IOPS | 3,000 |
| Storage Autoscaling | Enabled |
| Maximum Storage Threshold | 1,000 GiB |

### Backups

| Parameter | Configuration |
|---|---|
| Automated Backups | Enabled |
| Retention Period | 7 days |
| Backup Target | AWS Cloud |

RDS Multi-AZ provides a standby database instance in a separate Availability Zone and supports automatic failover.

## 6. Shared Storage — Amazon EFS

### File System

| Parameter | Configuration |
|---|---|
| Name | `aws-wordpress-efs` |
| Availability | Regional |
| Performance Mode | General Purpose |
| Throughput Mode | Bursting |
| Encryption at Rest | Enabled |
| Automatic Backups | Disabled |

### Lifecycle Management

| Parameter | Configuration |
|---|---|
| Transition to Infrequent Access (IA) | 30 days since last access |
| Transition to Archive | Not configured |
| Transition to Standard | Not configured |

### Mount Targets

Two Mount Targets provide access to the shared file system from both Availability Zones.

| Availability Zone | Subnet | Security Group |
|---|---|---|
| `us-east-1a` | `subnet-private-app-us-east-1a` | `wordpress-efs-sg` |
| `us-east-1b` | `subnet-private-app-us-east-1b` | `wordpress-efs-sg` |

### EC2 Mount Configuration

The custom AMI includes the EFS mounting utility and persistent mount configuration.

| Parameter | Configuration |
|---|---|
| Mount Directory | `/mnt/wordpress` |
| Mount Utility | `amazon-efs-utils` |
| TLS | Enabled |
| Persistent Mounting | Configured through `/etc/fstab` |

EFS provides shared storage for WordPress files across application instances, including media uploads, plugins, and themes.

## 7. Security Group References

The following Security Groups are associated with the infrastructure components. Their inbound and outbound rules are documented separately in `security.md`.

| Security Group | Associated Resource |
|---|---|
| `wordpress-alb-sg` | Application Load Balancer |
| `wordpress-ec2-sg` | EC2 application instances |
| `wordpress-rds-sg` | Amazon RDS |
| `wordpress-efs-sg` | EFS Mount Targets |
