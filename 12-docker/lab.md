# Lab 12

In this lab we will install Docker and redeploy a few services to run in Docker
containers. Playbook for this lab should be named `lab12_docker.yaml`.

We'll learn how to launch a container from Docker Hub image, and also how to
build the Docker image locally.

Some hints common for all tasks:
 - should you need to debug someting inside the container you can run most
   standard Linux commands there as `docker exec <container-name> <command>`,
   for example `docker exec grafana cat /etc/resolv.conf`
 - container name can be found in the `docker ps` output (last column)

You may -- but don't have to -- install all services mentioned below on the one
Docker host. You may use one or all your virtual machines for that.


## Task 1

Add another role named `docker` that will install Docker on your managed hosts.

Note: double check the pakcgae name you are installing! Package named `docker`
in Ubuntu package repository has nothing to do with containers. You will need a
package named `docker.io` (yes, with dot). You can find the package description
with thes commands:

	apt show docker
	apt show docker.io

You will also need to install another package to allow Ansible to manage Docker
resources. The package name is `python3-docker`, it is a Python library that
allows Ansible modules to execute Docker commands.

Ensure that the Docker daemon is running and is enabled to start on system boot.

You can check if Docker daemon is running with this command run as root on the
managed host:

	docker info


## Task 2

Stop and feel free to delete the old Grafana introduced in
[lab 7](../07-grafana/lab.md) -- it can always be restored from Ansible and
backups if needed. Remove the old `grafana` role from the plays so that Ansible
won't restart the old Grafana accidentally.

Install the new Grafana that runs in Docker conatiner.

