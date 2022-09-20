### PERSISTING DATA IN KUBERNETES

#### INTRODUCTION

Containers are stateless by design, which means that data does not persist in the containers. Even when you run the containers in kubernetes pods, they still remain stateless unless you ensure that your configuration supports statefulness. The pods created in Kubernetes are ephemeral, they don't run for long. When a pod dies, any data that is not part of the container image will be lost when the container is restarted because Kubernetes is best at managing stateless applications which means it does not manage data persistence. To ensure data persistent, the PersistentVolume resource is implemented to acheive this.

Prerequisite: Setting Up AWS Elastic Kubernetes Service With EKSCTL

* **kubectl** – A command line tool for working with Kubernetes clusters.

* **eksctl** – A command line tool for working with EKS clusters that automates many individual tasks. This guide requires that you use version 0.112.0 or later. For more information, see Installing or updating eksctl.

* **Required IAM permissions** – The IAM security principal that you're using must have permissions to work with Amazon EKS IAM roles and service linked roles,

### Step 1: Setting up Amazon Elastic Kubernetes Service with EKSCTL

#### To install or update eksctl on Linux

* Download and extract the latest release of eksctl with the following command: 
````
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
````
* Move the extracted binary to /usr/local/bin.````sudo mv /tmp/eksctl /usr/local/bin````

* Test that your installation was successful with the following command: ````eksctl version````

#### To install or update eksctl on Windows

* Install  Chocolatey first

* Install the binaries with the following command: ````choco install -y eksctl````

* If they are already installed, run the following command to upgrade: ````choco upgrade -y eksctl````

* Test that your installation was successful with the following command. ````eksctl version````

![image](https://user-images.githubusercontent.com/87030990/191236745-5da0410a-3765-4dc0-a620-7fff2d430541.png)


#### Creating Amazon EKS cluster and nodes

````
$ eksctl create cluster \
  --name obafema-cluster \
  --version 1.21 \
  --region us-east-2 \
  --nodegroup-name worker-nodes \
  --node-type t2.medium \
  --nodes 2 \
  ````
![image](https://user-images.githubusercontent.com/87030990/191243041-491fcf5d-22c5-4d53-9ef0-cf70d18f49c1.png)
![image](https://user-images.githubusercontent.com/87030990/191243148-d01139b7-f329-44a2-8547-c009e91b6cf2.png)

### Step 2: Creating Persistent Volume Manually For The Nginx Application

* Creating a deployment manifest file for the Nginx application and applying it:

````
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF
````

* Task
Verify that the pod is running
Check the logs of the pod
Exec into the pod and navigate to the nginx configuration file /etc/nginx/conf.d
Open the config files to see the default configuration.

![image](https://user-images.githubusercontent.com/87030990/191245474-b1788305-6fe3-4600-97b7-146c20427859.png)
![image](https://user-images.githubusercontent.com/87030990/191246130-3f30eb33-f0c6-4227-8e55-1238d944f78b.png)

NOTE: There are some restrictions when using an awsElasticBlockStore volume:

The nodes on which pods are running must be AWS EC2 instances.
Those instances need to be in the same region and availability zone as the EBS volume
EBS only supports a single EC2 instance mounting a volume

* When creating a volume it must exists in the same region and availability zone as the EC2 instance running the pod. To confirm which node is running the pod:````kubectl get pod nginx-deployment-6fdcffd8fc-cppkv -n obafema -o wide````

* To check the Availability Zone where the node is running:````kubectl descride node ip-192-168-45-154.us-east-2.compute.internal -n obafema````

![image](https://user-images.githubusercontent.com/87030990/191248774-cabe165f-64f5-44e7-a9ba-3505b0ae08ed.png)

* Creating a volume in the Elastic Block Storage section in AWS in the same AZ as the node running the nginx pod which will be used to mount volume into the Nginx pod.

![image](https://user-images.githubusercontent.com/87030990/191250018-414f375b-d142-4134-9163-6855f4d526ca.png)

Copy the volume ID, update the deployment configuration with the volume spec and apply changes using ````kubectl apply -f nginx-pod.yaml -n obafema````

````
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "vol-0a0fe7d926616b8d9"
          fsType: ext4
````
![image](https://user-images.githubusercontent.com/87030990/191251920-5be40a9c-5b53-492d-8681-b6e393ae954d.png)

* Run **kubectl describe** on both the pod and deployment to verify EBS volume attachment

![image](https://user-images.githubusercontent.com/87030990/191256723-aa3eb351-f1c6-45f5-887a-509ed4c8dcb0.png)
![image](https://user-images.githubusercontent.com/87030990/191257358-1b71b5e0-ab38-4d11-8238-01f2b4a5b89b.png)

The **volumeMounts** section was added to the deployment yaml manifest to specify where Volume will be mounted inside the container to store all data written into the directory:


