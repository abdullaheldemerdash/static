## Introduction

This is my proposed solution for the task provided by Scopic Software company to deploy an Inventory app including its decoupled parts (backend, frontend, and database) to AWS. In this project, I took the approach of deploying the frontend to an S3 bucket while deploying the backend where the heavy lifting is occurring to an autoscaling group of EC2 instances directly connected to an RDS DB to persist the data and retrieve it whenever needed.

## Architecture Design & Technical steps

1- **Frontend on S3 bucket** 

Why I  chose that? Amazon S3 is reliable, scalable, and secure with no real configuration needed. It is an unlimited cloud storage with super capabilities, including 99.99 percent of durability and high availability, object-based storage acting as an API and with the ability to host our static website. Amazon S3 Cloud storage can is a serverless application without much effort which makes it option number one in my opinion.

I wanted to separate all static content in this app(images, docs, etc.) to Amazon S3. This split allows distributing the application’s requests in parallel, dynamic content served by the EC2 instance (web server) and the rest with Amazon S3.

Technically, I moved the static content manually (aws s3 cp)into the bucket after building the frontend code (the backend was embedded inside (ALB pointing to my webapp EC2 instances). I made it available for public access using the site hosting feature then enabled CORS to communicate with the backend api endpoint,I set the IAM role/permissions to allow read access. I'd have leveraged Roue53 beside CDN with all its benefits if my IAM role provided by Scopic allwoed me to do that. That would make it a practical secure solution which includes using HTTPS instead of HTTP besides the caching and other advantages.

2- **Backend with AWS Auto Scaling Group and a Load Balancer**

In my design, I took the approach of deploying the backend on an AWS Auto Scaling group which scales dynamically behind a load balancer, which is defined by a set of scaling policies (thresholds/limits) that controls the process of scale-up and down. The instances can scale up and down based on predefined criteria for CPU, RAM I made carefully to make the start cost as low as possible (chose free-tier t2.micro size for my instances also to reduce costs), taking into account that it will scale up efficiently in case the load increased allowing us to save money and unneeded resources. I would also consider using reserved and Spot instances in my EC2 ASG to reduce the costs but this depends on other factors includes decision-makers.

Security-wise, the autoscaling group is deployed in private subnets (alongside the RDS DB) while they can reach the public internet (for example: for fetching updates) via the Nat gateway which used an Elastic IP address. I used two AZ for High Availability reason and added one bastion host in case we needed to ssh to any of the webapp instances.

The AMI used in the launch configuration was based on a 64-bit Ubuntu 18 LTS instance where I built the backend code, made it ready with an entrypoint (npm run start:production) as one step in the running time (user data section). In case we needed to update the code in the future we might have a Jenkins pipeline with AWS plugin to build the AMI and update the variable of "ImageID" which I used in the deployment scripts. 

Technically, I built my own CloudFormation scripts (attached to the repo)to deploy the infrastructure, which initialized the network as a ground floor first then built the EC2 servers where I hosted my backend web application while I deployed the RDS DB manually from the management console.

3- ** MySQL based RDS DB instance ** 

I started with free-tier DB instance to make the costs the lowest but enabled autoscaling in case we needed more storage for our persisted data. It is worth mentioning that there is no additional cost for RDS Storage Auto Scaling. You pay only for the RDS resources needed to run your applications (pay as you go).

I am not using multi-AZ feature because it will cost us more money while we can depend on the automated backups to restore it in case of any disaster cases.

### Work Theory

It is simple and clear from the diagram I attached as PDF document in the repo that the users will have the URL of our S3 bucket because we are not allowed to use Route53 or CDN instead of a normal registered domain URL. Their browsers requests will either be served by the S3 directly for static contents or it will go to the ALB as an interface of our Autoscaled backend webapp instances where the logic happens and the connection to the database is made.

<a href="https://ibb.co/4N950Ph"><img src="https://i.ibb.co/x5PRrJx/Scopic-Inventory-Arch-Diagram.png" alt="Scopic-Inventory-Arch-Diagram" border="0"></a>

### Base Cost

Using the AWS Pricing Calculator I got this estimate report: https://calculator.aws/#/estimate?id=6aee1ba87be7cdd2e3ec473e8377e62071d6d070

### Autoscaled Costs (10,100,1000) user

I took the approach of measuring the load with JMeter apache tool, and simulated 100 users. I couldn't figure out a way to automate creating accounts for 1000 users with an SQL procedure but i think it is doable. I have created 100 manually from the UI then updated the emailVerified to 1 in users table.

When I ran the load testing, I was checking CPU and memory utilization from the management console in AWS and it was about 20% CPU for simultaneous 100 users, so I expect we are safe to use more 2 instances for the 1000 user because it might scale to maximum 10x to be 4 instances total.

I also needed to check database load too (Monitoring section in AWS RDS), and see how the number of user affects its load and how much resources are used and found the CPU utilization was under 2%. With regards to scaling the storage I couldn't find a way to simulate this and didn't add more than the default 20GB storage coming with the instance.

Finally, I used the packets sent and received to calculate the data in and out of the region from the S3 bucket and I noticed it didn't make any real difference except with the 1000 users.

After I knew the cpu/memory/storage/data usage per user, I can now do the calculations for the estimated ec2, S3 and RDS instances sizes for each number of users by adding these values to the AWS cost estimator.

and here are the results for the 1000 users: https://calculator.aws/#/estimate?id=f2395b31d7cbaceac60b8420be6ec0e2bedffe80

### Application URL:

http://scopic-inventory-frontend.s3-website.ap-northeast-2.amazonaws.com


