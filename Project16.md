### Project 16 Automate Infrastructure With IAC using Terraform Part 1

**Objective:**: To use Terraform to build an infrastructure that can serve 2 or more different websites that is resilient to Web Server’s failures, can accomodate increased traffic, at the lowest possible infrastructure and cloud cost while still satisfying high availability and security.

S3 bucket named was created to store Terraform state file and programmatic CLI access to AWS was also confirmed running **aws s3 ls** command

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


#### Step 1: Create VPC, Subnets and Security Groups

AWS Provider and resource to create VPC was added to **main.tf** file created:
![image](https://user-images.githubusercontent.com/87030990/172056782-8a5999e5-4eef-491c-bc91-703eefbc5852.png)

Plugin for AWS Provider was downloaded using **terraform init** command

![image](https://user-images.githubusercontent.com/87030990/172057476-c3361a61-487d-4500-86d9-acb911ca872c.png)

**aws_vpc** resource was created using **terraform apply** after checking what terraform intends to created using **terraform plan** command
![image](https://user-images.githubusercontent.com/87030990/172057738-4d758054-a8b3-44bc-b31e-9a5c70d08e80.png)
