### AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 2

Objective: 

#### Step 1: Create Resources

Having created 2 public subnets, 4 private subnets were then created according to the design

````bash
# Create private subnets
resource "aws_subnet" "private" {
  count                   = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index + 2)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
````

Creating 4 subnets using in a region with 3 Availability Zones using **Data Source: aws_availability_zones** returned **Invalid index** error as shown below

![image](https://user-images.githubusercontent.com/87030990/173038032-ecd81a39-d102-496e-ae27-283a04bada58.png)

Resolution: **random_shuffle** resource was used to shuffle the availability zones

````bash
# Create private subnets
resource "aws_subnet" "private-subnets" {
  vpc_id                  = aws_vpc.main.id
  count                   = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index + 2)
  map_public_ip_on_launch = false
  availability_zone       = random_shuffle.az_list.result[count.index]

  tags = {
    Name = "PrivateSubnet"
  }

}
````
![image](https://user-images.githubusercontent.com/87030990/173048977-d93c1077-c32d-4589-b858-e84f432c37a8.png)


Running terraform plan gives the output below

![image](https://user-images.githubusercontent.com/87030990/173048414-55d4f347-d36f-4907-a6d6-8eb389799f13.png)
![image](https://user-images.githubusercontent.com/87030990/173048483-d40303b2-8952-4b20-8687-ddc75a31fbb7.png)
![image](https://user-images.githubusercontent.com/87030990/173048558-b09756e9-c5bb-4473-81e8-90532ee57b2f.png)
![image](https://user-images.githubusercontent.com/87030990/173048650-40502f1d-7321-41fc-9f0b-f855f6617a6e.png)

#### Step 2: Tagging of Resources for effective management

Multiple tags was added as a default set to the **terraform.tfvar** file e.g

````bash
tags = {
  Enviroment      = "production" 
  Owner-Email     = "example.io"
  Managed-By      = "Terraform"
  Billing-Account = "1234567890"
}
````
![image](https://user-images.githubusercontent.com/87030990/173054277-2c991094-48e6-4ad9-96ab-9053b16df49e.png)

Resources were tagged using the format below:

````bash
tags = merge(
    var.tags,
    {
      Name = format("%s-PrivateSubnet-%s", var.name, count.index)
    },
  )

}
````
Variable tags were then declared in the variable.tf file using the format below

````bash
variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
````

Internet Gateway was created in a separate Terraform file named **internet_gateway.tf**


1 NAT Gateways and 1 Elastic IP (EIP) address were created

Terraform plan gives the output below with NAT-gw, IGW and EIP created

![image](https://user-images.githubusercontent.com/87030990/173062736-cfb0490b-964b-42a5-ab73-05211eb78a95.png)

AWS Route
A file was created named **route_tables.tf** and used to create routes for both public and private subnets

````bash
# create private route table
resource "aws_route_table" "private-rtb" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Private-Route-Table", var.name)
    },
  )
}

# create route for the private route table and attach the internet gateway
resource "aws_route" "private-rtb-route" {
  route_table_id         = aws_route_table.private-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_nat_gateway.nat.id
}

# associate all private subnets to the private route table
resource "aws_route_table_association" "private-subnets-assoc" {
  count          = length(aws_subnet.private-subnets[*].id)
  subnet_id      = element(aws_subnet.private-subnets[*].id, count.index)
  route_table_id = aws_route_table.private-rtb.id
}

# create route table for the public subnets
resource "aws_route_table" "public-rtb" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Public-Route-Table", var.name)
    },
  )
}

# create route for the public route table and attach the internet gateway
resource "aws_route" "public-rtb-route" {
  route_table_id         = aws_route_table.public-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}

