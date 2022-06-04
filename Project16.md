### Project 16 Automate Infrastructure With IAC using Terraform Part 1

**Objective:**: To use Terraform to build an infrastructure that can serve 2 or more different websites that is resilient to Web Server’s failures, can accomodate increased traffic, at the lowest possible infrastructure and cloud cost while still satisfying high availability and security.

S3 bucket named was created to store Terraform state file

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

#### Step 1: Create VPC
