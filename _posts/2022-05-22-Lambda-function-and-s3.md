---
layout: post
title:  "Containerized AWS Lambda"
date:   2022-05-22 09:46:54 +0530
categories: aws
---

## Lambda Deployment Packages
Lambda supports two types of deployment packages: container images and .zip file archives. 
- A container image includes the base operating system, the runtime, Lambda extensions, your application code and its dependencies. 
- A .zip file archive includes your application code and its dependencies. When you author functions using the Lambda console or a toolkit, Lambda automatically creates a .zip file archive of your code.

## Creating Lambda container image
The activity can be achieved in the following steps:

1. Create a project directory for your new function on your system.

2. Create a directory named app in the project directory, and then add your function handler code to the app directory in a file called app.rb.

3. Create a new Dockerfile with the content as below:

   ```Dockerfile
   FROM public.ecr.aws/lambda/ruby:2.7

   # Copy function code
   COPY app.rb ${LAMBDA_TASK_ROOT}

   # Copy dependency management file
   COPY Gemfile ${LAMBDA_TASK_ROOT}

   # Install dependencies under LAMBDA_TASK_ROOT
   ENV GEM_HOME=${LAMBDA_TASK_ROOT}
   ENV HOME="/tmp"

   RUN yum install -y gcc make gcc-c++ git unzip &&\
    # Install gem dependencies with bundler
    bundle install &&\
    # Uninstall yum packages that will no longer be needed
    yum remove -y gcc make gcc-c++ unzip &&\
    # Clear out yum cache to save additional space
    yum clean all &&\
    rm -rf /var/cache/yum


   # Set the CMD to your handler (could also be done as a parameter override outside of the Dockerfile)
   CMD [ "app.lambda_handler" ]
   ```

4. Build your Docker image with the docker build command. Enter a name for the image. The following example names the image hello-world.

    `docker build -t hello-world .   `

5. Start the Docker image with the docker run command. For this example, enter hello-world as the image name.

    `docker run -p 9000:8080 hello-world `

6. Test your application locally using the runtime interface emulator. From a new terminal window, post an event to the following endpoint using a curl command:

    `curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{}'`

    This command invokes the function running in the container image and returns a response.

## Upload the image to the Amazon ECR repository

In the following commands, **replace 123456789012 with your AWS account ID** and set the region value to the region where you want to create the Amazon ECR repository.

   > Note
   > In Amazon ECR, if you reassign the image tag to another image, Lambda does not update the image version.

1. Authenticate the Docker CLI to your Amazon ECR registry.

   `aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com    `

2. Create a repository in Amazon ECR using the create-repository command.

   `aws ecr create-repository --repository-name hello-world --image-scanning-configuration scanOnPush=true --image-tag-mutability MUTABLE`

3. Tag your image to match your repository name. 

   `docker tag  hello-world:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/hello-world:latest`

4. Deploy the image to Amazon ECR using the docker push command.

   `docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/hello-world:latest`

Now that your container image resides in the Amazon ECR container registry, you can create and run the Lambda function.

## References
1. [Lambda Deployment Packages - AWS](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-package.html)
2. [Creating Lambda Container images - AWS](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)
3. [Deploying Lambda function as container images - AWS](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-images.html)
