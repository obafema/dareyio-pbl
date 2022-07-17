### AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 4 – TERRAFORM CLOUD

Objective: To migrate the previously written terraform codes for creating AWS Cloud Solution for 2 company websites using a reverse proxy technology to Terraform cloud and managing our AWS Infrastructure from there.

Tasks
- Create Amazon Machine Images (AMIs) for our instances using Packer and define our shell script to create the required instances inside it.
- Migrate our terraform codes to Terraform Cloud and provision our infrastructure from there.
- Use GitLab to run Terraform commands triggered from our git repository.
- Configure our infrastructire using Ansible

#### Step 1: Migrate your terraform codes to Terraform Cloud

A repository named Terraform-cloud was created in Gitlab and the previously developed terraform codes pushed into it for connection to the version control workflow

![image](https://user-images.githubusercontent.com/87030990/179385759-65b3e29c-276c-480b-bb11-8d0c89c9d7ef.png)

Terraform Cloud account and an organisation were created. Also configured was a workspace named terraform-cloud with version control workflow to run Terraform commands triggered from our git repository.

![image](https://user-images.githubusercontent.com/87030990/179385493-064cd907-027a-4b0c-a843-d8752c9bc960.png)


