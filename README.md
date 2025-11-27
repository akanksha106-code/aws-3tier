
# AWS 3-Tier Architecture

## Project Executive Summary

**Problem**
Monolithic architectures often host database, application, and web layers on single instances. This creates Single Points of Failure (SPoF), limits scalability, and poses security risks by exposing the database layer to the public internet.

**Action**
I designed and provisioned a decoupled 3-tier architecture on AWS. The environment is segregated into three distinct layers:

1.  **Presentation Layer (Web Tier):** Nginx web servers handling HTTP requests.
2.  **Logic Layer (App Tier):** Node.js application servers processing API requests.
3.  **Data Layer (Database Tier):** Amazon RDS (MySQL) storing application data.

**Outcome**
The result is a fault-tolerant system where the database is strictly isolated in a private subnet, accessible only by the application server. The architecture supports independent scaling of tiers and utilizes Multi-AZ deployment for high availability.

-----

## Architecture Design

  * **VPC:** Custom Virtual Private Cloud spanning two Availability Zones.
  * **Web Tier:** Public subnets hosting Nginx web servers behind an Internet-facing Application Load Balancer.
  * **App Tier:** Private subnets hosting Node.js servers behind an Internal Load Balancer.
  * **Database Tier:** Private subnets hosting Amazon RDS (MySQL) with Standby Replica.
  * **Security:** Restricted Security Groups ensuring traffic flow follows the path: Internet → Web ALB → Web Tier → Internal ALB → App Tier → RDS.

-----

## Technical Implementation

### Phase 1: Network & Security

  * **VPC Configuration:** Created public subnets for the NAT Gateway and Web Tier, and private subnets for the App Tier and Database.
  * **IAM Roles:** Configured `AmazonS3ReadOnlyAccess` roles for EC2 instances to securely retrieve code artifacts from S3 without hardcoding credentials.

### Phase 2: Database Layer

Deployed Amazon RDS MySQL in a private subnet. The Security Group was configured to accept traffic on port 3306 solely from the App Tier Security Group.

**Schema Initialization:**
Connected via bastion host to initialize the database:

```sql
CREATE DATABASE webappdb;
USE webappdb;

CREATE TABLE IF NOT EXISTS transactions(
  id INT NOT NULL AUTO_INCREMENT,
  amount DECIMAL(10,2),
  description VARCHAR(100),
  PRIMARY KEY(id)
);
```

### Phase 3: Application Tier (Backend)

Configured Node.js application servers to interface between the Web Tier and Database.

**Configuration:**
Updated `DbConfig.js` to point to the RDS endpoint:

```javascript
module.exports = Object.freeze({
    DB_HOST: 'RDS-ENDPOINT',
    DB_USER: 'admin',
    DB_PWD: 'DB_PASSWORD',
    DB_DATABASE: 'webappdb'
});
```

**Deployment:**
Utilized PM2 for process management to ensure the application remains active and restarts on failure.

```bash
# PM2 Setup
npm install -g pm2
pm2 start index.js
pm2 startup
pm2 save
```

**Internal Routing:**
Deployed an Internal Application Load Balancer (ALB) to distribute traffic across App Tier instances.

### Phase 4: Web Tier (Frontend)

Deployed Nginx as a reverse proxy to forward API requests to the internal application layer, keeping the backend isolated.

**Nginx Configuration:**
Configured `nginx.conf` to proxy traffic to the Internal ALB:

```nginx
location /api/ {
    proxy_pass http://INTERNAL-ALB-DNS:80/;
}
```

### Phase 5: Production Readiness

  * **External Load Balancer:** Configured an Internet-facing ALB to route traffic to the Web Tier.
  * **SSL/TLS:** Attached an ACM certificate to the ALB listener for HTTPS traffic.
  * **DNS:** Mapped the custom domain to the External ALB using a Route 53 Alias record.

-----

## Resource Teardown Checklist

To maintain cost efficiency, resources are terminated in the following order:

1.  Auto Scaling Groups
2.  Load Balancers and Target Groups
3.  EC2 AMIs and Snapshots
4.  RDS Instance
5.  S3 Bucket
6.  Route 53 Records and ACM Certificates
7.  NAT Gateways and Elastic IPs
8.  VPC
