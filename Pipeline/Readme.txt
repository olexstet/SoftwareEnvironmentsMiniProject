
# Pipeline 

## Asset composition

- Stage environment running in VM (from stage_env folder) 
	- A vagrant VM specification (i.e. Vagrant file)
	- Tomcat scripts configuration 
	- Tomcat server running in VM 
	- Binary of the product running into the tomcat for testing purpose retrieved from CI Server 

- CI Server Environment running in VM (from ci-sevrver_env folder)
	- A vagrant VM specification (i.e. Vagrant file)
	- Ansible playbooks to provision the VM
	- GitLab as VCS and CI running into the VM 
	- Docker service running into the VM 

- Production Environment running in VM (from prod_env folder)
	- A vagrant VM specification (i.e. Vagrant file)
	- Tomcat scripts configuration 
	- Ansible playbooks to provision the VM
	- Tomcat server running in VM 
	- Product running in tomcat server in VM 


## Prerequisites

### Hardware

1. Laptop with at least 8 Gb memory (recommended 16 Gb, ideally 32 Gb)


### Software

1. Any Unix-base OS (Tested on MacOS)
2. VirtualBox(v 6.0, or higher)
3. Vagrant (v 2.2.5, or higher)
4. Ansible (v 2.7.5, or higher)

### Remarks 
1) By preference, ones any VM is created and accessed, you should not switch off the VM because we will need them to run at the same time 
2) When you have a test case, it also must be executed as any of the steps because they guarantee that everything is working fine 

3) Video tutorial is also available for this README where the author present all the steps in the following README.txt file. URL: https://drive.google.com/file/d/1p4ue5dsXB6Ob00i8Ok-xjXbSKfxlyKB8/view?usp=sharing
Don't hesitate to look on it, if you encounter any problems or misunderstandings. 

## Guidelines

Step 1: CI Server Setup 

1.1) Get to the working directory
cd ~/<root_folder>/Pipeline/ci-server_env/integration-server

1.2) Create a VM which is used for CI Server 
vagrant up  

Remark: In case of failure of gitlab which may not arrive. Please destroy the VM and run everything again. 

1.3) Enter into the VM 
vagrant ssh

1.4) Reconfigure Gitlab by editing /etc/gitlab/gitlab.rb file 

	1.4.1) Find location of gitlba.rb file by 
	cd /etc/gitlab

	1.4.2) Open gitlab.rb file modification 
	sudo nano gitlab.rb

	1.4.3) Change external url: external_url http://hostname to external_url 'http://192.168.33.9/gitlab'

	1.4.4) Replace and uncomment unicorn['port'] = 8080 by unicorn['port'] = 8088 (don't forgot to uncomment) 

	1.4.5) Save the modification and close the gitlab.rb

	1.4.5) Reconfigure Gitlab based on previous changes 
	sudo gitlab-ctl reconfigure
	sudo gitlab-ctl restart unicorn
	sudo gitlab-ctl restart


### **** Test Case ****

Initial conditions: none

Test Steps:
1. Go to http://192.168.33.9/gitlab (Maybe not accessible some seconds, wait a bit or reload page until it works) 

Post conditions:
- GitLab is accessible at the indicated URL
- It asks to enter password for the root creedentials

1.5) Access http://192.168.33.9/gitlab and register in Gitlab by entering the password, example: devopsse 

### **** Test Case ****

Initial conditions: You have successfully entered a password for the root credentials

Test Steps: 
1. Go to http://192.168.33.9/gitlab
2. Log in using as user name "root" and password the one entered in the previous step.


Post conditions:
- You have successfully logged in as administrator and should see the Gitlab Projects user interface. 

1.6) Configure Docker 
	1.6.1) Go to the terminal where you have our VM and add a user to the docker group in order to be able to access the docker CLI ($YOUR_USERNAME = vagrant)
	sudo usermod -aG docker $YOUR_USERNAME 
	
	1.6.2) Validate the installation and access by running a hello world contained 
	docker run --name hello-world hello-world

	1.6.3) Remove the docker container and image:
	docker rm hello-world
	docker rmi hello-world
	
1.7) Return to initial directory 
exit 
vagrant ssh 

