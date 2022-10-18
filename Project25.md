### DEPLOYING AND PACKAGING APPLICATIONS INTO KUBERNETES WITH HELM

**Objective:** To deploy DevOps tools, toubleshoote faults, tweak helm values files to automate the configuration of the applications deployed and how to use the deployed tools and relate with the DevOps cycle and how they fit into the entire ecosystem.

In this project **Jfrog Artifactory** is used as a private registry for the organisation’s Docker images and Helm charts to satisfy company’s corporate security policies 

Prerequisute:

* Create EKS Cluster 

* Create **kubeconfig file** using awscli: ````aws eks update-kubecofig --name tooling-app-eks --region us-east-2 --kubeconfig kubeconfig````

![image](https://user-images.githubusercontent.com/87030990/195843577-1b89b2db-b867-48a3-84f4-111dc5a9c1f6.png)


* While trying to connect to the cluster to check nodes, I gotthe error below so the **kubeconfig file** for the cluster was updated using: ````aws eks --region us-east-2 update-kubeconfig --name tooling-app-eks````

![image](https://user-images.githubusercontent.com/87030990/195843775-46a04d13-279e-4911-aadf-686f9413386b.png)


* Number nodes created was confirmed using: ````kubectl get nodes````

![image](https://user-images.githubusercontent.com/87030990/195843978-db9ec051-b429-4330-89c9-d3fc348c859f.png)


#### Step 1: Deploy Jfrog Artifactory into Kubernetes

The best approach to easily get Artifactory into kubernetes is to use helm.

* Add the jfrog remote repository on your laptop/computer ````helm repo add jfrog https://charts.jfrog.io````

* namespace called **tools** was created for deployment of Devops tools: ````kubectl create ns tools````
 
![image](https://user-images.githubusercontent.com/87030990/195846309-bbb619a1-d1e9-4497-b6f1-9f82fca2a358.png)

* Update the helm repo index on your laptop/computer: ````helm repo update````

![image](https://user-images.githubusercontent.com/87030990/195846556-adda2657-ed20-49bf-b34e-3a8d24516aab.png)

* Install artifactory in the newly created namespace: ````helm upgrade --install artifactory jfrog/artifactory --version 107.38.10 -n tools````

![image](https://user-images.githubusercontent.com/87030990/195847462-411a2546-96f8-468c-adc3-1ca16e64eab3.png)


Checking the pod status: ````kubectl get pod -n tools````

![image](https://user-images.githubusercontent.com/87030990/196150419-b1bfacbe-a5c5-437e-b961-87f84f13eb97.png)

Artifactory was uninstalled and then downgraded by installing a lower version. The pod is running but not ready.

* Doing ````kubectl describe po artifactory-0 -n tools```` returned "startup probe failed" warning

![image](https://user-images.githubusercontent.com/87030990/196151300-2d930623-43ed-442c-9852-606afde616ad.png)


![image](https://user-images.githubusercontent.com/87030990/196150249-7e327ce1-a101-45e3-98f9-2030f4259287.png)


#### Step 2: Deploy Nginx Ingress Controller

Note: This controller is maintained by Kubernetes, there is an official guide the installation process. Hence, **artifacthub.io** was not used for the deployment.

* Install Nginx Ingress Controller in the ingress-nginx namespace

````
helm upgrade --install ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx --create-namespace
````

![image](https://user-images.githubusercontent.com/87030990/196229192-cdadda4b-cf64-4079-b0d0-5b317979e91f.png)

* A few pods should start in the ingress-nginx namespace:````kubectl get pods --namespace=ingress-nginx````

* After a while, they should all be running. The following command will wait for the ingress controller pod to be up, running, and ready:

````
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
````

* Check to see the created load balancer in AWS. ````kubectl get service -n ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/196230736-42f51273-5706-4190-abe9-ede634f76304.png)

* Check the IngressClass that identifies this ingress controller: ````kubectl get ingressclass -n ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/196232174-30b8c7ff-19b3-4313-9f7d-475097c49a80.png)






