hello-world
============

## Running locally

Build and run using Docker Compose:

`git clone https://github.com/runticm/tq-work.git`  
`cd tq-work/`  
`docker-compose up`  
`localhost:80`  


## Deploying to ECS

In AWS create key pair (if you already have it skip it)  
After download change permissions (if you already have it skip it)  
`chmod 400 <key_name>` (if you already have it skip it)  
`git clone https://github.com/runticm/tq-work.git`  
`cd tq-work/`  
In AWS account run this CloudFormation template <ecs-cluster.template>, set name to EcsClusterStack and select key name. All other settings leave default  
When its done add another CloudFormation template <ecs-jenkins-demo.template>, set name to Jenkins and leave all other setting on that page as default  
When its done create ECR repository (private) and name it hello-world  
In EC2 console look for ssh to Jenkins instance and take public ip  
`ssh -i <key> ec2-user@<ip_of_jenkins>`  
`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`(and copy password)    
Go to your favorite browser and paste public hostname of Jenkins server  
Paste password which you copy from previous step  
Choose Install suggested plugins  
Create your first admin user  
Go to tab Manage Jenkins and then Manage Plugins  
On Available tab choose Amazon ECR and CloudBees Docker Build and Publish plugin  
Then select Download now and install after restart  
Select Restart Jenkins when installation is complete and no jobs are running  
Refresh page after one min  
	
## Set GIT accept changes (if you already have it skip it)
In the top right corner of any page in GIT, choose your profile picture, then choose settings  
In the user settings sidebar, choose SSH and GPG keys  
Choose New SSH key and add Title and past pub key  
If prompted, confirm your GitHub password  

## Create a GIT repository (if you already have it skip it)
Create a repository  
On your PC do next  
Go into tq-work folder  
Delete the hidden .git directory with command 
`rm -fR .git`  
Reinitialize the repository and push the contents to your new GitHub repository using SSH by running the following command  
`git init`  
`git add .`  
`git commit -m "Initial commit"`  
If you are using SSH, run the following command:  
`<git remote add origin 'git@github.com:<your_repo>.git'>`  
If you are using HTTPS, run the following command:  
`<git remote add origin 'https://github.com/<your_repo>.git'>`  
Example: git remote add origin 'https://github.com/runticm/tq-work.git'  
`git push -u origin master`  
    
## Enable automatic trigger in Jenkins by adding webhook
In GitHub repo, click settings (Not main settings! It is Repo settings)  
Under settings, select Webhooks  
Add Payload URL of your Jenkins public hostname and add sufix github-webhook  
Example: http://ec2-3-120-237-218.eu-central-1.compute.amazonaws.com/github-webhook/  
Check pushes and pull requests under let me select individual events to trigger this webhook  
Check active box before click add webhook  
Add webhook  

## Configure Jenkins job
Create a freestyle project in Jenkins and add name  
Under source code management, select git and type the name of your GitHub repository, https://github.com/your_repo.git (if it is public then this is done if not add Credentials)  
Select Branch Specifier to */master  
Under build triggers, select Github hook trigger for GITScm polling in order to connect with Github webhook (as soon as we push our script from local environment to Github, Jenkins will be triggered sponteneously)  
Besides, delete workspace before build starts checked under build environment  
Select execute shell under add a build step. In the command field, type or paste the following text: (ECR_REPO and REGION accordingly to your account)  

```
#!/bin/bash
set -x
sudo groupadd docker
sudo usermod -aG docker $USER
sudo chmod 777 /var/run/docker.sock
PATH=$PATH:/usr/local/bin; export PATH
REGION=eu-central-1
ECR_REPO="008125993587.dkr.ecr.eu-central-1.amazonaws.com/hello-world"
#$(aws ecr get-login --region ${REGION})
aws ecr get-login --no-include-email --region ${REGION}>>login.sh
sh login.sh
```

Select docker build and publish under add a build step  
On Repository Name add: 008125993587.dkr.ecr.eu-central-1.amazonaws.com/hello-world (accordingly to your account)  
On TAG add: v_$BUILD_NUMBER  
On Docker registry URL add: http://008125993587.dkr.ecr.eu-central-1.amazonaws.com/hello-world (accordingly to your account)  
Select execute shell under add a build step. In the command field, type or paste the following text: (REGION accordingly to your account)  

```
#!/bin/bash
set -x
#Constants
PATH=$PATH:/usr/local/bin; export PATH
REGION=eu-central-1
REPOSITORY_NAME=hello-world
CLUSTER=typeqast-workshop
FAMILY=`sed -n 's/.*"family": "\(.*\)",/\1/p' taskdef.json`
NAME=`sed -n 's/.*"name": "\(.*\)",/\1/p' taskdef.json`
SERVICE_NAME=${NAME}-service
env
aws configure list
echo $HOME
#Store the repositoryUri as a variable
REPOSITORY_URI=`aws ecr describe-repositories --repository-names ${REPOSITORY_NAME} --region ${REGION} | jq .repositories[].repositoryUri | tr -d '"'`
#Replace the build number and respository URI placeholders with the constants above
sed -e "s;%BUILD_NUMBER%;${BUILD_NUMBER};g" -e "s;%REPOSITORY_URI%;${REPOSITORY_URI};g" taskdef.json > ${NAME}-v_${BUILD_NUMBER}.json
#Register the task definition in the repository
aws ecs register-task-definition --family ${FAMILY} --cli-input-json file://${WORKSPACE}/${NAME}-v_${BUILD_NUMBER}.json --region ${REGION}
SERVICES=`aws ecs describe-services --services ${SERVICE_NAME} --cluster ${CLUSTER} --region ${REGION} | jq .failures[]`
#Get latest revision
REVISION=`aws ecs describe-task-definition --task-definition ${NAME} --region ${REGION} | jq .taskDefinition.revision`
#Create or update service
if [ "$SERVICES" == "" ]; then
  echo "entered existing service"
  DESIRED_COUNT=`aws ecs describe-services --services ${SERVICE_NAME} --cluster ${CLUSTER} --region ${REGION} | jq .services[].desiredCount`
  if [ ${DESIRED_COUNT} = "0" ]; then
    DESIRED_COUNT="1"
  fi
  aws ecs update-service --cluster ${CLUSTER} --region ${REGION} --service ${SERVICE_NAME} --task-definition ${FAMILY}:${REVISION} --desired-count ${DESIRED_COUNT}
else
  echo "entered new service"
  aws ecs create-service --service-name ${SERVICE_NAME} --desired-count 1 --task-definition ${FAMILY} --cluster ${CLUSTER} --region ${REGION}
fi
```

## Test everything
On your PC do folowing  
`git add .`  
`git commit -m "initial commit"`  
`git push`  
On Jenkins web page we see that job is triggered  
On AWS go to ECS  
Select Cluster typeqast-workshop  
On Task bar select task  
Expand under Containers hello-world  
Take external link address and here is our deployment which is builded from scratch  
On top of that we go to Auto Scaling Groups  
Go to EcsClusterStack  
On Load balancing go to edit  
Choose Add a new load balancers  
Type is Application Load Balancer and Internet-facing  
Add a name on Loadbalancer (Name length must be between 1 - 32 characters)  
On Listeners and routing > Default routing (forward to) choose Create a target group and pick proposed  
Go to Load Balancers and search for ELB which you just created  
Copy DNS name  
Go to Route53, then hosted zones and pick your domain  
There create a record  
Record name can be test-deploy and Record type is CNAME  
Paste into Value field ELB dns name which you copy from previus step and Create records  
Go to your web page and see what we did  

	:) Hello world (:
