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







