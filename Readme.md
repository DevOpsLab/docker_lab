# Getting the Docker Big Picture by Example

## Objective
The objective of this exercise is to begin building an app the Docker way.

We'll start at the bottom of the hierarchy of such an app, which is a **container**.

Above this level is a **service**, which defines how a container behaves in production.

Finally, at the top level is the **stack**, defining the interactions of all the services.

## More about Services
In a distributed application, different pieces of the app are called **services**. For example, if you imagine a video sharing site it probably includes a service for storing application data in a database, a service for video transcoding in the background after a user uploads something, a service for the front-end, and so on.

Services are really just “containers in production.” A service only runs one image, but it codifies the way that image runs—what ports it should use, how many replicas of the container should run so the service has the capacity it needs, and so on. Scaling a service changes the number of container instances running that piece of software, assigning more computing resources to the service in the process.

Luckily it's very easy to define, run, and scale services with the Docker platform by just writing a `docker-compose.yml` file. A `docker-compose.yml` file is a YAML file that defines how Docker containers should behave in production.

## The Action Script / Steps

**Step 1: Your new dev environment - define a container with a Dockerfile.**

Dockerfile will define what goes on in the environment inside your container. Access to resources like networking interfaces and disk drives is virtualized inside this environment, which is isolated from the rest of your system, so you have to map ports to the outside world, and be specific about what files you want to “copy in” to that environment. However, after doing that, you can expect that the build of your app defined in this Dockerfile will behave exactly the same wherever it runs.
```
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

**Step 2: Code your app**

Ensure that its working/root directory matches the value of `WORKDIR` in your Dockerfile.

**Step 3: Build your app**

When we are ready to build the app, make sure you are still at the top level of your new directory. That is, your project root should have the Dockerfile that we defined in Step 1. Good to go?

Now run the Docker build command like below to create a docker image with a tag name that we can refer to. 
`docker build -t friendlyhi .`

Where is your built image? It's in your machine's local Docker image registry thaat you can look at like below:
`docker images` or newer `docker image ls`

**Step 4: Run the app**

Run the app, mapping your machine's port 4000 to the container's published port 80 using -p:
`docker run -p 4000:80 friendlyhi`

To run the app in detached mode `docker run -d -p 4000:80 friendlyhi`

Note: This port remapping of `4000:80` is to demonstrate the difference between what you `EXPOSE` within the Dockerfile, and what you publish using `docker run -p`.

**Step 5: Manually test your app**

Go to the host machine's browser and hit the url http://localhost:4000 to verify the app iss running good. 

**Step 6: Manually stopping the app/container**

Hit Ctrl+C in the terminal. Note, in case of windows, this wouldn't stop the container. You merely get the prompt back at your disposal. Then type `docker container ls` to list the running containers, followed by `docker container stop container_NAME_or_ID` to stop the container.

**Step 7: Push your image to remote repo/registry**

A registry is a collection of repositories, and a repository is a collection of images—sort of like a GitHub repository, except the code is already built. An account on a registry can create many repositories. 

The docker CLI uses Docker’s public registry by default. We'll be using Docker's public registry here just because it's free and pre-configured, but there are many public ones to choose from, and you can even set up your own private registry using Docker Trusted Registry.

Sign up for one at `http://cloud.docker.com` and login from  your docker cli like below:
`docker login` #username: sirkarthik

Tag  the image like:
Syntax: `docker tag image username/repo:tag`
Ex.: `docker tag friendlyhi sirkarthik/hello-pyworld:v1`

Publish the image
Syntax: `docker push containername:tag`
Ex.: `docker push sirkarthik/hello-pyworld:v1`

You can now use `docker run` to run your app from any machine with command like below:
`docker run -p 4000:80 sirkarthik/hello-pyworld:v1`

Note:  If you don't specify the :tag portion of these commands, the tag of :latest will be assumed, both when you build and when you run images. Docker will use the last version of the image that ran without a tag specified (not necessarily the most recent image).

To be sure your image works as a deployed container, run this command, slotting in your info for `username`, `repo`, and `tag`: `docker run -p 80:80 username/repo:tag`, then visit `http://localhost/`.

-------------------------------------------------------------------------------------------------------------------------------
Note: While typing docker run is simple enough, the true implementation of a container in production is running it as a service. Services codify a container’s behavior in a Compose file, and this file can be used to scale, limit, and redeploy our app. Changes to the service can be applied in place, as it runs, using the same command that launched the service: docker stack deploy.
-------------------------------------------------------------------------------------------------------------------------------

**Step 8: Create Compose file defining services behaviour**

Create a `docker-compose.yml` file in the project directory (or wherever you want).

**Step 9: Run your new load-balancecd app**

Setup Swarm (A swarm is a group of machines that are running Docker and joined into a cluster.):
`docker swarm init`

Let's give our app a name - getstartedlab. We configured our single service stack to runn 5 replica container instances of our deployed image on one host
`docker stack deploy -c docker-compose.yml getstartedlab`

Let's investigate a bit on how things are like below:

`docker service ls` # You’ll see output for the web service, prepended with your app name. The service ID is listed as well, along with the number of replicas, image name, and exposed ports. 

`docker service ps getstartedlab_web` # A single container running in a service is called a task. Tasks are given unique IDs that numerically increment, up to the number of replicas you defined in docker-compose.yml.

