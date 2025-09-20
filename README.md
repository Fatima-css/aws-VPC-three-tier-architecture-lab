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

<img src="vpc/IAM%role.png" alt="IAM role" width="800"/>

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

1. **Engine & Creation**  
   - Standard create → Aurora MySQL-Compatible Edition.  
   - Defaults for engine options.  
   <img src="vpc/config%aurora%pt1.png" alt="Aurora configuration step 1" width="600"/>  

2. **Template & Credentials**  
   - Template: Dev/Test (non-production).  
   - Custom master username + password (stored securely).  
   <img src="db/config-aurora-pt2.png" alt="Aurora configuration step 2" width="600"/>  

3. **Availability & Connectivity**  
   - Multi-AZ deployment → created reader replica in another AZ.  
   - Selected `3-tier-vpc` and `DB-subnet-group`.  
   - Public access = No (DB stays private).  
   <img src="db/config-aurora-pt3.png" alt="Aurora configuration step 3" width="600"/>  

4. **Security & Authentication**  
   - Attached custom DB SG (`DB-SG`).  
   - Chose password authentication.  
   - Created the database.  
   > By default, RDS attaches an auto-created SG. Replaced with custom DB-SG to restrict access only to App Tier.  

**Result:**  
- Aurora DB deployed across two AZs.  
- One Writer (primary) for read/write.  
- One Reader (replica) for read-only scaling.  
- High availability and fault tolerance achieved.  

<img src="db/created-DB.png" alt="Aurora DB created" width="600"/>  

**Database Endpoints:**  
- **Writer Endpoint:** Used for INSERT, UPDATE, DELETE (application writes).  
- **Reader Endpoint:** Used for SELECT (read-only queries). Since our app modifies data (e.g., transactions table), Writer Endpoint was essential.  
