### BUILDING ELASTIC KUBERNETES SERVICE (EKS) WITH TERRAFORM

**Objective:** To create EKS Cluster and get it up and running using Terraform.

### Task

* Create a Kubernetes EKS cluster using Terraform and dynamically add scalable worker nodes
* Deploy multiple applications using HELM


### Step 1: Configuring The Terraform Module For EKS

* A directory named **eks** was created and cd into

* **AWS S3 bucket** was created using AWS CLI in order to store the Terraform state: 

````aws s3api create-bucket \
    --bucket eks-terraform-s3bucket \
    --region us-east-2 \
    --create-bucket-configuration LocationConstraint=us-east-2
````
*  **backend.tf** file was created to configure the backend for remote state in S3 

````
#############################
##creating bucket for s3 backend
#########################
resource "aws_s3_bucket" "terraform_state" {
  bucket = "eks-terraform-s3bucket"

force_destroy = true
}

resource "aws_s3_bucket_versioning" "versioning_terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.bucket

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}

# terraform {
#   backend "s3" {
#     bucket         = "eks-terraform-s3bucket"
#     key            = "global/s3/terraform.tfstate"
#     region         = "us-east-2"
#     dynamodb_table = "terraform-locks"
#     encrypt        = true
#   }
# }
````

* **network.tf** file was created to provision Elastic IP for Nat Gateway, VPC, Private and public subnets.

````
# reserve Elastic IP to be used in our NAT gateway
resource "aws_eip" "nat_gw_elastic_ip" {
vpc = true

tags = {
Name            = "${var.cluster_name}-nat-eip"
iac_environment = var.iac_environment_tag
}
}

module "vpc" {
source  = "terraform-aws-modules/vpc/aws"

name = "${var.name_prefix}-vpc"
cidr = var.main_network_block
azs  = data.aws_availability_zones.available_azs.names

private_subnets = [
# this loop will create a one-line list as ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20", ...]
# with a length depending on how many Zones are available
for zone_id in data.aws_availability_zones.available_azs.zone_ids :
cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) - 1)
]

public_subnets = [
# this loop will create a one-line list as ["10.0.128.0/20", "10.0.144.0/20", "10.0.160.0/20", ...]
# with a length depending on how many Zones are available
# there is a zone Offset variable, to make sure no collisions are present with private subnet blocks
for zone_id in data.aws_availability_zones.available_azs.zone_ids :
cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) + var.zone_offset - 1)
]

# Enable single NAT Gateway to save some money
# WARNING: this could create a single point of failure, since we are creating a NAT Gateway in one AZ only
# feel free to change these options if you need to ensure full Availability without the need of running 'terraform apply'
# reference: https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/2.44.0#nat-gateway-scenarios
enable_nat_gateway     = true
single_nat_gateway     = true
one_nat_gateway_per_az = false
enable_dns_hostnames   = true
reuse_nat_ips          = true
external_nat_ip_ids    = [aws_eip.nat_gw_elastic_ip.id]

# Add VPC/Subnet tags required by EKS
tags = {
"kubernetes.io/cluster/${var.cluster_name}" = "shared"
iac_environment                             = var.iac_environment_tag
}
public_subnet_tags = {
"kubernetes.io/cluster/${var.cluster_name}" = "shared"
"kubernetes.io/role/elb"                    = "1"
iac_environment                             = var.iac_environment_tag
}
private_subnet_tags = {
"kubernetes.io/cluster/${var.cluster_name}" = "shared"
"kubernetes.io/role/internal-elb"           = "1"
iac_environment                             = var.iac_environment_tag
}
}
````

* **variables.tf** was created store variables

````
# create some variables
variable "cluster_name" {
type        = string
description = "EKS cluster name."
}
variable "iac_environment_tag" {
type        = string
description = "AWS tag to indicate environment name of each infrastructure object."
}
variable "name_prefix" {
type        = string
description = "Prefix to be used on each infrastructure object Name created in AWS."
}
variable "main_network_block" {
type        = string
description = "Base CIDR block to be used in our VPC."
}
variable "subnet_prefix_extension" {
type        = number
description = "CIDR block bits extension to calculate CIDR blocks of each subnetwork."
}
variable "zone_offset" {
type        = number
description = "CIDR block bits extension offset to calculate Public subnets, avoiding collisions with Private subnets."
}
````

