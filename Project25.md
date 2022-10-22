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

* The instance type was then scaled up from **t3.medium** to **t3.large** for improved performance and artifactory-0 became ready

![image](https://user-images.githubusercontent.com/87030990/196540443-6b8dceb1-608d-47e9-88b5-7cbc6512cbd6.png)

* artifactory-nginx however was still not ready.

![image](https://user-images.githubusercontent.com/87030990/197289528-56dcb257-848d-4972-a415-4122ea34bdb4.png)

* Kubectl describe done on the artifactory-nginx pod returned **startup probe failed** error but no output returned with ````kubectl logs````

![image](https://user-images.githubusercontent.com/87030990/197289954-5582346e-80de-4588-aab5-43c4c4d2082c.png)
![image](https://user-images.githubusercontent.com/87030990/197290050-37e3b877-b01f-4014-894f-18c1e68c7808.png)

* Further troubleshooting was carried out by login in to the container inside the **artifact-nginx** pod. Connection was terminated abruptly wjile returning error code 137 which is related to memory issue:

* The instance type again scaled up from **t3.large** to **t3.2xlarge** for improved performance and artifactory-nginx became ready

![image](https://user-images.githubusercontent.com/87030990/197291057-ef347747-222d-4697-b935-3e1cf8abb001.png)

*  The command ````kubectl get svc artifactory-artifactory-nginx -n tools```` was ran to retrieve the Load Balancer URL

![image](https://user-images.githubusercontent.com/87030990/197291586-4054019b-71e5-49c1-8483-5f0b0562d746.png)

* Artifactory was accessed using the Load Balancer URL and the default username and password:

![image](https://user-images.githubusercontent.com/87030990/197291775-a3d7f99e-0169-48f4-a25e-08fa8c796eb8.png)
![image](https://user-images.githubusercontent.com/87030990/197292152-e18a4447-6c8b-45d8-9ee3-748e30590c22.png)


Setting the service type to **Load Balancer** is the easiest way to get started with exposing applications running in kubernetes externally. But provissioning load balancers for each application can become very expensive over time, and more difficult to manage especially when tens or even hundreds of applications are deployed.

* Let us explore **ClusterIP** service type which is relatively cheaper to run

* To get the postgresql paasword: ````POSTGRES_PASSWORD=$(kubectl get secret -n tools artifactory-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)```` was ran

![image](https://user-images.githubusercontent.com/87030990/197293829-615f85ed-8fc8-4a77-be7e-2eb3a6d52fab.png)


* Convert Artifactory-nginx service-type from Load Balancer to ClusterIP

```` helm upgrade artifactory jfrog/artifactory --set postgresql.postgresqlPassword=${POSTGRES_PASSWORD} --namespace tools --set databaseUpgradeReady=true --set nginx.service.type=ClusterIP````

![image](https://user-images.githubusercontent.com/87030990/197293896-20071af0-7cbd-475f-a95a-3bbfef3e4f7f.png)

* Port forwarding to access Artifactory from the UI was down: ````kubectl port-forward svc/artifactory 8082:8082 -n tools````

![image](https://user-images.githubusercontent.com/87030990/197293975-9983b8fe-f2cd-4fa9-baa7-a7be80e51ac6.png)

* Artifactory was accessed using ````http://127.0.0.1:8082/ or http://localhost:8082 `````

![image](https://user-images.githubusercontent.com/87030990/197294101-3917c767-9862-4dce-b540-ab86f270071f.png)
![image](https://user-images.githubusercontent.com/87030990/197294217-b8c53fac-b8af-4463-8a3c-5bbadcc9cc20.png)



#### Step 2: Deploy Nginx Ingress Controller and managing Ingress Resources

* Setting the service type to **Load Balancer** is the easiest way to expose application running in Kubernetes externally, the best approach is to use **Kubernetes Ingress** instead. But to do that, we will have to deploy an **Ingress Controller**. A huge benefit of using the **ingress controller** is that we will be able to use a **single load balancer** for different applications we deploy.


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

* Add repo: ````helm repo add bitnami https://charts.bitnami.com/bitnami````

![image](https://user-images.githubusercontent.com/87030990/197180729-c12d7603-6e40-4dbd-8d73-9ee773df9df3.png)

* Update repo if required: ````helm repo update````

* Install Chart: ````````helm install nginx-ingress-controller bitnami/nginx-ingress-controller -n ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/197295504-c77058d9-ca72-4a23-836e-83ff75408763.png)

* A few pods should start in the ingress-nginx namespace: ````kubectl get pods --namespace=ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/197295567-5fe85958-be5e-4afc-a890-6ab9394c764b.png)

* After a while, they should all be running. The following command will wait for the ingress controller pod to be up, running, and ready:

````
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
````

* Check to see the created load balancer in AWS. ````kubectl get service -n ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/197295660-7f6657ef-5848-4ba9-9cb5-09924e07de58.png)
![image](https://user-images.githubusercontent.com/87030990/197295756-b764863d-cf97-474b-9e32-bae0d1891041.png)

Note: The ingress-nginx-controller service that was created is of the type LoadBalancer. That will be the load balancer to be used by all applications which require external access, and is using this ingress controller.

* Check the IngressClass that identifies this ingress controller: ````kubectl get ingressclass -n ingress-nginx````

![image](https://user-images.githubusercontent.com/87030990/197182189-7d7b01d3-9773-4427-bc49-ade6ded9c7b3.png)


#### Step 3: Deploy Artifactory Ingress

* Artifactory Ingress was configured for traffic to be routed to the Artifactory internal service through the ingress controller’s load balancer

* Artifactory-ingress.yaml file was created for deployment of artifactory ingress

````
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: artifactory
spec:
  ingressClassName: nginx
  rules:
  - host: "artifactory.agoone.link"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 8082
 ````
 
 * Artifactory ingress was deployed using ````kubectl apply -f artifactory-ingress.yaml -n tools````

#### Step 3: Register Domain and Configure DNS

A domain named agoone.link was registered and Route53 records with subdomain **artifactory** added was created to the ingress controller’s loadbalancer

![image](https://user-images.githubusercontent.com/87030990/197337507-db215942-46bb-4225-ba96-909fc10fc046.png)

* DNS record was confirmed to be properly propergated by visiting https://dnschecker.org

![image](https://user-images.githubusercontent.com/87030990/197337417-676fc68e-92b6-429c-8ecd-03aa8191ebe6.png)


* Visiting the application from the browser

On Chrome browser

The website is reachable but insecured 

![image](https://user-images.githubusercontent.com/87030990/197337401-8a77b3b1-88b9-473f-aafa-d352a606b1b7.png)


* Nginx Ingress Controller does configure a default TLS/SSL certificate which is not trusted because it is a self signed certificate that browsers are not aware of.

![image](https://user-images.githubusercontent.com/87030990/197345164-cc2fbd8b-038c-4180-8a4f-3347ba25cc61.png)
![image](https://user-images.githubusercontent.com/87030990/197345226-5e6d7a32-a647-4aae-8a31-36fd2af8683c.png)


* Explore Artifactory Web UI

* Get the default username and password using: ````helm test artifactory -n tools````

![image](https://user-images.githubusercontent.com/87030990/197337920-97513a3b-9aab-47da-8a6b-44b1f805280d.png)

* Insert the username and password to Get Started:

![image](https://user-images.githubusercontent.com/87030990/197337493-d43039d6-dd33-49a9-b903-cbcd9c16e9b3.png)


* Change your admin password to your choice, Activate Artifactory Licence, Set the Base URL and Ensure to use https. Skip other configuration part. 

![image](https://user-images.githubusercontent.com/87030990/197338464-258360ae-d12c-4108-846f-b5286ddcbea2.png)

![image](https://user-images.githubusercontent.com/87030990/197338500-8442c160-3f25-423d-b9a0-c8ad03967682.png)


#### Step 4: Deploying Cert-Manager and Managing TLS/SSL for Ingress

* Deploying Cert-manager

* Installing the Chart

* Install first, the **cert-manager CustomResourceDefinition** resources that helps to uninstall and reinstall cert-manager without deleting your installed custom resources. ````$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.crds.yaml````

![image](https://user-images.githubusercontent.com/87030990/197343637-bed1c3fa-79fd-4967-8ee7-5e232c082005.png)

* To install the chart

* Add cert-manager repo: ````helm repo add jetstack https://charts.jetstack.io````

* Update repo: ````helm repo update````

* Install CustomResourceDefinitions: ````kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.crds.yaml````

![image](https://user-images.githubusercontent.com/87030990/197352587-effbf402-5a5a-4e26-8934-2324af05f8bb.png)


* Install the cert-manager helm chart:

````
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.10.0
````

![image](https://user-images.githubusercontent.com/87030990/197356381-e8c44bc4-f1f6-4fb1-b527-906fc22d1f8b.png)


* Certificate Issuer. Cluster Issuer was used so it could be scoped globally

![image](https://user-images.githubusercontent.com/87030990/197346550-620d76d0-3a46-4ba9-b06e-166e847754c0.png)


#### Step 5: Configuring Ingress for TLS

* Update the Artifactory Ingress manifest with TLS specific configurations and redeploy.

````
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    kubernetes.io/ingress.class: nginx
  name: artifactory
spec:
  rules:
  - host: "artifactory.agoone.link"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 8082
  tls:
  - hosts:
    - "artifactory.agoone.link"
    secretName: "artifactory.agoone.link"
 ````
 ![image](https://user-images.githubusercontent.com/87030990/197347024-933a703e-9531-4b44-acbf-b766864f3d70.png)

* Run ````kubectl get certificate```` to verify deployment

![image](https://user-images.githubusercontent.com/87030990/197360187-4602842b-7435-4e44-854e-99931fdfb0d3.png)

 
![image](https://user-images.githubusercontent.com/87030990/197361906-cecb5d5b-bc7f-4f51-8dca-e8bb0bdc35e3.png)

![image](https://user-images.githubusercontent.com/87030990/197361931-b02f498b-effc-4940-8c74-75f7a0a645e8.png)

![image](https://user-images.githubusercontent.com/87030990/197361974-8d198577-414f-4911-ab2c-30aceccaee82.png)




