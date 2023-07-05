Prerequisites:
AWS Account,
GitHub account,
Jenkins,
Nexus,
SonarQube,
Slack

Flow of Execution:-
Login to AWS Account.
Create Your Key Pair.

Create Security Group.
a. Jenkins, Nexus and Sonarqube

Create EC2 Instances with User-data.
a. Jenkins, Nexus and Sonarqube

Post Installation.
a. Jenkins setup and Plugin
b. Nexus Setup and repository setup
c. Sonarqube login test

Git.
a. Create a github repository and migration code
b. integrate github repo with Vs-code and test it

Build Job with Nexus integration.
Github Webhooks.
Sonarqube Server integration stage.
Nexus Artifact upload stage.

Slack Notification.
Step1 and 2: Log into your AWS account and Create Key Pair
Navigate to EC2, Key pair and Create a keypair. Download the private key to your local system. The Created key will be used to ssh into your servers once provisioned.
Step3: Create Security Groups for Jenkins, Nexus and SonarQube
Jenkins Security Group
Configure inbound rule to allow ssh login on port 22 and Allow Jenkins on port 8080 from anywhere. We will be updating the Jenkins security group rule to allow traffic on port 8080 via the sonerqube security group.

Name: Jenkins-SG
Allow: SSH from MyIP on port 22
Allow: 8080 from Anywhere
Nexus Security Group
Configure inbound rule to allow ssh login on port 22, also configure port 8081 to be accessible from the browser and port 8081 from the the configured Jenkins-SG.

Name: Nexus-SG
Allow: SSH on port 22 from MyIP
Allow: 8081 from MyIP and Jenkins-SG
SonarQube Security Group
Configure the inbound rule allow SSH on port 22, allow access from My IP on port 80 and also from the configure Jenkins-SG

Name: SonarQube-SG
Allow: SSH from MyIP
Allow: 80 from MyIP and Jenkins-SG
Once SonarQube-SG is created edit Jenkins-SG inbound rule to allow traffic from sonarqube security group. This allows for sonarqube send results back to our Jenkins server.

Step4: Create EC2 instances
We will create 3 instances for Jenkins, Nexus and SonarQube using user-data scripts

Jenkins Server Setup
Create Jenkins- Server
Name: jenkins-server
AMI: Ubuntu 20.04
Security Group: jenkins-SG
InstanceType: t2.small
KeyPair: awskeypair
Additional Details: copy and paste below jenkins-setup script
Jenkins Userdata script


Nexus Server and repository Setup:
Create Nexus-server lunching an Amazon Linux-2
Name: nexus-server
AMI: Amazon Linux-2
InstanceType: t2.medium
SecGrp: nexus-SG
KeyPair: awskeypair (ensure to use your created keypair here)
Additional Details: copy and paste below Nexus-setup script
Nexus Userdata script

Name: Sonarqube-server
AMI: Ubuntu 18.04
InstanceType: t2.medium
SecGrp: SonarQube-SG
KeyPair: awskeypair (ensure to use your created keypair here)
Additional Details: copy and paste below Sonarqube-setup script
Sonarqube Userdata script

Step5: Post Installation Steps
Jenkins Server setup and plugins :
SSH into the Jenkins server and validate the status of Jenkins with the below command. Status should be active (running).

sudo -i
systemctl status jenkins

Open a browser and enter the your jenkins-server IP address on port 8080 To get the initial Admin password from directory use the below command

cat /var/lib/jenkins/secrets/initialAdminPassword

Install recommended plugins and more importantly the below listed plugins as we will be using them on this project

Maven Integration
Github Integration
Nexus Artifact Uploader
SonarQube Scanner
Slack Notification
Build Timestamp

Nexus Server and repository setup:
SSH into nexus server, check system status for nexus.

sudo -i
systemctl status nexus
Status should be active(running) as displayed below.


Open your browser, input your nexus-server public IP address on port 8081 . To sign in you need to use this command cat opt/nexus/sonatype-work/nexus3/admin.password to access the initial password . Follow the wizard to update your new password and ensure to disable anonymous access.



