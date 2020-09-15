# Guide to AWS Fargate deployment
This guide describes how to deploy a load balanced app on AWS Fargate. This draft is a work in progress.

## Dockerize your apps
For ease of deployment and continuous integration's sake, dockerize your apps and host them somewhere accessible online, such as GitHub Packages or Dockerhub. 

<hr>

## Add a Docker private registry secret
To allow AWS to access your private image registry, you need to add access tokens to AWS Secrets Manager. Let's see how to create these first.

### Hosting on Github Packages
If you decided to use Github Packages to host your images, you need to create a Personal Access Token that will let AWS pull your Github images. To do that, head to Github, select the global settings panel (n.b. not the repository settings), click on "developer settings", "personal access tokens", and finally "generate new token". Under "scopes", you only need to give it permissions to read and write packages. Click on create and the token will be displayed. Take a note of the token as it will disappear once you navigate away. 

### Create a secret on AWS
Navigate to the AWS Secrets Manager, choose to store a new key/value secret selecting the secret type "Other type of secrets". Add as key your Github **username**, while as value the previously created access token. Next, give it a meaningful name (e.g. github-packages) and an optional description. Reach the final page and finally store the secret.

<hr>
## Create AWS IAM users, roles and policies
AWS is quite strict about priviledges, which is a good thing. The bad thing is that it can be quite chaotic to face this amount of complexity for the first time.
Head to the AWS IAM panel to start this journey.


### Policy
Under "Access management", select "Policies", then click "Create Policy". Under Service, search for "secret" to find and select the Secret Manager service. Below, under Actions expand the Read option and select "GetSecretValue". Under Resources, you can give permission to this policy to read any stored secrets (click on the checkbox "Any"), or only specific secrets (add the specific ARNs of the secrets). Next, name the policy something meaningful (e.g. secret-policy or more specifically github-secret-policy) and hit create.

### User
If using AWS for the first time, it is recommended to create a user with restricted priviledges. At the very least, it is good practice to create an Admin type of user to manage all deployemnt related things. We will cover this last use case.

under "Access management", click on "Users". Here, click on "add user". Name the user "Administrator", and under access type select "Console password". Select the preferred way of password creation.

Go next to the permission tab and select "Attach existing policies directly". Here you can attach the existing permission AdministratorAccess. Reach the final screen and create the user.


### Role
We need to create a new IAM role that has privileges to access the previously created secret (has the right policy) and, therefore, pull images from our private Docker registry. This role will only be used at a later stage by the Fargate service.

Under "Access management", select "Roles". Create a new role, choose "AWS Service" and then the use case "Elastic Container Service". Following, select "Elastic Container Service Task".

On the next page, we need to attach the policies needed to run AWS ECS (Fargate) related tasks. Search for "ecs" and select "AmazonECSTaskExecutionRolePolicy".  Then, search for the previously created secret policy (e.g. secret-policy) and attach it.

You can give this role the default name "ecsTaskExecutionRole", review, and create.

<hr>

## Add an Application Load Balancer
...

<hr>

## Connect a domain using WHM
From the AWS load balancer control panel, select the previously created load balancer and navigate to the "Integrated services" tab.
Here, attach an AWS Global Accelerator to the load balancer, which will provide it with a set of static IP addresses. Then, from the WHM top-level control panel select DNS Functions -> DNS Zone Manager. Select the wanted domain and add an A Record that points the root domain name (without "www") to the AWS static IP address. Add one A Record for each static IP address.