1.8) Create new project in Gitlab for our product 
	1.8.1) Go to Gitlab user interface and access Projects 
	1.8.2) Click "New project" button and create blank project 
	1.8.3) Enter Project name as "product" and click "Create Project" 

1.9) Push our Product to "product" repository
	1.9.1) Open new terminal and access the "Product" folder: 
	cd ~/<root_folder>/Product
	
	1.9.2) Ensure that no git reference exists already in project, by removing git: 
	rm -rf .git


	1.9.3) Push project to "product" repository
	git init
	git remote add origin http://192.168.33.9/gitlab/root/product.git
	git add .
	git commit -m "Initial commit"
	git push -u origin master
	 

### **** Test Case ****

Initial conditions: Pushing Product to "product" repository 

Test Steps:
1. Access project or reload the project page if already in the project repository

Post conditions:
- You should see all the files and folders present in Product folder (local) in our "product" repository 


1.10) GitLab Runner Setup in our integration VM 
	1.10.1) Go into our terminal VM 
	1.10.2) Install GitLab Runner 
	curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
	sudo apt-get install gitlab-runner
	
	1.10.3) Register the runner 
	sudo gitlab-runner register

	1.10.4) Enter Information as following: 
	For GitLab instance URL enter: 
	http://192.168.33.9/gitlab/
	For the gitlab-ci token enter the generated token: (You can find token in gitlab GUI, Admin Area -> runners->"And this registration token:" our token)
	Example Y1Bnz3rXNYPxJ6io_rS7 
	For a description for the runner enter: 
	[integration-server] docker
	For the gitlab-ci tags for this runner enter:
	integration
	For the executor enter:
	docker
	For the Docker image enter:
	alpine:latest

	1.10.4) Restart the runner:
	sudo gitlab-runner restart


 	### **** Test Case ****

	Initial conditions: Perform all steps in 1.10.1) to 1.10.4) 

	Test Steps:
	1. Access Gitlab GUI -> Admin Area -> Runners (or reload the page if you were already on it)

	Post conditions:
	- You should see the runner with the tag integration 


	1.10.5) Go into Runners and find our new runner. Click on Edit and select "Run untagged jobs" and click "Save changes"

1.11) Create simple pipeline for building and testing our Product from "product" repository 
	1.11.1) Access Gitlab user interface. Go into Projects -> product -> CI/CD -> Pipeline 
	1.11.2) Stop current(default) pipeline if running by clicking on "Cancel" and then "Stop pipeline)
	1.11.3) Access CI/CD -> Editor and click "Create new CI/CD pipeline"
	1.11.4) Paste the following code: 

Code for step 1.11.4: 

image: maven:latest

stages:
  - build
  - test-unit
  - test-integration
  - package
  - deploy

cache:
  paths:
    - target/

build_app:
  stage: build
  script:
    - mvn compile

test_app_test:
  stage: test-unit
  script:
    - mvn test -Dtest=WelcomeControllerUnitTest
    
test_app_integration:
  stage: test-integration
  script:
    - mvn test -Dtest=WelcomeControllerIntegrationTest


package_app:
  stage: package 
  script:
    - mvn package

