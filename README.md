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

**1. Install MariaDB Client**
- This command installs the tool that lets your App Instance talk to your database.
> Note: The workshop originally used an older command to install MySQL:```sudo yum install mysql -y```. Amazon Linux updates frequently. The older `mysql` package is deprecated; `mariadb105` provides the MySQL-compatible client for connecting to Aurora.
- Researched and installed the correct package for Amazon Linux 2023:
```
sudo yum install mariadb105 -y
```

<img src="vpc/connect 2 sudo install error+fix.png" alt="mariadb install" width="800"/>

---

**2. Connecting to Aurora RDS (Writer Endpoint)**
 
- This command uses the tool you just installed to open a connection to your database (Aurora RDS).
```
mysql -h <RDS_WRITER_ENDPOINT> -u <DB_USERNAME> -p
```
Replace <RDS_WRITER_ENDPOINT> with your Aurora writer endpoint and <DB_USERNAME> with the the master username(admin).
- Enter the database password when prompted.  

> **Why:** Connecting to the **writer endpoint** allows the App Tier to perform read/write operations (INSERT, UPDATE, DELETE).

---

**3. Create Database**

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
**The goal of this section was to get the web application running on the App Instance.** 

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
**4. Download App Code from S3 to the App instance**
```
cd ~/
aws s3 cp s3://BUCKET_NAME/app-tier/ app-tier --recursive
```
This copies the application code from our cloud storage (S3) and places it on the App Instance, allowing the server to access the files and run the application.

<img src="vpc/config app instance1.png" alt="app instance1" width="600"/>

---

** 5. Install Dependencies and Start the App**
```
cd ~/app-tier
npm install
pm2 start index.js
```

- ```npm install``` installs required libraries for the app.

- ```pm2 start index.js``` launches the app with PM2.

<img src="vpc/config app instance2.png" alt="app instance2" width="600"/>

---

**6. Verify the App is Running**
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

--

## Internal Load Balancing and Auto Scaling  

In this section, we create an **Amazon Machine Image (AMI)** of the App Tier instance, then set up **Auto Scaling** with an **Internal Load Balancer (ALB)** to make the App Tier highly available.  

---

### Learning Objectives  
- Create an AMI of the App Tier  
- Create a Launch Template  
- Configure Auto Scaling  
- Deploy an Internal Load Balancer  

---

## 1. App Tier AMI
We first create an **Amazon Machine Image (AMI)** of our existing app tier instance. 
The AMI is a snapshot or a "cookie cutter" of a server. It contains the operating system and all of your application's code and software already installed. 

This allows us to spin up **identical instances** automatically in the future.
- Navigate to **Instances** in the EC2 dashboard.  
- Select the App Tier instance -> **Actions -> Image and templates -> Create Image**.  
- Give the image a **name and description**.  
- Monitor status under EC2 -> AMIs in the left-hand menu.  
<img src="vpc/App tier AMI.png" alt="App tier AMI" width="600"/>

---

### 2. Target Group  
The **Target Group** defines the set of instances that the ALB will route requests to.  

While the AMI is being created:  

- Navigate to **Target Groups** under **Load Balancing**.  
- Click **Create Target Group**.  
- Select **Instances** as the target type and give it a name.  
- Set protocol: **HTTP**, port: **4000** (this is the port the Node.js app runs on).  
- Choose the same VPC.  
- Set health check path to **/health** (app’s health check endpoint). Ensure only healthy instances get requests. 
- Skip registering targets for now, and finish creating the target group.  
<img src="vpc/App Tier target group.png" alt="App Tier target group" width="600"/>

---

### 3. Internal Load Balancer  
We’ll now deploy an **internal ALB** to forward requests from the web tier to the app tier.
- Go to **Load Balancers -> Create Load Balancer**.  
- Choose **Application Load Balancer** (ALB).  
- Select **Internal** (not internet-facing).  
- Pick the correct VPC and **private subnets**.  
- Attach the security group for the internal ALB.  
- Listener: **HTTP (port 80)** -> forward traffic to the target group created earlier.  
- Create the load balancer.  

