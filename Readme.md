# Getting the Docker Big Picture by Example

## Objective
The objective of this exercise is to begin building an app the Docker way.

1. Defining Docker Container using Dockerfile
2. Deploying Docker Container on a Single Host using Compose (aka Docker-Compose)
3. Create VMs hosting Docker, using Docker-Machine
4. Deploying Docker Containers on many VMs/Hosts/Nodes in a same Network/Docker-Cluster using Swarm and Stack

## Undedrstanding the Big-picture

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

To be sure your image works as a deployed container, run this command, slotting in your info for `username`, `repo`, and `tag`: `docker run -p 80:80 username/repo:tag`, then visit `http://localhost/`. If that doesn't work, try the ip of the machine/container where the app is deployed as URL.

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

**Step 12: Create VMs dynamically using Docker-Machine**

Create VMs using docker-machine:
* `docker-machine create --driver virtualbox myvm1`
* `docker-machine create --driver virtualbox myvm1`

List to see the details of above VM:
`docker-machine ls`

Get ip of a vm machine by name
`docker-machine ip vm_name`
`docker-machine ip myvm1`
`docker-machine ip myvm2`

**Step 13: Create Swarm -- Initialize the swarm in one of the VMs and add notes to it**

The first machine will act as the manager, which executes management commands and authenticates workers to join the swarm, and the second will be a worker.

You can send commands to your VMs using `docker-machine ssh`. Instruct `myvm1` to become a swarm manager with `docker swarm init`:
`docker-machine ssh myvm1 "docker swarm init --advertise-addr <myvm1 ip>"` requires you to manually find the machine-ip (in my cases, 192.168.99.101) and fill it in place of `<myvm1 ip>`. Is there a better way where this can be automated?

To script the above command: `docker-machine ssh myvm1 "docker swarm init --advertise-addr $(docker-machine ip myvm1)"` where the command within `$()` gets executed in the host machine where docker-machine resides to find out the ip of the vm by its name. 

Executing the above command will return an output something that looks like below:
```
Swarm initialized: current node (n8oqj08mlkthdvn40bpgdeh84) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2t1tlwyj8aeutuqy0wnp9pm7zz0w5ac4cnfydrovd7lj1mqt4d-1g0sixf92fyvczd02k6ofl1w6 192.168.99.101:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
As you can see, the response to docker swarm init contains a pre-configured docker swarm join command for you to run on any nodes you want to add. Copy this command, and send it to myvm2 via docker-machine ssh to have myvm2 join your new swarm as a worker:
```
$ docker-machine ssh myvm2 "docker swarm join \
--token <token> \
<ip>:2377"

This node joined a swarm as a worker.
```
Congratulations, you have created your first swarm!

Run docker node ls on the manager to view the nodes in this swarm:
```
$ docker-machine ssh myvm1 "docker node ls"
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
brtu9urxwfd5j0zrmkubhpkbd     myvm2               Ready               Active
rihwohkh3ph38fhillhhb84sk *   myvm1               Ready               Active              Leader
```

**Step 14: Deploy app on the swarm cluster**

Configure a docker-machine shell to the swarm manager:

So far, you’ve been wrapping Docker commmands in docker-machine ssh to talk to the VMs. Another option is to run docker-machine env <machine> to get and run a command that configures your current shell to talk to the Docker daemon on the VM. This method works better for the next step because it allows you to use your local docker-compose.yml file to deploy the app “remotely” without having to copy it anywhere.

Type `docker-machine env myvm1`, then copy-paste and run the command provided as the last line of the output to configure your shell to talk to `myvm1`, the swarm manager.

Run `docker-machine env myvm1` to get the command to configure your shell to talk to `myvm1`.
```
$ docker-machine env myvm1
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/sam/.docker/machine/machines/myvm1"
export DOCKER_MACHINE_NAME="myvm1"
# Run this command to configure your shell:
# eval $(docker-machine env myvm1)
```
Now, run the given command to configure your shell to talk to `myvm1`:
`$ eval $(docker-machine env myvm1)`

Run `docker-machine ls` to verify that `myvm1` is now the active machine, as indicated by the asterisk next to it.
```
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
myvm1   *        virtualbox   Running   tcp://192.168.99.100:2376           v17.06.2-ce   
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v17.06.2-ce   
```

You are connected to `myvm1` by means of the `docker-machine` shell configuration, and you still have access to the files on your local host. Make sure you are in the same directory as before, which includes the `docker-compose.yml` file you created earlier.

Now that you have my `myvm1`, you can use its powers as a swarm manager to deploy your app by using the same `docker stack deploy` command you used in Step 10 to `myvm1`, and your local copy of `docker-compose.yml`.

Just like before, run the following command to deploy the app on `myvm1`: 
`docker stack deploy -c docker-compose.yml getstartedlab`

And that’s it, the app is deployed on a swarm cluster! Only this time you’ll see that the services (and associated containers) have been distributed between both `myvm1` and `myvm2`.
```
$ docker stack ps getstartedlab
ID            NAME                  IMAGE                   NODE   DESIRED STATE
jq2g3qp8nzwx  getstartedlab_web.1   john/get-started:part2  myvm1  Running
88wgshobzoxl  getstartedlab_web.2   john/get-started:part2  myvm2  Running
vbb1qbkb0o2z  getstartedlab_web.3   john/get-started:part2  myvm2  Running
ghii74p9budx  getstartedlab_web.4   john/get-started:part2  myvm1  Running
0prmarhavs87  getstartedlab_web.5   john/get-started:part2  myvm2  Running
```

Accessing your cluster: You can access your app from the IP address of either `myvm1` or `myvm2`. The network you created is shared between them and load-balancing. Run `docker-machine ls` to get your VMs' IP addresses and visit either of them on a browser, hitting refresh (or just curl them). In my case it turns out to be `http://192.168.99.101/` or `http://192.168.99.102/`