Select gear symbol and create repository. We will be using the repository to store our release artifacts.

maven2 hosted
Name: vprofile-release
Version policy: Release

Create another repository for maven2 proxy. This will store any dependencies required for our project. when the dependency needed, the proxy repository will checked in Nexus and downloaded. Proxy repository will download the dependencies from maven2 central repository URL provided

maven2 proxy
Name: vpro-maven-central
remote storage: https://repo1.maven.org/maven2/
Create another repository to store snapshot artifacts. All artifact with snapshot extension will be stored in this repository. Ensure to change the version policy to snapshot.

maven2 hosted
Name: vprofile-snapshot
Version policy: Snapshot
Lastly create a repository with the below configuration to group all your created repository together.

Repository type: maven2 group
Name: vpro-maven-group
Member repositories: 
 - vpro-maven-central
 - vprofile-release
 - vprofile-snapshot

SonarQube Server login test:
ssh into your sonarqube server and validate sonar service status. Status should be active(running).

Open a new browser enter the public IP address of your sonarqube-server instance. Use username admin and password admin to login. Ensure to change your password. This is best the practice as default password aren’t save.


Step6: Create a repository in Github
Create a private repository in the Github for the project, then clone content from this project repo shown below:
git clone -b ci-jenkins git@github.com:Adutoby/vprofile-project-all.git

Step7: Build Job with Nexus Integration
We will require some dependencies to build our job on Jenkins server. This include JDK8 and Maven.

Navigate tp Manage Jenkins in your Jenkins UI, then to Global Tool Configuration. Under JDK, select Add JDK > Name it, uncheck install Automatically and under JAVA HOME provide the path for JDK-8.(this is discussed below)

We also need jdk8 installed and to do that we will need to ssh into our Jenkins server (instance) and run the following commands

sudo apt update -y
sudo apt install openjdk-8-jdk -y
Since our application is using JDK8, we need to install Java8 on Jenkins. Follow these steps: Manage Jenkins > Global Tool Configuration select Install JDK8 manually, and specify its PATH in there.

Paste /usr/lib/jvm/java-1.8.0-openjdk-amd64 in the path section to install jdk8

Enter MAVEN3 and leave rest as they are and Save the configuration

We also need to add Nexus credentials to be able upload our artifacts, Therefor we will to add Nexus login credentials to Jenkins by navigating to Manage Jenkins > Manage credentials > Global > Add Credentials used below information for the configuration

username: admin
password: enter the password you setup for nexus
ID: nexuslogin
description: nexuslogin

Next is to create a Jenkinsfile to build our pipeline. The variables mentioned in the pom.xml repository and settings.xml will be declared in Jenkinsfile and their values to be used during execution.

Write the Pipeline code, save, commit and push to GitHub.
Moving forward, we will then create a New Job in Jenkins with the below property and save them.

Pipeline from SCM 
Git
URL: <url_from_project> I will use SSH link
Crdentials: we will create github login credentials
#### add Jenkins credentials for github ####
Kind: SSH Username with private key
ID: Gitpass
Description: Gitpass
Username: git
Private key file: paste your private key here
#####
Branch: */ci-jenkins
path: Jenkinsfile
Login Jenkins server via ssh and complete host-key checking step with the command below.The host-key will be stored in .ssh/known_host file. Only then will error be fixed.

sudo -i
sudo su - jenkins
git ls-remote -h git@github.com:Adutoby/vprociproject.git HEAD

Build the pipeline see that ran and built successfully.


Step8: GitHub Webhook
We will automate the process of manual build done on the Jenkins UI by using Github webhook so that whenever a developer makes changes or/and commit to the repository, a pipeline job is triggered automatically.

To setup the webhooks, Copy your Jenkins URL, then navigate to your Github repository settings. Go to the webhooks and follow the process below.

Click setting > Webhooks > Add your <JenkinsURl>/github-webhook/ at the end of the JenkinsURL.

Go back to Jenkins UI, to configure your pipeline job to set the build trigger to Build Trigger: GitHub hook trigger for GITScm polling and save the configuration

Next we will update the Jenkinsfile by adding a post action to our pipeline script and commit/push changes to GitHub. Once this is done an automatic build should be triggered if the configuration is okay.


Build job was triggered successfully and automatically


The Job was also built successfully!


Step9: SonarQube Sever Integration and code analysis
Recall we added a unit job in the previous pipeline. This Unit test and code coverage reports generated are located under Jenkins workspace target directory. The reports are not human readable and so we will be needing a tool to scan and analyze the code to a human readable format the that will be done by our SonarQube server and a toll called sonarScanner.

To set this up:

Navigate to Manage Jenkins > Click Global Tool Configuration and add the sonar scanner.

Add: SonarQube scanner
Name: sonarscanner
and tick install automatically option
Next go to Configure system under the manage Jenkins and find sonarqube sever section and follow below configuration. Remember to save

Check mark the environment variables box
Add sonarqube
Name: sonarserver
Server URL: http://<private_ip_of_sonar_server>
Server authentication token: create the token from sonar website
Next we add sonar token to global credentials section. To do this navigate to the sonarqube UI, click on the admin name > my account > security > give your token a name and generate the token. copy the token and continue your global credentials configuration on Jenkins where you will follow below. Note that you will have to save without the secret and return to input secret to complete the configuration for it to work. Ensure to save.

Kind: secret text
Secret: paste the token generated 
name: sonartoken
description: sonartoken
To add sonarQube code for our pipeline and commit/push changes to GitHub.

Observer to see our job was triggered successfully.


Job was also built successfully


On the SonarQube UI we should now see our project with the quality gate results Passed.


To create our own Quality Gates and add it to our project, we will need create a Webhook in SonarQube to send the analysis results to Jenkins.

To do this. Navigate on the sonarqube UI to Quality gate > click on create > give your Quality gate a name “vprofileQG” and save. The click on add condition to give your rules. Select on over all codes set your rule and add condition.

Then attached the quality gate to your project. Go back to your project > project settings > select quality gate > and select the quality gate you just created.

Now setup the webhooks by navigating to project settings > select webhooks > select create > give a name “jenkinswebhook” use below sample url for in the URL section and create.

http://<private_ip_of_jenkins>:8080/sonarqube-webhook

Update the Jenkinsfile by adding a new stage named “Quality Gate” to the pipeline and commit changes to GitHub.
Run the job and Build was successful!


Step10: Publish Artifact to Nexus Repo
Here we will be automating the process of publishing the latest artifact to Nexus repository after a successful build and to do this we will need to add Build-Timestamp to the artifact name to get unique artifact each time a job is successfully built.

Click Manage Jenkins > Configure System under Build Timestamp, update the pattern as we desire.

yy-MM-dd_HHmm

Now we will add another stage to our pipeline called UPLOAD ARTIFACT save our changes, commit the changes and see the result.


Job was again built successfully!


Our artifact was uploaded to Nexus repository.


Step11: Slack Notification
Login to slack and create a workspace. We will then create a channel called jenkins-cicd-tabnob in our workspace.

Next we will add Jenkins app to slack. The step to do this available Searching on the web so lets Google Slack app and then search for Jenkins, select the “Jenkins CI”. Choose the channel jenkins-cicd-tabnob you just created. It will give a setup instructions, then copy the Integration token credential ID . Ensure to save the settings before you leave the page.

Return to your Jenkins dashboard, Click Manage Jenkins > System configuration > Slack and Use the guides below.

Workspace:  ( the workspace you created )
credential: give a credential name
default channel: #jenkins-cicd-tabnob
Next is to add our Slack token to Jenkins global credentials. Again you the guide

Kind: secret text
Secret: the token copied should be pasted here.
name: slacksecret
description: slacksecret

Now update our Jenkinsfile script with a post installation step, commit and push the changes to GitHub which will automatically trigger a build job and we should receive the notification on slack accordingly.

Notification was sent and received on slack. Great Job making it through the lab.


Finally Clean-up
Ensure to delete all resources created throughout the project.
