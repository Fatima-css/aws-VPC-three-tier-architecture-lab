# aws-VPC-three-tier-architecture-lab

## Objective  
To build a secure, highly available, three-tier web application architecture on AWS.  

---

## Part 0: Initial Setup & Foundational Resources  

### S3 Bucket Creation  
**Implementation:** Created an S3 bucket named `vpc-three-tier-project`.  

**Purpose:** This bucket will store the application code that our EC2 instances will download and run.  

### IAM EC2 Instance Role Creation  
**Implementation:** Created an IAM role with two key policies attached:  

- `AmazonS3ReadOnlyAccess`: Grants permission for EC2 instances to download code from the S3 bucket.  
- `AmazonSSMManagedInstanceCore`: Enables secure management of EC2 instances through AWS Systems Manager Session Manager (no SSH keys needed!).  

**Purpose:** IAM roles provide temporary security credentials to instances, eliminating the need to store long-term access keys on instances—a critical security best practice that follows the principle of least privilege.  

<img src="vpc/IAM role.png" alt="IAM role" width="800"/>

---

## Part 1: Networking & Security  

### VPC and Subnets  

**VPC Creation:** Created a VPC `3-tier-vpc` with the CIDR block `10.0.0.0/16`.  

**What is a VPC?** Virtual Private Cloud (VPC) is an isolated virtual network in the cloud that allows you to launch and manage resources.  

**Subnet Creation:**  

The plan is to create six subnets across two Availability Zones (`us-east-1a` and `us-east-1b`) for each tier (Web, App, DB). Carve the large VPC CIDR into smaller, non-overlapping blocks using a `/20` mask (each providing 4,096 IPs).  

> **Mistake:** While creating the third subnet, I accidentally tried to use a `/19` CIDR block (`10.0.32.0/19`). This block was too large and overlapped with the range of my first subnet (`10.0.0.0/20`), causing an error.  

<img src="vpc/CIDR%2019%20error.png" alt="CIDR overlap error" width="800"/>  

> **Fix & Lesson:** Corrected the CIDR to `10.0.32.0/20`. This was a great hands-on lesson in how CIDR blocks work—it’s essential to avoid IP address overlap.  

**Final Subnet Setup:**  

- Public Web Subnets: `10.0.0.0/20` (AZ1), `10.0.48.0/20` (AZ2)  
- Private App Subnets: `10.0.16.0/20` (AZ1), `10.0.64.0/20` (AZ2)  
- Private DB Subnets: `10.0.32.0/20` (AZ1), `10.0.80.0/20` (AZ2)  

<img src="vpc/all-subnets-created.png" alt="All subnets created" width="600"/>  

---

### Internet Connectivity  

**Internet Gateway (IGW):**  
A VPC component that enables communication between your VPC and the internet, allowing resources in public subnets to have direct inbound and outbound access.  

- **Purpose:** It allows resources in the public subnets (like web servers) to have direct inbound and outbound internet access.  
- **Implementation:** Created and attached an Internet Gateway named `IGW-3-tier` to the VPC.  

<img src="vpc/created-IGW.png" alt="Internet Gateway created" width="600"/>  

**NAT Gateway:**  
A managed AWS service that allows instances in private subnets to initiate outbound connections to the internet while preventing unsolicited inbound connections.  

- **Purpose:** In our 3-tier architecture, the NAT Gateway enables resources in the private subnets (App Tier) to securely access the internet for tasks like downloading updates, while still remaining inaccessible from the internet.  
- **Implementation:** Created one NAT Gateway in each public subnet (Web Tier AZ1 and AZ2) so that private subnets in each AZ can route traffic through their local NAT Gateway. This improves availability and keeps traffic within the same AZ.  

<img src="vpc/created-NAT-gateway.png" alt="NAT Gateway created" width="600"/>  

---

### Routing Configuration  

Routing tables tell network traffic where to go. A subnet doesn’t know if it’s public or private until you assign a route table.  

**Public Route Table:**  

- Created a route table named `Public-Route-Table`.  
- Added a route sending all non-VPC traffic (`0.0.0.0/0`) to the Internet Gateway (IGW).  
- Explicitly associated the two Public Web Subnets with this route table.  

<img src="vpc/edit-routes-IGW.png" alt="Route to IGW" width="600"/>  
<img src="vpc/edit-subnet-association-web-subnets.png" alt="Subnet association for web tier" width="600"/>  

> **Why explicit association?** A route table is just a set of directions. You must manually link it to the subnets that should use those directions. Without this, the web subnets would never use the IGW.  

**Private Route Tables:**  