# associate all public subnets to the public route table
resource "aws_route_table_association" "public-subnets-assoc" {
  count          = length(aws_subnet.public-subnets[*].id)
  subnet_id      = element(aws_subnet.public-subnets[*].id, count.index)
  route_table_id = aws_route_table.public-rtb.id
}
````
**terraform apply** was ran after verifying that the code is valid with **terraform validate** and confirming the resources that will be created with **terraform plan**
![image](https://user-images.githubusercontent.com/87030990/173074934-ca87226c-b536-46ff-941b-c90667b60da8.png)
![image](https://user-images.githubusercontent.com/87030990/173075246-941e76dd-41de-4a0b-b10f-461e5fade34d.png)
![image](https://user-images.githubusercontent.com/87030990/173075418-a2c1e3d7-f5d3-4aa7-b0fe-51cf8baec129.png)
![image](https://user-images.githubusercontent.com/87030990/173075568-72127ea3-5fe8-4679-bd3c-04ff034cb7b4.png)
![image](https://user-images.githubusercontent.com/87030990/173075704-84022ae5-bca4-4279-970d-fb852d5d227c.png)
![image](https://user-images.githubusercontent.com/87030990/173075946-5e2b7744-df65-4a0a-9d14-b6b46cd459de.png)

AWS Identity and Access Management
IaM and Roles
We want to pass an IAM role our EC2 instances to give them access to some specific resources, so we need to do the following:

Create AssumeRole
Assume Role uses Security Token Service (STS) API that returns a set of temporary security credentials that you can use to access AWS resources that you might not normally have access to. These temporary credentials consist of an access key ID, a secret access key, and a security token. Typically, you use AssumeRole within your account or for cross-account access.

Add the following code to a new file named roles.tf



Step 2: Register a Domain and configure secured connection using SSL/TLS certificates
CREATE CERTIFICATE FROM AMAZON CERIFICATE MANAGER
Create cert.tf file and add the following code snippets to it.

CREATE SECURITY GROUPS

N.B:  the aws_security_group_rule was used to reference another security group in a security group.
![image](https://user-images.githubusercontent.com/87030990/173116266-062464c8-9fed-4ee8-8c33-4f5734b0934d.png)



Create an external (Internet facing) Application Load Balancer (ALB)
Create a file called alb.tf

**terraform apply** was ran after verifying that the code is valid with **terraform validate** and confirming the resources that will be created with **terraform plan**
![image](https://user-images.githubusercontent.com/87030990/173116409-0e77c545-4f10-483a-af87-566863ee125b.png)

![image](https://user-images.githubusercontent.com/87030990/173114517-2b061b60-acc4-47f8-a860-338868be7293.png)
![image](https://user-images.githubusercontent.com/87030990/173116477-61688198-52c0-4fbd-981d-3ce58f8fa464.png)

CREATING AUSTOALING GROUPS

Create asg-bastion-nginx.tf and paste all the code snippet below


STORAGE AND DATABASE
Create Elastic File System (EFS)
In order to create an EFS you need to create a KMS key.


Error while creating rds subnet group. The 3rd and 4th subnets were shuffled into the same AZ
![image](https://user-images.githubusercontent.com/87030990/173157057-99366f11-72d0-4663-b010-87d8de5923b9.png)
![image](https://user-images.githubusercontent.com/87030990/173161956-2a767290-3927-49c6-abb3-70d6ad41f56e.png)

![image](https://user-images.githubusercontent.com/87030990/173161920-850210a0-26de-4c10-a265-fc8c431fb93d.png)

Create Relational Database System(RDS)
Create subnet group first
![image](https://user-images.githubusercontent.com/87030990/173162025-cb803b72-4970-4805-a773-fe811c3cdc71.png)

![image](https://user-images.githubusercontent.com/87030990/173162045-f8200da5-d47e-4155-97c7-cca9073406c3.png)

![image](https://user-images.githubusercontent.com/87030990/173162071-8e47f573-6886-43bd-8485-c17a60397669.png)

![image](https://user-images.githubusercontent.com/87030990/173162127-28f8b02c-5a92-4959-9418-c4b2604164ab.png)

![image](https://user-images.githubusercontent.com/87030990/173162166-f3a86ff8-2d53-48b5-b67e-c552634e3dd9.png)

![image](https://user-images.githubusercontent.com/87030990/173162210-605b54b9-bfa0-4ac5-8918-2ecb84a3655c.png)

![image](https://user-images.githubusercontent.com/87030990/173162272-5ddf0585-09ed-4486-90c5-b6671b24df59.png)



