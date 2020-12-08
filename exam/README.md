Exam
----

Our application is ready to be realesed!

Today is the release day, link to Agama app was published in the Internet and
you start to serve your first happy customers. You monitor all your
insfastructure components and ready to react on any problems.

Of course everything that could go wrong -- goes wrong! Suddenly one of your DNS
instances just died without any reason. While you were checking the logs one
HAProxy crashed! And Docker containers with Agama app just start to disappear!

Luckily, you did a great job in the last few months to build fault-tolerant
infrastructure and happy customers didn't notice any problems with Agama app
during that time. Great job, you got many benefits and congratulations. But the
most important thing: now you can tell everyone that you sucessfully finished
"IT Infrastructure services" course in TalTech.

Sounds great?

There is one last step though -- put all the bits and pieces together to build
this infrastructure one last time.


Infrastructure
--------------

You will have 3 virtual machines to use: 1 for internal services and 2 for the
main application stack.

Each application stack machine should have these services set up and running:
 - HAProxy
 - Keepalived
 - Agama in the Docker container
 - MySQL (master on one machine, replica on the another)
 - Bind slave
 - Prometheus exporters for HAProxy, Keepalived, MySQL and Bind slave

Remaining machine should have these services set up and running:
 - Bind master
 - InfluxDB
 - Telegraf
 - Prometheus
 - Grafana
 - Nginx as frontend for Grafana and Prometheus


Differences with previous tasks
-------------------------------

We tried to make this infrastructure as similar as possible to what you did
during the course, however there are a few differences worth pointing out.

There is one Bind master and _two_ Bind slaves now; you don't need to add master
address to `/etc/resolv.conf` -- only both slave addresses. For example, in this
setup:

	192.168.42.35   # Bind slave
	192.168.42.87   # Bind slave
	192.168.42.124  # Bind master

`/etc/resolv.conf` could look like this:

	nameserver 192.168.42.35
	nameserver 192.168.42.87
	search mydomain.tld

Also, as there are three machines now, so your Grafana dashboards, backup
scripts and documentation should reflect and support it.


Task
----

Write the Ansible playbook named `infra.yaml` to deploy the infrastrusture
described above.

Infrastructure should be fully set up and online by running exactly this
command:

	ansible-playbook infra.yaml

Note: it is still allowed to set up MySQL replication manually as we didn't show
how to set it up with Ansible.

Provide the backup SLA document for the entire infrastructure (`backup_sla.md`)
and backup restore instructions for every service that is being backed up
(`backup_restore.md`).

**Be ready to rebuild your infrastructure from scratch on empty instances!**


Requirements
------------

1. After the infrastructure is deployed, running `ansible-playbook infra.yaml`
   command again should not produce any changes on the managed hosts
2. Every variable should be defined exactly once -- in `hosts`,
   `group_vars/all.yaml` or some other file if you feel needed
3. No active plain text passwords should be found anywhere in your repository
   (including history); note that it's okay to have old passwords in the code
   -- but only if you changed them already
4. No IP addresses should be used in configurations, except Bind and Keepalived
   configuration and `/etc/resolv.conf`. Should you address local machine, use
   `localhost` -- not `127.0.0.1`!


Testing
-------

If you have run the Ansible command mentioned above, and it did not trigger any
changes, you are ready to present your solution.

This is what we (teachers) may do to test your infrastructure.


**1. Kill any of the services mentined above**

You should be able to restore the infrastructure in exactly one Ansible run.

Note: to _restore_ the services with Ansible here and in other cases you may --
but don't have to -- use Ansible tags.


**2. Kill one of the highly available service instances**

Agama in Docker container, Bind (master or slave), Docker daemon, HAProxy,
Keepalived, MySQL.

The application should remain accessible for the clients, i. e. public URL of
Agama should remain working as nothing happened.

Your monitoring (Grafana) should be able to detect which component is down.

You should be able to restore the infrastructure in exactly one Ansible run.


**3. Kill one of the services and break its configuration file**

We will only break the files that are (or expected to be) managed with Ansible.

You should be able to restore the infrastructure in exactly one Ansible run.


**4 Break one of your services and ask you to find the problem**

We may tell you which service is broken, or you may need to identify that
yourself -- your monitoring (Grafana) will be helpful here.

We will only produce the damage that can be easily detected with commands
exaplined on the "Troubleshooting" lecture and lab, by checking
 - service status
 - service configuration syntax
 - service logs
 - ability to communicate with other services

For example, we may:
 - Make a typo in the HAProxy configuration
 - Change Bind configuration so that name resoution does not work
 - Change MySQL user, password, port
 - Stop Prometheus
 - etc.

You should be able to find the problem and explain what is broken and, ideally,
how to fix it.


**4. Reboot any of your VMs**

After the machine is started all your services there should be started
automatically **without any Ansible runs**, and of course without any manual
intervention.


**6. Ask you to change some configuration value**

Username, password, port, file permissions etc. for any of the service mentioned
above.

This change should be then fully applied in a single Ansible run.

> We will not ask you to make changes that silently break your infrastrucutre.
> If we do ask a breaking change -- we'll make it openly.


**7. Stop one of your services and resore it from the backup**

We will only use the instructions from your `backup_restore.md` document.

Note that _we_ will restore the service, not you.

**8. Ask you questions about your infrastructure**

A few examples:

 - How to check if the service X is working (multiple ways)?
 - What files does service X work with? Where are configuration files, where is
   working directory etc.
 - What other services does service X communicate to?
 - What services will be affected and how if machine A is turned off?
 - How do you back up your infrastructure (coverage, retention, versioning,
   usability, RPO/RTO etc.)?
 - When was the last backup for serice X created?

And others questions of similar complexity and detail level: we won't ask you to
disassemble the binary files -- but you should have a basic understanding of how
the deployed services work.

Expected answer is usually a few, most often one-two, sentences.


**9. Ask you questions about your code**

Be ready to explain every line of your code.

You don't need to learn the documentation by heart precisely. You can always
check it any time -- it's an open-everything exam. We well accept cited online
references as answers -- if they answer the question, of course :)


Good luck!
----------

![](./unicorn.png)
