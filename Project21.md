# ORCHESTRATING CONTAINERS ACROSS MULTIPLE VIRTUAL SERVERS WITH KUBERNETES. PART 1

**Objective:** To setup Kubernetes cluster manually from the scratch without any automation for better understanding of the components and stages involve in setting up a cluster and how they are interrelated.

### **Prerequisites:**

The following Client tools were installed before bootstrapping the Cluster

* **awscli** – A unified tool to provision and manage the AWS services (**aws-cli/2.2.41**)
* **kubectl** – Main control tool to manage the K8s cluster (**v1.24.2**).
* **cfssl** – An open source toolkit by Cloudflare. Used to setup a Public Key Infrastructure(PKI) for generating, signing and bundling TLS/SSL certificates
* **cfssljson** – A program, which takes the JSON output from the cfssl and writes **certificates**, **keys**, **CSRs**, and bundles to disk.


* Install and configure AWS CLI

A user with programmable access was configure on AWS Identity and Access Management (IAM) for access to all AWS services used

AWS CLI was downloaded, installed and configured using aws configure command and the appropriate keys, region and output format provided

````
 aws configure --profile %your_username%
AWS Access Key ID [None]: <Access_Key_ID>
AWS Secret Access Key [None]: <secret_key>
Default region name [None]: us-west-2
Default output format [None]: json
````

AWS CLI was tested for proper configuration was by running: ````aws ec2 describe-vpcs````

