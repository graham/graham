## Why this document

1. Creating documents for __my future self__ is something I've rarely regretted, this is part of that.
2. As impressive as tools like Terraform and others are, it bothers me if I don't understand what is actually happening. I've found these management tools often are even harder to for me to reason about because of this.
3. If I do build a system, I'd be nice to have something to point people at that clearly and comfortably describes what I built.

## What I can't teach you in AWS
### Security Groups
If you're a total AWS beginner, I'd recommend trying to get a grasp for security groups, they are the thing that are hard to get right without any understanding and have far reaching impact if configured incorrectly. I'd recommend less restrictive and more open groups at the start, while this is "less secure" it gives you more room to understand when the thing you're configuring is broken or if you've just prevent access to it with a bad security group configuration.

Security groups determine what services and machines in your AWS infrastructure can talk to each other, spend some time learning about them.
### Target Groups
These are not as complicated, but spend a little time understanding that target groups are collections of machines or services that can be __targeted__ by load balancer or other request creator. Just take a moment to familiarize yourself with the basics.
### Docker
I assume you know enough docker to create a container that hosts a webserver on port 8080 and that can respond with an HTTP 200 that says 'hello world'. Docker is a great resource, but I won't be focusing on it in this document.
### The command line
I assume you know how to use the command line and build docker images, you will also need to install the aws cli if you want to support "kicking off updates" from the command line.

Ok that's it, lets get started.

----
# The basics

1. We will have an ALB Application Load balancer that hosts a internet public listener and forwards that traffic into our AWS network. If you know what you're doing and are receiving traffic in some other way, the only thing you'll need to ensure you have is a target group. If you already have a load balancer this solution will play nicely with that and simply be an additional routing rule in your load balancer.
2. We will have a ECR (Elastic Container Registry) that holds our images, you may be able to use dockerhub or some other image service, but AWS has a container registry and that simplifies things significantly (especially because they are often default private).
3. We have a single container that hosts our web service, we won't discuss databases or other containers in this example (maybe later). It's completely possible to start a managed sql database and give the container access to that database via a different security group. Since you can scale up the cpu and number of container instances this should get you much of the way toward your first round of scale.

---

### Step 1: 
Create a container registry, https://us-west-2.console.aws.amazon.com/ecr/private-registry/repositories?region=us-west-2 (or whatever region you're in), choose whatever options work for you.
### Step 2: 
Make sure you have a target group https://us-west-2.console.aws.amazon.com/ec2/home?region=us-west-2#TargetGroups **that is currently empty and holds IP Addresses**, this will be automatically populated by the service we will create. If you'd like to skip this step, you can have the cluster service (created later) auto create this for you. If you're not very familiar with creating Target Groups, skip to step 4.
### Step 3: 
Configure a ALB or other load balancer and have it direct requests to this target group.
### Step 4: 
Visit https://us-west-2.console.aws.amazon.com/ecs/v2/clusters?region=us-west-2 and click create cluster, I'd recommend fargate if you're new, if you want to control your own image runners with ec2 you can.
### Step 5: 
On the left side bar of the ECS aws application, click on "Tasks", this task is going to reference the image you've uploaded, so make sure you have the arn handy, all other options are up to you, but make sure you have the correct port configuration.
### Step 6: 
**This contains the step that is easiest to miss**, view the detail of the cluster you just created, and under `Services` **create**; Choose "Launch Type" as your compute option, you are deploying a **Service** not a task, and make sure that you cover 2 things in the configuration,
  a) You must make sure that you configure your security group correctly so that the load balancer can reach this container. Do this in the **Networking** panel.
  b) You must make sure you choose the target group you created in step 2 (or whatever empty IP based target group). Do this in the **Load Balancing** panel. If you skipped step 2 you can have this configurator create the target group for you.
### Step 7: 
With all these new words (cluster, service, task), know that when you want to update your deployment, you'll visit the Service and click Update Service, this will create a new deployment, add the new deployment to the target group, remove the old deployment from the target group, and then drain and remove the previous container deployment.

This can be done via the AWS cli with something like:
> aws ecs update-service --cluster my-fargate-cluster --service web-service --force-new-deployment > last_build_output.txt

Updating the ECR is also, pretty simple:
> docker build -t webimage . 
> docker tag webimage:latest 12345.dkr.ecr.us-west-2.amazonaws.com/myecrrepo:latest 
> docker push 12345.dkr.ecr.us-west-2.amazonaws.com/myecrrepo:latest 

If you have a load balancer that targets this target group you should now be able to access the container's web service.

----

## Scratch Pad

Commentary:

Basically, security group one allows connections from anywhere to get to the ELB, and the second security group allows connections FROM the elb to anywhere in my aws network (you may want to make this more restrictive, but I leave that up to you)

Security group one could be called `internet-to-elb` and the second could be called `elb-to-private-network`; these will be important later because our container will need to be apart of the second group so that the elb can forward traffic to it.

Broad Highlights

1. Create a registry in ECR for your containers, this is where you will push your docker container.
2. Create a ECS Cluster, choose fargate.
3. create a 'service',
    1. compute type launch type
    2. deployment application type service
    3. choose your container and latest.
    4. give it a real service name.
    5. make sure to add it to a security group that your existing load balancer can reach
    6. make sure you select the load balancer and that you've created a target group that has nothing else in it. Each deployment will have a new ip and you need the service description to be able to update the target group for you while it swaps deployments.