deploy_app:
    stage: deploy
    script:
    - echo "Deploy review app"
    artifacts:
        name: "my-app"
        paths:
        - target/*.war

	
	1.11.5) Commit .gitlab-ci.yml file by clicking "Commit changes" at he bottom 


### **** Test Case ****

Initial conditions: Commit .gitlab-ci.yml file with previously provided code 
Test Steps:
1. Access CI/CD -> Pipelines and wait until the end of the pipeline

Post conditions:
- You should see 5 stages which are all passed 
- These stages allows to compile, perform unit tests, perform integration tests, package (in war format) and obtain an artifact which is our binary 


Step 2: Setup stage environment and improve the pipeline with acceptance tests stage 

Step 2 prerequisites: integration VM must running 

2.1) Open new terminal and access stage_env folder: 
cd ~/<root_folder>/Pipeline/stage_env

2.2) Create stage VM: 
vagrant up 

You should see "default: Done" at the end 

2.3) Access stage VM: 
vagrant ssh 


### **** Test Case ****

Initial conditions: none
Test Steps:
1. Access in our browser and check if you can access the following urls: 
http://192.168.33.17:8080
http://192.168.33.17:8080/manager/html (username: admin, password: admin)

2.(If Problem in 1.) In case these URLs cannot be reached, then try to fix it by restarting tomcat and access again:
sudo /opt/tomcat/bin/shutdown.sh
sudo /opt/tomcat/bin/startup.sh

Post conditions:
- You should see for http://192.168.33.17:8080 a page where at the top you have Apache Tomcat/9.0.54
- You should see Tomcat Web Application Manager page 

2.4) Install GitLab Runner:
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner

2.5) Register the GitLab Runner 
sudo gitlab-runner register

Enter Information as following: 
For GitLab instance URL enter: 
http://192.168.33.9/gitlab/
For the gitlab-ci token enter the generated token: (You can find token in gitlab GUI, Admin Area -> runners->"And this registration token:" our token)
Example Y1Bnz3rXNYPxJ6io_rS7 
For a description for the runner enter: 
[stage-vm] shell
For the gitlab-ci tags for this runner enter:
stage-vm-shell
For the executor enter:
shell

2.6) Restart the runner: 
sudo gitlab-runner restart

2.7) Grant sudo permissions to the gitlab-runner: 
sudo usermod -a -G sudo gitlab-runner
sudo visudo

2.8) Add the following line at the bottom of the file: 
gitlab-runner ALL=(ALL) NOPASSWD: ALL

2.9) Reload stage VM:
exit 
vagrant reload
vagrant ssh 

### **** Test Case ****

Initial conditions: Create and restart GitLab Runner 

Test Steps:
1. Access Gitlab GUI -> Admin Area -> Runners (or reload the page if you were already on it)

Post conditions:
- You should see the runner with the tag stage-vm-shell
- Remark: You should be able to see right now 2 runners in total 

2.10) Improve the pipeline with new code for .gitlab-ci.yml file in order to include execution of acceptance tests 
	2.10.1) Go into Projects -> product -> CI/CD -> Editor 
	2.10.2) Replace completely previous code (from 1.11.4) by the following new code: 

Code for 2.10.2): 

image: maven:latest

stages:
  - build
  - test-unit
  - test-integration
  - package
  - deploy-war
  - deploy-stage
  - test-acceptance


variables:
  # This will suppress any download for dependencies and plugins or upload messages which would clutter the console log.
  # `showDateTime` will show the passed time in milliseconds. You need to specify `--batch-mode` to make this work.
  MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
  # As of Maven 3.3.0 instead of this you may define these options in `.mvn/maven.config` so the same config is used
  # when running from the command line.
  # `installAtEnd` and `deployAtEnd` are only effective with recent version of the corresponding plugins.
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"

  STAGE_BASE_URL: "http://192.168.33.17:8080"


  # Define Tomcat's variables on Staging environment
  APACHE_HOME: "/opt/tomcat"
  APACHE_BIN: "$APACHE_HOME/bin"
  APACHE_WEBAPPS: "$APACHE_HOME/webapps"

  # Define product's variables
  PRODUCT_SNAPSHOT_NAME: "controller-testing-0.0.1-SNAPSHOT.war"
  RESOURCE_NAME: "controller.war"


cache:
  paths:
    - .m2/repository
    - target/

build_app:
  stage: build
  script:
    - mvn compile

test_app_test:
  stage: test-unit
  script:
    - mvn test -Dtest=WelcomeControllerUnitTest
    
test_app_integration:
  stage: test-integration
  script:
    - mvn test -Dtest=WelcomeControllerIntegrationTest

package_app:
  stage: package 
  script:
    - mvn package

deploy_app:
    stage: deploy-war
    script:
    - echo "Deploy review app"
    artifacts:
        name: "my-app"
        paths:
        - target/*.war

deploy-stage:
  stage: deploy-stage
  tags:
    - stage-vm-shell
  script:
    - echo "Shutdown Tomcat"
    - sudo sh $APACHE_BIN/shutdown.sh
    
    - echo "Deploy generated product into the stage-vm environment" 
    - sudo cp target/$PRODUCT_SNAPSHOT_NAME $APACHE_WEBAPPS
    
    - echo "Set right name"
    - sudo mv $APACHE_WEBAPPS/$PRODUCT_SNAPSHOT_NAME $APACHE_WEBAPPS/$RESOURCE_NAME
    
    - echo "Set user asnd group rights"
    - sudo chown tomcat:tomcat $APACHE_WEBAPPS/$RESOURCE_NAME
    
    - echo "Start up Tomcat"
    - sudo sh $APACHE_BIN/startup.sh

test-acceptance:
  stage: test-acceptance
  tags:
    - integration
  script:
    - mvn $MAVEN_CLI_OPTS -Denv.BASEURL=$STAGE_BASE_URL test -Dtest=WelcomeControllerAcceptanceTest



	2.10.3) Click on "Commit changes" in order to create new pipeline 

### **** Test Case ****

Initial conditions: Commit .gitlab-ci.yml file with previously provided code 
Test Steps:
1. Access CI/CD -> Pipelines and wait until the end of the pipeline

Post conditions:
- You should see 7 stages which are all passed 
- These stages allows to compile, perform unit tests, perform integration tests, package (in war format), obtain an artifact, deploy product to tomcat in stage and perform acceptance tests (in WelcomeControllerAcceptanceTest class) 


### **** Test Case ****

Initial conditions: Pipeline for new .gitlab-ci.yml is completed 
Test Steps:
1. http://192.168.33.17:8080/manager/html (username: admin, password: admin)
2. Check if in Applications(Path) there is a name "controller"
3. Go and check if product is running in tomcat with following url: http://192.168.33.17:8080/controller/welcome

Post conditions:
- You should be able to see that our product is running in tomcat 

 
Step 3: Setup production environment and deploy the product in this environment 

3.1) Open new terminal and go into prod-vm folder: 
cd ~/<root_folder>/Pipeline/prod_env/prod-vm 

3.2) Create prod VM:
vagrant up 

You should see at the end: "default: Done."

3.3) Access prod VM: 
vagrant ssh 

3.4) Install GitLab Runner: 
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner

3.5) Register a new runner in gitlab for production VM: 
sudo gitlab-runner register

Enter Information as following: 
For GitLab instance URL enter: 
http://192.168.33.9/gitlab/
For the gitlab-ci token enter the generated token: (You can find token in gitlab GUI, Admin Area -> runners->"And this registration token:" our token)
Example Y1Bnz3rXNYPxJ6io_rS7 
For a description for the runner enter: 
[prod-vm] shell
For the gitlab-ci tags for this runner enter:
prod-vm-shell
For the executor enter:
shell

3.6) Restart GitLab Runner:
sudo gitlab-runner restart

3.7) Grant sudo permissions to the gitlab-runner: 
sudo usermod -a -G sudo gitlab-runner
sudo visudo

3.8) Add the following line at the bottom of the file: 
gitlab-runner ALL=(ALL) NOPASSWD: ALL

3.9) Reload prod VM:
exit 
vagrant reload
vagrant ssh 

### **** Test Case ****

Initial conditions: Create and restart GitLab Runner 

Test Steps:
1. Access Gitlab GUI -> Admin Area -> Runners (or reload the page if you were already on it)

Post conditions:
- You should see the runner with the tag prod-vm-shell
- Remark: You should be able to see right now 3 runners in total 

3.10) Deploy the product in tomcat for prod-vm (production environment) 
	3.10.1) Go into Gitlab user interface, then Projects -> product -> CI/CD -> Editor
	3.10.2) Replace completely previous .gitlab-ci.yml code but the following new code 

Code for 3.10.2) 

image: maven:latest

stages:
  - build
  - test-unit
  - test-integration
  - package
  - deploy-war
  - deploy-stage
  - test-acceptance
  - deploy-prod


variables:
  # This will suppress any download for dependencies and plugins or upload messages which would clutter the console log.
  # `showDateTime` will show the passed time in milliseconds. You need to specify `--batch-mode` to make this work.
  MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
  # As of Maven 3.3.0 instead of this you may define these options in `.mvn/maven.config` so the same config is used
  # when running from the command line.
  # `installAtEnd` and `deployAtEnd` are only effective with recent version of the corresponding plugins.
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"

  STAGE_BASE_URL: "http://192.168.33.17:8080"


  # Define Tomcat's variables on Staging environment
  APACHE_HOME: "/opt/tomcat"
  APACHE_BIN: "$APACHE_HOME/bin"
  APACHE_WEBAPPS: "$APACHE_HOME/webapps"

  # Define product's variables
  PRODUCT_SNAPSHOT_NAME: "controller-testing-0.0.1-SNAPSHOT.war"
  RESOURCE_NAME: "controller.war"


cache:
  paths:
    - .m2/repository
    - target/

build_app:
  stage: build
  script:
    - mvn compile

test_app_test:
  stage: test-unit
  script:
    - mvn test -Dtest=WelcomeControllerUnitTest
    
test_app_integration:
  stage: test-integration
  script:
    - mvn test -Dtest=WelcomeControllerIntegrationTest

package_app:
  stage: package 
  script:
    - mvn package

deploy_app:
    stage: deploy-war
    script:
    - echo "Deploy review app"
    artifacts:
        name: "my-app"
        paths:
        - target/*.war

deploy-app-stage:
  stage: deploy-stage
  tags:
    - stage-vm-shell
  script:
    - echo "Shutdown Tomcat"
    - sudo sh $APACHE_BIN/shutdown.sh
    
    - echo "Deploy generated product into the stage-vm environment" 
    - sudo cp target/$PRODUCT_SNAPSHOT_NAME $APACHE_WEBAPPS
    
    - echo "Set right name"
    - sudo mv $APACHE_WEBAPPS/$PRODUCT_SNAPSHOT_NAME $APACHE_WEBAPPS/$RESOURCE_NAME
    
    - echo "Set user asnd group rights"
    - sudo chown tomcat:tomcat $APACHE_WEBAPPS/$RESOURCE_NAME
    
    - echo "Start up Tomcat"
    - sudo sh $APACHE_BIN/startup.sh

test-acceptance:
  stage: test-acceptance
  tags:
    - integration
  script:
    - mvn $MAVEN_CLI_OPTS -Denv.BASEURL=$STAGE_BASE_URL test -Dtest=WelcomeControllerAcceptanceTest


deploy-app-production:
  stage: deploy-prod
  tags:
    - prod-vm-shell
  script:
    - echo "Shutdown Tomcat"
    - sudo sh $APACHE_BIN/shutdown.sh
    
    - echo "Deploy generated product into the prod-vm environment" 
    - sudo cp target/$PRODUCT_SNAPSHOT_NAME $APACHE_WEBAPPS
    
    - echo "Set right name"
    - sudo mv $APACHE_WEBAPPS/$PRODUCT_SNAPSHOT_NAME $APACHE_WEBAPPS/$RESOURCE_NAME
    
    - echo "Set user asnd group rights"
    - sudo chown tomcat:tomcat $APACHE_WEBAPPS/$RESOURCE_NAME
    
    - echo "Start up Tomcat"
    - sudo sh $APACHE_BIN/startup.sh


	3.10.3) Click "Commit changes" for committing the .gitlab-ci.yml file 

### **** Test Case ****

Initial conditions: Commit .gitlab-ci.yml file with previously provided code 
Test Steps:
1. Access CI/CD -> Pipelines and wait until the end of the pipeline

Post conditions:
- You should see 8 stages which are all passed 
- These stages allows to compile, perform unit tests, perform integration tests, package (in war format), obtain an artifact, deploy product to tomcat in stage and perform acceptance tests (in WelcomeControllerAcceptanceTest class). The last stage deploy our product in tomcat which runs on prod-vm. This is done in the same way as for stage-vm. 


### **** Test Case ****

Initial conditions: Pipeline for new .gitlab-ci.yml is completed 
Test Steps:
1. http://192.168.33.7:8080/manager/html (username: admin, password: admin)
2. Check if in Applications(Path) there is a name "controller"
3. Go and check if product is running in tomcat with following url: http://192.168.33.7:8080/controller/welcome?name=John

Post conditions:
- You should be able to see that our product is running in tomcat 


Congratulations, you just created a pipeline for our product which build, execute tests(unit, integration, acceptance) and deploy to production environment automatically without human interaction from the commit of developer.  

It is the end of Readme about how to create this pipeline. 

Thank you 