# Guide to AWS Fargate deployment
This guide describes how to deploy a load balanced app on AWS Fargate. This draft is a work in progress.

## Dockerize your apps
For ease of deployment and continuous integration's sake, dockerize your apps and host them somewhere accessible online, such as GitHub Packages or Dockerhub. 

<hr>

## Add a Docker private registry secret
To allow AWS to access your private image registry, you need to add an access token to AWS Secrets Manager. Let's see how to create these first.

### Hosting on Github Packages
If you decided to use Github Packages to host your images, you need to create a Personal Access Token that will let AWS pull your Github images. To do that, head to Github, select the global settings panel (n.b. not the repository settings), click on "developer settings", "personal access tokens", and finally "generate new token". Under "scopes", you only need to give it permissions to read and write packages. Click on create and the token will be displayed. Take a note of the token as it will disappear once you navigate away. 

### Create a secret on AWS
Navigate to the AWS Secrets Manager, choose to store a new key/value secret selecting the secret type "Other type of secrets". Add as key your Github **username**, while as value the previously created access token. Next, give it a meaningful name (e.g. github-packages) and an optional description. Reach the final page and finally store the secret.

<hr>

## Create AWS IAM users, roles and policies
AWS is quite strict about privileges, which is a good thing. The bad thing is that it can be quite chaotic to face this amount of complexity the first time.
Head to the AWS IAM panel to start this journey.


### Policy
Under "Access management", select "Policies", then click "Create Policy". Under Service, search for "secret" to find and select the Secret Manager service. Below, under Actions expand the Read option and select "GetSecretValue". Under Resources, you can give permission to this policy to read any stored secrets (click on the checkbox "Any"), or only specific secrets (add the specific ARNs of the secrets). Next, name the policy something meaningful (e.g. secret-policy or more specifically github-secret-policy) and hit create.

### User
If using AWS for the first time, it is recommended to create a user with restricted privileges. At the very least, it is good practice to create an Admin type of user to manage all deployment related things. We will cover this last use case.

under "Access management", click on "Users". Here, click on "add user". Name the user "Administrator", and under access type select "Console password". Select the preferred way of password creation.

Go next to the permission tab and select "Attach existing policies directly". Here you can attach the existing permission AdministratorAccess. Reach the final screen and create the user.


### Role
We need to create a new IAM role that has privileges to access the previously created secret (has the right policy) and, therefore, pull images from our private Docker registry. This role will only be used at a later stage by the Fargate service.

Under "Access management", select "Roles". Create a new role, choose "AWS Service" and then the use case "Elastic Container Service". Following, select "Elastic Container Service Task".

On the next page, we need to attach the policies needed to run AWS ECS (Fargate) related tasks. Search for "ecs" and select "AmazonECSTaskExecutionRolePolicy".  Then, search for the previously created secret policy (e.g. secret-policy) and attach it.

You can give this role the default name "ecsTaskExecutionRole", review, and create.

<hr>

## Create a VPC

TODO: write about creating a VPC and subnets (both need to be created individually, subnets need to belong to VPC)

1 - Create vpc with CIDR ending in /16 to obtain a network with range of addresses from x.x.0.0 to x.x.255.255
2 - create and attach 2 subnets to vpc, with subnets CIDR blocks 10.0.0.0/24 and 10.0.1.0/24
3 - create internet gateway to allow network to pull images from the internet
4 - attach internet gateway to vpc from the internet gateway control panel
5 - find vpc route table, go to routes, then edit routes. Add a route with destination 0.0.0.0/0 and with target the newly created internet gateway.

<hr>

## Add an Application Load Balancer (Optional)
Head to the AWS EC2 control panel. From the left menu select "load balancer" under "load balancing", then click on "create load balancer". Pick Application Load Balancer for your Fargate deployment.

Give a name to your load balancer. Under Listeners, add a HTTP listener on port 80 if not already present; then, add a HTTPS listener on port 443.


<hr>

## AWS ECS

### Create a cluster
From the AWS ECS control panel, select Clusters and click on "Create Cluster". Select the "networking only" mode that is powered by Fargate. Give a name to the cluster and select "Create VPC", which is a Virtual Private Cloud that contains a logically isolated section where you to launch AWS resources. Leave the default values and end by clicking create.

### Create a Task Definition
An AWS Task defines the environment in which the containers need to run. From the AWS ECS control panel, select Task Definitions and click on "Create new Task Definition". Here, select Fargate to power your app. Name the new task, then, under "task role" select the default role "ecsTaskExecutionRole", which has the right set of privileges to run ECS tasks. 

Below, under "Task execution IAM role", select "ecsTaskExecutionRole" again. If you followed the guide correctly, this role has the necessary privileges to pull the container images of you apps.

Select the task memory size and the CPU power for the task, which should include the total memory and CPU power needed to run all of your containers. You can choose how to distribute these resources in the next step.

Now, select "add container" to add the images of your containers. Give the container a name, and under "image" paste the link to your hosted image (e.g. docker.pkg.github.com/user/repo-name/image-name:1.0.0 for Github Packages). Tick the box next to "private repository authentication" and paste the ARN of the Secret Manager, which you can get if you navigate to the AWS Secret Manager and click on the name of your secret.
Set a memory soft limit and add the ports of the container that need to be accessible. Leave everything else as is and click "add". Repeat the process for your other containers and hit "create" when done.

### Create a service
Navigate to the Clusters panel and select the newly created cluster. Under the services tab, select "Create". This allows to create a Fargate service responsible for provisioning the task (the containers) that we defined. 

Select the Fargate lauch type. Then, under "task definition" select the task you created and the latest revision. Select the cluster where you want the containers to run. Name the service and select the number of tasks that will be run by the service (1 as default).

Select the VPC created earlier, as well as the two subnets.

Click "Edit" on the security group. Here you can create a new security group that will allow traffic to ingress your VPC. Select "Create new security group" and give the group a name. Under "Inbound rules for security group" you will find port 80 is open to the public by default. Remove it if not necessary, and add a rule for each of the ports of your containers that need to be accessible publicly via the internet. In most cases, you will want to select the type "Custom TCP" and specify the port under "Port range".

Optionally, here you can select the load balancer previously created. Go next and you will be able to select Auto Scaling if needed (not covered by this guide at this stage). Go next and you will be able to create your service.

<hr>

## Connect a domain using WHM
From the AWS load balancer control panel, select the previously created load balancer and navigate to the "Integrated services" tab.
Here, attach an AWS Global Accelerator to the load balancer, which will provide it with a set of static IP addresses. Then, from the WHM top-level control panel select DNS Functions -> DNS Zone Manager. Select the wanted domain and add an A Record that points the root domain name (without "www") to the AWS static IP address. Add one A Record for each static IP address.
