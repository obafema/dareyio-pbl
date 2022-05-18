### AWS CLOUD SOLUTION FOR 2 COMPANY WEBSITES USING A REVERSE PROXY TECHNOLOGY

**Objective:** To build an infrastructure that can serve 2 or more different websites that is resilient to Web Server’s failures, can accomodate increased traffic, at the lowest possible infrastructure and cloud cost while still satisfying high availability and security.

#### Step 1: Set up a Virtual Private Network (VPC) needed for the implementation

Virtual Private Cloud (VPC) was created
![image](https://user-images.githubusercontent.com/87030990/168436231-d7b97b53-d5ca-4328-9bf2-5f27741f5769.png)

6 Subnets were created (2 Public and 4 Private Subnets) in 2 Availability Zones

![image](https://user-images.githubusercontent.com/87030990/168438811-984bcf19-a833-46fd-8f7a-a454b92dee5d.png)

Internet Gateway was created for accessibility by the public from the internet and attached to the VPC

![image](https://user-images.githubusercontent.com/87030990/168439139-1d3d82f4-1554-418e-a773-8f5bbea53ec6.png)

Route table for public subnets was created and associated with the 2 public subnets

![image](https://user-images.githubusercontent.com/87030990/168440094-ec5c7a76-bac9-4ef4-9e9c-a47df8b5f127.png)

Route table for private subnet was created and associated with the 4 private subnets

![image](https://user-images.githubusercontent.com/87030990/168440289-ccaaddfa-3ace-43e2-82fa-68928d5d54cb.png)

**Note:** We need 3 Elastic IPs for this implementation. 1 for the NAT Gateway and the other 2 for the Bastion server

NAT Gateway was created in the public subnet and an Elastic IP allocated to it

![image](https://user-images.githubusercontent.com/87030990/168487053-cbb58df9-6d25-4b5b-8220-2bc73494e37f.png)

Security Group for External Public Facing Application Load Balancer, Nginx Servers, Bastion Servers, Internal Non Public Facing Application Load Balancer, Webservers and Data Layer were created

![image](https://user-images.githubusercontent.com/87030990/168500344-726d7efd-9e1a-4585-ac05-e448fda093b2.png)

Create Certificate

![image](https://user-images.githubusercontent.com/87030990/168501077-1ba5a3a7-053c-43ba-944d-4aa624e0db80.png)


Create EFS

![image](https://user-images.githubusercontent.com/87030990/168502266-93b09fe8-baaa-458b-b35d-28b6820ed293.png)


Create RDS

Prerequisite for creating RDS

1ST Create KMS Key

![image](https://user-images.githubusercontent.com/87030990/168502204-65a1ead3-6558-4ba0-87c0-45dec2f97819.png)

Create RDS Subnet Group

![image](https://user-images.githubusercontent.com/87030990/168502629-e08a220d-4f6c-4580-bae5-8ec4133570da.png)


![image](https://user-images.githubusercontent.com/87030990/168443526-1b3ba60d-3957-411e-a9ae-a7ca198bdcac.png)


#### Step 2: Set up Compute Resources inside the VPC

For a start a base imsge was chosen to help in building our own images (CentOS was chosen from AWS Market Place)

A base-ami from amazon market place was launched and used to create my images

![image](https://user-images.githubusercontent.com/87030990/168446486-132d6bad-e07d-44e9-a6cd-bc071ce8d763.png)
![image](https://user-images.githubusercontent.com/87030990/168446495-a877f7dd-4be8-4962-accb-44296d5eeb3b.png)

Launch Templates were created for bastion, nginx, tooling webserver and wordpress webserver:

![image](https://user-images.githubusercontent.com/87030990/168448295-fbb0586e-f937-4c12-b009-7262a43b15fc.png)

External and Internal Load Balancers and the target groups were created

![image](https://user-images.githubusercontent.com/87030990/168461959-91e6e1cb-157f-411c-9a04-706a0680b83f.png)

A subnet group was created for Amazon Aurora

![image](https://user-images.githubusercontent.com/87030990/169141490-3fe248bb-416e-443a-b747-9f9ecd6bbac1.png)

RDS Database was created

Database for tooling and wordpress named toolingd and wordpressdb were created by login into the rds from the bastion server

![image](https://user-images.githubusercontent.com/87030990/169141762-6cff404c-f0ee-4fbd-af11-cad7fcf19989.png)

Auto Scaling Groups was created

In the Route 53, Records were created for tooling and wordpress Load Balancing

![image](https://user-images.githubusercontent.com/87030990/169151283-2347b07e-2ad0-4b92-8100-48ba81e6287b.png)




Wordpress target group healthy
![image](https://user-images.githubusercontent.com/87030990/169155769-34b99ea2-e064-419d-8f46-587d7ca1bdb3.png)