Use Docker image named
[grafana/grafana](https://hub.docker.com/r/grafana/grafana)) from the Docker
Hub.

> Note that the image is named `grafana/grafana`. This is unofficial (from
> Docker Hub point of view) image hosted by user `grafana` (Grafana development
> team), and the image name is also `grafana`. If it were hosted by user `elvis`
> image would be called `elvis/grafana`.

Ansible module
[docker_container](https://docs.ansible.com/ansible/2.9/modules/docker_container_module.html)
may be helpful here.

Add the new role named `grafana_docker` for this task. Keep the existing role
`grafana` in your repository -- do not delete it.

On the Docker host create a working directory that will be later mounted to
Grafana container:

	/opt/docker/grafana

Note that Grafana is running in the container with specific and documented user
id:
[472](https://grafana.com/docs/grafana/latest/installation/docker/#migrate-to-v51-or-later).
This means that working directory (`/opt/docker/grafana` on the Docker host and
`/var/lib/grafana` in the container) should be owned by the **same** user:

	$ ls -la /opt/docker/
	...
	drwxr-xr-x  4  root  root  4096  Nov 15 21:90  some-dir
	drwxr-xr-x  4   472   472  4096  Nov 15 21:09  grafana         # <-- this
	drwxr-xr-x  4  root  root  4096  Nov 15 21:09  some-other-dir
	...

> Don't worry that the direcotry is owned by just a 'number'. You don't need to
> create the user with this UID explicitly. Once container is started the
> correct user with the correct UID will be added there.

You will need to mount Grafana working directory to the container and allow
Grafana user to write there -- provide these paramneters to your
`docker_container` taks module:

	volumes:
	  - /opt/docker/grafana:/var/lib/grafana

Make sure to expose ('publish') from the container the port that Grafana is
listening on. Use port 3001 on the Docker host, and make this number a variable:

	published_ports:
	  - "{{ grafana_port }}:3000"

Also make sure to update the Nginx proxy configuration accordingly, for example:

	proxy_pass: http://localhost:{{ grafana_port }};

In the previous labs you've achieved this by modifying the Grafana configuration
file and adding a few settings to the `[server]` section. There is another way
to do it that may be more convenient for Docker setups -- environment variables:

	env:
	  GF_SERVER_ROOT_URL: "http://localhost:{{ grafana_port }}/grafana"
	  GF_SERVER_SERVE_FROM_SUB_PATH: "true"

Note that `"true"` must be in quotes here for Ansible to interpret it as string,
not as boolean. Just `true` will not work.

Related docs -- none of them has the exact solution but they pretty well explain
all the parts that you'll need:
 - https://grafana.com/tutorials/run-grafana-behind-a-proxy -- Grafana
   configuration to run behind reverse proxy
 - https://grafana.com/docs/grafana/latest/administration/configuration/#configure-with-environment-variables
   -- Grafana configuration with environment variables that override the
   configuration file

You can check that Grafana container is running with this command:

	docker ps

If the container is not starting or not working as expected you can check its
logs with this command:

	docker logs grafana

Hint: it also supports follow mode:

	docker logs -f grafana  # exit with Ctrl+C

Once installed, new dockerized Grafana should be accessible on the same URL as
the old one before, via Nginx proxy.


## Task 3

New shiny -- but unfortunately empty -- Grafana is now running in Docker. But
your old one has backups, right?

Restore the data from the old Grafana to the new one using the backup restore
procedure for Grafana described in your `backup_restore.md` document you've
composed in the [lab 10](../10-backups/lab.md).

**Make sure to stop the Grafana container before restoring the data!**

Note that directory to restore to is `/opt/docker/grafana` now instead of
`/var/lib/grafana`, and file
ownership may need to be changed accordingly. You can use this command (run as
root or with sudo) to change the owner of all files in the directory
recursively:

	chown -R 472:472 /opt/docker/grafana

It accepts both user names and UIDs.

Once the data is restored, run Ansible again -- it should bring the Grafana
container back online, and the Grafana should read in the data restored from
the backup.

Once done and Grafana was started successfully with restored data, update your
`backup` role to also backup the new Grafana directory. Do not replace the
existing procedure -- just add a new one and make sure these backups are kept
separate.

Update `backup_restore.md` and include the required steps to restore the
Grafana data for the Grafana running in Docker container. Do not replcase the
old procedure for Grafana backup restore -- just add a new one.

If needed, also update the `backup_sla.md` accordingly.


## Task 4

Stop and feel free to delete the old Agama stack: uWSGI and Agama files
introduced in [lab 3](../03-web-app/lab.md) -- they can always be restored from
Ansible and backups if needed. Remove the old `uwsgi` and `agama` roles from the
plays so that Ansible won't restart the old Agama accidentally.

Install the new Agama that runs in Docker container.

As in previous tasks, create a new role named `agama_docker` and keep the
previous role in the repository.

Stop and delete uWSGI service, and also all files related to Agama from
`/opt/agama` and `/etc/uwsgi`.

Agama does not have an image in the Docker Hub so it needs to be built on the
managed host.

Dockerfile can be found in the Agama repository:
https://github.com/hudolejev/agama/blob/master/Dockerfile -- click 'Raw' to get
the direct download URL. You will need to download it to the managed host --
Ansible module
[get_url](https://docs.ansible.com/ansible/2.9/modules/get_url_module.html)
may be useful.

Save this file as `/opt/agama/Dockerfile`.

Docker image can be built from this file using Ansible module
[docker_image](https://docs.ansible.com/ansible/2.9/modules/docker_image_module.html)

It has many arguments but don't get too excited with that. Something simple like
this would work just fine:

	docker_image:
	  name: agama
	  source: build
	  build:
	    path: /opt/agama

 - the build directory would be `/opt/agama`
 - image name could be `agama`

Please **do not** push the image to Docker Hub.

Once done, use `docker_container` module to start the container from the built
image, similarly as you did for the Grafana. Make sure to use the same image
name (`agama`) in `docker_image` and `docker_container` modules so that Docker
would use your local image instead of downloading it from Docker Hub.

Note that you will need to provide additional environment variable to the
container with MySQL connection URL for agama, similarly as you did for uWSGI in
the [lab 4](../04-troubeshooting/lab.md#task-8-web-application):

	env:
	  AGAMA_DATABASE_URI: mysql://<...>

Note that Docker container exposes the port 8000 and the service listening there
is HTTP server -- not uWSGI.

Expose the container port as some unused port on Docker host -- can be even the
same 8000 if you want but this port number should be a variable. Tweak your
Nginx proxy configuration accordingly.

Once installed, new dockerized AGAMA should be accessible on the same URL as the
old one before, via Nginx proxy.


## Expected result

Your repository contains these files:

	backup_restore.md
	lab12_docker.yaml
	roles/agama_docker/tasks/main.yaml
	roles/docker/tasks/main.yaml
	roles/grafana_docker/tasks/main.yaml

Your Agama application is accessible on its public URL from
[this list](http://193.40.156.86/vms.html).

Your Grafana is accessible on its public URL.

Agama and Grafana are both running in Docker containers; no local services named
`grafana-server` or `uwsgi` are running on any of your managed hosts.