![image](https://user-images.githubusercontent.com/87030990/183634120-4f6cefb6-3356-4221-99c8-d065eadb057a.png)


* Install kubectl on Local System

Kubectl version (1.24.2) included in earlier installed Docker Desktop installation was used for this project

![image](https://user-images.githubusercontent.com/87030990/183635398-6a31fa73-fde8-44ac-802d-f3945cce3352.png)

**N.B:** You can however download the binary and follow the steps below:

Downloading the binary: ````$ wget https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl````

Making it executable: ````$ chmod +x kubectl````

Moving the file to the Bin directory: ````$ sudo mv kubectl /usr/local/bin/````

Verifying that kubectl version 1.21.0 or higher is installed: ````$ kubectl version --client````


* Install CFSSL and CFSSLJSON

**cfssl** was configured as a **Certificate Authority** which will issue the certificates required to spin up the Kubernetes cluster.

The binary was downloaded
````
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
````
Made executable: ````chmod +x kubectl````

And moved to the Bin directory: ````sudo mv kubectl /usr/local/bin/````

![image](https://user-images.githubusercontent.com/87030990/183637075-f7cefc34-597b-4a3c-94ca-d7d76d3bd5e9.png)



### Step 1 – Configure Network Infrastructure for Kubernetes Cluster


AWS CLI was used in creating the AWS Resources for our implementation

A directory named **k8s-cluster-from-ground-up** was created and it was cd into: ````$ mkdir k8s-cluster-from-ground-up && cd k8s-cluster-from-ground-up````

![image](https://user-images.githubusercontent.com/87030990/183642054-7bbc6b4b-5408-4442-8b8f-ab526a1e6936.png)


* Virtual Private Cloud – VPC

VPC was created within **k8s-cluster-from-ground-up** folder and the ID stored as a variable for subsequent use:

````
VPC_ID=$(aws ec2 create-vpc \
--cidr-block 172.31.0.0/16 \
--output text --query 'Vpc.VpcId'
)
````

The VPC was appropriately tagged:

````
NAME=k8s-cluster-from-ground-up

aws ec2 create-tags \
  --resources ${VPC_ID} \
  --tags Key=Name,Value=${NAME}
````

![image](https://user-images.githubusercontent.com/87030990/183646972-22c70a36-22bd-436b-a7bc-d1567bfb4f87.png)

* Domain Name System – DNS

DNS support for the VPC was enabled and set to true

````
aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-hostnames '{"Value": true}'
````

DNS support for hostnames was also enabled and set to true

````
aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-hostnames '{"Value": true}'
````

![image](https://user-images.githubusercontent.com/87030990/183650452-3d019a94-b9e1-469e-adff-c5f3ddb2dfde.png)



* AWS Region

The required region was set and the value set as a variable

````
AWS_REGION=us-east-2
````

* Dynamic Host Configuration Protocol – DHCP

DHCP Options Set was configured:

**N.B:** AWS automatically creates and associates a **DHCP option set** for your Amazon VPC upon creation and sets two options: domain-name-servers (defaults to AmazonProvidedDNS) and domain-name (defaults to the domain name for your set region). AmazonProvidedDNS is an Amazon Domain Name System (DNS) server, and this option enables DNS for instances to communicate using DNS names.

Although by default EC2 instances have fully qualified names but for this implimentation the DHCP options were set and tagged as below

* Configuring DHCP Options Set:
* 
````
DHCP_OPTION_SET_ID=$(aws ec2 create-dhcp-options \
  --dhcp-configuration \
    "Key=domain-name,Values=$AWS_REGION.compute.internal" \
    "Key=domain-name-servers,Values=AmazonProvidedDNS" \
  --output text --query 'DhcpOptions.DhcpOptionsId')
  ````

* Tagging

  ````
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


* Subnet

Subnet were created within the VPC and appropriately tagged:

````
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 172.31.0.0/24 \
  --output text --query 'Subnet.SubnetId')
  ````
  
  ````
  aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=${NAME}
  ````
  
  ![image](https://user-images.githubusercontent.com/87030990/183667787-a19bc066-83a2-4c9b-8ddb-18daea3c78c2.png)


* Internet Gateway – IGW

The Internet Gateway was created, appropriately tagged and attached to the VPC:

````
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


* Route tables

Route table was created, associated to the subnet, and a route to allow external traffic to the Internet created through the Internet Gateway:

````
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


* Security Groups

Required Security Groups were configured:

````
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


* Network Load Balancer

A Network Load balancer was created,

````
LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
--name ${NAME} \
--subnets ${SUBNET_ID} \
--scheme internet-facing \
--type network \
--output text --query 'LoadBalancers[].LoadBalancerArn')
````

![image](https://user-images.githubusercontent.com/87030990/183676161-53628b9b-3cbb-4a87-8dec-383b0db8639a.png)


* Tagret Group

Target group was created: (It is expected to be unhealthy initially as there are no real targets yet.)

````
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name ${NAME} \
  --protocol TCP \
  --port 6443 \
  --vpc-id ${VPC_ID} \
  --target-type ip \
  --output text --query 'TargetGroups[].TargetGroupArn')
  ````

![image](https://user-images.githubusercontent.com/87030990/183677656-8c1a6888-0542-4843-9f74-4b92a0b4a07a.png)


Register targets: (Because there was no real targets yet, target IP addresses were hardcoded so that when the nodes become available, they will be used as targets.)

````
aws elbv2 register-targets \
  --target-group-arn ${TARGET_GROUP_ARN} \
  --targets Id=172.31.0.1{0,1,2}
  ````
 
![image](https://user-images.githubusercontent.com/87030990/183678168-4cde90c4-0cc4-4e17-b052-37ef4eb36dbd.png)


A listener to listen for requests was created and forward to the target nodes on TCP port 6443

````
aws elbv2 create-listener \
--load-balancer-arn ${LOAD_BALANCER_ARN} \
--protocol TCP \
--port 6443 \
--default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
--output text --query 'Listeners[].ListenerArn'
````

![image](https://user-images.githubusercontent.com/87030990/183678778-ed72d684-331f-47d6-96cb-850a766477d4.png)


* K8s Public Address

Get the Kubernetes Public address needed for the Load Balancer

````
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
--load-balancer-arns ${LOAD_BALANCER_ARN} \
--output text --query 'LoadBalancers[].DNSName')
````

![image](https://user-images.githubusercontent.com/87030990/183688957-3b0b7851-3380-4978-adf3-3466dbf19012.png)


### Step 2 – Create Compute Resources

* AMI (Amazon Machine Image)

Get an image to create EC2 instances:

````
IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
  --filters \
  'Name=root-device-type,Values=ebs' \
  'Name=architecture,Values=x86_64' \
  'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*' \
  | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
  ````

![image](https://user-images.githubusercontent.com/87030990/183704138-7b4f6d91-efed-452d-a851-5590acf78283.png)

![image](https://user-images.githubusercontent.com/87030990/183704433-195c04e0-d5e2-499d-8818-5708d21675f1.png)


* SSH key-pair

An SSH Key-Pair was created

````
mkdir -p ssh

aws ec2 create-key-pair \
  --key-name ${NAME} \
  --output text --query 'KeyMaterial' \
  > ssh/${NAME}.id_rsa
chmod 600 ssh/${NAME}.id_rsa
````

![image](https://user-images.githubusercontent.com/87030990/183754089-1f0180bc-6ea6-4e14-8b3e-b631ba23bd12.png)


* EC2 Instances for Controle Plane (Master Nodes)

3 EC2 Instances were created for Master nodes (t2.micro was used)

````
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


* EC2 Instances for Worker Nodes

3 EC2 Instances also created for Worker nodes:

````
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

![image](https://user-images.githubusercontent.com/87030990/184155551-6ac368fc-985a-4f2c-b01d-f63aa642b4d5.png)


### Step 3 Prepare The Self-Signed Certificate Authority And Generate TLS Certificates for each components in the Kubernetes Cluster

* Self-Signed Root Certificate Authority (CA)

Here, a Certificate Authority was created as root CA to generate and sign additional TLS certificates for Kube-apiserver, Kube-controller-manager, Kube-scheduler, etcd in the master nodes, likewise Kube-proxy and Kubelet in worker nodes.

The PKI Infrastructure is provisioned using cfssl which will have a Certificate Authority which will then generate certificates for all the individual components:

A directory named **ca-authority** was created and then cd into it: ````$ mkdir ca-authority && cd ca-authority````

![image](https://user-images.githubusercontent.com/87030990/184192786-37b92a31-80e0-4025-852e-33bf4b78f010.png)

The CA configuration file, Root Certificate, and Private key were generated

````
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
 ````
![image](https://user-images.githubusercontent.com/87030990/184192977-1ad99895-3f1a-4257-948b-36729dec5bf6.png)

![image](https://user-images.githubusercontent.com/87030990/184193313-dd108ea0-cbf6-402e-b261-23039c3da9b3.png)

List the directory to see the created files ````ls -ltr````


TLS Certificates For Client and Server were generated

The Certificate Signing Request (CSR), Private Key and the Certificate for the Kubernetes Master Nodes were generated.

````
{
cat > master-kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
   "hosts": [
   "127.0.0.1",
   "172.31.0.10",
   "172.31.0.11",
   "172.31.0.12",
   "ip-172-31-0-10",
   "ip-172-31-0-11",
   "ip-172-31-0-12",
   "ip-172-31-0-10.${AWS_REGION}.compute.internal",
   "ip-172-31-0-11.${AWS_REGION}.compute.internal",
   "ip-172-31-0-12.${AWS_REGION}.compute.internal",
   "${KUBERNETES_PUBLIC_ADDRESS}",
   "kubernetes",
   "kubernetes.default",
   "kubernetes.default.svc",
   "kubernetes.default.svc.cluster",
   "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  master-kubernetes-csr.json | cfssljson -bare master-kubernetes
}
````

![image](https://user-images.githubusercontent.com/87030990/184196337-a0bfbe09-2b0a-40f5-998a-1a86ba42257b.png)


Creating the other certificates: for the following Kubernetes components **(Scheduler Client Certificate, Kube Proxy Client Certificate, Controller Manager Client Certificate, Kubelet Client Certificates, K8s admin user Client Certificate**)

* kube-scheduler Client Certificate and Private Key

````
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:kube-scheduler",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
````

![image](https://user-images.githubusercontent.com/87030990/184198046-eb3438ef-7047-4e5d-80f5-30f1ada9422c.png)


* kube-proxy Client Certificate and Private Key

````
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:node-proxier",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
````

![image](https://user-images.githubusercontent.com/87030990/184215001-642c38f4-1730-419d-b304-d1c04dca7cba.png)


* kube-controller-manager Client Certificate and Private Key

````
{
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:kube-controller-manager",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
````

![image](https://user-images.githubusercontent.com/87030990/184216034-241bdbc0-c0fd-4fc6-9a2c-ff680bad6779.png)


* kubelet Client Certificate and Private Key

Similar to how you configured the api-server's certificate, Kubernetes requires that the hostname of each worker node is included in the client certificate.

 ```
 for i in 0 1 2; do
  instance="${NAME}-worker-${i}"
  instance_hostname="ip-172-31-0-2${i}"
  cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance_hostname}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:nodes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  internal_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PrivateIpAddress')

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${instance_hostname},${external_ip},${internal_ip} \
    -profile=kubernetes \
    ${NAME}-worker-${i}-csr.json | cfssljson -bare ${NAME}-worker-${i}
done
 ````
 
![image](https://user-images.githubusercontent.com/87030990/184227122-0aeeebfe-740a-42ce-b643-79c58d249f65.png)

 
 * Finally, kubernetes admin user's Client Certificate and Private Key
 
 ````
 {
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:masters",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
}
 ````

![image](https://user-images.githubusercontent.com/87030990/184227484-345a3f21-546e-41c5-a3b7-0ef0eff57a2a.png)


* Client Certificate and Private Key for Token Controller was also generated

**Token Controller** is a part of the Kubernetes Controller Manager responsible for generating and signing service account tokens which are used by pods or other resources to establish connectivity to the api-server
 
 ````
 {

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
}
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184227959-d0841e98-9e26-49df-a0f0-4d39a760d3e1.png)

 
 ### Step 4 – Distributing the Client and Server Certificates

* Worker Nodes

The following files were copied securely to the worker nodes using scp utility

Root CA certificate – ca.pem
X509 Certificate for each worker node
Private Key of the certificate for each worker node

 ````
 for i in 0 1 2; do
  instance="${NAME}-worker-${i}"
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
    ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/; \
done
 ````
 
![image](https://user-images.githubusercontent.com/87030990/184240614-1ffb07fe-a67f-4ca8-84e7-89127e882a2b.png)

 
 * Master or Controller node: – Note that only the **api-server** related files are sent over to the master nodes.
 
 ````
 for i in 0 1 2; do
instance="${NAME}-master-${i}" \
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
    ca.pem ca-key.pem service-account-key.pem service-account.pem \
    master-kubernetes.pem master-kubernetes-key.pem ubuntu@${external_ip}:~/;
done
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184240448-cfe3ec2b-3f05-4710-a534-c9257fb672a3.png)

 
 N.B: The kube-proxy, kube-controller-manager, kube-scheduler, and kubelet client certificates will be used to generate client authentication configuration files later
 


### Step 5: Use `KUBECTL` To Generate Kubernetes Configuration Files for Authentication
 
Here, **kubeconfig** files that will enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers were created

For start, a few environment variables were created for reuse by multiple commands.

````KUBERNETES_API_SERVER_ADDRESS=$(aws elbv2 describe-load-balancers --load-balancer-arns ${LOAD_BALANCER_ARN} --output text --query 'LoadBalancers[].DNSName')````

 ![image](https://user-images.githubusercontent.com/87030990/184242327-c58a75b8-78b4-422b-a723-0708607ebabb.png)
 

* Worker Nodes
 
The kubelet kubeconfig file was generated for each of the worker nodes:

N.B: Below command must be run in the directory where all the certificates were generated (ca-authority).
  
````
for i in 0 1 2; do

instance="${NAME}-worker-${i}"
instance_hostname="ip-172-31-0-2${i}"

 # Set the kubernetes cluster in the kubeconfig file
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://$KUBERNETES_API_SERVER_ADDRESS:6443 \
    --kubeconfig=${instance}.kubeconfig

# Set the cluster credentials in the kubeconfig file
  kubectl config set-credentials system:node:${instance_hostname} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

# Set the context in the kubeconfig file
  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:node:${instance_hostname} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184249197-1b609b63-3fc4-43d1-945d-fa7caffe4b11.png)

 
 List the output: ````ls -ltr *.kubeconfig ````
 
 
 ![image](https://user-images.githubusercontent.com/87030990/184249684-588143fe-4971-4dfd-9c4c-bcbaddb18624.png)

 
 * Generate the kube-proxy kubeconfig
 
 ````
{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
 ````
 
* Generate the kube-proxy kubeconfig
 
 ````
 {
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184253064-8e87b661-25f7-4969-bc82-68905df73765.png)
 

* Generate the Kube-Controller-Manager kubeconfig

Notice that the --server is set to use 127.0.0.1. This is because, this component runs on the API-Server so there is no point routing through the Load Balancer.

 ````
 {
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184255382-2426034e-a1a3-43b4-a633-abd60eba3e24.png)

 
 
 * Generating the Kube-Scheduler Kubeconfig
 
 ````
 {
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
 ````

* Finally, generate the kubeconfig file for the admin user
 
 ````
 {
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184259109-22ea763c-adf8-4484-be83-8af062bb0199.png)

 
### The files were distributed to their respective servers using **scp** and a for loop. 

* Worker Nodes

 ````
 for i in 0; do
>   instance="${NAME}-worker-${i}"
>   external_ip=$(aws ec2 describe-instances \      
>     --filters "Name=tag:Name,Values=${instance}" \
>     --output text --query 'Reservations[].Instances[].PublicIpAddress')
>   scp -i ../ssh/${NAME}.id_rsa \
>     kube-proxy.kubeconfig k8s-cluster-from-ground-up-worker-0.kubeconfig ubuntu@${external_ip}:~/; \
> done
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184259214-de715a93-3df7-4f10-a40d-bb8b496b5dfc.png)

 
N.B: The procedure above was repeated individually for worker 1 and 2 as the files are unique to each server.


* Master Node (Control Plane)

 ````
 for i in 0 1 2; do
>   instance="${NAME}-master-${i}"
>   external_ip=$(aws ec2 describe-instances \
>     --filters "Name=tag:Name,Values=${instance}" \
>     --output text --query 'Reservations[].Instances[].PublicIpAddress')
>   scp -i ../ssh/${NAME}.id_rsa \
>     admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ubuntu@${external_ip}:~/; \
> done
````

![image](https://user-images.githubusercontent.com/87030990/184260591-ba1b3c88-e325-4c34-93f2-2480ee8d7705.png)
 


### Step 6: Prepare the ETCD Database for Encryption at Rest

Kubernetes uses etcd (A distributed key value store) to store variety of data which includes the cluster state, application configurations, and secrets. By default, the data that is being persisted to the disk is not encrypted. Any attacker that is able to gain access to this database can exploit the cluster since the data is stored in plain text. Hence, it is a security risk for Kubernetes that needs to be addressed.

To mitigate this risk, we must prepare to encrypt etcd at rest. "At rest" means data that is stored and persists on a disk. "In-flight" encryption is done through TLS.

* Generate the encryption key and encode it using base64

````
ETCD_ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64) 
````

![image](https://user-images.githubusercontent.com/87030990/184261132-c3b73e8a-bb08-4d01-9c51-26cff9d30e4f.png)


* Create an **encryption-config.yaml** file as documented officially by kubernetes
 
 ````
 cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ETCD_ENCRYPTION_KEY}
      - identity: {}
EOF
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184261366-d166f757-ac0b-490a-b444-31d16f144de6.png)
                                    

* Send the encryption file to the Controller nodes using scp and a for loop.

````
for i in 0 1 2; do
   instance="${NAME}-master-${i}"
   external_ip=$(aws ec2 describe-instances \
     --filters "Name=tag:Name,Values=${instance}" \
     --output text --query 'Reservations[].Instances[].PublicIpAddress')
   scp -i ../ssh/${NAME}.id_rsa \
      encryption-config.yaml ubuntu@${external_ip}:~/; \
 done
 ````

### Bootstrap etcd cluster
                                    
It is advisable to use a terminal multi-plexer to reduce the risk of error of omission as the task is enormous on multiple servers.
                                    

* SSH into the controller server

Master-1

````
master_1_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-0" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_1_ip}
````
                                    
Master-2

````
master_2_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-1" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_2_ip}
````                                    
                                    
Master-3

````
master_3_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-2" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_3_ip}                                  
 ````
                                    
![image](https://user-images.githubusercontent.com/87030990/184268582-530447f0-1889-404a-8b07-5901407b34f1.png)

                                    
* Download and install etcd
                                    
 ````
  wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
 ````
 
 * Extract and install the etcd server and the etcdctl command line utility:
                                    
 ````
{
tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
}
 ````
 
 Configure the etcd server
                                    
 ````
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo chmod 700 /var/lib/etcd
  sudo cp ca.pem master-kubernetes-key.pem master-kubernetes.pem /etc/etcd/
}
 ````
 
The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance:
 
 ````
export INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
````
 
* Each etcd member must have a unique name within an etcd cluster. Set the etcd name to node Private IP address so it will uniquely identify the machine:
 
 ````
 ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)

echo ${ETCD_NAME}
 ````
 
* Create the etcd.service systemd unit file:
                                    
The flags are well documented here
 
 ````
 cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster master-0=https://172.31.0.10:2380,master-1=https://172.31.0.11:2380,master-2=https://172.31.0.12:2380 \\
  --cert-file=/etc/etcd/master-kubernetes.pem \\
  --key-file=/etc/etcd/master-kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/master-kubernetes.pem \\
  --peer-key-file=/etc/etcd/master-kubernetes-key.pem \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
 ````
 
* Start and enable the etcd Server

 ````
{
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
}
 ````
 
 * Verify the etcd installation
 
 ````
 sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/master-kubernetes.pem \
  --key=/etc/etcd/master-kubernetes-key.pem
 ````
 
 Output:
 
 ![image](https://user-images.githubusercontent.com/87030990/184269761-377b584e-1faa-42e8-a3e5-c512df0a2649.png)
![image](https://user-images.githubusercontent.com/87030990/184982236-349e565e-0fb0-4864-a7b8-05f58ba82aa0.png)

 

### Bootstrap the Control Plane
          
The components for the control plane was configured on the master/controller nodes.

* Create the Kubernetes configuration directory:

 ````
sudo mkdir -p /etc/kubernetes/config
````
 
* Download the official Kubernetes release binaries:

 Kube-apiserver version (v1.24.2) that matches the version of Kubectl on the local system was downloaded and installed:
  
 ````
wget -q --show-progress --https-only --timestamping \
"https://storage.googleapis.com/kubernetes-release/release/v1.24.2/bin/linux/amd64/kube-apiserver" \
"https://storage.googleapis.com/kubernetes-release/release/v1.24.2/bin/linux/amd64/kube-controller-manager" \
"https://storage.googleapis.com/kubernetes-release/release/v1.24.2/bin/linux/amd64/kube-scheduler" \
"https://storage.googleapis.com/kubernetes-release/release/v1.24.2/bin/linux/amd64/kubectl"
 ````
          
* Install the Kubernetes binaries:
          
````
{
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
````
 
 ![image](https://user-images.githubusercontent.com/87030990/184818628-349d0615-c75d-409b-b498-3b638d6c58ec.png)

 
 * Configure the Kubernetes API Server:

 ````
{
sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem master-kubernetes-key.pem master-kubernetes.pem \
service-account-key.pem service-account.pem \
encryption-config.yaml /var/lib/kubernetes/
}
 ````
          
 * Error: No such file or directory
 
 ![image](https://user-images.githubusercontent.com/87030990/184815208-8736947e-fb82-4ba2-809e-727ac61de2a6.png)
 
* Encryption.yaml files was copied into the master nodes from ca-authority directory
 
 ````
  for i in 0 1 2; do
   instance="${NAME}-master-${i}"
   external_ip=$(aws ec2 describe-instances \
     --filters "Name=tag:Name,Values=${instance}" \
     --output text --query 'Reservations[].Instances[].PublicIpAddress')
   scp -i ../ssh/${NAME}.id_rsa \
     encryption-config.yaml ubuntu@${external_ip}:~/; \
 done
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184816579-3e4589cb-43ed-46a2-a15d-b53c8941343e.png)
 

 * Encryption.yaml file was then moved to **/var/lib/kubernetes/** directory
 
 ````
 sudo mv encryption-config.yaml /var/lib/kubernetes/
 ````
 
 The instance internal IP address will be used to advertise the API Server to members of the cluster. The internal IP address was retrieved for the current compute instance:
 
 ````
 export INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
 ````
 
 * Create the kube-apiserver.service systemd unit file:
 
Note:  In this project, **systemd** is used to configure kubernetes
 
 ````
 cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/master-kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/master-kubernetes-key.pem\\
  --etcd-servers=https://172.31.0.10:2379,https://172.31.0.11:2379,https://172.31.0.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/master-kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/master-kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${INTERNAL_IP}:6443 \\
  --service-cluster-ip-range=172.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/master-kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/master-kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
 ````
 
 #### Configure the Kubernetes Controller Manager:

Move the kube-controller-manager kubeconfig into place:

 ````
 sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
 ````
 
 * Export some variables to retrieve the vpc_cidr – This will be required for the bind-address flag:
 
 ````
 export AWS_METADATA="http://169.254.169.254/latest/meta-data"
export EC2_MAC_ADDRESS=$(curl -s $AWS_METADATA/network/interfaces/macs/ | head -n1 | tr -d '/')
export VPC_CIDR=$(curl -s $AWS_METADATA/network/interfaces/macs/$EC2_MAC_ADDRESS/vpc-ipv4-cidr-block/)
export NAME=k8s-cluster-from-ground-up
 ````
 
 * Create the kube-controller-manager.service systemd unit file:
 
````
 
export VPC_CIDR=$(curl -s $AWS_METADATA/network/interfaces/macs/$EC2_MAC_ADDRESS/vpc-ipv4-cidr-block/)
export NAME=k8s-cluster-from-ground-up
Create the kube-controller-manager.service systemd unit file:

cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=${VPC_CIDR} \\
  --cluster-name=${NAME} \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --authentication-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --authorization-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=172.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
 ````
 
#### Configure the Kubernetes Scheduler:

* Move the kube-scheduler kubeconfig into place:

 ````
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
sudo mkdir -p /etc/kubernetes/config
````
 
 * Create the kube-scheduler.yaml configuration file:

 ````
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
 ````
 
 * Create the kube-scheduler.service systemd unit file:

 ````
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
 ````
 
 * Start the Controller Services
 
 ````
{
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
 }
 ````
 
* Check the status of the services. Start with the kube-scheduler and kube-controller-manager. It may take up to 20 seconds for kube-apiserver to be fully loaded.

````
{
sudo systemctl status kube-apiserver
sudo systemctl status kube-controller-manager
sudo systemctl status kube-scheduler
}
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184826995-2e0510e1-6089-43a0-a59a-357c656b5bc4.png)

 

### Step 7: Test that Everything is working fine
 
 To get the cluster details run:
 
 ````
kubectl cluster-info  --kubeconfig admin.kubeconfig
 ````
 
 * To get the current namespaces:
 
 ````
kubectl get namespaces --kubeconfig admin.kubeconfig
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/184828057-92accebd-f723-43f6-b142-3799d184fc61.png)
 
 * To reach the Kubernetes API Server publicly
 
 ````
 curl --cacert /var/lib/kubernetes/ca.pem https://$INTERNAL_IP:6443/version
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185149333-76877d18-47a2-42a8-8170-f59be637536a.png)


* To get the status of each component:
 
 ````
kubectl get componentstatuses --kubeconfig admin.kubeconfig
````
 
 Error: The scheduler was observed to be unhealthy but the rest were. 
 
 **Error message:** scheduler unhealthy get "https://127.0.0.1:10259/healthz": dial tcp 127.0.0.1:10259: connect: connection refused
 
 ![image](https://user-images.githubusercontent.com/87030990/185000626-b2e74f41-bfee-474e-a2d9-63efc6ef542e.png)

 
 Resolution: The apiVersion in the **kube-scheduler.yaml** was changed from **kubescheduler.config.k8s.io/v1beta1** instead of **kubescheduler.config.k8s.io/v1beta2**
 after carrying out further troubleshooting using ````journalctl -xe | grep scheduler.service```` and checking the internet for possible solution to the error.
 
 ![image](https://user-images.githubusercontent.com/87030990/185001326-754a7d42-6619-4f43-b46b-bcc1c9f63295.png)

 
 ### Step 8: Configuring Role Based Access Control
 
On one of the controller nodes, configure Role Based Access Control (RBAC) so that the api-server has necessary authorization for the kubelet.

* Create the ClusterRole:
 
 ````
 cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
 ````
 
 * Create the ClusterRoleBinding to bind the kubernetes user with the role created above:

 ````
cat <<EOF | kubectl --kubeconfig admin.kubeconfig  apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
PREV
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185001930-17a4a3a6-8ba1-44e6-b200-104ee5e537ef.png)

 
* Configure Role Based Access (RBAC) for Kubelet Authorization:

Configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods. Create the system:kube-apiserver-to-kubelet ClusterRole with permissions to access the Kubelet API and perform most common tasks associated with managing pods on the worker nodes:


Run the below script on the Controller node:

 ````
 cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185002765-87c8a66f-b730-40ce-a241-c1a047dc8657.png)

 
 * Bind the system:kube-apiserver-to-kubelet ClusterRole to the kubernetes user so that API server can authenticate successfully to the kubelets on the worker nodes:
 
 ````
 cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185002875-bfe575c9-fb21-47e7-9a05-332482144cfa.png)

 
### Step 9: Bootstraping components on the worker nodes
 
The following components will be installed on each node (**kubelet, kube-proxy, Containerd or Docker and Networking plugins**)

 
 * SSH into the worker nodes and install OS depemdences
 
 Worker-1

````
worker_1_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-worker-0" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${worker_1_ip}
````

Worker-2

````
worker_2_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-worker-1" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${worker_2_ip}
````

Worker-3

````
worker_3_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-worker-2" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${worker_3_ip}
 ````
  
 ![image](https://user-images.githubusercontent.com/87030990/185003497-5db7d79a-7edb-4d33-9aee-f6639be40488.png)
 
* Install OS dependencies:

````
{
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset
}
````
 
* Disable Swap
If swap) is not disabled, kubelet will not start. It is highly recommended to allow Kubernetes to handle resource allocation.

* Test if swap is already enabled on the host: ````sudo swapon --show````
 
 * Disable swap by running: ````sudo swapoff -a````
 
 * Download and install a container runtime. (Containerd was preferred to Docker as Docker is now deprecated, and Kubernetes no longer supports the Dockershim codebase from v1.20 release)
 
 Download binaries for runc, cri-ctl, and containerd

````
 wget https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64 \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.21.0/crictl-v1.21.0-linux-amd64.tar.gz \
  https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz 
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185004741-30e96471-d0ec-47bb-b072-c0b1171036bf.png)

 
 * Configure containerd:
 
 ````
 {
  mkdir containerd
  tar -xvf crictl-v1.21.0-linux-amd64.tar.gz
  tar -xvf containerd-1.4.4-linux-amd64.tar.gz -C containerd
  sudo mv runc.amd64 runc
  chmod +x  crictl runc  
  sudo mv crictl runc /usr/local/bin/
  sudo mv containerd/bin/* /bin/
}
 ````
 
* Creating containerd directory: ````sudo mkdir -p /etc/containerd/````
 
![image](https://user-images.githubusercontent.com/87030990/185005046-3dc95548-0c36-4384-bc07-ec9b66fddc40.png)

 * Inserting the following code snippets in it
 
 ````
 cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185005347-dfb30c1d-878d-4b48-989b-9b2c1ee4975e.png)

 
 * Create the containerd.service systemd unit file:

 ````
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185005518-7e258bfc-44ed-4dbc-9e7b-dcf3064e014b.png)

 
 * Create directories for to configure kubelet, kube-proxy, cni, and a directory to keep the kubernetes root ca file:
 
 ````
sudo mkdir -p \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubernetes \
  /var/run/kubernetes
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185005681-8bd0948d-46e7-4f93-b262-1264e96cf86c.png)
 

 * Downloading Container Network Interface(CNI) plugins available from container networking’s GitHub repo:
 
 ````
 wget -q --show-progress --https-only --timestamping \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz
 ````
 
 * Install CNI into /opt/cni/bin/
 
 ````
 sudo tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
 ````
 
  ![image](https://user-images.githubusercontent.com/87030990/185006066-7ee6d544-c874-46d2-836b-f9dc70ca26eb.png)

 
 * Download binaries for kubectl, kube-proxy, and kubelet
 
 ````
 wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubelet
 ````
 
 * Install the downloaded binaries
 
 ````
{
  chmod +x  kubectl kube-proxy kubelet  
  sudo mv  kubectl kube-proxy kubelet /usr/local/bin/
}
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185006506-7aa4d937-5135-4043-8e0d-7a1fd02ca094.png)

 
 ````
 wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubelet
 ````
 
 * Install the downloaded binaries
 
 ````
{
  chmod +x  kubectl kube-proxy kubelet  
  sudo mv  kubectl kube-proxy kubelet /usr/local/bin/
}
 ````
 
 * Configure kubelet:
 
 ![image](https://user-images.githubusercontent.com/87030990/185007807-e0d72230-9456-4358-a905-2869b34cbee5.png)



### Step 10: Configuring the network

Get the POD_CIDR that will be used as part of network configuration

 ````
POD_CIDR=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^pod-cidr" | cut -d"=" -f2)
echo "${POD_CIDR}"
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185008147-ce064e84-34aa-4a87-9087-642da1d08b22.png)


* Configure the bridge and loopback networks

Bridge:
 
 ````
 cat > 172-20-bridge.conf <<EOF
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
 ````
 
 Loopback:

 ````
cat > 99-loopback.conf <<EOF
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185009097-8811081e-0fe1-4e8a-bd40-532e65605947.png)

 
 * Move the files to the network configuration directory:
 
 ````
sudo mv 172-20-bridge.conf 99-loopback.conf /etc/cni/net.d/
 ````
 
 * Store the worker’s name in a variable:
 
 ````
 NAME=k8s-cluster-from-ground-up
WORKER_NAME=${NAME}-$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${WORKER_NAME}"
 ````
 
 * Move the certificates and kubeconfig file to their respective configuration directories:
 
 ````
sudo mv ${WORKER_NAME}-key.pem ${WORKER_NAME}.pem /var/lib/kubelet/
sudo mv ${WORKER_NAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185009643-a6d1cf69-9193-422e-8bd3-063021042aee.png)

 
 * Create the kubelet-config.yaml file

Ensure the needed variables exist:

````
NAME=k8s-cluster-from-ground-up
WORKER_NAME=${NAME}-$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${WORKER_NAME}"
 ````
 
 ````
 cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta2
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${WORKER_NAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${WORKER_NAME}-key.pem"
EOF
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185010056-6d878617-259d-40a2-800c-0bfc4ffefc9f.png)
 
 
 * Configure the kubelet systemd service
 
 ````
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service
[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --cluster-domain=cluster.local \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
````
 
 ![image](https://user-images.githubusercontent.com/87030990/185010580-a9127c84-9b2c-4eec-9cbf-bbda32e3601f.png)

 
 * Create the kube-proxy.yaml file

 ````
 cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "172.31.0.0/16"
EOF
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185011168-eccadcef-1d4f-41f1-8f8c-6f5f1e67ced3.png)

 
 * Configure the Kube Proxy systemd service
 
 ````
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/185012444-07139d66-8614-44a5-b9ce-69dc512071e6.png)

 
 * Reload configurations and start both services
 
 ````
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy
}
 ```` Kubernetes cluster From-Ground-Up
 
 ![image](https://user-images.githubusercontent.com/87030990/185012629-84003d35-311a-4016-9b59-e2c0efbf863a.png)

 
 * Checking the readiness of the worker nodes on all master nodes: ````$ kubectl get nodes --kubeconfig admin.kubeconfig -o wide```
 

![image](https://user-images.githubusercontent.com/87030990/185015505-ea648881-5ff7-4cd6-9b83-1c95a80938c9.png)

**Conclusion:** The Kubernetes cluster was successfully built From-Ground-Up as shown above. The worker nodes joined to the cluster and are in a READY state.

