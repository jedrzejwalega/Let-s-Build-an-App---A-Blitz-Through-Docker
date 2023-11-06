# What's a Docker?
**Docker** is an open source technology used in app deployment. Whenever you develop an app, you do it with chosen operating system in mind (like a particular release of Linux), as well as a language of choice, associated tooling and libraries, which we can collectively refer to as **dependencies**. Whenever you run your app on some machine (for example on a rented cloud compute instance) you need to make sure that all of the dependencies are met for the app to run properly. Docker provides you an opportunity to do away with that headache by introducing a concept known as **containers**. Putting your app inside one is called **containerization**.

## Core Docker Concepts
### Container

**Containers** can be thought of as boxes where your app will run, isolated from the host's operating system. Inside them you can freely define your dependencies and stop worrying about which host your app will run on. Thanks to the containers you can rest easy knowing that inside the box your app will have all the dependencies it needs to work properly. The host only needs to have Docker installed to launch your container with an app.

It is important to note that containers have their own **isolated**, minimal file systems inside, which house your app's code. Due to this isolation containers also have their own set of ports, which often need to be configured to communicate with your local host.
### Image
Whenever you spawn a container, it is spawned based on an **image**, which is another core Docker concept. It contains all the code and dependencies necessary to deploy your app inside a container. 

Images are usually uploaded to image registries, which can be thought of as Github equivalents for images. They allow you to track the versioning of the image. 

Some of the popular image registries are: Docker Hub, Google Container Registry (GCR), Amazon Elastic Container Registry (ECR) and Azure Container Registry (ACR). The last three are integrated with the most popular cloud providers and for a good reason - images are the current standard in app deployment in the cloud.
### Dockerfile
Creating an image (commonly referred to as building) is performed by your local Docker installation based on a set of instructions in a so-called **Dockerfile**. These steps are defined by you and should not only copy the app code from your local machine to the image, but also install all the necessary tooling and libraries. They should also provide some basic information about the image, like which command to run upon container start up or which ports in the container should be exposed.

The filename "Dockerfile" is obligatory and is recognized by Docker as an instruction to build an image. Those instructions are defined in Docker specific syntax.

A Dockerfile is subject to the same version control system as other files in a project, which means that any changes in image setup can be easily tracked and documented.

## Containerization Thought Map
In other words, if you want to put your app inside a container, the steps towards doing so are:
1) Define the dependencies for your app
2) Write a Dockerfile where those dependencies will be addressed
3) Build an image
4) Launch a container based on that image

# Base Docker Commands
After you [install Docker](https://docs.docker.com/engine/install/) on your local machine, you should get familiar with some of the basic Docker commands to work with containers:
1) Build an image called image_name based on a Dockerfile in the current directory.
```bash
docker build -t image_name .
```
2) Launch a new container based on an image in an interactive mode (you will be able to interact with it from the terminal, which is very useful for debugging) 
```bash
docker run -it --name container_name image_name
```
3) Launch an interactive shell to get inside the container and interact with the files and logs. **Extremely** useful in debugging.
```bash
docker exec -it container_name bash // start an 
```
4) Stop a running container. It will still be left on your system and you'll be able to restart it.
```bash
docker stop container_name
```
5) Remove a container from your system. If it's running it will be stopped first.
```bash
docker rm container_name
```
5) List the images currently present in your system
```bash
docker images
```
6) List currently running containers
```bash
docker ps
```
# Let's Build an App
## Baduk.ninja 
In this exemplary app containerization I will share a Dockerfile from [baduk.ninja](http://baduk.ninja/) - a website dedicated to all aspiring Go players aiming to improve their skills. While the source code for this project is currently private, it can still be used as a valid example for building a simple image for an app.
## Project Structure
The project can be summarised by the following directory hierarchy:
- baduk.ninja
    - frontend
        - node_modules
        - src
        - public
        - package-lock.json
        - package.json
    - backend
        - src
        - Cargo.lock
        - Cargo.toml
    - sgfs
    - Dockerfile
    - run_services.sh
- rust-goban-fork
## Defining Dependencies
The frontend is written in JavaScript and uses React, with directory structure as expected by the npm package manager. It listens on port 80 and communicates with backend on port 4000.

The backend is written in Rust and is structured to work with cargo package manager. Aside from official library releases, it uses a fork of the goban library, which we will have to handle while writing a Dockerfile. It also uses some sgfs files from a separate directory.

In the main baduk.ninja directory we can also see a run_services.sh file, which will launch both fronted and backend. It will track the status of those two services and if at least one of them exits with an error, the entire process will, too. This will come in handy later, as run_services.sh will be the main entry script for our future containers. If either of our running services exits inside the container, we want the container itself to go down, too.

## Image Concept
We will build a single image that will run both the frontend and backend of our app. Since we're using ports 80 and 4000, our future containers will have to expose those two ports to communicate with the host.

In the image we are also looking for a way to build both the frontend and backend and use those executables in starting the services for an optimized performance.

## Picking a Base Image
### What is a Base Image?
It is important to note that when building an image we don't do so from an absolute scratch. The first step is to pick a **base image**, which provides a basic file structure and some preinstalled tooling, and add our own changes on top of those. An example of a base image can be as simple as a minimal installation of Ubuntu, or a more robust Node or Rust base images. The last two come with a particular version of the language installed, along with most common tooling. They are publically available on Docker Hub and save a lot of time in not having to install all of the above features into the image on your own.

### Intermediate Images
While the robust images are great, they can be quite memory heavy and ideally we would like to get the best of both worlds:
- When building the baduk.ninja image we would like to use robust base images. This will allow us to build frontend and backend without losing time for installing languages and tooling.
- We would like the final image to be constructed based on a minimal Linux setup to save memory

To achieve that we will use three base images:
- rust:1.73.0 - base image for Rust builds
- node:14 - base image for Node.js builds
- frolvlad/alpine-glibc:latest - a minimal Linux Alpine build

The first two will be used as **intermediate base images**. They will be mounted and used to build backend and frontend respectively. We will grab the optimized executables from the intermediate images and copy them to the lightweight Linux Alpine base image. Afterwards the robust intermediate images will be discarded.

The major steps for our Dockerfile can be summarized as follows:
1) Frontend build
    - Mount Node base image
    - Copy local, unoptimized frontend source code inside the image
    - Install libraries with npm
    - Build frontend code
2) Backend build
    - Mount Rust base image
    - Clone the goban fork directory from Github
    - Copy local, unoptimized backend and sgfs code inside the image
    - Build backend code
3) Final build
    - Mount Linux Alpine base image
    - Copy only executables from backend and frontend intermediates
    - Install npm and serve tooling (necessary to launch frontend)
    - Copy sgfs directory for optimized backend to work properly
4) Add information in the image which ports should be exported
5) Define a default command to launch upon container start

# Dockerfile Code
--To be added--