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
  
  Each module shall contain following files:
  
- main.tf (or %resource_name%.tf) file(s) with resources blocks
- outputs.tf (optional, if you need to refer outputs from any of these resources in your root module)
- variables.tf (as we learned before - it is a good practice not to hard code the values and use variables)

