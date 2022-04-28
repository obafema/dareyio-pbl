## Experience Continuous Integration with Jenkins | Ansible | Artifactory | Sonarqube | PHP

Objective: To create a pipeline that simulates continuous integration and delivery for a PHP based application

### Step 1: Spin up an EC2 instance to serve as Jenkins_Ansible server and clone ansible-config-mgt repository from github on it.

An EC2 instance for installation of Jenkins and Ansible was spinned up and named P14_Jenkins_Ansible:

![image](https://user-images.githubusercontent.com/87030990/165818583-a40885bd-2f9f-4476-ac22-ccbb39545861.png)

git was installed on the server

![image](https://user-images.githubusercontent.com/87030990/165818664-100fb922-2f44-4515-9b08-9c31bc534c95.png)

ansible-config-mgt repository from github was cloned to the Jenkins-Ansible server

![image](https://user-images.githubusercontent.com/87030990/165818723-b7f974e5-6b07-4d04-a9a2-2c8a3e05dcb2.png)

### Step 2: Configuring Ansible For Jenkins Deployment

Before installing Jenkins on the server, Jenkins Redhat Packages and Jenkins dependencies (epel release, remirepository and Java) were installed on the EC2 instance serving as Jenkins_ansible server.

N.B: Jenkins requires Java to run, yet certain distributions do not include this by default.

wget, Jenkins Redhat Packages, epel release, remirepository and Java were installed on P14_Jenkins_Ansible EC2 instance
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

epel release was installed
````bash
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
````

![image](https://user-images.githubusercontent.com/87030990/165819386-ac028f0e-f1fd-42c8-8f06-b11583e0e945.png)

remirepository also installed
````bash
yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
````

![image](https://user-images.githubusercontent.com/87030990/165819492-8384afc2-0b35-4251-895d-7fb1a53f9875.png)

Java was equally installed

````bash
sudo yum install java-11-openjdk
````

![image](https://user-images.githubusercontent.com/87030990/165819854-57650df0-397b-4490-92fb-79856c0ef8c8.png)

bash profile was opened after installing java so that at any given time the instance is restarted this code is also export

![image](https://user-images.githubusercontent.com/87030990/165819987-477dd668-327d-4b6e-8141-ca8d69cfb55e.png)

bash profile was then reloaded

````bash
source ~/.bash_profile
````
![image](https://user-images.githubusercontent.com/87030990/165820136-1e88f50b-7457-4b0e-b58b-c03efcb7e0f4.png)

Jenkins was then installed on the instance after installing the dependencies.

````bash
sudo yum install jenkins
````
![image](https://user-images.githubusercontent.com/87030990/165820347-c772584f-c103-4726-852c-c6c5459871b0.png)

Jenkins was then started and enabled. The status was confirmed to be active and running and was reloaded
![image](https://user-images.githubusercontent.com/87030990/165820468-504843db-385e-4c64-8a15-9d6a7cc58f25.png)

Jenkins was launched by navigating to Jenkins URL, unlocked by providing the password and suggested plugins installed

![image](https://user-images.githubusercontent.com/87030990/165820563-9ae1d2ba-e15f-4f0d-aa27-c10f48f817b4.png)

Blue Ocean Jenkins Plugin was install and opened to create a pipeline

![image](https://user-images.githubusercontent.com/87030990/165820646-431de417-98d6-4b19-8cb7-b927db13b3d6.png)
![image](https://user-images.githubusercontent.com/87030990/165820707-ecd8ad1b-ca55-4c01-96d8-a295cfd27bd2.png)

Access Token from GitHub was generated, copied and pasted to connect Jenkins to GitHub

![image](https://user-images.githubusercontent.com/87030990/165820821-371e90a9-3c7e-48a2-ac41-11e3211b410e.png)

A pipeline was created after successful connection:
![image](https://user-images.githubusercontent.com/87030990/165820895-23c65f5e-cfd7-4cec-9cc8-bdc42d31a8f7.png)

click on Administration to exit the Blue Ocean console.

![image](https://user-images.githubusercontent.com/87030990/165821007-f2ca822d-5fb1-48a4-91f9-a38c21172a1a.png)

Jenkinsfile was then created inside a new directory named deploy within the ansible project. 
Code snippet below was then added to start building the Jenkinsfile gradually. 

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

Changes to deploy/Jenkinsfile were added, committed and pushed to GitHub.

````bash
git add .
git commit -m "add jenkinsfile"
git push
````
In the ansible-config-mgt pipeline’s Build Configuration section in Jenkins, the location of the Jenkinsfile was specified:

![image](https://user-images.githubusercontent.com/87030990/165821802-32782839-9ce1-48d7-b348-9a84388674e6.png)

The build job started automatically after applying and saving the script path

![image](https://user-images.githubusercontent.com/87030990/165821903-267195ec-2642-4c2b-b159-b444da95ed85.png)

Note: To really appreciate and feel the difference of Cloud Blue UI, it is recommended to try triggering the build again from Blue Ocean interface.

Click on Blue Ocean, Select ansible-config-mgt project and Click on the play button against the branch

![image](https://user-images.githubusercontent.com/87030990/165822054-d9d12475-9ecc-4487-9c05-7137b91f7f19.png)

Notice that this pipeline is a multibranch one. This means, if there were more than one branch in GitHub, Jenkins would have scanned the repository to discover them all and we would have been able to trigger a build for each branch.

A new git branch named feature/jenkinspipeline-stages was created to serve as our working branch and to explore the multibranch capability of the pipeline

![image](https://user-images.githubusercontent.com/87030990/165822499-8382d7ba-20ea-4e1a-b15c-f8ff1b4c759d.png)

A new stage called Test was added by pasting the code snippet below and the changes pushed to GitHub.

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

    stage('Test') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }
    }
}
````

![image](https://user-images.githubusercontent.com/87030990/165822826-afc908bf-469b-49b3-b334-ae66702bf950.png)

````bash
git add .
git commit -m "add jenkinsfile"
git remote add origin https://github.com/<your-name>/ansible-config-mgt.git
git push --set-upstream origin feature/jenkinspipeline-stages
````

Note: To make your new branch show up in Jenkins, we need to tell Jenkins to scan the repository.
#### Click on the "Administration" button and then “Scan Repository Now”. Refresh the page to view the two branches

![image](https://user-images.githubusercontent.com/87030990/165823541-af83902f-d3be-4989-a27e-3f3685e61570.png)

Blue Ocean also has both branches too.

![image](https://user-images.githubusercontent.com/87030990/165823834-71bb1fa4-a715-4e5f-af9f-134b662cc579.png)

Below is how Jenkinsfile has caused a new step in the pipeline launch build for the new branch.

![image](https://user-images.githubusercontent.com/87030990/165824087-49e30b34-b60b-4eba-bd59-55d96baacc78.png)

The changes were merged to the main branch on GitHub and the latest changes pulled to the server.

More stages were added into the Jenkins file to simulate below phases.
   1. Package 
   2. Deploy 
   3. Clean up

Jenkinsfile for Quick Task
==================================

````javascript
pipeline {
    agent any

  stages {
    stage("Initial cleanup") {
          steps {
            dir("${WORKSPACE}") {
              deleteDir()
            }
          }
        }
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }

    stage('Test') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }

    stage('Package'){
      steps {
        script {
          sh 'echo "Packaging App" '
        }
      }
    }

    stage('Deploy'){
      steps {
        script {
          sh 'echo "Deploying to Dev"'
        }
      }

    }
    
    stage("clean Up"){
       steps {
        cleanWs()
     }
    }
     
    }
}
````
All the stages were verified to be working in Blue Ocean to have the pipeline below:

![image](https://user-images.githubusercontent.com/87030990/165827204-91b3a1b3-9327-41e9-85a8-905425b3971d.png)

### Running Ansible Playbook from Jenkins

To run Ansible Playbook from Jenkins, the followings were done:
* Ansible was installed on Jenkins
* Ansible plugin was Installed in Jenkins UI
* Jenkinsfile was created from scratch by deleting previous code snippet

Installing Ansible

````bash
sudo yum install ansible
````
![image](https://user-images.githubusercontent.com/87030990/165827720-51b4713d-e1da-4adb-9eb9-a2ace10443fb.png)

Then install dependencies that will make ansible playbook work

````bash
pythom3 -m pip install --upgrade setuptools
pythom3 -m pip install --upgrade pip
pythom3 -m pip install PyMySQL
pythom3 -m pip install mysql-connector-python
pythom3 -m pip install psycopg2==2.7.5 --ignore-installed
````

Install Ansible community

````bash
ansible-galaxy collection install community.postgresql
````

Installing Ansible plugin in Jenkins UI

![image](https://user-images.githubusercontent.com/87030990/165828097-49c559ac-6b36-4183-8af5-0ba3a6f4e57e.png)

Global Configuration was updated with details (Name and path to ansible executable directory)

![image](https://user-images.githubusercontent.com/87030990/165828170-06385be4-d1e9-46c4-b563-4568c6bda52b.png)

Credentials was added

![image](https://user-images.githubusercontent.com/87030990/165828243-a9db7cfa-2b1f-4080-a5ed-a12b889dc6b5.png)

Possible errors to watch out for:

* Ensure that the git module in Jenkinsfile is checking out SCM to main branch instead of master. 
Note: ansible.cfg file was put alongside Jenkinsfile in the deploy directory so Jenkins could export the ANSIBLE_CONFIG environment variable for Ansible to know where to find Roles. This way, anyone can easily identify that everything in there relates to deployment.  
Using the Pipeline Syntax tool in Ansible, the syntax to create environment variables to set was generated. But because you will possibly run Jenkins from different git branches, the location of Ansible roles will change. Linux Stream Editor (sed) was used to handle this dynamically to update the section roles_path each time there is an execution.

````javascript
stage('Prepare Ansible For Execution') {
            steps {
                sh 'echo ${WORKSPACE}' 
                sh 'sed -i "3 a roles_path=${WORKSPACE}/roles" ${WORKSPACE}/deploy/ansible.cfg'  
                sh 'export ANSIBLE_CONFIG=${WORKSPACE}/deploy/ansible.cfg'
            }
        }
````
*  Ensure that you start the Jenkinsfile with a clean up step to always delete the previous workspace before running a new one so Jenkins could download the latest code from GitHub

* Always verify that you are running a pipeline from right branch by logging onto the Jenkins box to check the workspace, and run git branch command to confirm that the branch you are expecting is there.

The Dev environment was confirmed to have an up-to-date configuration.

![image](https://user-images.githubusercontent.com/87030990/165828854-a6a783e0-ce94-41ff-98d4-05f9860307aa.png)

### Parameterizing Jenkinsfile For Ansible Deployment

To deploy to other environments, we will need to use parameters. Update sit inventory with new servers
````bash
[tooling]
<SIT-Tooling-Web-Server-Private-IP-Address>

[todo]
<SIT-Todo-Web-Server-Private-IP-Address>

[nginx]
<SIT-Nginx-Private-IP-Address>

[db:vars]
ansible_user=ec2-user
ansible_python_interpreter=/usr/bin/python

[db]
<SIT-DB-Server-Private-IP-Address>
````

pentest inventory file

````bash
[pentest:children]
pentest-todo
pentest-tooling

[pentest-todo]
<Pentest-for-Todo-Private-IP-Address>

[pentest-tooling]
<Pentest-for-Tooling-Private-IP-Address>
````

Update Jenkinsfile to introduce parameterization. Below is just one parameter. It has a default value in case if no value is specified at execution. It also has a description so that everyone is aware of its purpose.

````javascript
pipeline {
    agent any

    parameters {
      string(name: 'inventory', defaultValue: 'dev',  description: 'This is the inventory file for the environment to deploy configuration')
    }
...
````
![image](https://user-images.githubusercontent.com/87030990/165829374-a3709eab-d1bc-4761-80ab-dc5715450314.png)

In the Ansible execution section, the hardcoded inventory/dev was removed and replace with `${inventory}

![image](https://user-images.githubusercontent.com/87030990/165829497-c7fe1ad9-e5c6-4f46-b842-5b1d220e669f.png)

![image](https://user-images.githubusercontent.com/87030990/165829595-58420c3d-66fc-4d30-889a-5d0fd95be281.png)

### CI/CD Pipeline for TODO Application

#### Phase 1 – Prepare Jenkins

Fork the repository below into your GitHub account 
````bash
https://github.com/darey-devops/php-todo.git
````

Clone Todo repository to the Jenkins_Ansible server and add to your workspace

![image](https://user-images.githubusercontent.com/87030990/165829989-33cc031a-ec41-4c99-975b-ae9d3b04e716.png)

On you Jenkins server, PHP, its dependencies and Composer tool was installed

````bash
yum module reset php -y
yum module enable php:remi-7.4 -y
yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
systemctl start php-fpm
systemctl enable php-fpm
systemctl status php-fpm
````
![image](https://user-images.githubusercontent.com/87030990/165830090-56183bd3-c25b-4976-9074-ce75e1f4ab6e.png)

Composer was installed and moved to /bin so it could be run globally

![image](https://user-images.githubusercontent.com/87030990/165830174-da5b4083-c62f-48a7-b649-2f5dc17668b0.png)

Jenkins plugins (Plot plugin and artifactory) were installed

* Plot plugin
* Artifactory plugin

![image](https://user-images.githubusercontent.com/87030990/165830475-848f947c-69de-434d-8709-280656dd4248.png)

Note: Plot plugin was used to display tests reports, and code coverage information while the Artifactory plugin was used to easily upload code artifacts into an Artifactory server.

Spin up a new EC2 instance and install artifactory:

![image](https://user-images.githubusercontent.com/87030990/165830633-20e15cef-8836-486a-a7e1-4c7aba8f1a49.png)
![image](https://user-images.githubusercontent.com/87030990/165830720-1ddf84d4-0ca4-4796-aae9-53c9c440c7f2.png)

Launch artifactory on the browswer and also configured in Jenkins UI

![image](https://user-images.githubusercontent.com/87030990/165831066-23297c27-6f81-4061-9b8f-3d71a386979f.png)

#### Phase 2 – Integrate Artifactory repository with Jenkins

Create a dummy Jenkinsfile in the repository 

![image](https://user-images.githubusercontent.com/87030990/165831239-5ad85050-d7e8-48bc-9000-550453238999.png)

Using Blue Ocean, create a multibranch Jenkins pipeline 

On the database server, create database and user

Create database homestead;
CREATE USER 'homestead'@'%' IDENTIFIED BY 'pass';
GRANT ALL PRIVILEGES ON * . * TO 'homestead'@'%';

![image](https://user-images.githubusercontent.com/87030990/165831867-fa5bc9b3-298f-43dc-896e-b1982e316263.png)

mysql was installed on the Jenkins-Ansible server

![image](https://user-images.githubusercontent.com/87030990/165831950-2216adf3-1b9e-4916-9096-30e9f5bb1f2c.png)

mysql server’s bind address was set to 0.0.0.0 and the server restarted

![image](https://user-images.githubusercontent.com/87030990/165832036-74c447bc-c328-4a13-8925-ed167a3ee795.png)

DB connection inside the php-todo file was updated

![image](https://user-images.githubusercontent.com/87030990/165832116-6f6151a5-5a04-4d32-856b-dda8f86bd7d3.png)
![image](https://user-images.githubusercontent.com/87030990/165832165-a24b1619-6789-463e-8ac0-85b5942f6e5c.png)

Jenkinsfile was updated with proper pipeline configuration

````javascript
pipeline {
    agent any

  stages {

     stage("Initial cleanup") {
          steps {
            dir("${WORKSPACE}") {
              deleteDir()
            }
          }
        }

    stage('Checkout SCM') {
      steps {
            git branch: 'main', url: 'https://github.com/darey-devops/php-todo.git'
      }
    }

    stage('Prepare Dependencies') {
      steps {
             sh 'mv .env.sample .env'
             sh 'composer install'
             sh 'php artisan migrate'
             sh 'php artisan db:seed'
             sh 'php artisan key:generate'
      }
    }
  }
}
````

![image](https://user-images.githubusercontent.com/87030990/165832318-574052c9-13c2-4a9b-a80c-718dccb7bb8d.png)

Notice the Prepare Dependencies section
* The required file by PHP is.env so we are renaming.env.sample to.env 
* Composer is used by PHP to install all the dependent libraries used by the application
* php artisan uses the.env file to setup the required database objects – (After successful run of this step, login to the database, run show tables and you will see the tables being created for you)
 
Jenkinsfile was updated to include Unit tests step

````javascript
   stage('Execute Unit Tests') {
      steps {
             sh './vendor/bin/phpunit'
      }
````

![image](https://user-images.githubusercontent.com/87030990/165832617-753e0051-fc19-4e86-8841-e502c0c0e721.png)

#### Phase 3 – Code Quality Analysis

This is one of the areas where developers, architects and many stakeholders are mostly interested in as far as product development is concerned. As a DevOps engineer, you also have a role to play. Especially when it comes to setting up the tools.

#### Install phpunit, phploc and x-debug
````bash
sudo dnf --enablerepo=remi install php-phpunit-phploc
wget -O phpunit https://phar.phpunit.de/phpunit-7.phar
chmod +x phpunit
sudo yum install php-xdebug

![image](https://user-images.githubusercontent.com/87030990/165833086-c06100b4-acdf-4fb3-9b29-d64ef9c3c79c.png)

No need to type git add ., git commit and git push again 
For PHP the most commonly used tool for code quality analysis is phploc.
The data produced by phploc can be plotted on graphs in Jenkins.
Add the code analysis step in Jenkinsfile. The output of the data will be saved in build/logs/phploc.csv file

````javascript
stage('Code Analysis') {
  steps {
        sh 'phploc app/ --log-csv build/logs/phploc.csv'

  }
}

````
Plot the data using plot Jenkins plugin.

This plugin provides generic plotting (or graphing) capabilities in Jenkins. It will plot one or more single values variations across builds in one or more plots. Plots for a particular job (or project) are configured in the job configuration screen, where each field has additional help information. Each plot can have one or more lines (called data series). After each build completes the plots’ data series latest values are pulled from the CSV file generated by phploc.

````javascript
   stage('Plot Code Coverage Report') {
      steps {

            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)                          ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'A - Lines of code', yaxis: 'Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Directories,Files,Namespaces', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'B - Structures Containers', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Average Class Length (LLOC),Average Method Length (LLOC),Average Function Length (LLOC)', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'C - Average Length', yaxis: 'Average Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Cyclomatic Complexity / Lines of Code,Cyclomatic Complexity / Number of Methods ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'D - Relative Cyclomatic Complexity', yaxis: 'Cyclomatic Complexity by Structure'      
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Classes,Abstract Classes,Concrete Classes', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'E - Types of Classes', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Methods,Non-Static Methods,Static Methods,Public Methods,Non-Public Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'F - Types of Methods', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Constants,Global Constants,Class Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'G - Types of Constants', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Test Classes,Test Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'I - Testing', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Logical Lines of Code (LLOC),Classes Length (LLOC),Functions Length (LLOC),LLOC outside functions or classes ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'AB - Code Structure by Logical Lines of Code', yaxis: 'Logical Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Functions,Named Functions,Anonymous Functions', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'H - Types of Functions', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Interfaces,Traits,Classes,Methods,Functions,Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'BB - Structure Objects', yaxis: 'Count'

      }
    }
````

Bundle the application code for into an artifact (archived package) upload to Artifactory

````javascript
stage ('Package Artifact') {
    steps {
            sh 'zip -qr php-todo.zip ${WORKSPACE}/*'
     }
    }
````

To start this step, install zip first
````bash
sudo yum install zip -y
````

Publish the resulted artifact into Artifactory

![image](https://user-images.githubusercontent.com/87030990/165833738-e4ecee5c-7a23-42cd-9861-e7f6bf6a2fa3.png)

![image](https://user-images.githubusercontent.com/87030990/165833822-0c90fda2-ceec-434d-a738-37dae3c803fc.png)

Deploy the application to the dev environment by launching Ansible pipeline

![image](https://user-images.githubusercontent.com/87030990/165833947-33d3d6ba-6d87-4c07-a16f-792f124a5512.png)

### SonarQube Installation

SonarQube was installed on Ubuntu EC2 instance using Ansible role with PostgreSQL as Backend Database using Ansible role

Java was installed on the EC2 instance for Sonarqube

Install OpenJDK and Java Runtime Environment (JRE) 11
```bash
sudo apt-get install openjdk-11-jdk -y
sudo apt-get install openjdk-11-jre -y
````
 
Set default JDK – To set default JDK or switch to OpenJDK enter below command:
````bash
sudo update-alternatives --config java
````

JAVA Version was verified with
````bash
java -version
````

PostgreSQL software was downloaded
````bash
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
````

PostgreSQL Database Server installed
````bash
sudo apt-get -y install postgresql postgresql-contrib
 ````
 
PostgreSQL Database Server was started and enabled to start automatically at boot time
````bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
````

Password for default postgres user was changed
````bash
sudo passwd postgres
````

Switch to the postgres user
````bash
su - postgres
````

A new user was created by typing
```bash
createuser sonar
````

Switch to the PostgreSQL shell
````bash
psql
````
 
Password was set for the newly created user for SonarQube database

````bash
ALTER USER sonar WITH ENCRYPTED password 'sonar';
````

A new database for PostgreSQL database was created by running:
````bash
CREATE DATABASE sonarqube OWNER sonar;
````

All privileges was granted to sonar user on sonarqube Database.
````bash
grant all privileges on DATABASE sonarqube to sonar;
````

Exit from the psql shell:
````bash
\q
````

Switch back to the sudo user by running the exit command.
````bash
exit
````

Sonarqube was installed using Ansible Playbook

![image](https://user-images.githubusercontent.com/87030990/165834870-c3d0689e-813b-4c11-8b64-828729bdfce5.png)

#### Access SonarQube

SonarQube was accessed using browser with default administrator username and password – admin
http://server_IP:9000 OR http://localhost:9000

![image](https://user-images.githubusercontent.com/87030990/165835067-c16ecc48-492a-46eb-af35-bb00d6cf2d4d.png)

SonarScanner plugin was installed in Jenkins and configure as shown

![image](https://user-images.githubusercontent.com/87030990/165835168-eb677c3d-8a53-4314-ab42-5c838176a3eb.png)

#### Configure SonarQube and Jenkins for Quality Gate

In Jenkins, install SonarScanner plugin and navigate to configure system in Jenkins. 

Add SonarQube server, setup SonarQube scanner from Jenkins and create webhook


Setup SonarQube scanner from Jenkins – Global Tool Configuration
Manage Jenkins > Configure System
 
On SonarQube UI

![image](https://user-images.githubusercontent.com/87030990/165835482-66eea5ac-25a6-4420-a95d-71a930cb10ce.png)

Jenkins pipeline was updated with below snippet to include SonarQube scanning and Quality Gate
Below is the snippet for a Quality Gate stage in Jenkinsfile

````javascript
   stage('SonarQube Quality Gate') {
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner"
            }

        }
    }
````

NOTE: The above step will fail because we have not updated `sonar-scanner.properties
 
Configure sonar-scanner.properties

 – From the step above, Jenkins will install the scanner tool on the Linux server. You will need to go into the tools directory on the server to configure the properties file in which SonarQube will require to function during pipeline execution.
 
 ````bash
 cd /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/
 ````
 
 ![image](https://user-images.githubusercontent.com/87030990/165835812-76d48dc9-54eb-4ed7-a560-2ab945fdcb57.png)


Open sonar-scanner.properties File

````bash
sudo vi sonar-scanner.properties
````
````bash
sonar.host.url=http://<SonarQube-Server-IP-address>:9000
sonar.projectKey=php-todo
#----- Default source code encoding
sonar.sourceEncoding=UTF-8
sonar.php.exclusions=**/vendor/**
sonar.php.coverage.reportPaths=build/logs/clover.xml
sonar.php.tests.reportPath=build/logs/junit.xml
````

Add configuration related to php-todo Project

**Challenge:** The pipeline failed after updating sonar-scanner.properties file.
**Error message:** soar.source must be defined

**Resolution:** sonar.sources was then added to the configuration.

````bash
sonar.host.url=http://<SonarQube-Server-IP-address>:9000
sonar.projectKey=php-todo
#----- Default source code encoding
sonar.sourceEncoding=UTF-8
sonar.php.exclusions=**/vendor/**
sonar.php.coverage.reportPaths=build/logs/clover.xml
sonar.php.tests.reportPath=build/logs/junit.xml
sonar.sources=
````

**HINT:** To know what exactly to put inside the 
sonar-scanner.properties file, SonarQube has a configurations page where you can get some directions.

![image](https://user-images.githubusercontent.com/87030990/165836419-507ea767-0369-4f48-8092-ff6d9a590ed3.png)

The quality gate we just included has no effect. Why? Well, because if you go to the SonarQube UI, you will realise that we just pushed a poor-quality code onto the development environment.Navigate to php-todo project in SonarQube
 
List the content to see the scanner tool sonar-scanner. That is what we are calling in the pipeline script.
Output of 
````bash
ls -latr
````

````bash
ubuntu@ip-172-31-16-176:/var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/bin$ ls -latr
total 24
-rwxr-xr-x 1 jenkins jenkins 2550 Oct  2 12:42 sonar-scanner.bat
-rwxr-xr-x 1 jenkins jenkins  586 Oct  2 12:42 sonar-scanner-debug.bat
-rwxr-xr-x 1 jenkins jenkins  662 Oct  2 12:42 sonar-scanner-debug
-rwxr-xr-x 1 jenkins jenkins 1823 Oct  2 12:42 sonar-scanner
drwxr-xr-x 2 jenkins jenkins 4096 Dec 26 18:42 .
````

![image](https://user-images.githubusercontent.com/87030990/165836718-909d70c5-2882-4549-957a-5776e3431c21.png)

So far you have been given code snippets on each of the stages within the 
Jenkinsfile. But, you should also be able to generate Jenkins configuration code yourself.

* To generate Jenkins code, navigate to the dashboard for the php-todo pipeline and click on the Pipeline Syntax menu item
* Click on Steps and select withSonarQubeEnv
* – This appears in the list because of the previous SonarQube configurations you have done in Jenkins. Otherwise, it would not be there.

#### Conditionally deploy to higher environments

In the real world, developers will work on feature branches in a repository (e.g., GitHub or GitLab). There are other branches that will be used differently to control how software releases are done. You will see such branches as:
Develop
Master or Main 
(The * is a place holder for a version number, Jira Ticket name or some description. It can be something like Release-1.0.0)
Feature/*
Release/*
Hotfix/*
etc.

There is a very wide discussion around release strategy, and git branching strategies which in recent years are considered under what is known as GitFlow (Have a read and keep as a bookmark – it is a possible candidate for an interview discussion, so take it seriously!)

There is a very wide discussion around release strategy, and git branching strategies which in recent years are considered under what is known as GitFlow (Have a read and keep as a bookmark – it is a possible candidate for an interview discussion, so take it seriously!)
Assuming a basic gitflow implementation restricts only the develop branch to deploy code to Integration environment like sit
.
Let us update our Jenkinsfile to implement this:

First, we will include a When condition to run Quality Gate whenever the running branch is either develop, hotfix, release, main, or master
````javascript
when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
````

Then we add a timeout step to wait for SonarQube to complete analysis and successfully finish the pipeline only when code quality is acceptable.
````javascript
timeout(time: 1, unit: 'MINUTES') {
        waitForQualityGate abortPipeline: true
    }
 
The complete stage will now look like this:
   stage('SonarQube Quality Gate') {
      when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner -Dproject.settings=sonar-project.properties"
            }
            timeout(time: 1, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }
````

To test, create different branches and push to GitHub. You will realise that only branches other than develop, hotfix, release, main, or master will be able to deploy the code.

**Challenge:** A defect was encountered when sonar server URL was configured (in Configure System) on Jenkins  with a trailing slash (http://3.135.217.42:9000/) causing java.lang.IllegalStateException: “Unable to parse response from”

![image](https://user-images.githubusercontent.com/87030990/165837979-dfe6bf50-6a43-45d6-9082-2ed545ff24c1.png)

Resolution: The trailing slash in the sonarqube server (url http://3.135.217.42:9000) on Jenkins was removed to resolve the problem

![image](https://user-images.githubusercontent.com/87030990/165838154-2010ecf8-e508-4838-8822-22322e265fa2.png)

Notice that with the current state of the code, it cannot be deployed to Integration environments due to its quality. In the real world, DevOps engineers will push this back to developers to work on the code further, based on SonarQube quality report. Once everything is good with code quality, the pipeline will pass and proceed with sipping the codes further to a higher environment.

#### Complete the following tasks to finish Project 14
* Introduce Jenkins agents/slaves – Add 2 more servers to be used as Jenkins slave. Configure Jenkins to run its pipeline jobs randomly on any available slave nodes.

![image](https://user-images.githubusercontent.com/87030990/165838350-98879db4-13bb-4396-b319-ae4d3cfc721a.png)

* Configure webhook between Jenkins and GitHub to automatically run the pipeline when there is a code push.

![image](https://user-images.githubusercontent.com/87030990/165838445-0ac6f898-b4ec-4dd1-937f-6d486b2de624.png)

Deploy the application to all the environments


