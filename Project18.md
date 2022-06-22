### Automate Infrastructure With IaC using Terraform-Part 3

**Objective:**

REFACTOR YOUR PROJECT USING MODULES
Terraform codes were broken down to have all resources in their respective modules. Resources of a similar type were combined into directories within a ‘modules’ directory as discribed below

  - ALB: For Apllication Load balancer and similar resources
  - EFS: For Elastic file system resources
  - RDS: For Databases resources
  - Autoscaling: For Autosacling and launch template resources
  - compute: For EC2 and rlated resources
  - VPC: For VPC and netowrking resources such as subnets, roles, e.t.c.
  - security: for creating security group resources
  
  Each module shall contain following files:vpc-0c8cf1db6ce27cf80 | ACS-VPC
  
- main.tf (or %resource_name%.tf) file(s) with resources blocks
- outputs.tf (optional, if you need to refer outputs from any of these resources in your root module)
- variables.tf (as we learned before - it is a good practice not to hard code the values and use variables)

Introducing Backend on S3

S3 bucket as a backend where the state file can be accessed remotely by other DevOps team members was configured


Navigate to the DynamoDB table inside AWS and leave the page open in your browser. Run terraform plan and while that is running, refresh the browser and see how the lock is being handled:
![image](https://user-images.githubusercontent.com/87030990/175098157-01ea2b9c-917c-4c8b-b2d3-0d019fb6e0b8.png)

After terraform plan completes, refresh DynamoDB table.
![image](https://user-images.githubusercontent.com/87030990/175098866-928f5d2e-229c-43a8-b01f-724dd4cb4663.png)

Let us run terraform apply
Terraform will automatically read the latest state from the S3 bucket to determine the current state of the infrastructure. Even if another engineer has applied changes, the state file will always be up to date.

Now, head over to the S3 console again, refresh the page, and click the grey “Show” button next to “Versions.” You should now see several versions of your terraform.tfstate file in the S3 bucket:
![image](https://user-images.githubusercontent.com/87030990/175122841-53fa3896-f965-41bc-b416-510afdfa1556.png)

Final output
![image](https://user-images.githubusercontent.com/87030990/175133019-3c60786f-6295-4b73-aefe-47c44c818c7b.png)
![image](https://user-images.githubusercontent.com/87030990/175133225-4bdfc733-b034-4325-a1a6-7fdc10da11cb.png)
![image](https://user-images.githubusercontent.com/87030990/175133321-d81bc274-9036-4540-94a9-7d7ca5a140fc.png)
![image](https://user-images.githubusercontent.com/87030990/175133476-f112abff-5b99-4764-a6c5-524af8df1f6a.png)
![image](https://user-images.githubusercontent.com/87030990/175133688-3c41b1a0-cb4c-4676-8403-ff1bbc2f0bae.png)
![image](https://user-images.githubusercontent.com/87030990/175133864-f995582f-ec0e-4aba-9f2a-4d23a4a117a5.png)
![image](https://user-images.githubusercontent.com/87030990/175134030-fc71b04d-de99-46a9-981c-75a16574f935.png)
![image](https://user-images.githubusercontent.com/87030990/175134133-f6d43908-3bd9-412b-aa36-59337769c84b.png)
![image](https://user-images.githubusercontent.com/87030990/175134654-04c5200b-0f7e-4c6e-b36d-2bdaa76624f1.png)
![image](https://user-images.githubusercontent.com/87030990/175134886-c7731eeb-d346-4a1f-be90-2e110f0b9d1a.png)
![image](https://user-images.githubusercontent.com/87030990/175135361-9ff96dab-c2c5-4695-89fb-c5d85a9689f8.png)
![image](https://user-images.githubusercontent.com/87030990/175135543-fd2bcfdd-fcfd-48df-8600-9ac76a54e8bc.png)
![image](https://user-images.githubusercontent.com/87030990/175135834-924bd76f-53bd-4db9-b6b2-37ccfb976da6.png)
![image](https://user-images.githubusercontent.com/87030990/175136056-1b4578bb-1d3b-4a0c-9df9-446f932b7117.png)



