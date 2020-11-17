# Lab 12 demo: Docker basics

## Intro

First, we will need to install Docker daemon and client tools. There are
multiple ways how to do it, but we will use the simplest possible approach for
the demo:

	- name: Docker package
	  apt:
	    name: docker.io

	- name: Docker service
	  service:
	    name: docker
	    state: started
	    enabled: yes

Note the package name, `docker.io` -- this name is inspired by previous Docker
website name and chosen not to conflict with another package in Debian and
Ubuntu repositories called `docker`; the latter has nothing to do with
containers:

	Package: docker
	Description: System tray for KDE3/GNOME2 docklet applications

	Package: docker.io
	Description: Linux container runtime

Also note that if you decide to install the package from the Docker own
repository as described
[here](https://docs.docker.com/install/linux/docker-ce/ubuntu/) the DEB package
name will be different again: `docker-ce`.

For this demo a bit older Docker version from Ubuntu `universe` repository is
just fine.

Next, let's make sure that Docker container runtime itself is up and running:

	systemctl status docker

... and the Docker client is working:

	docker info


## First Docker container

Docker team has prepared one very simple container image to verify the Docker
installation. Let's download and run it:

	docker run hello-world

It should print something like

	Unable to find image 'hello-world:latest' locally
	latest: Pulling from library/hello-world
	0e03bdcc26d7: Pull complete 
	Digest: sha256:8c5aeeb6a5f3ba4883347d3747a7249f491766ca1caa47e5da5dfcf6b9b717c0
	Status: Downloaded newer image for hello-world:latest

	Hello from Docker!
	This message shows that your installation appears to be working correctly.

	...

Read the output carefully. It explains in details what Docker just did.

Docker Hub is a public Docker registry, in simple words, it is the "package
repository" for Docker where package is the container image.

The action above has left some artifacts on the Docker host, namely the
downloaded container image:

	docker images

Now if you run `docker run hello-world` again it will not re-download the
container but use the local version instead:

	docker run hello-world


## Second Docker container

We cannot do much with this `hello-world` app. So let's run another container:

	docker run -it alpine /bin/sh

 - `alpine` here stands for image name; this is an image for container with
    [Alpine Linux](https://alpinelinux.org/), a minimalist Linux distribution
    quite popular for containerized applications;
 - `-it` means `--interactive --tty` which instructs Docker daemon to start the
   interactive session with the container and allocate the pseudo-terminal for
   that; check `docker run --help` for more details;
 - `/bin/sh` is the command we asked Docker to run inside the container

As a result, terminal prompt should change to something like

	/ #

This is the shell inside the container! Let's look around there:

	cat /etc/issue
	> Welcome to Alpine Linux 3.12
	> Kernel \r on an \m (\l)

	uname -a
	> Linux 6dbd86ae0998 4.15.0-76-generic #86-Ubuntu SMP Fri Jan 17 17:24:28 UTC 2020 x86_64 Linux

The first command will print the system identification -- the system running in
the container as seen by applications is Alpine Linux v3.12.

The second command will print the Linux kernel version, and you can easily
notice that the kernel (Ubuntu) does not quite match the userspace (Alpine).

This is how operating system virtualization (or containerization) work: both
host and guest systems share the same Linux kernel but their userspaces are
isolated from each other. Let's make some damage in the container:

	# Make sure to run this in the container, *not* on the Docker host!
	rmdir /opt
	exit

Note that `/opt` directory is still present on the host; it was only removed
from the container:

	ls -la /opt

Also note that the Linux kernel version on the host system is the same as was in
the container:

	uname -a


## Containerized service example

Docker containers were designed to run services. To be more precise, every
container is expected to run exactly one process. In previous example the
process was `/bin/sh` and it was running in interactive mode. Once we terminated
the session the process was terminated as well. You can list the terminated
Docker processes with

	docker ps -a

Running processes can be listed with

	docker ps

Note that there are none. So let's start on of the processes in Alpine Linux
container and leave it running in daemon mode. This process could be a `top`
utility:

	docker run -d alpine top

`-d` here means `--detach`: Docker will launch the container and leave it
running without keeping any interactive session open.

The process should now appear in the list of running Docker containers:

	docker ps

You can attach to the running container using its id:

	docker attach 4e411b27a685

You should see the `top` process running. Note that it is the only running
process in this container; although it is technically possible to launch more
than one process in the same container it is still a bit witchcraft, and Docker
discourages these attempts.

You can detach from the process by pressing `Ctrl+C`. The process will be
terminated, and so the container that hosted it:

	docker ps -a


## Some real services

Now once we know the basics of the Docker container anatomy we can try running
some real services, containerized. One possible candidate is Nginx web server:

	docker run -d nginx
	docker ps

It will take some time to download, but in the end you should see the Nginx
container running and listening on port 80/tcp. But is you check the local
processes listening on port 80 you will find none:

	netstat -lnpt

(If there is something listening on port 80, it may be another Nginx installed
locally on one of the previous labs -- not the one from the container.)

This is another example of process isolation. By default Docker containers
will not bind to host system ports automatically, this needs to be allowed
explicitly. Let's destroy the running Nginx container and launch it again, this
time providing the port configuration:

	docker stop bb5f196693ea
	docker run -d -p8081:80 nginx

`-p8081:80` here means `--publish 8081:80` -- this exposes container's port 80
and binds it to host's port 8081. This is done by another service called Docker
proxy that listens on port 8081 on the Docker host and forwards all the incoming
traffic to the port 80 of the container:

	netstat -lnpt

It should now be possible to communicate with the Nginx process running in the
container via port 8081 on the Docker host:

	curl http://localhost:8081

should return the page served by Nginx.


## Grafana

Let's get it further and deploy the entire Grafana in Docker container:

	docker run -d -p3001:3000 --name=grafana grafana/grafana

Note the `--name=grafana` part. We are naming the container `grafana` --
otherwise Docker will generate some random name for it. You can always find the
container name in the last column of `docker ps` output.

Grafana will listen on the port 3000 in the container but we've instructed
Docker proxy to bind to port 3001 on the Docker host and forward all the
incoming traffic to the port 3000 in the container:

	netstat -lnpt

We can access Grafana login page from the same host now:

	curl http://localhost:3001
	curl -L http://localhost:3001

But it's not really usable that way. As you remember we've set up a Nginx proxy
on the previous labs to serve Grafana from a custom path like
`http://193.40.156.86:xx80/grafana`, and that required additional Grafana
configuration. We achieved that by updating the Grafana configuration file that
contained something like this:

	[server]
	root_url = http://localhost:3000/grafana
	serve_from_sub_path = true

Although it is possible to do it inside Docker container as well there is a
simpler way. Grafana can also be configured with environment variables (we did
that already with Agama as you remember).

If we would be running Grafana locally and use environment variables for
configuration we would do something like this:

	#
	# Do not run this code please; it's just an example
	#

	export GF_SERVER_ROOT_URL=http://localhost:3000/grafana
	export GF_SERVE_FROM_SUB_PATH=true
	./grafana-server

Environment variables are not copied to Docker container by default -- remember,
the whole idea of the container is to _isolate_ the process -- but can be
provided explicitly. Let's kill the existing container and start a new one, with
better configured Grafana:

	docker stop grafana
	docker rm grafana

	docker run -d -p3001:3000 --name=grafana \
	  -eGF_SERVER_ROOT_URL=http://localhost:3000/grafana \
	  -eGF_SERVER_SERVE_FROM_SUB_PATH=true \
	  grafana/grafana

It will take some time to start and initialize, maybe several seconds.

Note that Grafana will now redirect to the login page at `/grafana/login`:

	curl http://localhost:3001


## Managing Docker containers with Ansible

At this point it makes sense to stop doing the damage manually and switch to
Ansible. Yes, there is a module to manage Docker containers:
[docker_container](https://docs.ansible.com/ansible/2.9/modules/docker_container_module.html)!

Let's take the last demonstarted `docker run` commands and reimplement then with
Ansible:

	- name: Grafana container
	  docker_container:
	    name: grafana
	    image: grafana/grafana
	    env:
	      GF_SERVER_ROOT_URL: http://localhost:3000/grafana
	      GF_SERVER_SERVE_FROM_SUB_PATH: "true"
	    ports:
	      - 3001:3000

And also update the Nginx proxy setting so that it'll serve Grafana from the
Docker container. We'll need to change the proxy port for that:

    location /grafana {
        proxy_pass http://localhost:3001;  # <-- here
        proxy_set_header Host $http_host;
    }

After Ansible is run to apply the changes Grafana should be available on one of
your public URLs:

	http://193.40.156.86:xxx80/grafana

Default login credentials are `admin:admin`.

If you have another version of Grafana running locally (not in Docker) you may
stop it to be sure that the Docker one is being served:

	systemctl stop grafana-server

Given that InfluxDB is set up on the same machine, let's try adding the
datasource in Grafana.

 - URL `http://localhost:8086` will fail because Grafana is running in the
   isolated container, and `localhost` can only reach the services inside the
   _same container_ -- not on the same host anymore
 - URL `http://influxdb.<your-domain>:8086` will probably work because by
   default Docker container inherits DNS settings from Docker host

You can verify that by running a command inside the container. This is done on
the Docker host. As you rememebr `docker attach` will break it, but this one
would do the trick:

	docker exec c3ef3473f0c0 cat /etc/resolv.conf  # resolver settings inside container
	cat /etc/resolv.conf                           # resolver settings on Docker host


## Sharing files between host and container

There is one unsolved problem though. Once we destroy the container with

	docker stop grafana
	docker rm grafana

-- all our modifications in Grafana will be lost forever.

This can be solved by sharing a directory from Docker host to the running
container. This is called 'mounting', and the container will handle this
directory as a volume (attached disk). Good thing about this approach is that
these volumes are not deleted once container is. Directories can be mounted to
the containers with `-v` parameter:

	docker run -v/opt/docker/grafana:/var/lib/grafana <...>

...or with `docker_container.volumes` attribute in Ansible:

	docker_container:
	  ...
	  volumes:
	    - /opt/docker/grafana:/var/lib/grafana

In both examples directory `/opt/docker/grafana` from Docker host is mounted as
`/var/lib/grafana` inside the container. Nothe that there may be
`/var/lib/grafana` on the Docker host as well if the Grafana is also installed
locally. Usually it is a good idea to keep these directories separated, so that
Grafana from Docker container won't ruin the data of the local Grafana.

It is recommended to keep the data directories for the container in
`/opt/docker` or `/srv/docker`. Not a rule, just a personal recommendation from
Juri.

Run the changed Ansible task. Is the container still up?

	docker ps
	docker ps -a

It is not. We can eaasily find the problem by checking the container logs:

	docker logs grafana

Where we'll see something like

	mkdir: can't create directory '/var/lib/grafana/plugins': Permission denied

On the Docker host you may notice that the `/opt/docker/grafana` directory is
created but is owned by `root`:

	ls -la /opt/docker

So this directory `/opt/docker/grafana` mounted into the container as
`/var/lib/grafana` is not writeable for the Grafana user. With that, Grafana
process fails to start, and the container dies.

But how do we know what's the user name and UID in the container?

The log mentioned above also contains the link to documentation (you can also
easily find it with `grafana docker` query in your web search engine of choice):
http://docs.grafana.org/installation/docker/#migration-from-a-previous-version-of-the-docker-container-to-5-1-or-later

This document  mentions the UID configured in Docker image, so every Grafana
container built from this image will have exactly the same UID for the Grafana
user.

Let's create this directory and set the correct ownership before starting the
Grafana container:

	- name: Grafana directory
	  file:
	    name: /opt/docker/grafana
	    state: directory
	    owner: "472"
	    group: "472"
	    recurse: true

Run the Ansible again. Once the changes are applied Grafana container should
start successfully, and Grafana should welcome you on the public URL.

You can now make changes in the Grafana, kill and restart containers as many
times as you want:

	docker stop grafana
	docker rm grafana

After you run the Ansible again the container will be restored and should read
in all the data that the previous container had stored in `/opt/docker/grafana`
on the Docker host.

Now make sure you won't wipe this directory. It is a wise idea to backup it :)


## Summary

In this demo we have discussed main Docker container lifecycle  stages.

We've also learned how to

 - start Docker containers with `docker run`
 - publish container ports with `docker run -p`
 - mount directories to the containers with `docker run -v`
 - list available Docker images with `docker images`
 - list running Docker containers with `docker ps` and failed containers with
   `docker ps -a`
 - run commands inside containers with `docker exec`
 - stop containers with `docker stop`
 - delete containers with `docker rm`
 - read container logs with `docker logs`


## Final notes

This is a very basic demo on how to handle Docker containers. We did not create
any container images, yet, but only used available, ready made images from
Docker Hub.

Images we used:
 - https://hub.docker.com/_/hello-world
 - https://hub.docker.com/_/alpine
 - https://hub.docker.com/_/nginx/
 - https://hub.docker.com/r/grafana/grafana

**SECURITY NOTE ABOUT DOCKER HUB**

Do not blindly trust images from Docker Hub!

If downloading an image, make sure it is an official Docker image!

See also: https://docs.docker.com/docker-hub/official_images

Sometimes it is hard to tell if the image is released buy Docker team or the
product team. Check out these two and carefully inspect the pages:

 - https://hub.docker.com/_/nginx
 - https://hub.docker.com/r/grafana/grafana

Only one of them is official Docker image as it is built by Docker team.
Another, although also called 'official', is not provided by Docker but by the
product developers. Both teams interpret the word 'official' differently; you
should understand this difference if working with Docker Hub.

**Only use Docker Hub images from the authors whom you trust!**
