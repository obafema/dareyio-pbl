## Experience Continuous Integration with Jenkins | Ansible | Artifactory | Sonarqube | PHP

### Objective: To create a pipeline that simulates continuous integration and delivery for a PHP based application

### Step 1: Spin up an EC2 instance to serve as Jenkins_Ansible server and clone ansible-config-mgt repository from github on it.

#### An EC2 instance for installation of Jenkins and Ansible was spinned up and named P14_Jenkins_Ansible:

![image](https://user-images.githubusercontent.com/87030990/165818583-a40885bd-2f9f-4476-ac22-ccbb39545861.png)

git was installed on the server

![image](https://user-images.githubusercontent.com/87030990/165818664-100fb922-2f44-4515-9b08-9c31bc534c95.png)

ansible-config-mgt repository from github was cloned to the Jenkins-Ansible server

![image](https://user-images.githubusercontent.com/87030990/165818723-b7f974e5-6b07-4d04-a9a2-2c8a3e05dcb2.png)

### Step 2: Configuring Ansible For Jenkins Deployment

#### Before installing Jenkins on the server, Jenkins Redhat Packages and Jenkins dependencies (epel release, remirepository and Java) were installed on the EC2 instance serving as Jenkins_ansible server.

N.B: Jenkins requires Java to run, yet certain distributions do not include this by default.

#### wget, Jenkins Redhat Packages, epel release, remirepository and Java were installed on P14_Jenkins_Ansible EC2 instance
````bash
sudo wget -O /etc/yum.repos.d/jenkins.repo\https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade
# Add required dependencies for the jenkins package
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum install java-11-openjdk
sudo yum install jenkins
sudo systemctl daemon-reload
````

![image](https://user-images.githubusercontent.com/87030990/165819144-2dbd0cfb-823f-4cae-9a10-1bdc16c8946e.png)

#### epel release was installed
````bash
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
````

![image](https://user-images.githubusercontent.com/87030990/165819386-ac028f0e-f1fd-42c8-8f06-b11583e0e945.png)

#### remirepository also installed
````bash
yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
````

![image](https://user-images.githubusercontent.com/87030990/165819492-8384afc2-0b35-4251-895d-7fb1a53f9875.png)

#### Java was equally installed

````bash
sudo yum install java-11-openjdk
````

![image](https://user-images.githubusercontent.com/87030990/165819854-57650df0-397b-4490-92fb-79856c0ef8c8.png)

#### bash profile was opened after installing java so that at any given time the instance is restarted this code is also export

![image](https://user-images.githubusercontent.com/87030990/165819987-477dd668-327d-4b6e-8141-ca8d69cfb55e.png)

#### bash profile was then reloaded

````bash
source ~/.bash_profile
````
![image](https://user-images.githubusercontent.com/87030990/165820136-1e88f50b-7457-4b0e-b58b-c03efcb7e0f4.png)

#### Jenkins was then installed on the instance after installing the dependencies.

````bash
sudo yum install jenkins
````
![image](https://user-images.githubusercontent.com/87030990/165820347-c772584f-c103-4726-852c-c6c5459871b0.png)

#### Jenkins was then started and enabled. The status was confirmed to be active and running and was reloaded
![image](https://user-images.githubusercontent.com/87030990/165820468-504843db-385e-4c64-8a15-9d6a7cc58f25.png)

#### Jenkins was launched by navigating to Jenkins URL, unlocked by providing the password and suggested plugins installed

![image](https://user-images.githubusercontent.com/87030990/165820563-9ae1d2ba-e15f-4f0d-aa27-c10f48f817b4.png)

#### Blue Ocean Jenkins Plugin was install and opened to create a pipeline

![image](https://user-images.githubusercontent.com/87030990/165820646-431de417-98d6-4b19-8cb7-b927db13b3d6.png)
![image](https://user-images.githubusercontent.com/87030990/165820707-ecd8ad1b-ca55-4c01-96d8-a295cfd27bd2.png)

#### Access Token from GitHub was generated, copied and pasted to connect Jenkins to GitHub

![image](https://user-images.githubusercontent.com/87030990/165820821-371e90a9-3c7e-48a2-ac41-11e3211b410e.png)

#### A pipeline was created after successful connection:
![image](https://user-images.githubusercontent.com/87030990/165820895-23c65f5e-cfd7-4cec-9cc8-bdc42d31a8f7.png)

#### click on Administration to exit the Blue Ocean console.

![image](https://user-images.githubusercontent.com/87030990/165821007-f2ca822d-5fb1-48a4-91f9-a38c21172a1a.png)

#### Jenkinsfile was then created inside a new directory named deploy within the ansible project. 
#### Code snippet below was then added to start building the Jenkinsfile gradually. 
#### Code snippet below was then added to start building the Jenkinsfile gradually. 

````javascript
pipeline {
    agent any

  stages {
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }
    }
}
````
![image](https://user-images.githubusercontent.com/87030990/165821587-dcde257c-df3a-448e-9c94-693168f85ba1.png)

#### Changes to deploy/Jenkinsfile were added, committed and pushed to GitHub.

````bash
git add .
git commit -m "add jenkinsfile"
git push
````
#### In the ansible-config-mgt pipeline’s Build Configuration section in Jenkins, the location of the Jenkinsfile was specified:

![image](https://user-images.githubusercontent.com/87030990/165821802-32782839-9ce1-48d7-b348-9a84388674e6.png)

#### The build job started automatically after applying and saving the script path

![image](https://user-images.githubusercontent.com/87030990/165821903-267195ec-2642-4c2b-b159-b444da95ed85.png)

#### Note: To really appreciate and feel the difference of Cloud Blue UI, it is recommended to try triggering the build again from Blue Ocean interface.

#### Click on Blue Ocean, Select ansible-config-mgt project and Click on the play button against the branch

![image](https://user-images.githubusercontent.com/87030990/165822054-d9d12475-9ecc-4487-9c05-7137b91f7f19.png)

