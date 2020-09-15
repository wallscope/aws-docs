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

## Add an Application Load Balancer
...

<hr>

## Connect a domain using WHM
From the AWS load balancer control panel, select the previously created load balancer and navigate to the "Integrated services" tab.
Here, attach an AWS Global Accelerator to the load balancer, which will provide it with a set of static IP addresses. Then, from the WHM top-level control panel select DNS Functions -> DNS Zone Manager. Select the wanted domain and add an A Record that points the root domain name (without "www") to the AWS static IP address. Add one A Record for each static IP address.
