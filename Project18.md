### Automate Infrastructure With IaC using Terraform-Part 3

**Objective:** To enhance previously developed AWS Infrastructure code using Terraform and run it remotely from alternative Terraform backends (s3 bucket).

#### Step 1: Introducing Backend on S3

**S3 bucket** as a backend where the state file can be accessed remotely by other DevOps team members was configured. **DynamoDB** was also configured which is required for State Locking feature of an s3 bucket. **State Locking** was used to lock the state for all operations that could write state. This prevents others from acquiring the lock and potentially corrupting the state file.

**N.B:** s3 bucket as a backend was chosen as we are using AWS. 

s3 and DynamoDB resource blocks were added before deleting the local state file

Terraform block was updated to introduce s3 bucket and locking

![image](https://user-images.githubusercontent.com/87030990/175301095-5eb87fbb-c3d4-4167-a252-d987165b6a2e.png)

Terraform was then re-initialized

The **local tfstate file** was deleted and the one in S3 bucket was checked

outputs was added and **terraform apply** ran
![image](https://user-images.githubusercontent.com/87030990/175379306-3b26b973-d1bb-4397-bddc-2cc60597eb90.png)

![image](https://user-images.githubusercontent.com/87030990/175135834-924bd76f-53bd-4db9-b6b2-37ccfb976da6.png)

**DynamoDB table** was created to handle locks and perform consistency checks
![image](https://user-images.githubusercontent.com/87030990/175300939-757fd8b7-6fd3-49ea-b325-257382c87562.png)

**terraform apply** was ran to provision resources(s3 bucket and DynamoDB) before configuring the backend


A file named **backend.tf** was created and below code added to replace the name of the S3 bucket created in previous project (**Project-16**) to configure s3 backend.

![image](https://user-images.githubusercontent.com/87030990/175295625-5f84ee27-c9ce-4621-bf42-20d3bf42c4da.png)

**terraform init** was ran to re-initialized the backend and the changes verified on AWS

- **tfstatefile** is now inside the S3 bucket

- **DynamoDB table** which we create has an entry which includes state file status

Navigate to the DynamoDB table inside AWS and leave the page open in your browser. Run **terraform plan** and while that is running, refresh the browser and see how the lock is being handled:

![image](https://user-images.githubusercontent.com/87030990/175098157-01ea2b9c-917c-4c8b-b2d3-0d019fb6e0b8.png)

After **terraform plan** completes, refresh DynamoDB table.
![image](https://user-images.githubusercontent.com/87030990/175098866-928f5d2e-229c-43a8-b01f-724dd4cb4663.png)

**output.tf file** was created in the root directory and the code snippets added so that the s3 bucket Amazon Resource Names ARN and DynamoDB table name can be displayed.

![image](https://user-images.githubusercontent.com/87030990/175306218-fbe5f3c2-99bc-4655-95b4-2c79ce528498.png)

**terraform apply** was then executed

Terraform will automatically read the latest state from the **s3 bucket** to determine the current state of the infrastructure. Even if another engineer has applied changes, the state file will always be up to date.

Now, head over to the S3 console again, refresh the page, and click the grey “Show” button next to “Versions.” You should now see several versions of the **terraform.tfstate** file in the S3 bucket:
![image](https://user-images.githubusercontent.com/87030990/175122841-53fa3896-f965-41bc-b416-510afdfa1556.png)

**N.B:** With help of remote backend and locking configuration that we have just configured, collaboration is no longer a problem. Others can now effectively access the **state file** remotely.

#### Step 2: Project Refactoring using modules

Terraform codes were broken down to have all resources in their respective modules. Resources of a similar type were combined into directories within a ‘modules’ directory as discribed below
````bash
  - ALB: For Apllication Load balancer and similar resources
  - EFS: For Elastic file system resources
  - RDS: For Databases resources
  - Autoscaling: For Autoscaling and launch template resources
  - compute: For EC2 and related resources
  - VPC: For VPC and netowrking resources such as subnets, roles, internet gateway, nat gateway, route tables e.t.c.
  - security: for creating security group resources
  ````
  
  ![image](https://user-images.githubusercontent.com/87030990/175339578-27ef296d-3603-4e50-886d-af5baa099a97.png)
  
  
  Each **module** shall contain following files:
````bash 
- main.tf (or %resource_name%.tf) file(s) with resources blocks
- outputs.tf (optional, if you need to refer outputs from any of these resources in your root module)
- variables.tf (as we learned before - it is a good practice not to hard code the values and use variables)
````
![image](https://user-images.githubusercontent.com/87030990/175377083-2c766773-b03e-40eb-87c8-44098ffbd46f.png)

Security Groups was refactored with dynamic block

![image](https://user-images.githubusercontent.com/87030990/175378269-ba4052e1-5fbd-47d5-85ff-59882dc8a781.png)
![image](https://user-images.githubusercontent.com/87030990/175378665-44d7c655-5380-46b7-ae5b-0a243d05d35c.png)
![image](https://user-images.githubusercontent.com/87030990/175378728-1258b655-a9d0-45b3-a8a8-b5f7e5a4a25a.png)


**providers** and **backends** sections were configured in separate files and placed in the root module.
![image](https://user-images.githubusercontent.com/87030990/175340473-e50709e8-eec2-44b0-9779-a12a5d54379c.png)

**Module** was imported into **main.tf** file in the root module as a source and access to its variables via **var** keyword was gotten:
e.g
````bash
module "VPC" {
  source = "./modules/VPC"
  region = var.region
  ...
  ````
  ![image](https://user-images.githubusercontent.com/87030990/175344805-998e0b8b-9f60-4051-b638-e02e327b536f.png)

**Terraform apply** was executed after the code has been validated using **terraform validate** and **terraforn plan** ran to confirm the resources that will be created

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

Conclusion: AWS Infrastructure was successfully developed using Terraform and was ran remotely from alternative Terraform backends (s3 bucket)