The internal ALB ensures traffic distribution across private app instances, improving fault tolerance and availability.
<img src="vpc/App Tier internal LB.png" alt="App Tier internal LB" width="600"/>
<img src="vpc/App Tier internal LB2nd.png" alt="App Tier internal LB 2" width="600"/>
<img src="vpc/App Tier internal LB3.png" alt="App Tier internal LB 3" width="600"/>


---

### 4. Launch Template  
The **Launch Template** defines how new app tier instances should be launched. 
- The Launch Template is a blueprint or a recipe. It doesn't contain any code itself. Instead, it provides the instructions for AWS on how to build a new server. It tells AWS which AMI to use and specifies other settings (AMI, security, IAM role) to ensure all auto-scaled instances are consistent. 

- Navigate to **Launch Templates -> Create Launch Template**.  
- Name the template.  
- Select the **App Tier AMI** created earlier.  
- Choose **t2.micro** instance type.  
- Skip key pair and network settings (not needed).  
- Assign the correct **App Tier security group**.  
- Under **Advanced details**, select the same IAM instance profile used before.  
<img src="vpc/App Tier launch template1.png" alt="launch template 1" width="600"/>
<img src="vpc/App Tier launch template2.png" alt="launch template 2" width="600"/>
<img src="vpc/App Tier launch template3.png" alt="launch template 3" width="600"/>

---

### 5. Auto Scaling  
Now we create an **Auto Scaling Group (ASG)** to automatically add/remove instances.

- Navigate to **Auto Scaling Groups → Create Auto Scaling Group**.  
- Name the group, select the **Launch Template**, and continue.  
- Set the VPC and private subnets for the App Tier.  
- Attach to the **internal load balancer’s target group**.  
- Set group size:  
  - **Desired capacity**: 2  
  - **Minimum capacity**: 2  
  - **Maximum capacity**: 2  
- Review and create the Auto Scaling Group.  
<img src="vpc/App Tier ASG1.png" alt="App Tier ASG 1" width="600"/>
<img src="vpc/App Tier ASG2.png" alt="App Tier ASG 2" width="600"/>
<img src="vpc/App Tier ASG3.png" alt="App Tier ASG 3" width="600"/>
<img src="vpc/App Tier ASG4.png" alt="App Tier ASG 4" width="600"/>

The ASG ensures there are always at least 2 running app instances. If one fails, it’s automatically replaced.

---

### ✅ Result  
- Check EC2 dashboard → You should see 2 new instances launched by the ASG.  
- Test high availability:  
  - Manually terminate 1 instance.  
  - The ASG should launch a new one automatically.
    
>> Note: The **original app instance** (used to create the AMI) is not part of the ASG. You’ll see 3 total instances. You can delete the original later, but keeping it for troubleshooting is recommended.


--

## Part 5: Web Tier Instance Deployment  

## !Imporant Error!: Connecting to the Wrong Instance  

During my deployment, I initially ran into an issue where I attempted to configure NGINX and the React.js web app **inside the App Tier instance** instead of the **Web Tier instance**.  

This mistake cost ~1–2 hours of troubleshooting because the workshop instructions did not clearly emphasize switching into the newly created **Web Tier EC2 instance** before executing commands.  

### Why this matters  
- Each **tier** (Web, App, Database) in a multi-tier architecture serves a specific role and must be configured independently.  
- Running Web Tier configuration commands (NGINX setup, React build) on the App Tier will fail, since that instance is not designed to serve frontend traffic.  
- Correctly separating responsibilities between tiers is a core AWS best practice for **security**, **scalability**, and **operational clarity**.  

---

The Web Tier is the internet-facing layer of the application. It is responsible for handling requests from users, serving the React website, and forwarding API calls to the Internal Load Balancer, which connects to the App Tier.  

## Learning Objectives  
- Update and upload NGINX configuration files  
- Create the Web Tier instance in a public subnet  
- Connect and verify instance connectivity  
- Configure the software stack (Node.js, React, and NGINX)  

---

## 1. Update Config File  

Before creating the Web Tier instance, we configure NGINX so that it knows how to forward API requests to the App Tier. This configures the Web Tier to act as a front door for your application, forwarding all API requests to the App Tier.

Steps:  
- Open the `nginx.conf` file from the project repository.  
- Scroll to line 58 and replace `[INTERNAL-LOADBALANCER-DNS]` with the DNS name of your Internal Load Balancer.  
- Upload the updated `nginx.conf` and the `web-tier/` folder to the S3 bucket created for this project.  
<img src="vpc/line 58.png" alt="line 58" width="600"/>

