### AWS CLOUD SOLUTION FOR 2 COMPANY WEBSITES USING A REVERSE PROXY TECHNOLOGY

**Objective:** To build an infrastructure that can serve 2 or more different websites that is resilient to Web Server’s failures, can accomodate increased traffic, at the lowest possible infrastructure and cloud cost while still satisfying high availability and security.

#### Step 1: Set up a Virtual Private Network (VPC) needed for the implementation

Virtual Private Cloud (VPC) was created
![image](https://user-images.githubusercontent.com/87030990/169170871-81f6737a-b812-4d0c-b81d-15b7c544410b.png)

6 Subnets were created (2 Public and 4 Private Subnets) in 2 Availability Zones

![image](https://user-images.githubusercontent.com/87030990/169170969-e3728757-7398-4bae-b959-da8914704b3e.png)

Internet Gateway was created for accessibility by the public from the internet and attached to the VPC

![image](https://user-images.githubusercontent.com/87030990/169171022-a93c5563-edc5-4500-9e51-cb2c8bd9c75b.png)

Route table for public subnets was created and associated with the 2 public subnets. Route for Internet Gateway(igw) was added to the routing table for public subnets

![image](https://user-images.githubusercontent.com/87030990/169610632-a161418e-cc7b-48e4-a1c1-7474f33f3665.png)

Route table for private subnet was created and associated with the 4 private subnets. 
![image](https://user-images.githubusercontent.com/87030990/169610709-d9cae8ca-e54a-461f-84ec-802951e2cfa9.png)

NAT Gateway was created in the public subnet and an Elastic IP allocated to it. Route for the NAT gateway was added to routing table for private subnets

![image](https://user-images.githubusercontent.com/87030990/169406792-80a6a197-de60-44ef-8f7d-4943bb578062.png)

Security Group for External Public Facing Application Load Balancer, Nginx Servers, Bastion Servers, Internal Non Public Facing Application Load Balancer, Webservers and Data Layer were created

![image](https://user-images.githubusercontent.com/87030990/169407228-a6569dad-cc86-494c-9648-979b7217a684.png)


#### Step 2: Register a Domain and configure secured connection using SSL/TLS certificates

Domain **(toolingobaf.ga)** was used for the implimentation

TLS Certificates was configured for ***.toolingobaf.ga** to handle secured connectivity to the Application Load Balancers (ALB)

![image](https://user-images.githubusercontent.com/87030990/168501077-1ba5a3a7-053c-43ba-944d-4aa624e0db80.png)


#### Step 3: Create the Data layer, which is comprised of Amazon Relational Database Service (RDS) and Amazon Elastic File System (EFS)

EFS was created with access points for both **tooling** and **wordpress**

![image](https://user-images.githubusercontent.com/87030990/169599495-b30fe8e8-33cb-411d-9b76-4ad47ed85ac6.png)

![image](https://user-images.githubusercontent.com/87030990/169597010-4b486e03-a232-4c2b-8199-fd6e27ee925d.png)


**Prerequisite for creating Relational Database System (RDS)**

Create a **KMS key from Key Management Service (KMS)** to be used to encrypt the database instance and RDS Subnet Group were created

![image](https://user-images.githubusercontent.com/87030990/168502204-65a1ead3-6558-4ba0-87c0-45dec2f97819.png)

![image](https://user-images.githubusercontent.com/87030990/168502629-e08a220d-4f6c-4580-bae5-8ec4133570da.png)

**RDS Databas**e was then created

![image](https://user-images.githubusercontent.com/87030990/169172480-4c41e9cc-8f35-48ce-a082-25034e30ff7c.png)


#### Step 4: Set up Compute Resources inside the VPC

For a start a EC2 instance based on RedHat was chosen from Amazon Market Place to help in building my own AMI and relevant software **(python, ntp, net-tools, vim, wget, telnet, epel-release, htop)** and licence installed (SSL). YThis was done for

* Bastion server (Userdata configured to update yum package repository and install Ansible and git)
* Nginx reverse-proxy (Userdata configured to update yum package repository and install nginx)
* Webservers (Userdata to update yum package repository. Install wordpress only required on the Wordpress launch template)


![image](https://user-images.githubusercontent.com/87030990/169170229-eedb60ef-a32d-430b-8fa1-8dea5158467c.png)


Target group(s) for nginx, tooling and wordpress were created

![image](https://user-images.githubusercontent.com/87030990/169605243-46e93532-a910-4d93-8864-84bb3181b8fc.png)

External Application Load Balancer to forward internet traffic to the nginx and Internal Application Load Balancers to route traffic to the Web Servers by assigning appropriate target groups were created

![image](https://user-images.githubusercontent.com/87030990/168461959-91e6e1cb-157f-411c-9a04-706a0680b83f.png)

Launch Templates were created with parameters (appropriate Userdata) to launch an instances for bastion, nginx, tooling webserver and wordpress webserver:

![image](https://user-images.githubusercontent.com/87030990/169170363-83adb468-d691-462f-9c46-ad655357b3cc.png)

Auto Scaling Groups for Bastion and nginx were created first

![image](https://user-images.githubusercontent.com/87030990/169576719-b318d248-6bcb-4f30-8712-6e01efd2ffe1.png)

Database for tooling and wordpress named **toolingdb** and **wordpressdb** were created by login into the RDS database from the bastion server

![image](https://user-images.githubusercontent.com/87030990/169141490-3fe248bb-416e-443a-b747-9f9ecd6bbac1.png)

![image](https://user-images.githubusercontent.com/87030990/169141762-6cff404c-f0ee-4fbd-af11-cad7fcf19989.png)

Auto Scaling Groups for tooling and wordpress were also created

![image](https://user-images.githubusercontent.com/87030990/169606056-9d24cef6-25cd-4bf9-9fe4-aff90f5d488f.png)

The health status of each insances were verified to be healthy:

For nginx

![image](https://user-images.githubusercontent.com/87030990/169608976-ca142750-4a9c-4920-bdd2-50d7b1546639.png)

For Wordpress

![image](https://user-images.githubusercontent.com/87030990/169614889-94c3f6af-6aa5-4977-9d0e-c8a3d270a556.png)

For Tooling

![image](https://user-images.githubusercontent.com/87030990/169172069-b82f545a-1d43-485f-ab68-0f639f479619.png)


#### Step 5: Configuring DNS with Route53

In the Route 53, A and alias Records were created for tooling and wordpress for accessibility from the internet

![image](https://user-images.githubusercontent.com/87030990/169151283-2347b07e-2ad0-4b92-8100-48ba81e6287b.png)

An alias record was created for the root domain and its traffic directed to the ALB DNS name while alias record was also created for tooling.toolingobaf.ga and its traffic directed to the ALB DNS name.

![image](https://user-images.githubusercontent.com/87030990/169607954-07aff919-0009-46af-9d98-31d4c8fd886b.png)

The infrastruction was tested and confirmed to be successful with results below for both websites. One of the nginx instance was stopped to test availability and there was not web failure.


#### Step 6: Check access to the websites from a browser

Wordpress susccesfully launched

![image](https://user-images.githubusercontent.com/87030990/169169246-b2d25919-8432-4a9c-a0f0-0a4562099522.png)

Tooling susccesfully launched

![image](https://user-images.githubusercontent.com/87030990/169570308-1a6389d9-a6e1-449c-98cc-d5f7d44bf5d9.png)


**Conclusion:** A secured, scalable and cost-effective infrastructure to host 2 enterprise websites using various Cloud services from AWS was successfully implemented
