### Deployment instructions

There are 3 github repos for this demo.

  * [Service A](https://github.com/kevinjcash/serviceA)
  * [Service B](https://github.com/kevinjcash/serviceB)
  * [Template](https://github.com/kevinjcash/microservicesTemplate)

You can take the cloudformation template from the Template repo and deploy it in your AWS environment. It will pull the images from Dockerhub. The template assumes you already have a keypair created; it will create everything else. The template will pull a file from my s3 bucket, which should be available to everyone.

### Overview
My microservices are small Flask python apps. [Service A](https://github.com/kevinjcash/serviceA) is the public facing service and queries [Service B](https://github.com/kevinjcash/serviceB), which is in the private subnet.  Each GitHub repo has a Dockerfile, which specifies build instructions for the image, and a TravisCI configuration file, which provides instructions to Travis continuous integration. Travis builds each docker image, and pushes the finished images to Dockerhub. Although this demo is simple, Travis can support complex build and deployment actions.

For the infrastructure, I've used a AWS Cloudformation template to specify the resources.  The template creates a VPC with one public and one private subnet. One instance is created in each subnet, one for service A in the public subnet and one for service B in the private. The startup script for these instances pulls down the docker images and runs them. There is a NAT instance in the public subnet to allow the Service B instance to reach the internet. The load balancer is also created in the public subnet, and contains instance A. 

I chose to use an ec2-linux AMI instead of Ubuntu because it is preconfigured to work with the cloudformation init scripts. These are available on Ubuntu, but Ubuntu doesn't provide an advantage in this scenario that makes the extra configuration worth it. The only configuration I built into the AMI was to format and mount the second drive for docker, since that was proving delicate to configure through the template.

### Security

Access to this environment is tightly controlled. Web traffic is allowed over the Load Balancer only to Service A and only ports 80->5000 are mapped. SSH traffic is restricted to the CIDR block specified in the parameters, and even then Service B can only be reached from either the NAT or Service A. All instances use the same key pair, which could be a vulnerability if that key is compromised and the ssh restriction is defeated. Multiple keys, or key rotation could mitigate this.

### Scalability

This application is simple, so scaling can be as simple as spinning up new machines. If we keep only on container on each instance for simplicity, we can use very small, cheap AWS instances. The load balancer can spread traffic between the instances. To improve response time and availability, the application can be replicated in multiple Availability Zones, or even multiple Regions, with Route 53 to direct users to the most appropriate Region.

### Rate Limiting

Rate limiting can be implemented in the app itself, in this case with a library such as [Flask-Limiter](http://flask-limiter.readthedocs.org/en/stable/). However, it would be more elegant to use a front end service such as Amazon's [API Gateway](https://aws.amazon.com/api-gateway/). This service acts as a gatekeeper, keeping excess traffic out of the environment, and provides features such as rate-limiting, caching, API versioning, and more. The architecture needs little change to use this, since the service can be configured to route traffic whereever it is needed, whether it be EC2 instances, AWS Lambda functions, or even outside resources.

### Updates

Seamless updates can be achieved by using a Blue/Green deployment. We can create the new stack, introduce the new instances into the load balancer, and then drain traffic to the old instances and remove them from the load balancer. This traffic is not disrupted, and in the case of a bug, the old instances can be put back in place.

![](https://s-media-cache-ak0.pinimg.com/originals/61/b7/1b/61b71b213343d1c82a2d78780ca1d618.jpg)