This step ensures that all API requests from the React app are directed through the internal load balancer instead of going directly to backend instances.  

---

## 2. Web Instance Deployment  

The Web Tier instance is created in the same way as the App Tier instance, with a few differences:  

- It must be provisioned in a **public subnet** (so it can be reached from the internet).  
- Auto-assign a **public IP address**.  
- Attach the security group that allows inbound HTTP traffic (port 80).  
- Attach the IAM role that allows access to S3 (instance-role).  
- Finally click Launch instance to create the instance. 
<img src="vpc/web instance" alt="web instace" width="600"/>

This design ensures the Web Tier is accessible to end users.  

---

## 3. Connect to Instance  

Once the Web Tier instance is running, connect to it and switch to the `ec2-user`.  

Run a simple ping command to confirm internet connectivity:
```
sudo -su ec2-user 
ping 8.8.8.8
```
If connectivity fails, check the route table associated with the public subnet.  

>> Note: During deployment, don't connect to the App Tier instance and attempt these steps there. This mistake can cause hours of confusion. Always confirm you are connected to the Web Tier instance before continuing.  

---

## 4. Configure Web Instance  

The Web Tier needs software installed and configured to serve the React frontend application.  

1. **Install Node.js with NVM**  
   - Node.js is the JavaScript runtime used to build the React application.  
   - NVM (Node Version Manager) makes it easier to install and manage Node.js versions.  
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
```
2. **Download Web Tier Code from S3**  
   - The necessary source code for the frontend application files are stored in S3 bucket, this command copies them to the Web Tier instance:
```
cd ~/
aws s3 cp s3://BUCKET_NAME/web-tier/ web-tier --recursive
```
<img src="vpc/web connect1" alt="web connect1" width="800"/>

3. **Build React Application**  
   - React code must be compiled into optimized static files before serving.  
   - Use `npm install` to install dependencies, then `npm run build` to create the production-ready files.  
```
cd ~/web-tier   
npm install      #The npm install command first downloads and sets up all the required dependencies. 
npm run build    #npm run build compiles the raw code into static files (HTML, CSS, and JavaScript).
```
<img src="vpc/web connect2" alt="web connect2" width="900"/>

4. **Install and Configure NGINX**  
   - NGINX is a lightweight and efficient web server.
   - We will be using it as a web server that we will configure to serve our application on port 80. It serves the React build files to users and also proxies API requests to the Internal Load Balancer.
     
1. To get started, install NGINX by executing the following command:
```
sudo amazon-linux-extras install nginx1 -y  # This command installs the NGINX software on the instance.
```

**2. Navigate to the Nginx configuration file with the following commands and list the files in the directory:**
```
cd /etc/nginx        # Navigate to the NGINX configuration directory
ls
```

**3. Next, you will configure NGINX. We need to delete the default configuration file and replace it with our own, which is stored in S3.:**
```
sudo rm nginx.conf                                 #Delete/remove the default configuration file.
sudo aws s3 cp s3://BUCKET_NAME/nginx.conf .       #Be sure to replace BUCKET_NAME with your actual bucket name.
```

**4. Then, restart Nginx with the following command:**
```
sudo service nginx restart     #you must restart the NGINX service to apply the changes.
```
 


5. **Set Permissions and Auto-Start**  

For NGINX to properly serve the React application files, you must grant it the correct permissions. This step allows NGINX to read and serve the files from the home/ec2-user directory
```
chmod -R 755 /home/ec2-user
```

Finally, you need to set up NGINX to start automatically every time the instance reboots.
```
sudo chkconfig nginx on
```

---

## Result  
Now, when you enter the public IP of your web tier instance in a browser, you should be able to see your website. If the database is connected and working correctly, you will also be able to add data.

When you navigate to the public IP address of the Web Tier instance in a browser:  
- You should see the React application being served.  
- If the database is properly connected, you will also see that the frontend can add and display data through the backend API.  

At this point, the three-tier architecture is complete:  

- **Web Tier**: Public-facing, serves the frontend, and routes API calls.  
- **App Tier**: Private layer running the Node.js API.  
- **Database Tier**: Stores persistent data.  
