### Project 16 Automate Infrastructure With IAC using Terraform Part 1

**Objective:**: To create and delete AWS Network infrastructure programmatically using Terraform

Programmatic access was configured from the workstation to connect to AWS using the access keys created and a Python SDK (boto3) installed.

S3 bucket named was created on AWS to store Terraform state file and programmatic CLI access to AWS from the workstation was also confirmed by running **aws s3 ls** command

```bash
aws s3 ls
````
![image](https://user-images.githubusercontent.com/87030990/172015771-f2b1ab67-6f01-4fef-9078-66c02e03b472.png)

or from >python:
````bash
import boto3
s3 = boto3.resource('s3')
for bucket in s3.buckets.all():
    print(bucket.name)
````
![image](https://user-images.githubusercontent.com/87030990/172017017-883ae25c-d4ca-4586-8e5f-7d5126793327.png)


#### Step 1: Create VPC and Subnets

Directory structure was created using Visual Studio Code by creating a folder named **PBL**. A file named **main.tf** was then created inside the PBL

AWS Provider and resource to create VPC was added to **main.tf** file created:
![image](https://user-images.githubusercontent.com/87030990/172056782-8a5999e5-4eef-491c-bc91-703eefbc5852.png)

Plugin for AWS Provider was downloaded using **terraform init** command

![image](https://user-images.githubusercontent.com/87030990/172057476-c3361a61-487d-4500-86d9-acb911ca872c.png)

**aws_vpc** resource was created using **terraform apply** after checking what terraform intends to create using **terraform plan** command
![image](https://user-images.githubusercontent.com/87030990/172057738-4d758054-a8b3-44bc-b31e-9a5c70d08e80.png)
![image](https://user-images.githubusercontent.com/87030990/172058611-110206cc-6c2b-4a2a-b050-9477f27dd5ee.png)

Resources to create Subnets (2 Public Subnets) in 2 Availability Zones was added to the **main.tf** file.

N.B: **vpc_id** argument was used to interpolate the value of the VPC id by setting it to **aws_vpc.main.id**. This way, Terraform knows inside which VPC to create the subnet.

The 2 public subnets were first created for a start with 2 resource blocks with hard coded values for the **availability_zone** and **cidr_block** and was confirmed successfully created on aws

![image](https://user-images.githubusercontent.com/87030990/172059095-766ddf75-9002-4f5e-8311-672305bdc606.png)
![image](https://user-images.githubusercontent.com/87030990/172058564-a326fbcf-039b-4253-8a17-9ece7f232b9f.png)
![image](https://user-images.githubusercontent.com/87030990/172058682-06efa10e-a78f-4343-abf1-f6cefd93e9aa.png)

N.B: Hardcoding values is not the best practice and so the code was then refactored to make it more dynamic

The infrastructure was destroyed by runnung **terraform destroy** command in order to begin the refactorng process
![image](https://user-images.githubusercontent.com/87030990/172059480-f189a926-0697-4718-b46f-fa2291c30301.png)

#### Step 2: Fixing Hard coding Problems By Code Refactoring

Variables and count arguments were introduced

Starting with the **provider block**, a **variable** named **region** was declared and given a **default value**. The **provider section** was then updated by making reference to the declared variable. Same was done to **cidr value** in the **vpc block**, and all the other arguments.

![image](https://user-images.githubusercontent.com/87030990/172060432-9a882989-75db-476e-bb77-0bb6616d7a12.png)

Multiple resource blocks was also fixed by introducing concept of Loops & Data sources to fetch information from AWS (outside of Terraform)
The code below was added to fetch Availability zones from AWS, and replace the hard coded value in the subnet’s availability_zone section.

````bash
 # Get list of availability zones
        data "aws_availability_zones" "available" {
        state = "available"
        }
 ````
To make use of this new data resource, a count argument was introduced in the subnet block to tell terraform the number of resources to be created:
````bash
# Create public subnet
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = "172.16.1.0/24"
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
 ````

To make the code more dynamic, **cidrsubnet(prefix, newbits, netnum)** function was introduced into the cidr block while the hardcoded count value was replaced with **length()** function (which basically determines the length of a given list, map, or string.)

````bash
# Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = length(data.aws_availability_zones.available.names)
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 8 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
````
A variable was declared to store the desired number of public subnets, and the default value set to 2 based on our design
````bash
variable "preferred_number_of_public_subnets" {
  default = 2
}
````
The count argument was updated with a condition for Terraform to check first if there is a desired number of subnets. If not to use the data returned by the lenght function

````bash
# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
````

Running terraform plan give the output below

![image](https://user-images.githubusercontent.com/87030990/172062746-9f38713e-62f1-4c7e-a82f-791ad0bf60f5.png)

#### Step3: Introducing variables.tf & terraform.tfvars

**Variables.tf** and **terraform.tfvars** file were introduced and declared variables and defaults values copied into them respectively

![image](https://user-images.githubusercontent.com/87030990/172063780-8d106d8d-d4e8-45e5-a7fe-eed3ee376a51.png)

terraform apply was run to create the vpc and the 2 public subnets after running terraform plan which was successful.

![image](https://user-images.githubusercontent.com/87030990/172063903-f5dadd43-95da-48c9-b6be-8f0a335703c2.png)
![image](https://user-images.githubusercontent.com/87030990/172063929-ffac891e-9ed9-4cc7-b092-bb8521594200.png)


Conclusion: AWS Network infrastructure was successfully created and deleted programmatically using Terraform
