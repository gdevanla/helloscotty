# helloscotty

This document describes the steps that were followed in creating this sample application. The purpose of this application is to demonstrate

1. Create a Scotty Web Application using stack.
2. Build the application using a default docker container (provided by Stack), to ensure all dependencies are met
3. Set up files to create a custom docker container
4. Creat a docker file.
5. Publish to Docker Hub
6. References to follow to host the docker container on Amazon Elastic Beanstalk

The document makes the following assumptions

1. Basic knowldge on how Docker works and docker is set up on the machine
2. Familiarity with Amazon Elastic Beanstalk instance
3. How stack works

## Create a Scotty Web Application

Using a template called `scotty-hello-world`, create a new stack project called `helloscotty`.

``` bash
 stack new helloscotty scotty-hello-world
```

The template currently uses earlier versions of ghc and scotty libraries. Therefore, make the following
changes

1. Changes to `helloscotty.cabal`.
   Update the build-depends, section to read `scotty >= 0.11`

2. If needed, fix the version of ghc referenced to `resolver: lts-7.14` in `stack.yaml`.1

Test the settings, and make sure the build is sucessful.

``` bash
    stack build
```

3. Simulate a typical web-application folder structure. Since out web application would involve
some static files, we create a 'static' folder under which we could have js and css files. We will
be referencing this folder later on when we create a docker image.

4. By now, you should be able to run the application, by refering to the executable create by `stack build` command.  Try running

``` bash
    stack exec helloscotty
```

So far, we have used a vanilla stack setup that did not involve any references to docker. If we wanted to deploy the executable on any server, that we might need dependencies like `libgmp` available on the server. Especially, if you were to use services like Elastic Beanstalk, where the VM is not self-hosted, that we cannot run our web application without the dependencies. Amazon, atleast,currently does not provide a default web application to run Haskell based applications. Currently, only language like Python, Java, Ruby, Go etc are supported.

Therefore, we are left with only one option. That is, we need to use a docker-based application to host our web application on Elastic Beanstalk.


### Dockerising our web application

1. Stack comes with in-built docker integration. You can refer to the docs at [Stack-Docker Integration](https://docs.haskellstack.org/en/stable/docker_integration/). In short, for the sample application
we want to build, we will add the following lines to the `stack.yaml` file.

``` yaml
docker:
  enable: true
  repo: fpco/stack-build
```

This will tell stack, to download an default image of docker that contains all the dependencies that will be needed to compile and later on run our web application.


After add these lines, we can build are application using this container, just by running

``` bash
    stack build
```

The compiled code can run using the same docker container with,

``` bash
    stack --docker-run-args='--net=bridge --publish=3000:3000' exec helloscotty
```

By running, this command we can make sure that our web application can run successfully within
the referenced docker container.

2. Creating the `Dockerfile`.

We want to be able to access our docker image from Amazon Elastic Beanstalk environment. One way to do this is to host the docker image on docker hub. Alternatively, you
could host it on any other private repository or on S3 itself. For our usecase, we will host our docker image on Docker Hub. If the docker image is private, then
we will follow a number of steps (referenced later on, in this document) to authorize Amazon EBS enviroment to be able to access the image.

We also need to add the executable and the static folder to the image. The basic docker file would look as follows:

``` dockerfile
# Docker file for helloscotty
FROM fpco/stack-build:lts-7.10
MAINTAINER your name <your@email.com>
ADD static static
ADD bin/helloscotty helloscotty
EXPOSE 3000
ENTRYPOINT ./helloscotty
```

This docker file will be access by the docker hub server to build the image, each time a change is push to this repo. Alternate, settings can be explored
on Docker Hub. Note, that we have to `ADD` statements in the docker file

1) ADD the static foloder
2) ADD the exectuble we build. For convienience, we can have a script that copies the executable to the bin folder everytime a new version is built.
3) EXPOSE exposes the port the helloscotty application will be listening on. Currently, the code is hard-coded to listen on this port.

3. Link the repository to Docker Hub

Create an account on Docker Hub, choose 'Create Automated Build` and refer to the repository, the web application is available at. DockerHub will
automatically build the image and be ready to be access by external services like Amazon EBS.

4. Setting up the enviroment of Amazon EBS involves a lot more steps that is not within the scope of this document. But, here are some helpful hints

a) Follow the steps provided at [Deploy Private Docker to Elastic Beanstalk](http://thequietlattice.com/docker/aws/elasticbeanstalk/2015/12/18/deploy-private-docker-to-elb.html).
Though, some of the instructions in steps 6 and 7 are outdates, the remaining steps clearly helps you through the process. For steps 6 and later, it is helpful to refer to Amazon EBS documentation
for find the settings page where the respective changes can be performed.