You’ll see five possible container IDs all cycling by randomly, demonstrating the load-balancing.

The reason both IP addresses work is that nodes in a swarm participate in an ingress routing mesh. This ensures that a service deployed at a certain port within your swarm always has that port reserved to itself, no matter what node is actually running the container. Here’s a diagram of how a routing mesh for a service called my-web published at port 8080 on a three-node swarm would look:
![alt text](https://docs.docker.com/engine/swarm/images/ingress-routing-mesh.png "Ingress Routing Mesh")

-----------------------------------------------------------------------------------------------------------------------------
**Connecting to VMs with `docker-machine env` and `docker-machine ssh`**
* To set your shell to talk to a different machine like `myvm2`, simply re-run `docker-machine env` in the same or a different shell, then run the given command to point to `myvm2`. This is always specific to the current shell. If you change to an unconfigured shell or open a new one, you need to re-run the commands. Use `docker-machine ls` to list machines, see what state they are in, get IP addresses, and find out which one, if any, you are connected to.
* Alternatively, you can wrap Docker commands in the form of `docker-machine ssh <machine> "<command>"`, which logs directly into the VM but doesn't give you immediate access to files on your local host.
* On Mac and Linux, you can use `docker-machine scp <file> <machine>:~` to copy files across machines, but Windows users need a Linux terminal emulator like [Git Bash](https://git-for-windows.github.io/) in order for this to work.

This tutorial demos both `docker-machine ssh` and `docker-machine env`, since these are available on all platforms via the `docker-machine` CLI.

-----------------------------------------------------------------------------------------------------------------------------

**Step 15: Iterating and scaling the app**

Scale the app by changing the `docker-compose.yml` file.

Change the app behavior by editing code, then rebuild, and push the new image.

In either case, simply run `docker stack deploy` again to deploy these changes.

You can join any machine, physical or virtual, to this swarm, using the same `docker swarm join` command you used on `myvm2`, and capacity will be added to your cluster. Just run `docker stack deploy` afterwards, and your app will take advantage of the new resources.

**Step 16: Cleanup and reboot**

You can tear down the stack with docker stack rm. For example:
`docker stack rm getstartedlab`

You can remove this swarm if you want to with `docker-machine ssh myvm2 "docker swarm leave"` on the worker and 
`docker-machine ssh myvm1 "docker swarm leave --force"` on the manager

You can unset the `docker-machine` environment variables in your current shell with the following command:
`eval $(docker-machine env -u)`
This disconnects the shell from docker-machine created virtual machines, and allows you to continue working in the same shell, now using native docker commands (for example, on Docker for Mac or Docker for Windows).

Restarting Docker machines: If you shut down your local host, Docker machines will stop running. You can check the status of machines by `running docker-machine ls`.
```
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL   SWARM   DOCKER    ERRORS
myvm1   -        virtualbox   Stopped                 Unknown
myvm2   -        virtualbox   Stopped                 Unknown
```

To restart a machine that’s stopped, run:
`docker-machine start <machine-name>`

To stop/kill the vm you created, you can use either `docker-machine stop vmname` or `docker-machine kill vmname`:
* `docker-machine stop myvm1`
* `docker-machine kill myvm2`

You can check the status of vm if its stopped or running like below: 
* `docker-machine status myvm1` 
* or by simply using `docker-machine ls`

You can start or restart like below:
* `docker-machine start myvm1`
* `docker-machine restart myvm2`

You can remove the VMs created using `docker-machine` by using `rm` command like below:
* `docker-machine rm myvm1`
* `docker-machine rm myvm2`

**Step 17: Add a new service and redeploy**

This step is about reachcing the top of the hierarchy of distributed applications: the **stack**.

-----------------------------------------------------------------------------------------------------------------------------
A **stack** is a group of interrelated services that share dependencies, and can be orchestrated and scaled together.

 A single stack is capable of defining and coordinating the functionality of an entire application (though very complex applications may want to use multiple stacks).

-----------------------------------------------------------------------------------------------------------------------------

 We technically have been using stacks from the earlier steps by creating a Compose file and executing the command `docker stack deploy`. But that was a single service stack running on a single host, which is not usually what takes place in production.

 Here, you will take what you've learned, make multiple services relate to each other, and run them on multiple machines.

It’s easy to add services to our `docker-compose.yml` file. First, let’s add a free visualizer service that lets us look at how our swarm is scheduling containers.

1. Open up docker-compose.yml in an editor and replace its contents with the following. Be sure to replace username/repo:tag with your image details.
```
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
    ports:
      - "80:80"
    networks:
      - webnet
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
networks:
  webnet:
```
The only thing new here is the peer service to `web`, named `visualizer`. You’ll see two new things here: a `volumes` key, giving the visualizer access to the host’s socket file for Docker, and a `placement` key, ensuring that this service only ever runs on a swarm manager – never a worker. That’s because this container, built from [an open source project created by Docker](https://github.com/ManoMarks/docker-swarm-visualizer), displays Docker services running on a swarm in a diagram.



## Issues N Fixes
* Issue: Docker for Windows 10 doesn't work for Windows 10 Home edition, because Hyper-V present in Pro edition is required as a prerequisite to run Docker natively in Windows.

  Fix: Use [Docker Toolbox](https://docs.docker.com/toolbox/overview/) instead that uses [VirtualBox](https://www.virtualbox.org) and [boot2docker](http://boot2docker.io/) linux distribution inside the VM as host machine for running docker containers.
 
* Issue: Couldn't connect to web-server instance running in Docker container from the browser in host machine using url - http://localhost:4000.

  Fix: The container port does get exposed at the Docker IP address, shown by `docker-machine ip Default`. Grab this ip of the docker container and substitute in place of localhost in the url. In my case the url turned  out to be `http://192.168.99.100:4000/`

* Issue: How do I open another terminal
  
  Fix: Simply double-click the "Docker Quickstart Terminal" in your host windows machine again :)

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

docker-machine create --driver virtualbox myvm1 # Create a VM (Mac, Win7, Linux)
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" myvm1 # Win10
docker-machine env myvm1                # View basic information about your node
docker-machine ssh myvm1 "docker node ls"         # List the nodes in your swarm
docker-machine ssh myvm1 "docker node inspect <node ID>"        # Inspect a node
docker-machine ssh myvm1 "docker swarm join-token -q worker"   # View join token
docker-machine ssh myvm1   # Open an SSH session with the VM; type "exit" to end
docker node ls                # View nodes in swarm (while logged on to manager)
docker-machine ssh myvm2 "docker swarm leave"  # Make the worker leave the swarm
docker-machine ssh myvm1 "docker swarm leave -f" # Make master leave, kill swarm
docker-machine ls # list VMs, asterisk shows which VM this shell is talking to
docker-machine start myvm1            # Start a VM that is currently not running
docker-machine env myvm1      # show environment variables and command for myvm1
eval $(docker-machine env myvm1)         # Mac command to connect shell to myvm1
& "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env myvm1 | Invoke-Expression   # Windows command to connect shell to myvm1
docker stack deploy -c <file> <app>  # Deploy an app; command shell must be set to talk to manager (myvm1), uses local Compose file
docker-machine scp docker-compose.yml myvm1:~ # Copy file to node's home dir (only required if you use ssh to connect to manager and deploy the app)
docker-machine ssh myvm1 "docker stack deploy -c <file> <app>"   # Deploy an app using ssh (you must have first copied the Compose file to myvm1)
eval $(docker-machine env -u)     # Disconnect shell from VMs, use native docker
docker-machine stop $(docker-machine ls -q)               # Stop all running VMs
docker-machine rm $(docker-machine ls -q) # Delete all VMs and their disk images
```

## References
* [Get Started, Part 1: Orientation and setup](https://docs.docker.com/get-started/) is a 6-part tutorial
* [QnA: “Get started” course: Can’t connect to web server on Windows 10](https://forums.docker.com/t/get-started-course-cant-connect-to-web-server-on-windows-10/35795)
* [Add shared directories](https://docs.docker.com/toolbox/toolbox_install_windows/#optional-add-shared-directories) to your VM for easy access to your project directory; because by default Docker Toolbox only has access to the `C:\Users` directory and mounts it into the VMs at `/c/Users`.