* **data.tf** file was created to pull the available AZs for use.

````
# get all available AZs in our region
data "aws_availability_zones" "available_azs" {
state = "available"
}
data "aws_caller_identity" "current" {} # used for accesing Account ID and ARN
````

* **eks.tf** file was developed to create EKS Cluster

````
module "eks_cluster" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 18.0"
  cluster_name    = var.cluster_name
  cluster_version = "1.22"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access = true

  # Self Managed Node Group(s)
  self_managed_node_group_defaults = {
    instance_type                          = var.asg_instance_types[0]
    update_launch_template_default_version = true
  }
  self_managed_node_groups = local.self_managed_node_groups

  # aws-auth configmap
  create_aws_auth_configmap = true
  manage_aws_auth_configmap = true
  aws_auth_users = concat(local.admin_user_map_users, local.developer_user_map_users)
  tags = {
    Environment = "prod"
    Terraform   = "true"
  }
}
````

* **local.tf** was developed to create local variables as Terraform does not allow assigning variable to variables

````
# render Admin & Developer users list with the structure required by EKS module
locals {
  admin_user_map_users = [
    for admin_user in var.admin_users :
    {
      userarn  = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/${admin_user}"
      username = admin_user
      groups   = ["system:masters"]
    }
  ]
  developer_user_map_users = [
    for developer_user in var.developer_users :
    {
      userarn  = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/${developer_user}"
      username = developer_user
      groups   = ["${var.name_prefix}-developers"]
    }
  ]

  self_managed_node_groups = {
    worker_group1 = {
      name = "${var.cluster_name}-wg"

      min_size      = var.autoscaling_minimum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      desired_size      = var.autoscaling_minimum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      max_size  = var.autoscaling_maximum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      instance_type = var.asg_instance_types[0].instance_type

      bootstrap_extra_args = "--kubelet-extra-args '--node-labels=node.kubernetes.io/lifecycle=spot'"

      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            delete_on_termination = true
            encrypted             = false
            volume_size           = 10
            volume_type           = "gp2"
          }
        }
      }

      use_mixed_instances_policy = true
      mixed_instances_policy = {
        instances_distribution = {
          spot_instance_pools = 4
        }

        override = var.asg_instance_types
      }
    }
  }
}
````

* Add more variables to variables.tf file

````
# create some variables
variable "admin_users" {
  type        = list(string)
  description = "List of Kubernetes admins."
}
variable "developer_users" {
  type        = list(string)
  description = "List of Kubernetes developers."
}
variable "asg_instance_types" {
  description = "List of EC2 instance machine types to be used in EKS."
}
variable "autoscaling_minimum_size_by_az" {
  type        = number
  description = "Minimum number of EC2 instances to autoscale our EKS cluster on each AZ."
}
variable "autoscaling_maximum_size_by_az" {
  type        = number
  description = "Maximum number of EC2 instances to autoscale our EKS cluster on each AZ."
}
````

* **terraform.tfvars** was created to set values for variables

````
cluster_name            = "tooling-app-eks"
iac_environment_tag     = "development"
name_prefix             = "obafema-io-eks"
main_network_block      = "10.0.0.0/16"
subnet_prefix_extension = 4
zone_offset             = 8

# Ensure that these users already exist in AWS IAM. Another approach is that you can introduce an iam.tf file to manage users separately, get the data source and interpolate their ARN.
admin_users                    = ["obafema", "osla"]
developer_users                = ["zigali", "moyo"]
asg_instance_types             = [ { instance_type = "t3.small" }, { instance_type = "t2.small" }, ]
autoscaling_minimum_size_by_az = 1
autoscaling_maximum_size_by_az = 10
````

* **provider.tf** file was created

````
provider "aws" {
  region = "us-west-1"
}

provider "random" {
}
````


* Run **terraform init** after developing all the **.tf** files

