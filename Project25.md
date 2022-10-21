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

![image](https://user-images.githubusercontent.com/87030990/196551578-5581a956-d119-4fe5-bbe3-50897ee00e5b.png)


#### Step 1: Deploy Jfrog Artifactory into Kubernetes

* In this project, Artifactory is deployed to serve as a private docker registry and repository for private helm charts.

* The best approach to easily get Artifactory into kubernetes is to use helm.

* Add the jfrog remote repository on your laptop/computer ````helm repo add jfrog https://charts.jfrog.io````

* namespace called **tools** was created for deployment of Devops tools: ````kubectl create ns tools````
 
![image](https://user-images.githubusercontent.com/87030990/195846309-bbb619a1-d1e9-4497-b6f1-9f82fca2a358.png)

* Update the helm repo index on your laptop/computer: ````helm repo update````

![image](https://user-images.githubusercontent.com/87030990/195846556-adda2657-ed20-49bf-b34e-3a8d24516aab.png)

* Install artifactory in the newly created namespace: ````helm upgrade --install artifactory jfrog/artifactory --version 107.38.10 -n tools````

![image](https://user-images.githubusercontent.com/87030990/195847462-411a2546-96f8-468c-adc3-1ca16e64eab3.png)


* Checking the pod status: ````kubectl get pod -n tools````

![image](https://user-images.githubusercontent.com/87030990/196150419-b1bfacbe-a5c5-437e-b961-87f84f13eb97.png)

* Running kubectl logs on the pod returned the output below:

````
kubectl logs artifactory-0 -n tools
Defaulted container "artifactory" out of: artifactory, delete-db-properties (init), remove-lost-found (init), copy-system-yaml (init), wait-for-db (init), migration-artifactory (init)
````

* Artifactory was uninstalled and re-installed. The pod was running but not ready.

![image](https://user-images.githubusercontent.com/87030990/196150249-7e327ce1-a101-45e3-98f9-2030f4259287.png)

* Doing ````kubectl describe po artifactory-0 -n tools```` returned "startup probe failed" warning

![image](https://user-images.githubusercontent.com/87030990/196151300-2d930623-43ed-442c-9852-606afde616ad.png)


* Doing kubectl logs on the pod returned connection refused. One of the two nodes the **artifactory-0** and **artifactory-postgresql-0** pods are running on goes to **NotReady** State and the pods get terminated afterwards and then get recreated:

![image](https://user-images.githubusercontent.com/87030990/196545077-16859dd7-5bf0-47fb-b314-da7499010309.png)

![image](https://user-images.githubusercontent.com/87030990/196546262-120dab0b-e399-4011-972a-ba4606ee49e0.png)

![image](https://user-images.githubusercontent.com/87030990/196546316-10b277d6-84db-4992-be94-f50ac71eb427.png)

* Doing kubectl describe node on the node that was failing intermettently returned **InSufficientMemory** error. 

* The instance type was then scaled up from **t3.medium** to **t3.large** for performance and artifactory-0 became ready

![image](https://user-images.githubusercontent.com/87030990/196540443-6b8dceb1-608d-47e9-88b5-7cbc6512cbd6.png)

* artifactory-artifactory-nginx pod was converted to ClusterIP service type from Load Balancer service type to achieve cost reduction strategy

* To get the postgresql paasword: ````POSTGRES_PASSWORD=$(kubectl get secret -n tools artifactory-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)```` was ran

* Convert Artifactory-nginx service-type from Load Balancer to ClusterIP

```` helm upgrade artifactory jfrog/artifactory --set postgresql.postgresqlPassword=${POSTGRES_PASSWORD} --namespace tools --set databaseUpgradeReady=true --set nginx.service.type=ClusterIP````

![image](https://user-images.githubusercontent.com/87030990/196540649-466b50a7-fb04-4143-ba3b-69c056e641d2.png)
![image](https://user-images.githubusercontent.com/87030990/196540772-d50e48ec-80ee-4bc0-918c-4e6a4af9e23f.png)

* Port forwarding to access Artifactory from the UI was down: ````kubectl port-forward svc/artifactory 8082:8082 -n tools````

![image](https://user-images.githubusercontent.com/87030990/196540943-04c5e997-410a-418a-b20b-8b0f30a3c60b.png)

* Artifactory was accessed using ````http://127.0.0.1:8082/`````

![image](https://user-images.githubusercontent.com/87030990/196534156-87f4687b-998f-4212-8b41-b105d1af2d8a.png)


![image](https://user-images.githubusercontent.com/87030990/196537110-3c48c862-41fa-4404-9f34-94dff5693b73.png)



#### Step 2: Deploy Nginx Ingress Controller

* Setting the service type to **Load Balancer** is the easiest way to get started with exposing applications running in kubernetes externally. But provissioning load balancers for each application can become very expensive over time, and more difficult to manage. Especially when tens or even hundreds of applications are deployed.

* The best approach is to use **Kubernetes Ingress** instead. But to do that, we will have to deploy an **Ingress Controller**. A huge benefit of using the **ingress controller** is that we will be able to use a **single load balancer** for different applications we deploy.

Note: This controller is maintained by Kubernetes, there is an official guide the installation process. Hence, **artifacthub.io** was not used for the deployment. 

* Install Nginx Ingress Controller in the ingress-nginx namespace

````
helm upgrade --install ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx --create-namespace
````

![image](https://user-images.githubusercontent.com/87030990/196229192-cdadda4b-cf64-4079-b0d0-5b317979e91f.png)

* A few pods should start in the ingress-nginx namespace:````kubectl get pods --namespace=ingress-nginx````

#### Self Challenge Task 

* Delete the above ingress-nginx installation and then re-install it using artifactory hub:

![image](https://user-images.githubusercontent.com/87030990/197180419-035b7e8b-3224-4860-9252-f48b94aefd28.png)
![image](https://user-images.githubusercontent.com/87030990/197180585-bb521eca-9de1-41d5-8cef-7b8304d5fd5a.png)

* Add repo: ````helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/197180729-c12d7603-6e40-4dbd-8d73-9ee773df9df3.png)

* Update repo if required: ````helm repo update````

* Install Chart: ````helm upgrade --install my-ingress-nginx ingress-nginx/ingress-nginx --version 4.3.0 -n ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/197180937-ca047d1f-8568-415e-868b-49a30b200584.png)

* A few pods should start in the ingress-nginx namespace: ````kubectl get pods --namespace=ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/197181267-d6c127eb-fd2b-49a5-8505-d67507ddf750.png)

* After a while, they should all be running. The following command will wait for the ingress controller pod to be up, running, and ready:

````
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
````

* Check to see the created load balancer in AWS. ````kubectl get service -n ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/197181768-de4843c9-dc37-4a82-98b1-c3c9bf1924b6.png)
![image](https://user-images.githubusercontent.com/87030990/197181573-1ba73f2b-7dd6-4096-99d1-3495d7a08a84.png)

Note: The ingress-nginx-controller service that was created is of the type LoadBalancer. That will be the load balancer to be used by all applications which require external access, and is using this ingress controller.

* Check the IngressClass that identifies this ingress controller: ````kubectl get ingressclass -n ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/197182189-7d7b01d3-9773-4427-bc49-ade6ded9c7b3.png)


#### Step 3: Deploy Artifactory Ingress