Tasks also show up if you just list all the containers on your system, though that will not be filtered by service:
`docker container ls -q`

**Step 10: Scaling the app**

You can scale the app by changing the replicas value in docker-compose.yml, saving the change, and re-running the docker stack deploy command:
`docker stack deploy -c docker-compose.yml getstartedlab`.

Docker will do an in-place update, no need to tear the stack down first or kill any containers.
`docker container ls`

**Step 11: Take down the app and the swarm**

Take the app down:
`docker stack rm getstartedlab`

Take down the swarm:
`docker swarm leave --force`

-------------------------------------------------------------------------------------------------------------------------------
It's as easy as that to stand up and scale your app with Docker. You've taken a huge step towards learning how to run containers in production. What we just did is take an app and define how it should run in production by turning it into a service and scaling it up 5x in the process.

Note: Compose files like this are used to define applications with Docker, and can be uploaded to cloud providers using [Docker Cloud](https://docs.docker.com/docker-cloud/), or on any hardware or cloud provider you choose with [Docker Enterprise Edition](https://www.docker.com/enterprise-edition).

Up next, you will learn how to run this app/servicec as a bonafide swarm on a cluster of Docker machines. That is, we will deploy this application onto a cluster, running it on multiple machines.

Multi-container, multi-machine applications are made possible by joining multiple machines into a “Dockerized” cluster called a **swarm**.

**Understanding Swarm clusters :** 
A swarm is a group of machines that are running Docker and joined into a cluster. After that has happened, you continue to run the Docker commands you’re used to, but now they are executed on a cluster by a swarm manager. The machines in a swarm can be physical or virtual. After joining a swarm, they are referred to as nodes.

Swarm managers can use several strategies to run containers, such as “emptiest node” – which fills the least utilized machines with containers. Or “global”, which ensures that each machine gets exactly one instance of the specified container. You instruct the swarm manager to use these strategies in the Compose file, just like the one you have already been using.

Swarm managers are the only machines in a swarm that can execute your commands, or authorize other machines to join the swarm as workers. Workers are just there to provide capacity and do not have the authority to tell any other machine what it can and cannot do.

Up until now, you have been using Docker in a single-host mode on your local machine. But Docker also can be switched into swarm mode, and that’s what enables the use of swarms. Enabling swarm mode instantly makes the current machine a swarm manager. From then on, Docker will run the commands you execute on the swarm you’re managing, rather than just on the current machine.
-------------------------------------------------------------------------------------------------------------------------------

## Issues N Fixes
* Issue: Docker for Windows 10 doesn't work for Windows 10 Home edition, because Hyper-V present in Pro edition is required as a prerequisite to run Docker natively in Windows.

  Fix: Use [Docker Toolbox](https://docs.docker.com/toolbox/overview/) instead that uses [VirtualBox](https://www.virtualbox.org) and [boot2docker](http://boot2docker.io/) linux distribution inside the VM as host machine for running docker containers.
 
* Issue: Couldn't connect to web-server instance running in Docker container from the browser in host machine using url - http://localhost:4000.

  Fix: The container port does get exposed at the Docker IP address, shown by `docker-machine ip Default`. Grab this ip of the docker container and substitute in place of localhost in the url. In my case the url turned  out to be `http://192.168.99.100:4000/`

## Tools
* In case of Windows, use [Visual Studio Code](https://code.visualstudio.com/) as IDE for convenience. Its light-weight and hass various plugins to support several languages and its frameworks.


## Docker Cheatsheet
```
docker -v # check docker version
docker images  # list locally available docker images
docker image  ls  # Newer API to list locally available docker images
docker run hello-world # test if docker works good

docker build -t friendlyname .  # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyname  # Run "friendlyname" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyname         # Same thing, but in detached mode
docker container ls                                # List all running containers
docker container ls -a             # List all containers, even those not running
docker container stop <hash>           # Gracefully stop the specified container
docker container kill <hash>         # Force shutdown of the specified container
docker container rm <hash>        # Remove specified container from this machine
docker container rm $(docker container ls -a -q)         # Remove all containers
docker image ls -a                             # List all images on this machine
docker image rm <image id>            # Remove specified image from this machine
docker image rm $(docker image ls -a -q)   # Remove all images from this machine
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
docker run username/repository:tag                   # Run image from a registry

docker stack ls                                            # List stacks or apps
docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
docker service ls                 # List running services associated with an app
docker service ps <service>                  # List tasks associated with an app
docker inspect <task or container>                   # Inspect task or container
docker container ls -q                                      # List container IDs
docker stack rm <appname>                             # Tear down an application
docker swarm leave --force      # Take down a single node swarm from the manager
```

## References
* [Get Started, Part 1: Orientation and setup](https://docs.docker.com/get-started/) is a 6-part tutorial
* [QnA: “Get started” course: Can’t connect to web server on Windows 10](https://forums.docker.com/t/get-started-course-cant-connect-to-web-server-on-windows-10/35795)
* [Add shared directories](https://docs.docker.com/toolbox/toolbox_install_windows/#optional-add-shared-directories) to your VM for easy access to your project directory; because by default Docker Toolbox only has access to the `C:\Users` directory and mounts it into the VMs at `/c/Users`.