![image](https://user-images.githubusercontent.com/87030990/191542608-afcc0485-981f-4ca7-bac1-f15f5f4425a9.png)


* Running the terraform apply command will cause the following the error at some point while creating the resources, that is because for us to connect to the cluster using the kubeconfig, Terraform needs to be able to connect and set the credentials correctly:

![image](https://user-images.githubusercontent.com/87030990/191562294-74f1a4f1-81f1-47b9-abbb-e3d9efe15b45.png)


* To fixed the error the following configuration is added in the data.tf and provider.tf file:

**data.tf**
````
# get EKS cluster info to configure Kubernetes and Helm providers
data "aws_eks_cluster" "cluster" {
  name = module.eks_cluster.cluster_id
}
data "aws_eks_cluster_auth" "cluster" {
  name = module.eks_cluster.cluster_id
}
````

**provider.tf**

````
# get EKS authentication for being able to manage k8s objects from terraform
provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}
````

* Then recreate the resources.

![image](https://user-images.githubusercontent.com/87030990/191564666-0692e3b4-d59c-4285-b088-6fa39bdc8fd2.png)

![image](https://user-images.githubusercontent.com/87030990/191565064-138a61c7-dab4-4f55-8791-8b706bfe7059.png)


* After creating the resources first, then uncomment the backend section,re-initialise terraform and apply again

 terraform {
   backend "s3" {
     bucket         = "eks-terraform-s3bucket"
     key            = "global/s3/terraform.tfstate"
     region         = "us-east-2"
     dynamodb_table = "terraform-locks"
     encrypt        = true
   }
 }

* Create kubeconfig file using awscli: ````aws eks update-kubecofig --name tooling-app-eks --region us-east-2 --kubeconfig kubeconfig````

![image](https://user-images.githubusercontent.com/87030990/191970356-989e176e-90d3-4e45-b54d-99b5377e61b7.png)

* While trying to connect to the cluster to check nodes, I gotthe error below:

![image](https://user-images.githubusercontent.com/87030990/192008909-63ab722e-1c65-47ca-93d4-344478342d59.png)

* The kubeconfig file for the cluster was updated using: ````aws eks --region us-east-2 update-kubeconfig --name tooling-app-eks````

![image](https://user-images.githubusercontent.com/87030990/192009507-cf35a795-a05b-4058-8afb-7d6b6604ee9e.png)

* A namespace called obafema was created for the implementation and connection to the cluster verified with: ````kubectl get nodes

![image](https://user-images.githubusercontent.com/87030990/192011010-d007268d-a4af-489f-a4f4-9a0202bad842.png)
![image](https://user-images.githubusercontent.com/87030990/192011118-afb260ba-3368-4253-9ef8-51a017d17e7f.png)

### Step 2: Installing Helm From Script

````
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 # Fetching the script:
chmod 700 get_helm.sh # Changing the permission of the script
./get_helm.sh # Executing the script
````

![image](https://user-images.githubusercontent.com/87030990/191601857-b634a437-f6cb-42a9-86c3-f757c15927fd.png)

### Step 3: Deploying Jenkins With Helm

* Visit **Artifact Hub** to find packaged applications as Helm Charts. Search for Jenkins

* Adding the Jenkins' repository to helm so it can be easily downloaded and deployed: ````helm repo add jenkins https://charts.jenkins.io````

* Updating helm repo: ````helm repo update````

![image](https://user-images.githubusercontent.com/87030990/191605169-1618178c-67ef-4b8a-b2aa-7873d86ac354.png)


* Install the chart: ````helm install jenkins jenkins/jenkins --kubeconfig kubeconfig -n obafema````

![image](https://user-images.githubusercontent.com/87030990/191608248-6cf18df5-faae-4504-881d-db6ae7ed421b.png)
![image](https://user-images.githubusercontent.com/87030990/192011730-77e1aff8-29d2-4a56-8def-ba3f6d14e6f1.png)


* Check the Helm deployment: ````helm ls --kubeconfig kubeconfig -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192013626-0adb04b5-35a3-4c80-9953-0f2be0650758.png)

* Check the pods: ````kubectl get pods --kubeconfig kubeconfig -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192014008-20a8261c-3b1f-441d-859b-0eff390a8f08.png)


* Describe the running pod: ````kubectl describe pod jenkins-0 --kubeconfig kubeconfig -n obafema````
* 
![image](https://user-images.githubusercontent.com/87030990/192019174-ad85e9d7-87fb-4740-8cd5-02d2f758a5e7.png)

* Check the logs of the running pod: ````kubectl logs jenkins-0 --kubeconfig kubeconfig -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192019868-4a614c23-85c5-47c6-a742-4ac80d3c3904.png)

Note: This is because the pod has a Sidecar container alongside with the Jenkins container called **config-reload**. The config-reload is mainly to help Jenkins to reload its configuration without recreating the pod.

* So, we specify the container we what to check its log: ````kubectl logs jenkins-0 -c jenkins --kubeconfig kubeconfig -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192022982-c8e6b785-c82d-48fd-8bfb-75e23ae438a3.png)

* In order to run the kubectl commands without specifying the kubeconfig file, a package manager for kubectl called krew is installed so that it will enable us to install plugins to extend the functionality of kubectl:

* Make sure that **git** is installed.

* Run this command to download and install **krew**:

````
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
````
* Add the $HOME/.krew/bin directory to your PATH environment variable. To do this, update your .bashrc or .zshrc file and append the following line:

````export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"````

![image](https://user-images.githubusercontent.com/87030990/192024851-0eaddf94-2cdc-487a-a234-f2172843e768.png)

* Installing Konfig plugin: ````kubectl krew install konfig````

![image](https://user-images.githubusercontent.com/87030990/192025847-c8f61bf2-1b0f-42a4-b209-ee9ec5bf1768.png)

* Import the kubeconfig into the default kubeconfig file: ````

* To show all the contexts for the clusters configured in my kubeconfig. ````kubectl config get-contexts````

![image](https://user-images.githubusercontent.com/87030990/192028513-fd84db27-34cb-4b1f-a481-bef7a99e1afa.png)


* Set the current context to use for all kubectl and helm commands: ````kubectl config use-context [name of EKS cluster]````

![image](https://user-images.githubusercontent.com/87030990/192029659-720c2ed5-be0b-4696-8023-577f79ba1327.png)


* Test that it is working without specifying the --kubeconfig flag: ```` kubectl get po -n obafema````

* Display the current context:````kubectl config current-context -n obafema````. This will let you know the context you are using to interact with Kubernetes

![image](https://user-images.githubusercontent.com/87030990/192030377-e7ba77dd-243e-49cc-956f-4747e3425c2e.png)

* To get the Jenkins administrator's password credential using the command provided on the screen when Jenkins was installed with Helm: 
* ````kubectl exec --namespace obafema -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo````


* Use port forwarding to access Jenkins from the UI

![image](https://user-images.githubusercontent.com/87030990/192038545-7e6575a1-8982-40b8-86a0-6bcbe53e05ed.png)


* Go to the browser localhost:8080 and authenticate with the username and password from number 1 above

![image](https://user-images.githubusercontent.com/87030990/192038332-a2f7e86b-7f27-4856-8ca0-5fa63791ee0b.png)


### Step 4: Deploying Artifactory With Helm

* Adding the Artifactory's repository to helm: ````helm repo add jfrog https://charts.jfrog.io````

* Updating helm repo: ````helm repo update````

* To install the chart with the release name artifactory:````helm upgrade --install artifactory --namespace artifactory jfrog/artifactory````

![image](https://user-images.githubusercontent.com/87030990/192040890-7786f9ab-e33b-4a12-9847-836bc25e8646.png)


### Step 5: Deploying Hashicorp Vault With Helm

* Adding the Hashicorp's repository to helm: ````helm repo add hashicorp https://helm.releases.hashicorp.com````

* Installing the chart: ````helm install vault hashicorp/vault -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192041675-064187ed-e18d-445a-ab50-a918d1557f91.png)


* To check the installation, run get pods and service on the namespaces

![image](https://user-images.githubusercontent.com/87030990/192042740-b727bb48-b07f-496b-948d-6b07f0f6292c.png)

* Port forwarding to access Hashicorp vault from the UI: ````kubectl port-forward svc/vault 8089:8200 -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192043308-b9f1f38d-7f22-44e7-a495-26ab24e82d0f.png)


* Accessing the app from the browser: ````http://localhost:8089````

![image](https://user-images.githubusercontent.com/87030990/192043249-6e33c08e-6b58-47dd-bea6-4fed33265b34.png)


### Step 6: Deploying Prometheus With Helm

* Adding the prometheus's repository to helm: ````helm repo add prometheus-community https://prometheus-community.github.io/helm-charts -n obafema````
* Updating helm repo: ````helm repo update````

![image](https://user-images.githubusercontent.com/87030990/192047187-9fc979f9-3ab3-40b5-a3f5-71944ba4f20a.png)

* Installing the chart: ````helm install prometheus prometheus-community/prometheus -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192048068-186f6411-8795-48ed-a725-33fa1eff7685.png)


* Inspecting the installation shows that there are various pods and services created

* Port forwarding to access prometheus for alert manager from the UI: ````kubectl port-forward svc/prometheus-alertmanager 8000:80 -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192049216-aecbb118-be1b-4cb0-8249-d3998574ac89.png)

* Accessing the app from the browser: ````http://localhost:8000````

![image](https://user-images.githubusercontent.com/87030990/192049038-14e53118-9217-40c5-a1ed-8042c14a4bda.png)

* Port forwarding to access prometheus for kube state metrics from the UI: ````kubectl port-forward svc/prometheus-kube-state-metrics 8000:8080 -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192053880-db954049-e023-482c-9e80-18f769ee520e.png)

* Accessing the app from the browser: ```http://localhost:8000````

![image](https://user-images.githubusercontent.com/87030990/192053818-b501c939-d4d9-4ca1-b1ce-f08b7e32306f.png)

* Port forwarding to access prometheus for pushgateway from the UI: ````kubectl port-forward svc/prometheus-pushgateway 8000:9091 -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192056161-cf691720-d328-403c-a44d-04afbee995ca.png)


* Accessing the app from the browser: ```http://localhost:8000````

![image](https://user-images.githubusercontent.com/87030990/192056090-7e57eec1-88b2-405d-8aea-3794b665fcfc.png)

### Step 6: Deploying Grafana With Helm

* Adding the grafana's repository to helm: ````helm repo add grafana https://grafana.github.io/helm-charts````
* Updating helm repo: ````helm repo update````

![image](https://user-images.githubusercontent.com/87030990/192056823-34a529c9-e100-40ea-b71e-f77bdac78bb0.png)

* Installing the Chart: ````helm install my-grafana grafana/grafana -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192057241-0a0fde01-1b0e-4874-b23b-27c3b077224c.png)

* Port forwarding to access grafana from the UI: ````kubectl port-forward svc/my-grafana 8000:80 -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192057561-c87e2e9e-f00c-4e08-9cdf-d503de04ee32.png)


* Accessing the app from the browser: ```http://localhost:8000````

![image](https://user-images.githubusercontent.com/87030990/192057473-2779d997-2a97-4d8c-a328-9a3155d9b11f.png)

![image](https://user-images.githubusercontent.com/87030990/192057975-9a2a98b5-b31e-4b78-8b04-4feb92183c7c.png)

### Step 7: Deploying Elasticsearch With Helm

* Adding the grafana's repository to helm: ````hhelm repo add bitnami https://charts.bitnami.com/bitnami````
* Updating helm repo: ````helm repo update````

![image](https://user-images.githubusercontent.com/87030990/192058863-fa3f9590-ac7a-4873-93ef-3bfeb9251f8b.png)

* Installing the Chart: ````helm install my-release bitnami/elasticsearch -n obafema````

![image](https://user-images.githubusercontent.com/87030990/192059263-b75667e1-d790-46a3-9f47-fbc52c16b844.png)


