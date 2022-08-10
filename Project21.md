# ORCHESTRATING CONTAINERS ACROSS MULTIPLE VIRTUAL SERVERS WITH KUBERNETES. PART 1

**Objective: ** 

**Prerequisites:**
Install Client tools before bootstrapping the Cluster

* **awscli** – is a unified tool to manage your AWS services
* **kubectl** – this command line utility will be your main control tool to manage your K8s cluster.
* **cfssl** – an open source toolkit for everything **TLS/SSL** from Cloudflare
* **cfssljson** – a program, which takes the JSON output from the cfssl and writes **certificates**, **keys**, **CSRs**, and bundles to disk.

Install and configure AWS CLI

A user with programmable access keys was configure on AWS Identity and Access Management (IAM) for access all AWS services used

Latest version of AWS CLI was downloaded, installed and configured using aws configure command and the appropriate keys, region and output format provided

````bash
 aws configure --profile %your_username%
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-2
Default output format [None]: json
````

Test your AWS CLI by running:

![image](https://user-images.githubusercontent.com/87030990/183634120-4f6cefb6-3356-4221-99c8-d065eadb057a.png)


Install kubectl

![image](https://user-images.githubusercontent.com/87030990/183635398-6a31fa73-fde8-44ac-802d-f3945cce3352.png)

Install CFSSL and CFSSLJSON

**cfssl** was configured as a **Certificate Authority** which will issue the certificates required to spin up a Kubernetes cluster.

![image](https://user-images.githubusercontent.com/87030990/183637075-f7cefc34-597b-4a3c-94ca-d7d76d3bd5e9.png)


### Step 1 – Configure Network Infrastructure for Kubernetes Cluster

AWS CLI was used in creating the AWS Resources for our implementation

Virtual Private Cloud – VPC

A directory named **k8s-cluster-from-ground-up** was created

![image](https://user-images.githubusercontent.com/87030990/183642054-7bbc6b4b-5408-4442-8b8f-ab526a1e6936.png)

VPC was created within **k8s-cluster-from-ground-up** folder and the ID stored as a variable for subsequent use:

````bask
VPC_ID=$(aws ec2 create-vpc \
--cidr-block 172.31.0.0/16 \
--output text --query 'Vpc.VpcId'
)
````

Tag the VPC so that it is named:

````bash
NAME=k8s-cluster-from-ground-up

aws ec2 create-tags \
  --resources ${VPC_ID} \
  --tags Key=Name,Value=${NAME}
````

![image](https://user-images.githubusercontent.com/87030990/183646972-22c70a36-22bd-436b-a7bc-d1567bfb4f87.png)

Domain Name System – DNS

DNS support for the VPC was enabled and set to true

````bash
aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-hostnames '{"Value": true}'
````
DNS support for hostnames was also enabled and set to true
````bash
aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-hostnames '{"Value": true}'
````
![image](https://user-images.githubusercontent.com/87030990/183650452-3d019a94-b9e1-469e-adff-c5f3ddb2dfde.png)

AWS Region

The required region was set

````bash
AWS_REGION=<region>
````

Dynamic Host Configuration Protocol – DHCP

DHCP Options Set was configured:

N.B: AWS automatically creates and associates a DHCP option set for your Amazon VPC upon creation and sets two options: domain-name-servers (defaults to AmazonProvidedDNS) and domain-name (defaults to the domain name for your set region). AmazonProvidedDNS is an Amazon Domain Name System (DNS) server, and this option enables DNS for instances to communicate using DNS names

Although by default EC2 instances have fully qualified names but for this implimentation the DHCP options were set and appropriately tagged as below

````bash
DHCP_OPTION_SET_ID=$(aws ec2 create-dhcp-options \
  --dhcp-configuration \
    "Key=domain-name,Values=$AWS_REGION.compute.internal" \
    "Key=domain-name-servers,Values=AmazonProvidedDNS" \
  --output text --query 'DhcpOptions.DhcpOptionsId')
  ````
  
  ````bash
  aws ec2 create-tags \
  --resources ${DHCP_OPTION_SET_ID} \
  --tags Key=Name,Value=${NAME}
  ````
  
  ![image](https://user-images.githubusercontent.com/87030990/183656843-ec7156be-0201-484b-9892-83d490bcadb1.png)

The DHCP Option set was associated with the VPC

````bash
aws ec2 associate-dhcp-options \
  --dhcp-options-id ${DHCP_OPTION_SET_ID} \
  --vpc-id ${VPC_ID}
  ````
  
  ![image](https://user-images.githubusercontent.com/87030990/183666727-c8819260-0f41-4d4b-866a-adf9eb0d5a0f.png)


Subnet

Subnet were created within the VPC and appropriately tagged:

````bash
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 172.31.0.0/24 \
  --output text --query 'Subnet.SubnetId')
  ````
  
  ````bash
  aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=${NAME}
  ````
  
  ![image](https://user-images.githubusercontent.com/87030990/183667787-a19bc066-83a2-4c9b-8ddb-18daea3c78c2.png)


Internet Gateway – IGW

The Internet Gateway was created, appropriately tagged and attached to the VPC:

````bash
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway \
  --output text --query 'InternetGateway.InternetGatewayId')
aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=${NAME}
  aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}
  ````

  ![image](https://user-images.githubusercontent.com/87030990/183671442-e0dbe233-cdaf-487b-b8d8-18d92048ab28.png)

Route tables

Create route tables, associate the route table to subnet, and create a route to allow external traffic to the Internet through the Internet Gateway:

````bash
ROUTE_TABLE_ID=$(aws ec2 create-route-table \
  --vpc-id ${VPC_ID} \
  --output text --query 'RouteTable.RouteTableId')
aws ec2 create-tags \
  --resources ${ROUTE_TABLE_ID} \
  --tags Key=Name,Value=${NAME}
aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID}
aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${INTERNET_GATEWAY_ID}
  ````
  
  ![image](https://user-images.githubusercontent.com/87030990/183672376-601d042e-ae93-47cc-b725-5f4150c2956b.png)

![image](https://user-images.githubusercontent.com/87030990/183672636-de1f6ffc-a012-487b-be5d-e790df9095d5.png)


Security Groups

Security groups were configured:

````bash
# Create the security group and store its ID in a variable
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name ${NAME} \
  --description "Kubernetes cluster security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')

# Create the NAME tag for the security group
aws ec2 create-tags \
  --resources ${SECURITY_GROUP_ID} \
  --tags Key=Name,Value=${NAME}

# Create Inbound traffic for all communication within the subnet to connect on ports used by the master node(s)
aws ec2 authorize-security-group-ingress \
    --group-id ${SECURITY_GROUP_ID} \
    --ip-permissions IpProtocol=tcp,FromPort=2379,ToPort=2380,IpRanges='[{CidrIp=172.31.0.0/24}]'

# # Create Inbound traffic for all communication within the subnet to connect on ports used by the worker nodes
aws ec2 authorize-security-group-ingress \
    --group-id ${SECURITY_GROUP_ID} \
    --ip-permissions IpProtocol=tcp,FromPort=30000,ToPort=32767,IpRanges='[{CidrIp=172.31.0.0/24}]'

# Create inbound traffic to allow connections to the Kubernetes API Server listening on port 6443
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 6443 \
  --cidr 0.0.0.0/0

# Create Inbound traffic for SSH from anywhere (Do not do this in production. Limit access ONLY to IPs or CIDR that MUST connect)
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# Create ICMP ingress for all types
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol icmp \
  --port -1 \
  --cidr 0.0.0.0/0
  ````
  
  ![image](https://user-images.githubusercontent.com/87030990/183675456-7a4a4677-c301-4297-a7ab-4a331e758ce1.png)


Network Load Balancer

Create a network Load balancer,

````bash
LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
--name ${NAME} \
--subnets ${SUBNET_ID} \
--scheme internet-facing \
--type network \
--output text --query 'LoadBalancers[].LoadBalancerArn')
````

![image](https://user-images.githubusercontent.com/87030990/183676161-53628b9b-3cbb-4a87-8dec-383b0db8639a.png)

Tagret Group

Create a target group: (For now it will be unhealthy because there are no real targets yet.)

````bash
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name ${NAME} \
  --protocol TCP \
  --port 6443 \
  --vpc-id ${VPC_ID} \
  --target-type ip \
  --output text --query 'TargetGroups[].TargetGroupArn')
  ````
  ![image](https://user-images.githubusercontent.com/87030990/183677656-8c1a6888-0542-4843-9f74-4b92a0b4a07a.png)

Register targets: (Just like above, no real targets. You will just put the IP addresses so that, when the nodes become available, they will be used as targets.)

````bash
aws elbv2 register-targets \
  --target-group-arn ${TARGET_GROUP_ARN} \
  --targets Id=172.31.0.1{0,1,2}
  ````
 
 ![image](https://user-images.githubusercontent.com/87030990/183678168-4cde90c4-0cc4-4e17-b052-37ef4eb36dbd.png)


Create a listener to listen for requests and forward to the target nodes on TCP port 6443

````bash
aws elbv2 create-listener \
--load-balancer-arn ${LOAD_BALANCER_ARN} \
--protocol TCP \
--port 6443 \
--default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
--output text --query 'Listeners[].ListenerArn'
````

![image](https://user-images.githubusercontent.com/87030990/183678778-ed72d684-331f-47d6-96cb-850a766477d4.png)


K8s Public Address

Get the Kubernetes Public address needed for the Load Balancer

````bash
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
--load-balancer-arns ${LOAD_BALANCER_ARN} \
--output text --query 'LoadBalancers[].DNSName')
````

![image](https://user-images.githubusercontent.com/87030990/183688957-3b0b7851-3380-4978-adf3-3466dbf19012.png)


### Step 2 – Create Compute Resources

AMI

Get an image to create EC2 instances:

![image](https://user-images.githubusercontent.com/87030990/183704138-7b4f6d91-efed-452d-a851-5590acf78283.png)

![image](https://user-images.githubusercontent.com/87030990/183704433-195c04e0-d5e2-499d-8818-5708d21675f1.png)


SSH key-pair

Create SSH Key-Pair

````bash
mkdir -p ssh

aws ec2 create-key-pair \
  --key-name ${NAME} \
  --output text --query 'KeyMaterial' \
  > ssh/${NAME}.id_rsa
chmod 600 ssh/${NAME}.id_rsa
````

![image](https://user-images.githubusercontent.com/87030990/183754089-1f0180bc-6ea6-4e14-8b3e-b631ba23bd12.png)


EC2 Instances for Controle Plane (Master Nodes)

Create 3 Master nodes: Note – Using t2.micro instead of t2.small as t2.micro is covered by AWS free tier

````bash
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name ${NAME} \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 172.31.0.1${i} \
    --user-data "name=master-${i}" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=${NAME}-master-${i}"
done
````

![image](https://user-images.githubusercontent.com/87030990/183761142-4c14d1a8-44b4-4ea9-a457-b96ec3962c93.png)

EC2 Instances for Worker Nodes

Create 3 worker nodes:

````bash
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name ${NAME} \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 172.31.0.2${i} \
    --user-data "name=worker-${i}|pod-cidr=172.20.${i}.0/24" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=${NAME}-worker-${i}"
done
````



### Step 3 Prepare The Self-Signed Certificate Authority And Generate TLS Certificates




