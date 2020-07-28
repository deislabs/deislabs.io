---
title: "Docker Mixin for Porter!"
description: New mixin to use the Docker CLI within the Porter manifest
date: "2020-07-28"
authorname:  Gauri Madhok
author: "@gaurimadhok"
authorlink: "https://twitter.com/gaurimadhok"
# image: "images/porter-with-docker.png" OPTIONAL CUSTOM IMAGE FOR PAGE
tags: ["porter"]
---
We are excited to announce the first release of a Docker mixin for Porter! 🐳

Mixins are critical building blocks for bundles, and we hope the Docker mixin will help ease the process of composing bundles. The Docker mixin installs Docker and provides the Docker CLI within bundles. Prior to the creation of this mixin, in order to use Docker within your bundle, you would have to create a custom Dockerfile and install Docker. Then, to run any Docker commands, you would need to use the exec mixin and call a bash script to execute the Docker commands. 

The Docker mixin abstracts this logic for you and allows you to specify the Docker commands with the arguments and flags that you want to execute directly in the Porter manifest. The commands currently provided by the Docker mixin are pull, push, run, build, login, and remove. To view all the syntax for the commands, take a look at the [README](https://github.com/deislabs/porter-docker).

Let's go through an example bundle to try out the mixin. First, we will use the Docker mixin to pull and run [docker/whalesay](https://hub.docker.com/r/docker/whalesay/). Then, we will write our own Dockerfile, build it, and push it to Docker Hub.

## Author the bundle
Writing a bundle with the Docker mixin has a few steps:


* [Create a bundle](#create-a-bundle)
* [Install the Docker mixin](#install-the-docker-mixin)
* [Add the Docker mixin to the Porter manifest](#add-the-docker-mixin-to-the-porter-manifest)
* [Set up credentials](#set-up-credentials)
* [Use Docker CLI](#use-docker-cli)

Let's run through these steps with our example bundle called docker-mixin-practice. First, set up a project:
```
mkdir docker-mixin-practice;
cd docker-mixin-practice;
```

### Create a bundle
Next, use the porter create command to generate a skeleton bundle that you can modify as we go through our example. Be sure to update the tag at the top of the porter.yaml file from getporter to the name of your own registry.
```console
$ porter create
```

### Install the Docker mixin
Next, you need to install the Docker mixin to extend the Porter client. To install the mixin, run the line below and you should see the output that it was installed.
```console
$ porter mixins install docker

installed docker mixin v0.1.0 (b660770)
```
This installs the docker mixin into porter, by default in ~/.porter/mixins.

### Add the Docker mixin to the Porter manifest
In order to use the Docker mixin within your bundle, you need to add it to the mixin list. In the create a bundle step, porter create added the exec mixin. Replace the exec line with docker. 
```
mixins:
- docker
```

### Set up credentials
This step is needed if you wish to use Docker push to push an image to a registry. In order to push to one of your registries, you need to login to Docker Hub. To set up your credentials to login to Docker Hub, make sure the environment variables DOCKER_USERNAME and DOCKER_PASSWORD are set on your machine. Next, add these lines in your porter.yaml. You may change the name to what you want it to be.
```
credentials:
  - name: DOCKER_USERNAME
    env: DOCKER_USERNAME
  - name: DOCKER_PASSWORD
    env: DOCKER_PASSWORD
``` 
Next, run the following line and select environment variable for where the credentials will come from.
```console
$ porter credentials generate
```
Your credentials are now set up. When you run install or upgrade or uninstall, you need to pass in your credentials using the `-c` or `--cred` flag. Here is an example: 
```console
$ porter install -c credentialName
```

### Use Docker CLI

Next, to run docker/whalesay, use the code below to pull the image and then run it with a command to say "Hello World". 
```
install:
- docker:
    description: "Install Whalesay"
    pull:
      name: docker/whalesay
      tag: latest
- docker:
    description: "Run Whalesay"
    run:
      name: dockermixin
      image: "docker/whalesay:latest"
      command: cowsay
      arguments:
        - "Hello World"
```
When you are ready to install your bundle, run the command below to identify the credentials and give access to the Docker Daemon. 

```console
$ porter install -c myCredentials --allow-docker-host-access
```
This is the output that should be generated after it runs. 
```
Run Whalesay
 _____________ 
< Hello World >
 ------------- 
    \
     \
      \     
                    ##        .            
              ## ## ##       ==            
           ## ## ## ##      ===            
       /""""""""""""""""___/ ===        
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
       \______ o          __/            
        \    \        __/             
          \____\______/   
```

Here is an example of what you can do in the uninstall step to remove the container we made above. 
```
uninstall:
- docker:
    description: "Remove dockermixin container"
    remove:
      container: dockermixin
```

Now, we will go through an example of how you can incorporate and build your own Docker image and then push it to Docker hub. First, you will need to create a Dockerfile. For example, here is a simple Dockerfile called Dockerfile-cookies.
```
FROM debian:stretch

CMD ["echo", "Everyone loves cookies"]
```
To build an image from this Dockerfile, you can use the code below and specify the name of your file and the tag you want. We created our Dockerfile in the same directory as the bundle. If you wish to create it somewhere else, you can specify the path to the file as an additional parameter under build. The default path is the current directory. If you want to push your image to a registry, you can add the code to login to Docker and then push the image. 
```
install:
- docker:
    description: "Build image"
    build:
      tag: "gmadhok/cookies:v1.0"
      file: Dockerfile-cookies
- docker:
    description: "Login to docker"
    login:
- docker:
    description: "Push image"
    push:
      name: gmadhok/cookies
      tag: v1.0
```
When you are ready to install your bundle, run the command below to identify the credentials and give access to the Docker daemon. 

```console
$ porter install -c myCredentials --allow-docker-host-access
```
After it runs, you should see output that the image was built and tagged successfully, the login succeeded, and the push to your repository happened.
```
Build image
Sending build context to Docker daemon  101.2MB
Step 1/2 : FROM debian:stretch
 ---> 614bb74b620e
Step 2/2 : CMD ["echo", "Everyone loves cookies"]
 ---> Using cache
 ---> e89d85933cc1
Successfully built e89d85933cc1
Successfully tagged *******/cookies:v1.0
Login to docker
Login Succeeded
Push image
The push refers to repository [docker.io/*******/cookies]
6885f9305c0a: Preparing
6885f9305c0a: Layer already exists
v1.0: digest: sha256:1fb89bd28f81c3e29ae16a44f077f4709f33ac410581faf23b42d9bd7a6f913b size: 529
execution completed successfully!
``` 

## Thank you for reading!
Please try out the mixin and let us know if you have any feedback to make it better! You can dig into the code [here](https://github.com/deislabs/porter-docker)  and create an issue [here](https://github.com/deislabs/porter-docker/issues/new).
