### AWS CLOUD SOLUTION FOR 2 COMPANY WEBSITES USING A REVERSE PROXY TECHNOLOGY

**Objective:** To build an infrastructure that can serve 2 or more different websites that is resilient to Web Server’s failures, can accomodate increased traffic, at the lowest possible infrastructure and cloud cost while still satisfying high availability and security.

#### Step 1: Set up a Virtual Private Network (VPC) needed for the implementation

Virtual Private Cloud (VPC) was created
![image](https://user-images.githubusercontent.com/87030990/169170871-81f6737a-b812-4d0c-b81d-15b7c544410b.png)

6 Subnets were created (2 Public and 4 Private Subnets) in 2 Availability Zones

![image](https://user-images.githubusercontent.com/87030990/169170969-e3728757-7398-4bae-b959-da8914704b3e.png)

Internet Gateway was created for accessibility by the public from the internet and attached to the VPC

![image](https://user-images.githubusercontent.com/87030990/169171022-a93c5563-edc5-4500-9e51-cb2c8bd9c75b.png)

Route table for public subnets was created and associated with the 2 public subnets

![image](https://user-images.githubusercontent.com/87030990/169171193-ffa2e91a-3301-4c8e-aa0d-c9191080c3fd.png)

Route table for private subnet was created and associated with the 4 private subnets. Internet gateway route was added to the routing table for public subnets

![image](https://user-images.githubusercontent.com/87030990/169171259-8ef97727-b3ff-4ad3-a051-524b94975a75.png)


NAT Gateway was created in the public subnet and an Elastic IP allocated to it. Route for the NAT gateway was added to routing table for private subnets

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

RDS Database was then created

![image](https://user-images.githubusercontent.com/87030990/169141490-3fe248bb-416e-443a-b747-9f9ecd6bbac1.png)

![image](https://user-images.githubusercontent.com/87030990/168443526-1b3ba60d-3957-411e-a9ae-a7ca198bdcac.png)

Wordpress Website
![image](https://user-images.githubusercontent.com/87030990/169168649-6144fa23-6f78-45e9-baaf-0cb0a8fecbe7.png)



#### Step 2: Set up Compute Resources inside the VPC

For a start a base imsge was chosen to help in building our own images (CentOS was chosen from AWS Market Place)

A base-ami from amazon market place was launched and used to create my images

![image](https://user-images.githubusercontent.com/87030990/169170229-eedb60ef-a32d-430b-8fa1-8dea5158467c.png)

Launch Templates were created for bastion, nginx, tooling webserver and wordpress webserver:

![image](https://user-images.githubusercontent.com/87030990/169170363-83adb468-d691-462f-9c46-ad655357b3cc.png)

External and Internal Load Balancers and the target groups were created

![image](https://user-images.githubusercontent.com/87030990/168461959-91e6e1cb-157f-411c-9a04-706a0680b83f.png)





RDS Database was created



Database for tooling and wordpress named toolingdb and wordpressdb were created by login into the rds from the bastion server

![image](https://user-images.githubusercontent.com/87030990/169141762-6cff404c-f0ee-4fbd-af11-cad7fcf19989.png)

Auto Scaling Groups was created

![image](https://user-images.githubusercontent.com/87030990/169170058-3f66f1fd-824a-41e5-90c0-ae645aacd2e1.png)


In the Route 53, Records were created for tooling and wordpress Load Balancing

![image](https://user-images.githubusercontent.com/87030990/169151283-2347b07e-2ad0-4b92-8100-48ba81e6287b.png)


Wordpress target group healthy
![image](https://user-images.githubusercontent.com/87030990/169155769-34b99ea2-e064-419d-8f46-587d7ca1bdb3.png)

Wordpress susccesfully installed and accessed

![image](https://user-images.githubusercontent.com/87030990/169169246-b2d25919-8432-4a9c-a0f0-0a4562099522.png)










