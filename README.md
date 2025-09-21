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
- **Purpose:** The “front door” for the app, that takes traffic from internet users.  
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

<img src="vpc/launch app instance1.png" alt="App Instance Launch step 1" width="600"/>  
<img src="vpc/launch app instance2.png" alt="App Instance Launch step 2" width="600"/>  
<img src="vpc/launch app instance3.png" alt="App Instance Launch step 3" width="600"/>  

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
<img src="vpc/connect 1.png" alt="ping" width="800"/>


## Database Configuration from App Instance

**Objective:** Connect the App Tier EC2 instance to Aurora RDS, create a database and table, and insert initial test data. You are essentially using the App Instance as a tool to connect to your database and build its structure.

### 1️. Install MariaDB Client
- This command installs the tool that lets your App Instance talk to your database.
> Note: The workshop originally used an older command to install MySQL:```sudo yum install mysql -y```. Amazon Linux updates frequently. The older `mysql` package is deprecated; `mariadb105` provides the MySQL-compatible client for connecting to Aurora.
- Researched and installed the correct package for Amazon Linux 2023:
```
sudo yum install mariadb105 -y
```

<img src="vpc/connect 2 sudo install error+fix.png" alt="mariadb install" width="800"/>

---

### 2️. Connecting to Aurora RDS (Writer Endpoint)
 
- This command uses the tool you just installed to open a connection to your database (Aurora RDS).
```
mysql -h <RDS_WRITER_ENDPOINT> -u <DB_USERNAME> -p
```
Replace <RDS_WRITER_ENDPOINT> with your Aurora writer endpoint and <DB_USERNAME> with the the master username(admin).
- Enter the database password when prompted.  

> **Why:** Connecting to the **writer endpoint** allows the App Tier to perform read/write operations (INSERT, UPDATE, DELETE).

---

### 3️. Create Database

- Create a database called webappdb with the following command using the MySQL CLI:

```
CREATE DATABASE webappdb;
```
 > **Note:** Encountered a **case sensitivity issue** when creating the database. Initially, the database name was typed inconsistently (`webappdB`). Fixed by using lowercase consistently (`webappdb`).

- Verify it was created:
```
SHOW DATABASES;
```

- Create a data table by first navigating to the database we just created:  
```
USE webappdb;
```

- Then, create the following transactions table by executing this create table command:
```
CREATE TABLE IF NOT EXISTS transactions (
  id INT NOT NULL AUTO_INCREMENT,
  amount DECIMAL(10,2),
  description VARCHAR(100),
  PRIMARY KEY(id)
);
```

- Verify the table was created:
```
SHOW TABLES;  
```

- Insert data into table for use/testing later:
```
INSERT INTO transactions (amount, description) VALUES (400, 'groceries');
```

- Verify that your data was added by executing the following command:
```
SELECT * FROM transactions;
```
- When finished, just type exit and hit enter to exit the MySQL client.

---
## Configure App Instance
### The goal of this section was to get the web application running on the App Instance. 

**1. Updating Application Credentials**
I opened the **DbConfig.js** file from the repository and updated it with my database credentials:

- **Hostname**: The Aurora writer endpoint  
- **User**: The master username (e.g., `admin`)  
- **Password**: The password created during database deployment  
- **Database**: `webappdb`  

**Why:**  
The application needs the correct credentials and endpoint so it can connect to the Aurora database.  

> **Note:** As mentioned in the workshop, storing credentials directly in a file is not a best practice in production. In real-world applications, services like **AWS Secrets Manager** are used for secure storage.  

---

**2. Upload the app-tier folder to the S3 bucket that you created in part 0.**

<img src="vpc/add app folder in S3 bucket.png" alt="upload folder to s3" width="600"/>

---

**3. Go back to the App Instance's SSM session to install the necessary software components to run the backend application**

- a. Install NVM (Node Version Manager):
NVM lets us install and switch between different Node.js versions.
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
source ~/.bashrc
```
- b. Install Node.js (version 16):
 Our app requires Node.js v16 to run.
```
nvm install 16
nvm use 16
```
- c. Install PM2:
PM2 is a process manager that keeps the Node.js app running in the background.
Even if we close the SSM session or the instance restarts, PM2 ensures the app stays alive.
```
npm install -g pm2
```
---
### 4. Download App Code from S3 to the App instance
```
cd ~/
aws s3 cp s3://BUCKET_NAME/app-tier/ app-tier --recursive
```
This copies the application code from our cloud storage (S3) and places it on the App Instance, allowing the server to access the files and run the application.

<img src="vpc/config app instance1.png" alt="app instance1" width="600"/>

---

### 5. Install Dependencies and Start the App
```
cd ~/app-tier
npm install
pm2 start index.js
```

- ```npm install``` installs required libraries for the app.

- ```pm2 start index.js``` launches the app with PM2.

<img src="vpc/config app instance2.png" alt="app instance2" width="600"/>

---

### 6. Verify the App is Running
```
pm2 list
```
- If online: the app is running.

- If error: run `pm2 logs`
---

### 7. Set Up PM2 for Auto-Restart

Right now, PM2 only keeps the app running in this session.
We need to make sure it auto-starts on reboot.
```
pm2 startup
```
This will output a custom command.
Copy-paste the command shown in your terminal

<img src="vpc/config app instance3.png" alt="app instance1" width="800"/>

Finally, save the PM2 process list so it reloads on reboot:
```
pm2 save
```
This ensures the app automatically restarts if the instance is rebooted or turned into an AMI.

<img src="vpc/config app instance4.png" alt="app instance4" width="800"/>

---
## Test App Tier
Will now run a couple of tests to confirm that the app is configured correctly and can connect to the database.


### 1. Health Check Endpoint
```
curl http://localhost:4000/health
```
A health check endpoint is a standard practice in web development to confirm a service is up and responding.

**Expected result:**
```
"This is the health check"
```

### 2. Database Connection Test
**Purpose:** This test verifies if the application tier is able to connect to the database, query data, and return a result. A successful response confirms that the networking, security, and database configurations are correct.

To test the database connection, I ran the following command in the terminal to hit the /transaction endpoint locally:
```
curl http://localhost:4000/transaction
```
**Expected result:**
The command should return the test data that was previously inserted into the database. The response confirms the application's ability to retrieve data.
```
{"result":[{"id":1,"amount":400,"description":"groceries"},{"id":2,"amount":100,"description":"class"},{"id":3,"amount":200,"description":"other groceries"},{"id":4,"amount":10,"description":"brownies"}]}
```

<img src="vpc/test app tier.png" alt="app tier test" width="800"/>
