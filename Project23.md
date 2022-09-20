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


* Although EBS volume has been attached to the pod, the **directory /usr/share/nginx/html** which holds the software/website code is still ephemeral, and if there is any kind of update to the **index.html** file, the new changes will only be there for as long as the pod is still running. If the pod dies after, all previously written data will be erased.

![image](https://user-images.githubusercontent.com/87030990/191274962-2d64ba4e-21de-48b9-a7b4-3b8563a2fa03.png)

* Changes was made to the index.html file as shown

![image](https://user-images.githubusercontent.com/87030990/191275173-9563a4e7-6ff3-45f3-bbc6-cc93b8089eb2.png)

* The pod was deleted and another one immediately created:

![image](https://user-images.githubusercontent.com/87030990/191275617-56610aeb-8696-473c-b414-1a0d865403c0.png)

* Refreshing the page returned it to the default message (Welcome to nginx)

![image](https://user-images.githubusercontent.com/87030990/191276020-b7d6aee4-615a-4993-b659-b16b3b87add1.png)


The **volumeMounts** section was added to the deployment yaml manifest to specify where Volume will be mounted inside the container to store all data written into the directory:

* Update the deployment manifest with volumeMount section and apply the changes:

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
        volumeMounts:
        - name: nginx-volume
          mountPath: /usr/share/nginx/
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "  vol-07b537651bbe68be0"
          fsType: ext4
 ````
 
* To complete the configuration, we will need to add another section called **volumeMounts** to the deployment yaml manifest. The value provided to name in **volumeMounts** must be the same value used in the **volumes section**. It basically means mount the volume with the name provided, to the provided mountpath

* But the problem with this configuration is that when we port forward the service and try to reach the endpoint, we will get a 403 error. This is because mounting a volume on a filesystem that already contains data will automatically erase all the existing data. To solve this issue is by implementing Persistent Volume(PV) and Persistent Volume claims(PVCs) resource.

### Step 3: Managing Volumes Dynamically With PV and PVCs

* PVs are volume plugins that have a lifecycle completely independent of any individual Pod that uses the PV. This means that even when a pod dies, the PV remains. A PV is a piece of storage in the cluster that is either provisioned by an administrator through a manifest file, or it can be dynamically created if a storage class has been pre-configured.

* PVs are resources in the cluster. PVCs are requests for those resources and also act as claim checks to the resource. PVs are cluster-based and not namespace scoped unlike PVCs which are namespace scoped. By default, in EKS, there is a default storageClass configured as part of EKS installation

* Run the command below to check if you already have a storageclass in your cluster ````kubectl get storageclass````

![image](https://user-images.githubusercontent.com/87030990/191308002-d629af6a-1e43-41a7-9576-8ce91644825b.png)

* If there is no storage class in your cluster, below manifest is an example of how one would be created

````
 kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: gp2
    annotations:
      storageclass.kubernetes.io/is-default-class: "true"
  provisioner: kubernetes.io/aws-ebs
  parameters:
    type: gp2
    fsType: ext4 
````

* 2 different approaches were used to create persistences for the nginx deployment:

* The first is to create a manifest file for a PVC, and based on the gp2 storageClass, a PV will be dynamically created:

````
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nginx-volume-claim
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi
      storageClassName: gp2
````

Apply the manifest file and you will get an output like below

* Checking the setup:````kubectl get pvc````

* Checking for the volume binding section: ````kubectl describe storageclass gp2````

![image](https://user-images.githubusercontent.com/87030990/191313355-d93aa0d8-2f2a-45ab-8292-82a1fe92c731.png)

* Checking for pv:````kubectl get pv````

No pv created yet

![image](https://user-images.githubusercontent.com/87030990/191314586-af6d41e5-cfef-4b9d-9150-66b3d94db9c2.png)

* To create pv, edit the **nginx-pod.yaml** file to create the PV and apply the change: ````kubectl apply -f nginx-pod.yaml -n obafema````

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
        volumeMounts:
        - name: nginx-volume-claim
          mountPath: "/tmp/obafema"
      volumes:
      - name: nginx-volume-claim
        persistentVolumeClaim:
          claimName: nginx-volume-claim
````



* To check the dynamically created PV: ````kubectl get pv````
![image](https://user-images.githubusercontent.com/87030990/191321108-eab7669c-5ebf-484b-af74-d82a7c2c58eb.png)

![image](https://user-images.githubusercontent.com/87030990/191323740-a1ca0c34-8c4e-4846-9136-ce1fa31d1777.png)


* Another approach is creating a **volumeClaimTemplate** within the Pod spec of **nginx-pod.yaml** file so rather than having 2 manifest files, everything will be defined within a single manifest:

````
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
spec:
  selector:
    matchLabels:
      tier: frontend
  serviceName: nginx-service
  replicas: 1  
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
        volumeMounts:
        - name: nginx-volume
          mountPath: /tmp/obafema
  volumeClaimTemplates:
  - metadata:
      name: mginx-volume
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
      storageClassName: standard
 ````

![image](https://user-images.githubusercontent.com/87030990/191327101-d5276875-3641-461b-ab11-1d89c5d5b7a9.png)

### Step 4: Using ConfigMap As A Persistent Storage

* ConfigMap is an API object used to store non-confidential data in key-value pairs. Using configMaps for persistence is not something you would consider for data storage. Rather it is a way to manage configuration files and ensure they are not lost as a result of Pod replacement.

* Remove the volumeMounts and PVC sections of the manifest and use kubectl to apply the configuration

* port forward the service and ensure that you are able to see the "Welcome to nginx" page

![image](https://user-images.githubusercontent.com/87030990/191348948-d9255fb8-334f-4d18-b88a-edd9e3a6bd23.png)

![image](https://user-images.githubusercontent.com/87030990/191348851-63a34949-ae15-478f-87a2-27e214ab93e3.png)

* To demonstrate this, the HTML file that came with Nginx will be used. This file can be found in /usr/share/nginx/html/index.html 

* Exec into the container and copying the HTML file: ````kubectl exec nginx-deployment-6fdcffd8fc-rpxjd -i -t bash -n obafema````

*  cat /usr/share/nginx/html/index.html 

![image](https://user-images.githubusercontent.com/87030990/191349377-876df6b3-8351-4f6e-b02c-00de5497cd0f.png)

* Creating the **ConfigMap** manifest file and customizing the HTML file and applying the change:````kubectl apply -f nginx-configmap.yaml -n obafema````
````
cat <<EOF | tee ./nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: website-index-file
data:
  # file to be mounted inside a volume
  index-file: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
EOF
````
![image](https://user-images.githubusercontent.com/87030990/191351597-cad88f35-507e-4e86-896e-d329cd6edae7.png)

* Update the deployment file to use the configmap in the volumeMounts section

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
        volumeMounts:
          - name: config
            mountPath: /usr/share/nginx/html
            readOnly: true
      volumes:
      - name: config
        configMap:
          name: website-index-file
          items:
          - key: index-file
            path: index.html
 ````
 
* Now the index.html file is no longer ephemeral because it is using a configMap that has been mounted onto the filesystem. This is now evident when you exec into the pod and list the /usr/share/nginx/html directory.

![image](https://user-images.githubusercontent.com/87030990/191353444-35befd43-c2dd-45d5-83ce-971a670e7ca2.png)

* the index.html is now a soft link to ../data

* Accessing the site will not change anything at this time because the same html file is being loaded through configmap. To see the change in effect, update the configmap manifest file by making any change to the content of the html file through the configmap, and restart the pod. All the changes should persist.

* To update the configmap. You can either update the manifest file, or the kubernetes object directly using  ````kubectl edit cm website-index-file```` and performing the Port forwarding command ```kubectl port-forward svc/nginx-service 8088:80 -n obafema```` and accessing it through the browser:

````
apiVersion: v1
data:
  index-file: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to AGOONE.IO!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
````

![image](https://user-images.githubusercontent.com/87030990/191356087-39c913ac-ee06-47ef-8415-a305a2770c52.png)
![image](https://user-images.githubusercontent.com/87030990/191356449-d7ba9afa-7334-492a-88e5-d61bfd4daaeb.png)

* Without restarting the pod, your site should be loaded automatically

![image](https://user-images.githubusercontent.com/87030990/191355872-856228cb-8f6d-4e66-984f-657240d39ea2.png)

* To restart the deployment for any reason, simply use the command: ````kubectl rollout restart deploy nginx-deployment````

![image](https://user-images.githubusercontent.com/87030990/191358351-74e8ebe1-412d-4b47-bc13-ca189cf3e4f8.png)

![image](https://user-images.githubusercontent.com/87030990/191358154-5b5a814b-f896-4ebb-ac20-f214e224b9c4.png)