- Created two private route tables: `Private-route-AZ1` and `Private-route-AZ2`.  
- For each, added a route sending all non-VPC traffic (`0.0.0.0/0`) to the NAT Gateway in its respective AZ.  

<img src="vpc/edit-routes-NAT-gateway.png" alt="Route to NAT Gateway" width="600"/>  
<img src="vpc/edit-subnet-association-private-app-subnet.png" alt="Subnet association for private app tier" width="600"/>  

- Associated `Private-route-AZ1` with the private App subnets in AZ1.  
- Associated `Private-route-AZ2` with the private App subnets in AZ2.  

**Purpose:** Ensures that if an app server in AZ1 needs internet, its traffic goes through the NAT Gateway in AZ1, keeping flow within the same AZ for performance and cost efficiency.  

---

### Security Groups  

Security Groups act as firewalls at the instance level. They decide who can talk to what.  

> **Note:** Always pick the `3-tier-vpc`, not the Default VPC.  

**External Load Balancer SG**  
- **Purpose:** The “front door” for the app—takes traffic from internet users.  
- **Rule:** Allow HTTP (80) from my IP (for testing).  

<img src="vpc/Load-balancer-SG.png" alt="Load Balancer Security Group" width="600"/>  

**Web Tier SG (Public Instances)**  
- **Purpose:** The actual web servers. They don’t talk to the internet directly—only through the load balancer.  
- **Rules:**  
  - Allow HTTP (80) from External LB SG.  
  - Allow HTTP (80) from my IP (for testing).  

<img src="vpc/web-tier-SG.png" alt="Web Tier Security Group" width="600"/>  

**Internal Load Balancer SG**  
- **Purpose:** Balances traffic between the web and app tiers. Only accepts traffic from Web Tier SG.  

<img src="vpc/internal-LB-SG.png" alt="Internal Load Balancer SG" width="600"/>  

**Private Instance SG (App Tier)**  
- **Purpose:** App servers running on port 4000.  
- **Rules:**  
  - Port 4000 from Internal LB SG.  
  - Port 4000 from my IP (for testing).  

<img src="vpc/private-SG.png" alt="Private Instance SG" width="600"/>  

> **Mistake encountered:** Couldn’t find my Internal LB SG in the dropdown. I was in the wrong VPC. Switched to `3-tier-vpc`, problem solved.  

**DB SG (Database Tier)**  
- **Purpose:** DB servers. Only allow MySQL/Aurora (3306) from Private Instance SG (App Tier).  

<img src="vpc/DB-SG.png" alt="DB Security Group" width="600"/>  

**End-to-End Flow:**  

Internet → External LB SG → Web Tier SG → Internal LB SG → App Tier SG (4000) → DB SG (3306).  

<img src="vpc/All-SG.png" alt="All Security Groups overview" width="700"/>  

---

## Part 2: Deploying Database  

### Database Subnet Group Creation  

**Purpose:** A DB subnet group ensures that RDS (or Aurora) instances are deployed into specific private subnets across multiple AZs, providing high availability and isolation from the public internet.  

**Implementation:**  
- Created DB Subnet Group: `DB-subnet-group`.  
- Added DB subnets:  
  - `10.0.32.0/20` (Private DB Subnet AZ1).  
  - `10.0.80.0/20` (Private DB Subnet AZ2).  

> Verified subnets are private and spread across two AZs for redundancy.  

**Result:** DB subnet group created successfully and used when deploying the database.  

---

### Database Deployment  

**Purpose:** Provide a highly available, fault-tolerant database layer for the 3-tier architecture.  

**Implementation:**  

1. **Template**  
   - Standard create: Aurora MySQL-Compatible Edition.  
   - Template: Dev/Test (non-production).  
   - Custom master username + password (stored securely).  
 <img src="vpc/config aurora pt1.png" alt="Aurora configuration step 1" width="600"/> 

3. **Availability & Connectivity**  
   - Multi-AZ deployment → created reader replica in another AZ.  
   - Selected `3-tier-vpc` and `DB-subnet-group`.  
   - Public access = No (DB stays private).   
<img src="vpc/config aurora pt2.png" alt="Aurora configuration step 2" width="600"/>  

4. **Security & Authentication**  
   - Attached custom DB SG (`DB-SG`).  
   - Chose password authentication.  
   - Created the database.  
   > By default, RDS attaches an auto-created SG. Replaced with custom DB-SG to restrict access only to App Tier.  
 <img src="vpc/config aurora pt3.png" alt="Aurora configuration step 3" width="600"/>
   
**Result:**  
- Aurora DB deployed across two AZs.  
- One Writer (primary) for read/write.  
- One Reader (replica) for read-only scaling.  
- High availability and fault tolerance achieved.  

<img src="vpc/created DB.png" alt="Aurora DB created" width="600"/>  

**Database Endpoints:**  
- **Writer Endpoint:** Used for INSERT, UPDATE, DELETE (application writes).  
- **Reader Endpoint:** Used for SELECT (read-only queries). Since our app modifies data (e.g., transactions table), Writer Endpoint was essential.  

---

## Part 3: App Tier Instance Deployment   

### App Instance Launch  

**Purpose:** The App Tier runs the backend logic (Node.js) and communicates with the Aurora database. It is deployed in private subnets, accessible only through the internal load balancer and NAT gateway for outbound internet.  

**Implementation:**  
- Launched an EC2 instance in the **Private App Subnet**.  
- Configuration:  
  - **AMI:** Amazon Linux 2 (HVM), Kernel 5.10  
  - **Instance Type:** `t2.micro` (Free Tier)  
  - **Network:** `3-tier-vpc`  
  - **Subnet:** Private App Subnet  
  - **Auto-assign Public IP:** Disabled  
  - **Security Group:** `PrivateInstanceSG` (port 4000 allowed)  
  - **IAM Role:** `EC2andS3Role` (with S3 + SSM permissions)  
- Did not use a key pair—connected via **AWS Systems Manager Session Manager**.  

<img src="vpc/launch instance1.png" alt="App Instance Launch step 1" width="600"/>  
<img src="vpc/launch instance2.png" alt="App Instance Launch step 2" width="600"/>  
<img src="vpc/launch instance3.png" alt="App Instance Launch step 3" width="600"/>  

---
### Connecting to Instance  

**Implementation:**  
- Navigate to **Instances**, select the running instance, and click **Connect, Session Manager, Connect**.  
- Used **Session Manager** to connect securely (no SSH keys).  
- Switched to **ec2-user** for full permissions.  

```
sudo -su ec2-user
```
- Verified outbound internet connectivity (through NAT Gateway):
```
ping 8.8.8.8
```
<img src="vpc/connect 1.png" alt="sudo and ping" width="600"/>


## Database Configuration from App Instance

**Objective:** Connect the App Tier EC2 instance to Aurora RDS, create a database and table, and insert initial test data.

### 1️. Install MySQL/MariaDB Client

**Implementation:**  
- The workshop originally used an older command to install MySQL:
```sudo yum install mysql -y```  
- Researched and installed the correct package for Amazon Linux 2023:
```
sudo yum install mariadb105 -y
```

> **Why:** Amazon Linux updates frequently. The older `mysql` package is deprecated; `mariadb105` provides the MySQL-compatible client for connecting to Aurora.

<img src="vpc/sudo install error+fix.png" alt="mariadb install error and fix" width="600"/>

---

### 2️. Connect to Aurora RDS (Writer Endpoint)

**Implementation:**  
- Connected to the Aurora RDS **writer endpoint** using the master username(admin) created during deployment:
mysql -h <RDS_WRITER_ENDPOINT> -u <DB_USERNAME> -p
Replace <RDS_WRITER_ENDPOINT> with your Aurora writer endpoint.  
- Entered the database password when prompted.  

> **Why:** Connecting to the **writer endpoint** allows the App Tier to perform read/write operations (INSERT, UPDATE, DELETE).

<img src="vpc/mysql -h.png" alt="MySQL DB connection" width="600"/>

---

### 3️⃣ Create Database

**Implementation:**  
-Create a database called webappdb with the following command using the MySQL CLI:

CREATE DATABASE webappdb;

 > **Note:** Encountered a **case sensitivity issue** when creating the database. Initially, the database name was typed inconsistently (`webappdB`). Fixed by using lowercase consistently (`webappdb`).
<img src="vpc/error case sensative.png" alt="Case sensitivity error" width="600"/>

Verify it was created:
SHOW DATABASES;

-Create a data table by first navigating to the database we just created:  
USE webappdb;


-Then, create the following transactions table by executing this create table command:

CREATE TABLE IF NOT EXISTS transactions (
  id INT NOT NULL AUTO_INCREMENT,
  amount DECIMAL(10,2),
  description VARCHAR(100),
  PRIMARY KEY(id)
);

-Verify the table was created:

SHOW TABLES;  

<img src="vpc/create webappdb.png" alt="Database created" width="600"/>

-Insert data into table for use/testing later:
INSERT INTO transactions (amount, description) VALUES (400, 'groceries');


Verify that your data was added by executing the following command:

SELECT * FROM transactions;




