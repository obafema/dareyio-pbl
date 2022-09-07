### Deploying Applications Into Kubernetes Cluster

**Objective:** To deploy containerised applications as pods in Kubernetes and access the application from the browser.

#### Step 1: Provisioing of the Infrastructure for the deployment using Terraform

AWS EKS Managed Node Group was used for the implementation

![image](https://user-images.githubusercontent.com/87030990/188833321-4a8383c3-c7c8-4c2e-b95f-73fea78f900f.png)
![image](https://user-images.githubusercontent.com/87030990/188833450-d6269c46-351f-4aff-991d-bda64bea551c.png)

The following nodes were verified to be created using terraform:

![image](https://user-images.githubusercontent.com/87030990/188835034-c1872b5a-52bd-4873-8d9b-c5cfd53fca81.png)

### Step 2: Creating A Pod For The Nginx Application

A director named **k8s** was created and cd into

A namespace named **obafema** was created for testing the deployment before deployment in dev environment:

![image](https://user-images.githubusercontent.com/87030990/188836083-51e609cf-c81f-4744-80f4-3c5ccd592228.png)

Nodes created in the eks cluster was confirmed as shown below:
![image](https://user-images.githubusercontent.com/87030990/188836313-af89685a-8dd6-4d21-b4fd-522b63968052.png)

* Manisfest to create a Pod for nginx Application named **nginx-pod.yaml** was written and the pod subsequently created withing **obafema namespace** using:

````kubectl apply -f nginx-pod.yaml````

nginx-pod.yaml manifest file

![image](https://user-images.githubusercontent.com/87030990/188837396-9124df40-7201-4537-a21e-8fedc46f2ec9.png)


The set up was inspected using:

````
$ kubectl get pod nginx-pod --show-labels

$ kubectl get pod nginx-pod -o wide

$ kubectl describe pod nginx-pod 
````

![image](https://user-images.githubusercontent.com/87030990/188840275-17445eb1-fd97-42f4-9d00-b16a938d76a5.png)


#### Step 3: Accessing the Nginx Application within the Kubernetes Cluster and also through a Browser

The Nginx Pod was accessed through its IP address from within the Kubernetes cluster using an image that already has curl software installed.

````$ kubectl run curl --image=dareyregistry/curl -i --tty```` was ran to run the container that has curl in it as a pod

curl command pointing it to the IP address of the Nginx Pod was ran: ````$ curl -v 172.16.3.17:80````

![image](https://user-images.githubusercontent.com/87030990/188859295-8d4a96d1-a22a-4326-9b70-64f8db4f56e9.png)



* To access the application through the browser, a service was created for the pod.

An nginx-service.yaml nanifest file was created

````
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-pod
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
````

![image](https://user-images.githubusercontent.com/87030990/188843055-b7edd264-8065-4ec2-9671-02e46af49588.png)


The service for the nginx pod was then created by applying the manifest file:````$ kubectl apply -f nginx-service.yaml````

![image](https://user-images.githubusercontent.com/87030990/188858100-79a48682-b658-4076-99e7-24ba36d7ef7c.png)

The service setup was inspected using the following commands:

````
$ kubectl get service nginx-service --show-labels

$ kubectl get svc nginx-service -o wide
````

![image](https://user-images.githubusercontent.com/87030990/188863299-1c51e8f0-cc6c-4936-a2b8-f56c7a02e328.png)


* The type of service created for the Nginx pod is a ClusterIP which cannot be accessed externally. Port-forwarding was done in order to map the machine's(my laptop) port to the ClusterIP service port i.e, tunnelling traffic through the machine's port number to the port number of the nginx-service: ````$ kubectl port-forward svc/nginx-service 8089:80````

![image](https://user-images.githubusercontent.com/87030990/188865117-6f858f21-2efb-4169-9152-658ec700e2c3.png)

* The Nginx application was accessed from the browser using ````http://localhost:8089````

![image](https://user-images.githubusercontent.com/87030990/188865397-7bdfad02-6212-48a2-8d64-e8ee47f54d48.png)


* To check logs on the pods, do ````kubectl logs nginx-pod```. To keep the logs yupdating do ````kubectl logs nginx-pod -n obafema --follow````

![image](https://user-images.githubusercontent.com/87030990/188867982-eeaf0e32-c60d-471a-8627-5389aae36008.png)


* Another way of accessing the Nginx app through browser is the use of **NodePort** which is a type of service that exposes the service on a static port on the node’s IP address and they range from 30000-32767 by default.

Note: Here, you can only access the application on the **worker node** the pod is running on.

* The **nginx-service.yml** manifest file was edited to expose the Nginx service in order to be accessible to the browser by adding NodePort as a type of service:

````
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx-pod
  ports:
    - protocol: TCP
      port: 80
      nodePort: 30080
````

````kubectl apply -f nginx-service.yaml````

![image](https://user-images.githubusercontent.com/87030990/188868791-3d8a0877-b109-4294-867a-b8ab4ce9e997.png)


Effort to access the application through the Public IP address of the worker node was not successful.

**Error message:** Site could not be reached. 3.23.23.173 took too long to respond.

![image](https://user-images.githubusercontent.com/87030990/188895217-53794e33-48a7-41f5-9d6a-622cd9158316.png)

Another cofiguration with LoadBalancer as the type of node:

````
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx-pod
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
 ````
 
 ````kubectl apply -f nginx-service.yaml````
 
![image](https://user-images.githubusercontent.com/87030990/188926864-4b74b6e9-714e-4423-873b-874f30bd7226.png)


* Accessing it through the browser:

![image](https://user-images.githubusercontent.com/87030990/188926521-daea7b7b-6dc3-406f-838c-68bcce1813db.png)

N.B: LoadBalancer type is too expensive to maintain. If you have 100 services, it is not cost-effective to have LoadBalancer for each of them. The best solution to use is the Cluster IP type with an ingress infront of it which is likely to be a LoadBalancer which will be used across all the services .running in the cluster

#### Step 4: Creating A Replica Set

ReplicaSet ensures desired number of Pods is always running to achieve availability in case one or two pods dies (e.g error, Node reboot or some other reason).

Deleting the nginx-pod:````kubectl delete pod nginx-pod````

Creating the replicaSet manifest file and applying it:````$ kubectl apply -f rs.yaml````

````
#Part 1 (actual replicaset creation)
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx-pod 
spec:
  # modify replicaset according to your case
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
#Part 2 (pod template used by replicaset to create pod)
  template:
    metadata:
      labels:
         app: nginx-pod
    spec:
      containers:
      - image: nginx:latest
        name: nginx
 ````
     

* To inspecting the replicaset setup:

````
$ kubectl get pods

$ kubectl get rs -o wide
````

![image](https://user-images.githubusercontent.com/87030990/188937180-3376e959-ca0e-408d-ac92-7249d551ef18.png)


* To verify that replicaset creation is working, two of the pods were deleted which led to the creation of two new pods:````$ kubectl delete pod nginx-rs-8xqvn nginx-rs-l9mlv -n obafema````

![image](https://user-images.githubusercontent.com/87030990/188939072-5e531624-953a-4d8e-996d-f1d601a42845.png)
![image](https://user-images.githubusercontent.com/87030990/188938594-5349136a-aa0d-4fd7-8cc9-1e531a77bfa1.png)

Note: ReplicaSet understands which Pods to create by using SELECTOR key-value pair.

* Scale ReplicaSet up and down:

There are two approaches of Kubernetes Object Management (scaling pods up or down): imperative and declarative: Imperative and Declarative

**Imperative method** is by running a command on the CLI: ````$ kubectl scale --replicas 5 replicaset nginx-rs```` to scale up

![image](https://user-images.githubusercontent.com/87030990/188942704-add86d56-4405-40a2-8bc5-552757fa9362.png)

* Scaling down will work the same way, so scale it down from 5 to 3 replicas for example using ````$ kubectl scale --replicas 3 replicaset nginx-rs````

![image](https://user-images.githubusercontent.com/87030990/188943557-3254227e-9ce5-4622-804e-b0951e75312f.png)



**Declarative method** is done by editing the **rs.yaml** manifest and changing to the desired number of replicas and applying the update

````
spec:
  replicas: 3
````

and applying the updated manifest:

````
kubectl apply -f rs.yaml
````

Note: There is another method – ‘ad-hoc’, it is definitely not the best practice and we do not recommend using it, but you can edit an existing ReplicaSet with following command:

````
kubectl edit -f rs.yaml
````

#### Step 5: Creating Deployment

A Deployment is another layer above ReplicaSets and Pods, It manages the deployment of ReplicaSets and allows for easy updating of a ReplicaSet as well as the ability to roll back to a previous version of deployment. It is declarative and can be used for rolling updates of micro-services, ensuring there is no downtime.

Steps to creating Deployment

The ReplicaSet that was created earlier were deleted: ````$ kubectl delete rs nginx-rs````

Deployment manifest file called **deployment.yaml** was developed and subsequently applied using:````$ kubectl apply -f deployment.yaml````

Deployment.yaml manifest

````

* Inspecting the setup:

````
$ kubectl get po

$ kubectl get deploy

$ kubectl get rs
````
![image](https://user-images.githubusercontent.com/87030990/188970318-a96b1093-f36e-4907-a3ba-49d05e49a72e.png)

* Replicas in the Deployment was scaled up to 15 Pods:

![image](https://user-images.githubusercontent.com/87030990/188971264-41c023fa-f5d4-4b20-937a-cdafcb1191fd.png)

* To exec into one of the pods:````kubectl exec nginx-deployment-6fdcffd8fc-x57f9 -i -t bash````

* List the files and folders in the Nginx directory

![image](https://user-images.githubusercontent.com/87030990/188973352-fc1a8535-dd27-44a6-8816-fcb86d102b06.png)

* Check the content of the default Nginx configuration file

![image](https://user-images.githubusercontent.com/87030990/188973834-a4ec3e1f-059f-4674-bd05-099f8330556d.png)

#### Step 6: Persisting Data for PODS

N.B: Deployments are stateless by design. Hence, any data stored inside the Pod’s container does not persist when the Pod dies. If you were to update the content of the **index.html** file inside the container, and the Pod dies, that content will not be lost since a new Pod will replace the dead one.

* Scale the Pods down to 1 replica

* Exec into the running container and install vim to be able to edit the index.html file

![image](https://user-images.githubusercontent.com/87030990/188976492-00ffaa08-2317-4f7c-8aa2-14967255e689.png)


Step 7: Deploying Tooling Application With Kubernetes

The tooling application that was containerised with Docker on Project 20, the following shows how the image is pulled and deployed as pods in Kubernetes:

Creating deployment manifest file for the tooling aplication called **tooling-deploy.yaml** and applying it:

````
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tooling-deployment
  labels:
    app: tooling-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tooling-app
  template:
    metadata:
      labels:
        app: tooling-app
    spec:
      containers:
      - name: tooling
        image: somex6/tooling:0.0.1
        ports:
        - containerPort: 80
   ````
   
   Creating Service manifest file for the tooling aplication called tooling-service.yaml and applying it
   
 ````
 apiVersion: v1
kind: Service
metadata:
  name: tooling-service
spec:
  selector:
    app: tooling-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
 ````
 
 
* Creating deployment manifest file for the MySQL database application called mysql-deploy.yaml and applying it

````
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mysql-deployment
 labels:
   tier: mysql-db
spec:
 replicas: 1
 selector:
   matchLabels:
     tier: mysql-db
 template:
   metadata:
     labels:
       tier: mysql-db
   spec:
     containers:
     - name: mysql
       image: mysql:5.7
       env:
       - name: MYSQL_DATABASE
         value: toolingdb
       - name: MYSQL_USER
         value: obafema
       - name: MYSQL_PASSWORD
         value: password123
       - name: MYSQL_ROOT_PASSWORD
         value: password1234
       ports:
       - containerPort: 3306
````
