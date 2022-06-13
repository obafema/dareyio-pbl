### AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 2

Objective: 

#### Step 1: Create Network Resources

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

![image](https://user-images.githubusercontent.com/87030990/173195601-70038092-2b24-4bd4-9b5f-a61480319c30.png)

1 NAT Gateways and 1 Elastic IP (EIP) address were created

![image](https://user-images.githubusercontent.com/87030990/173195616-d9225120-bcdd-4ce9-999b-bc77d92bf2cd.png)

Terraform plan gives the output below with NAT-gw, IGW and EIP created

![image](https://user-images.githubusercontent.com/87030990/173062736-cfb0490b-964b-42a5-ab73-05211eb78a95.png)

AWS Route
A file was created named **route_tables.tf** and used to create routes for both public and private subnets

Private route table was created with all private subnets associated was attached to the NAT Gateway. Public route table was also created with all public subnets associated with it and was attached to the Internet Gateway.

![image](https://user-images.githubusercontent.com/87030990/173195989-49765473-d34c-4366-bcc1-b03355ecad59.png)
 
**terraform apply** was ran after verifying that the code is valid with **terraform validate** and confirming the resources that will be created with **terraform plan**

Main vpc, 2 Public subnets, 4 Private subnets, Internet Gateway, NAT Gateway, EIP and 2 Route tables were created:

![image](https://user-images.githubusercontent.com/87030990/173074934-ca87226c-b536-46ff-941b-c90667b60da8.png)
![image](https://user-images.githubusercontent.com/87030990/173075246-941e76dd-41de-4a0b-b10f-461e5fade34d.png)
![image](https://user-images.githubusercontent.com/87030990/173075418-a2c1e3d7-f5d3-4aa7-b0fe-51cf8baec129.png)
![image](https://user-images.githubusercontent.com/87030990/173075568-72127ea3-5fe8-4679-bd3c-04ff034cb7b4.png)
![image](https://user-images.githubusercontent.com/87030990/173075704-84022ae5-bca4-4279-970d-fb852d5d227c.png)
![image](https://user-images.githubusercontent.com/87030990/173075946-5e2b7744-df65-4a0a-9d14-b6b46cd459de.png)

AWS Identity and Access Management

IAM roles was created and defined IAM Policy was attached to the role
The IAM role was passed on EC2 instances to give them access to some specific resources

A new file named **role.tf** was created and updated. An Instance Profile was also created and the IAM Role interpolated

![image](https://user-images.githubusercontent.com/87030990/173197689-de16d0dd-ba96-4245-aa0f-7f12c16bcd75.png)
![image](https://user-images.githubusercontent.com/87030990/173197770-1edc18da-7e3f-43fe-b25e-8c092acc549a.png)


### Step 2: Create Security Groups

A single file named **security.tf** was created for security groups (External Load Balancer, Internat Load Balancer, nginx, bastion, webservers and datalayer) and the security group referenced within each resources that needs it. Security group rule was also created for nginx, internal load balancer, webserver and datalayer.

N.B: The **aws_security_group_rule** was used to reference another security group within the security group.

````bash
# security group for alb, to allow acess from any where for HTTP and HTTPS traffic
resource "aws_security_group" "ext-alb-sg" {
  name        = "ext-alb-sg"
  vpc_id      = aws_vpc.main.id
  description = "Allow TLS inbound traffic"

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

 tags = merge(
    var.tags,
    {
      Name = "ext-alb-sg"
    },
  )

}

# security group for bastion, to allow access into the bastion host from the specified/privileged IP
resource "aws_security_group" "bastion_sg" {
  name        = "vpc_web_sg"
  vpc_id = aws_vpc.main.id
  description = "Allow incoming HTTP connections."

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

   tags = merge(
    var.tags,
    {
      Name = "Bastion-SG"
    },
  )
}

#security group for nginx reverse proxy, to allow access only from the external load balancer and bastion instance
resource "aws_security_group" "nginx-sg" {
  name   = "nginx-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

   tags = merge(
    var.tags,
    {
      Name = "nginx-SG"
    },
  )
}

resource "aws_security_group_rule" "inbound-nginx-http" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.ext-alb-sg.id
  security_group_id        = aws_security_group.nginx-sg.id
}

resource "aws_security_group_rule" "inbound-bastion-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.nginx-sg.id
}

# security group for ialb, to have acces only from nginx reverser proxy server
resource "aws_security_group" "int-alb-sg" {
  name   = "my-alb-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.tags,
    {
      Name = "int-alb-sg"
    },
  )

}

resource "aws_security_group_rule" "inbound-ialb-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.nginx-sg.id
  security_group_id        = aws_security_group.int-alb-sg.id
}

# security group for webservers, to have access only from the internal load balancer and bastion instance
resource "aws_security_group" "webserver-sg" {
  name   = "my-asg-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.tags,
    {
      Name = "webserver-sg"
    },
  )

}

resource "aws_security_group_rule" "inbound-web-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.int-alb-sg.id
  security_group_id        = aws_security_group.webserver-sg.id
}

resource "aws_security_group_rule" "inbound-web-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.webserver-sg.id
}

# security group for datalayer to alow traffic from websever on nfs and mysql port and bastiopn host on mysql port
resource "aws_security_group" "datalayer-sg" {
  name   = "datalayer-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

 tags = merge(
    var.tags,
    {
      Name = "datalayer-sg"
    },
  )
}

resource "aws_security_group_rule" "inbound-nfs-port" {
  type                     = "ingress"
  from_port                = 2049
  to_port                  = 2049
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.webserver-sg.id
  security_group_id        = aws_security_group.datalayer-sg.id
}

resource "aws_security_group_rule" "inbound-mysql-bastion" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.datalayer-sg.id
}

resource "aws_security_group_rule" "inbound-mysql-webserver" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.webserver-sg.id
  security_group_id        = aws_security_group.datalayer-sg.id
}
````
![image](https://user-images.githubusercontent.com/87030990/173116266-062464c8-9fed-4ee8-8c33-4f5734b0934d.png)


### Step 3: Register a Domain and configure secured connection using SSL/TLS certificates

Create Certificate from Amazon Certificate Manager

**cert.tf** file was created and the following code snippets added:

````bash
# The entire section create a certiface, public zone, and validate the certificate using DNS method

# Create the certificate using a wildcard for all the domains created in <domain name>
resource "aws_acm_certificate" "name" {
  domain_name       = "*.<domain name>"
  validation_method = "DNS"
}

# calling the hosted zone
data "aws_route53_zone" "<name>" {
  name         = "<domain name>"
  private_zone = false
}

# selecting validation method
resource "aws_route53_record" "<name>" {
  for_each = {
    for dvo in aws_acm_certificate.<name>.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.<name>.zone_id
}

# validate the certificate through DNS method
resource "aws_acm_certificate_validation" "<name>" {
  certificate_arn         = aws_acm_certificate.<name>.arn
  validation_record_fqdns = [for record in aws_route53_record.<name> : record.fqdn]
}

# create records for tooling
resource "aws_route53_record" "tooling" {
  zone_id = data.aws_route53_zone.<name>.zone_id
  name    = "tooling.<domain name>"
  type    = "A"

  alias {
    name                   = aws_lb.ext-alb.dns_name
    zone_id                = aws_lb.ext-alb.zone_id
    evaluate_target_health = true
  }
}

# create records for wordpress
resource "aws_route53_record" "wordpress" {
  zone_id = data.aws_route53_zone.<name>.zone_id
  name    = "wordpress.<domain name>"
  type    = "A"

  alias {
    name                   = aws_lb.ext-alb.dns_name
    zone_id                = aws_lb.ext-alb.zone_id
    evaluate_target_health = true
  }
}
````

#### Step 4:Create an external (Internet facing) Application Load Balancer and Internal Application Load Balancer

A file called **alb.tf** was created and loaded with the code snippets:

Application Load Balancer was created as well as the targets and listeners. 

N.B: ALB is required to balance the traffic between the Instances.
![image](https://user-images.githubusercontent.com/87030990/173200079-43dbdf68-43a8-4594-a0bc-d47b07a6b7f3.png)
![image](https://user-images.githubusercontent.com/87030990/173200149-53830f41-d23e-4fc4-b05a-cd769e1e392f.png)

**terraform apply** was ran after verifying that the code is valid with **terraform validate** and confirming the resources that will be created with **terraform plan**

![image](https://user-images.githubusercontent.com/87030990/173116409-0e77c545-4f10-483a-af87-566863ee125b.png)

![image](https://user-images.githubusercontent.com/87030990/173114517-2b061b60-acc4-47f8-a860-338868be7293.png)
![image](https://user-images.githubusercontent.com/87030990/173116477-61688198-52c0-4fbd-981d-3ce58f8fa464.png)


#### Step 5: Create AutoScaling Groups
Auto Scaling Group was created to scale the EC2 Instances out or in depending on the application traffic

**asg-bastion-nginx.tf** was created and the code snippet below pasted:

Before starting configuring an ASG, Launch Templates were created with parameters (appropriate Userdata) to launch an instances for bastion, nginx, tooling webserver and wordpress webserver using a random AMI from AWS.

SNS-topic and SNS-notification was created for all the AutoScaling Groups

**asg-bastion-nginx.tf** was created and code snippet for bastion and nginx pasted:
![image](https://user-images.githubusercontent.com/87030990/173200539-29540ebc-5062-45ae-94c0-34c8c5a6b19f.png)
![image](https://user-images.githubusercontent.com/87030990/173200593-835f87a1-3695-4694-969f-b77df885b36d.png)

#### Step 6: Create Storage and Database

Create Elastic File System (EFS)
In order to create an EFS, a KMS key was created.

An **efs.tf** file was created and code snippets for the creation of KMS, EFS, mount targets for EFS and access points for wordpress and tooling added:

![image](https://user-images.githubusercontent.com/87030990/173438709-ae23fc5d-1ee7-413c-89d0-4e5cc580075d.png)

Create MySQL Relation Database System (RDS)

**rds.tf** file was created and code snippet for creation of RDS was added
Note: the subnet group was created as a pre-requisite for RDS creation.

![image](https://user-images.githubusercontent.com/87030990/173439510-1b525043-b68f-4592-9b53-016b4ebd5df2.png)

All variables referenced in each of the resources were properly declared in the **variable.tf** file while values for the variables also declared in the **terraform.tfvars**

![image](https://user-images.githubusercontent.com/87030990/173440038-7b3ddf04-153b-4e03-8260-b1de3828421b.png)
![image](https://user-images.githubusercontent.com/87030990/173440387-1eb3477d-f0e0-425a-bd6b-175c8a0ceccc.png)




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



