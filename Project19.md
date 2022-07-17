### AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 4 – TERRAFORM CLOUD

**Objective:** To migrate the previously written terraform codes for creating AWS Cloud Solution for 2 company websites using a reverse proxy technology to Terraform cloud and managing our AWS Infrastructure from there.

Tasks
- Create Amazon Machine Images (AMIs) for our instances using Packer and define our shell scripts to create the required instances inside it.
- Migrate our terraform codes to Terraform Cloud after updating the terraform script with AMI IDs and provision our infrastructure from there.
- Use GitLab to run Terraform commands triggered from our git repository.
- Configure our infrastructire using Ansible


#### Step 1: Migrate your terraform codes to Terraform Cloud

A repository named Terraform-cloud was created in Gitlab and the previously developed terraform codes pushed into it for connection to the version control workflow

![image](https://user-images.githubusercontent.com/87030990/179385759-65b3e29c-276c-480b-bb11-8d0c89c9d7ef.png)

Terraform Cloud account and an organisation were created. Also configured was a workspace named terraform-cloud with version control workflow to run Terraform commands triggered from our git repository.

![image](https://user-images.githubusercontent.com/87030990/179385493-064cd907-027a-4b0c-a843-d8752c9bc960.png)

Variable was configured

Two environment variables (**AWS_ACCESS_KEY_ID** and **AWS_SECRET_ACCESS_KEY**) and their values were set to enable Terraform Cloud to apply the codes from our GitLab to create all necessary AWS resources (infrastructure) using Terraform Cloud.

![image](https://user-images.githubusercontent.com/87030990/179386033-bbb16de3-cbc2-4e9f-9e68-5975935a79a4.png)

**auto.tfvars** was used for the variable default values to make sure that every variable specified in the configuration is automatically uploaded to the workspace on Terraform Cloud instead of loading it manually.

Automatic speculative plan was checked to trigger speculative terraform plan for pull requests to the repository

#### Step 2: Develop packer files and define our shell script to create our AMIs

N.B: The shell scripts tells packer what needs to be picked inside the AMIs we are creating

Packer files (.pkr.hcl) for Bastion, nginx, webservers and ubuntu (sonarqube,jenkins and artifactory) and the respective shell script developed. See screenshot below for Bastion for the creation of our AMIs.

![image](https://user-images.githubusercontent.com/87030990/179386991-b2ec781f-7a14-4270-8ebf-cb64c20a89d2.png)

![image](https://user-images.githubusercontent.com/87030990/179386537-358f5cad-1a64-4067-bb0f-41ce844e1ac5.png)
![image](https://user-images.githubusercontent.com/87030990/179386767-22fba560-9a52-440f-9ad6-c8eba8834ac6.png)

Packer was installed on my loal machine in order to run packer commands

![image](https://user-images.githubusercontent.com/87030990/179388303-466fe58e-c225-49a6-b536-f7079aa57858.png)


Bastion AMIs was then created by running packer build command from the directory where we have the packer files

````bash
packer build .\bastion.pkr.hcl
````

Same was done for nginx, webservers and ubuntu(sonar)

![image](https://user-images.githubusercontent.com/87030990/179388563-12eafb9d-003e-4eef-ac24-d2944f65fa24.png)

#### Step 3: Create AWS Resources using terraform

Update the terraform script with the AMI IDs created with packer build

Comment out the listeners from the ALB file and the attachments for webservers in the Autoscaling group files to prevent the Load Balancers from forwarding traffic to the target groups before completing the configuration of the resources to prevent healthcheck failure.

N.B: The target groups will not have instances coming from the Autoscaling Group.

![image](https://user-images.githubusercontent.com/87030990/179389129-a1024dc0-8c83-4e44-a088-80a71a58d0b7.png)

![image](https://user-images.githubusercontent.com/87030990/179389152-2d1e9a86-07d0-416f-ac91-5397f6ddbe48.png)

![image](https://user-images.githubusercontent.com/87030990/179389195-ce1131fe-faff-424f-82a1-39fae8b210af.png)

![image](https://user-images.githubusercontent.com/87030990/179389220-8db6c193-5475-4bfa-9a71-6a928c669243.png)

![image](https://user-images.githubusercontent.com/87030990/179389253-2ef40496-ec5c-4b91-a7d8-bd0d24ea5680.png)


Run terraform script to create the resources

Changes to the code was saved, added, committed and pushed to GitLab to trigger creation of resources

The plan failed initially do to error in screenshot below. The name of the module was incorrect (module/security instead of module/Security)

![image](https://user-images.githubusercontent.com/87030990/179390810-c00d432d-b87b-45d8-882d-d5fd8397373b.png)

The error was corrected by changing the upper letter S in the security module to lower case s in the
**main.tf** in the root module

![image](https://user-images.githubusercontent.com/87030990/179390972-9a0121c8-5b62-4535-8421-c077e19f45cb.png)
![image](https://user-images.githubusercontent.com/87030990/179391382-02d2ea19-2734-453f-bab1-ceac87afe4a8.png)
![image](https://user-images.githubusercontent.com/87030990/179391533-c67c4b5b-8c99-4c58-8724-52b0d07ed401.png)


Changes to the code was saved, added, committed and again pushed to GitLab. Pipeline was passed as shown and 89 resources created

![image](https://user-images.githubusercontent.com/87030990/179391478-97bb2fb9-540c-49ce-b2fd-4e3677167e66.png)
![image](https://user-images.githubusercontent.com/87030990/179391820-dc2a4069-3aae-4c2b-9969-9addcbb52794.png)

![image](https://user-images.githubusercontent.com/87030990/179391735-c256fbda-44b0-488e-944e-52cea378da4c.png)

#### Step 4: Configure AWS Resources using Ansible

Logon to the Bastion server and clone ansible repository from GitLab to the bastion
Open the folder from Visual Studio Code and cd to the diectory

![image](https://user-images.githubusercontent.com/87030990/179392314-0d7c705e-7383-4f87-af56-33433bb90822.png)

Ansible was then configured to have access to our AWS account using aws configure command and providing the ACCESS KEY, SECRET KEY, the region and data format
````bash
aws configure
````

Access to aws account was verified by running **aws s3 ls**
````bash
aws s3 ls
````
![image](https://user-images.githubusercontent.com/87030990/179394365-b595aae4-0aaf-481e-a0fd-0a42c4f20264.png)


Ansible-inventory -i inventory/aws_ec2.yml --list to verify that Ansible can use the inventory which failed because plugin for ec2 dynamic inventory was not present

![image](https://user-images.githubusercontent.com/87030990/179392869-481d5b80-b244-4916-bc59-2908771ae8c0.png)

Boto3 and Botocore was then install using
````bash
sudo python3.8 -m pip install boto3 botocore
````

![image](https://user-images.githubusercontent.com/87030990/179394389-5ca7fc96-2c05-4db9-8967-2993957ef40b.png)


ansible-inventory -i inventory/aws_ec2.yml --graph was ran and it was verified that ansible can pull the IP address of our instances as shown below

![image](https://user-images.githubusercontent.com/87030990/179394436-c0623d59-4860-4268-9fa6-01a0b9559a6d.png)

RDS endpoint for tooling and wordpress was updated
![image](https://user-images.githubusercontent.com/87030990/179394678-1e407c1c-3acb-4006-a19a-484bca814642.png)

![image](https://user-images.githubusercontent.com/87030990/179394744-542c5154-a017-4412-9d3c-1fb688a65792.png)

Access point ID for wordpress and tooling was updated
![image](https://user-images.githubusercontent.com/87030990/179394715-59f43b6f-64e4-44a0-912b-6ed5412b2b9c.png)

![image](https://user-images.githubusercontent.com/87030990/179394764-d62bfa66-e4b5-401b-a2e2-0dc121fa10fb.png)


Internal load balancer DNS for nginx reverse proxy also updated

![image](https://user-images.githubusercontent.com/87030990/179394514-12cf5fcd-77fd-4b2b-beb4-0ff7498ac92f.png)

**ansible.cfg** file was updated with the role path 

![image](https://user-images.githubusercontent.com/87030990/179394870-44749f94-4553-4829-83b6-e2d82962ea68.png)

The command **export ANSIBLE_CONFIG=<ansible.cfg path>** was ran to tell ansible-playbook to look at the role path 

````bash
export ANSIBLE_CONFIG=/home/ec2-user/ansible-deploy-pbl-19/ansible.cfg (<ansible.cfg path>)
````
![image](https://user-images.githubusercontent.com/87030990/179395253-af88e837-46ad-428f-b825-46ef9ade2ac0.png)

Ansible-playbook command was then ran to configure the instances

````bash
ansible-playbook -i inventory/aws-ec2.yml playbooks/site.yml
````
![image](https://user-images.githubusercontent.com/87030990/179395267-14369d16-bba0-46e2-8c4e-943a6470d380.png)
![image](https://user-images.githubusercontent.com/87030990/179395286-88b97cd6-9380-4d74-ae33-a12c51d348ef.png)
![image](https://user-images.githubusercontent.com/87030990/179395301-efeadb52-a97d-4fc0-b4b2-f2185256b204.png)

#### Step 5: Verify that the resources are properly configured

After configuring our resources, we **ssh** to each of the resources configured using ansible-playbook to confirm that the resources were correctly configured and make neccessary adjustment if need be

Logon to the bastion server in order to access nginx and the webservers for confirmtion of status and configuration

![image](https://user-images.githubusercontent.com/87030990/179396133-a9ef509b-d1a5-4e1d-a6cb-5e6a3f9ff7d7.png)

From the bastion server, run **ssh ec2-user@<private_ip_address_of_nginx>** to access the nginx

run the following commands to check the status and the configurationn

````bash
sudo systemctl status nginx (to check the status of nginx)
sudo vi /etc/nginx/nginx.conf ( to confirm that nginx is configured properly)
````
![image](https://user-images.githubusercontent.com/87030990/179396215-4182bccb-d330-4bcc-8c57-8595bd8a24b0.png)

![image](https://user-images.githubusercontent.com/87030990/179396260-c236a978-521f-49fd-be3d-9c23e10c4b74.png)

Exit nginx and logon to the tooling from the bastion server, run **ssh ec2-user@<private_ip_address_of_tooling>** to access the nginx

````bash
df -h (to check for successful mounting)
sudo systemctl status httpd (to check the status of apache server)
cd /var/www/html
ls (to list the files and sub directories)
curl localhost (to confirm locally that the website is running)
````
![image](https://user-images.githubusercontent.com/87030990/179396313-09655866-3305-440d-a32f-fad6e650ec01.png)

![image](https://user-images.githubusercontent.com/87030990/179396337-be6fe481-c58b-4be7-a008-4d7c9fa2e855.png)

![image](https://user-images.githubusercontent.com/87030990/179396371-88d8d4b3-ea95-4234-a6c2-0fa83b2e7a32.png)

Exit tooling and logon to the wordpress from the bastion server, run **ssh ec2-user@<private_ip_address_of_wordpress>** to access the nginx

````bash
df -h (to check for successful mounting)
sudo systemctl status httpd (to check the status of apache server)
cd /var/www/html
ls (to list the files and sub directories)
curl -v localhost (to confirm locally that the website is running)
````
![image](https://user-images.githubusercontent.com/87030990/179396430-2f44ba27-010f-4fda-b424-585e1db13e87.png)

![image](https://user-images.githubusercontent.com/87030990/179396457-663e6370-3f22-419f-a72f-dd4d583ab00d.png)

When curl -v localhost was ran, an error with the message below was received.

</head>
<body id="error-page".
      "<div class="wp-die-message"><h1>Error establishing a database connection</h1></div></body> 
</html=
* closing connection 0
                                  
This was corrected by updating the database_name, database_user, database_password and database_host in the **wp-config.php** file in the html directory by running **sudo vi wp-config.php** and then run curl -v localhost again
                                  
````bash
sudo vi wp-config.php
````
                                  
![image](https://user-images.githubusercontent.com/87030990/179397101-60b34e4f-7ceb-4ed6-9e50-d28a0db310d8.png)
                               
#### Step 6: Uncomment the listeners from the ALB file and the attachments for webservers in the Autoscaling group files 
                 
![image](https://user-images.githubusercontent.com/87030990/179397631-0bbfd841-c21d-47e6-acf3-26d6bd77d2b9.png)
![image](https://user-images.githubusercontent.com/87030990/179397660-5d452a8d-0821-4fc5-9644-ca1bc62f572d.png)
![image](https://user-images.githubusercontent.com/87030990/179397686-bfcc1676-01d3-477b-b173-fa69d6185cf5.png)
![image](https://user-images.githubusercontent.com/87030990/179397709-214155e4-b63a-4522-910b-c8a0226b16af.png)
![image](https://user-images.githubusercontent.com/87030990/179397719-696c6e82-0229-41d8-a3ba-135254c9784d.png)

Changes to the code was saved, added, committed and pushed to GitLab to trigger updating of the resources

![image](https://user-images.githubusercontent.com/87030990/179397869-a322ddc7-a826-4bea-afed-78bd902bc31f.png)
![image](https://user-images.githubusercontent.com/87030990/179397902-fd48936a-2477-413c-83ee-6d35554394a5.png)


                                  
