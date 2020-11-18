# Lab 4

In this lab we'll improve setup from the [lab 3](../03-web-app/lab.md) by adding
a separate database server for our app. We'll also learn how to use Ansible
variables and Vault.

**Important!**

This and some following labs have tasks to handle secrets (passwords, keys
etc.). Make sure **not to commit plain text secrets to GitHub!**

Should you make this mistake, change the secret at once, encrypt it propperly
(details are provided below and in lecture slides) and push the next Git commit
that overwrites the secret. Leaked secret value will still remain in the Git
history but as you have changed it it's not a big problem anymore.

Note that your solution is not accepted if your Git history contains secrets
that are still valid (can be used to access your running services).


## Task 1: Ansible Vault

First, create a master password. It will be used to encrypt and decrypt other
secrets in your Ansible repository. Use any password generator that you like.

Then, save this password to a file **outside of your Ansible repository**. One
good choice is `~/.ansible/vault_password` -- but you can use other path if you
want. This file should contain just password, nothing more:

	$ cat ~/.ansible/vault_password
	y0ur_p4ssw0rd_h3r3

Make sure this file is readable only to you. You can use `chmod` command to set
the file permissions:

	chmod 600 ~/.ansible/vault_password

Finally, configure the Ansible to read the Vault password from this file --
update `ansible.cfg` and add the following setting to the `defaults` section:

	vault_password_file = ~/.ansible/vault_password

(modify as needed if you use a different path).

You can verify that you did everything correctly by running these commands in
the root of your Ansible repository. Run them exactly as written, without any
additional parameters:

	# Create a plain text file
	echo WORKS > ansible-vault-test.txt

	# Encrypt this file
	ansible-vault encrypt ansible-vault-test.txt
	# Should print 'Encryption successful'

	# Decrypt this file and print the decrypted text
	ansible-vault view ansible-vault-test.txt
	# Should print 'WORKS'

	# If that worked, delete the file, we don't need it anymore
	rm ansible-vault-test.txt

If these commands run without any errors and you could decrypt the file --
you're all set and good to go.


## Task 2: Ansible inventory

For this lab you'll need two virtual machines: one for an app set up in the
previous lab, and another for a standalone database server.

Empty VMs are created for you already and can be found
[here](http://193.40.156.86/vms.html) as usually. If you don't see a VM, then
first thing -- pleas make sure that your
[Git repository is set up correctly](http://193.40.156.86/students.html). If it
is, please contact teachers.

Update your Ansible inventory file and add the new VM there.

Note that your old VM address (specifically, port) is probably changed since the
last lab, so make sure to update it as well.

Add the new VM to the `db_servers` group and add a new VM there. Leave the old
VM as member of the existing groups (`app_servers` and `web_servers`).

Once done your inventory file should look similar to this:

	vm-1  ansible_host=... ...
	vm-2  ansible_host=... ...

	[app_servers]
	vm-1

	[db_servers]
	vm-2

	[web_servers]
	vm-1

Update the `test_ansible.yaml` playbook from the [lab 1](../01-intro/lab.md) to
match all hosts from the inventory (`hosts: all`). Run it and make sure Ansible
can connect to both VMs.


## Task 3: playbook

Create a playbook `lab04_web_app.yaml` -- just copy the `lab03_web_app.yaml`
from the previous lab. Add a new **play** there:

	- name: Database server
	  ...
	  hosts: db_servers
	  roles:
	    - mysql

	- name: Web app
	  ...
	  hosts: app_servers
	  roles:
	    - ...

This playbook will set up both database server and app server, in one run. This
is how playbooks work -- each play sets up a group of servers of the same role,
so if the application requires multiple servers of different roles (also called
_tiers_) to run then the playbook will contain multiple _plays_. Here we have
one play for database servers and another play for application servers.

Hint: you can comment out 'Web app' play while writing the code for MySQL. Once
done, uncomment the 'Web app' role and continue with the full setup.


## Task 4: MySQL server

Create an Ansible role named `mysql` that will install and configure MySQL
server.

Install the MySQL server. Ubuntu package `mysql-server` is just good enough for
this task.

By default this MySQL server daemon will bind to local interface only. This
means that only local connections (from the same host) will work. You can check
it by running this command on the managed host (3306 is the default MySQL port):

	$ netstat -lnpt | grep 3306
    tcp  0  0  127.0.0.1:3306  0.0.0.0:*  LISTEN  9001/mysqld
	           ^-------^
	            This

This MySQL server behavior is configured in `/etc/mysql/mysql.conf.d/mysqld.cnf`
file (if you installed the package mentioned above) by this setting:

	[mysqld]
	bind-address = 127.0.0.1

This behavior needs to be changed -- web application will connect to the
database from the different host, so MySQL server should bind to public
interface to accept external connections. Easiest way to achive this is to
configure MySQL to bind to `0.0.0.0` which means 'any possible public interface
on this host'.

Most of the tutorials in the Internet will suggest you to change
`/etc/mysq/mysql.cnf` or `/etc/mysql/mysql.conf.d/mysqld.cnf` or any other
similar file. It would work, but there is a better way -- you can _override_ the
configuration instead of changing it.

Add the `/etc/mysql/mysql.conf.d/override.cnf` file to the managed host with the
following content:

	[mysqld]
	bind-address = 0.0.0.0

File name may differ but the directory and extension are important. It will
override the existing `[mysqld]:bind-address` setting from the previous
configuration file -- no need to even touch that. Awesome!

MySQL server needs to be restarted to apply the change. Use Ansible handlers for
that -- you can find more info about Ansible handlers in the
[previous lecture slides](../03-web-app).

If you have done everything correctly MySQL server should start and bind to
public interface:

	$ netstat -lnpt | grep 3306
	tcp  0  0  0.0.0.0:3306  0.0.0.0:*  LISTEN  9001/mysqld
	           ^-----^
	            This


## Task 5: variables

Web application will use this MySQL server as a storage backend -- so the
application needs its own database, and credentials to access it.

MySQL connection parameters: host, database name, user and password -- are
different for different deployments, and are shared among different roles and
tasks. These are clear candidates for _variables_. Let's define them first.

Create a file named `group_vars/all.yaml` in your Ansible repository and define
the needed variables:

	mysql_host: 192.168.42.xxx
	mysql_database: agama
	mysql_user: agama
	mysql_password: !vault |
	          $ANSIBLE_VAULT;1.1;AES256
	          61383032323739633432663361343366396634613831346231303935396264623764306537373030
	          3565623834333662626562303533636364366665663630370a613562626463623263633162653634
	          62613637353161336437636663393338356437663933623061303438306634616434373837383439
	          3361303630633039340a323433646332316634643735613936386131306662346563313535386663
	          3132

Internal IP address (`192.168.42.xxx`) of your database server can be found
[here](http://193.40.156.86/vms.html). Note that this _internal_ address is
different from the one that you have added to the inventory file:
 - You (and Ansible) connect to the _public_ IP (`193.40.156.86`) of the managed
   host
 - Other hosts in the same network connect to _internal_ IP

Password **must** be encrypted. You can get the encrypted value using Ansible
Vault you have set up in the task 1:

	ansible-vault encrypt_string <mysql-password-for-agama-here>


## Task 6: MySQL database

Add another task to `roles/mysql/tasks/main.yaml` to create a MySQL database.
Ansible module
[mysql_db](https://docs.ansible.com/ansible/latest/modules/mysql_db_module.html)
will be helpful here.

Use the variables you have just defined:

	- name: MySQL database
	  mysql_db:
	    name: "{{ mysql_database }}"

Note the quotes around `{{ ... }}`. These are needed, otherwise Ansible will
fail to parse the code.

On a first try Ansible may fail with an error saying

> The PyMySQL (Python 2.7 and Python 3.X) or MySQL-python (Python 2.X) module is required.

Ansible needs a Python library on the managed host to connect to MySQL and make
required changes. This library is called PyMySQL and can be installed as
`python3-pymysql` package from the Ubuntu APT repository.

Another error you may see is

> unable to find /root/.my.cnf. Exception message: (1698, "Access denied for user 'root'@'localhost'")

This means that Ansible could not authorize in the MySQL to make the required
changes.

Don't hurry to create this file though. MySQL server (again, if installed from
the package mentioned above) is configured to authorize local `root` user
already. This is done via local UNIX socket file, all you need to do is to
instruct Ansible how to use it. Try this instead:

	- name: MySQL database
	  mysql_db:
	    name: "{{ mysql_database }}"
        login_unix_socket: /var/run/mysqld/mysqld.sock

Exact socket file location can be found in the MySQL configuration file:
`/etc/mysql/mysql.conf.d/mysqld.cnf`.


## Task 7: MySQL user

Add another task to `mysql` role to create a MySQL user for the web application.
Use Ansible module
[mysql_user](https://docs.ansible.com/ansible/latest/modules/mysql_user_module.html)
for that.

`login_unix_socket` trick from the previous task will also be useful here.

While creating the MySQL user for your application, make sure that it has access
**only** to its own database (not the other databases). This can be achived with
`priv` attribute of the `mysql_user` module, and variable you defined
prveviously, for example:

	priv: "{{ mysql_database }}.*:ALL"

Also, by default Ansible will create a MySQL user that will only be able to
login from the same machine (called `agama@localhost`). We need a remote user
that can login from a different host, it would be called `agama@%` in MySQL
where `%` means 'any host'. This can be configured using `host` attribute of the
`mysql_user` module:

	host: "%"

Once MySQL database and user are created you can verify that this user can login
by running this command (manually) on the database server (assuming user name is
`agama`):

	mysql -u agama -p

It will ask you for the password you encrypted previously (`mysql_password`
variable) and once authorized you should get into MySQL console:

	mysql> 

If that works, your MySQL server is set up correctly.

Type `exit` to quit the MySQL console.


## Task 8: web application

Finally, it's time to configure our web application to use MySQL as the storage
backend. This is an easy task now :)

Use the play and roles from the previous lab, it does almost everything needed
already. You will only need to change the `AGAMA_DATABASE_URI` in the uWSGI app
settings from hard-coded SQLite path to MySQL connection string.
[AGAMA docs](https://github.com/hudolejev/agama/#running) have the example how
to do it.

Note: as you will use variables in this configuration file now make sure to use
proper Ansible module
([template](https://docs.ansible.com/ansible/latest/modules/template_module.html),
not `copy`) and move the file to the correct location in your role directory
(`templates/`, not `files/`).

Note: when using MySQL backend AGAMA (namely, Python on which it's written)
needs additional library for MySQL. It's already familiar `python3-pymysql` from
the task 5. It also needs to be installed on the app server now.

Once new uWSGI configuration is uploaded, uWSGI needs to be restarted to apply
it. Use Ansible handlers fro that.

If you have done everything correctly AGAMA should be server from your web
server public interface. You can ensure that it uses the MySQL backend by doing
this:

 - Add some items, or delete the default ones
 - SSH to the app server and delete the default AGAMA SQLite database from
   as configured in the previous lab (if any)
 - Refresh the AGAMA page
 - If your changed items are still there, and the SQLite database was not
   created on server -- you are probably using the MySQL backend.


## Expected result

Your repository contains these files and directories:

	ansible.cfg
	group_vars/all.yaml
	hosts
	lab04_web_app.yaml
	roles/mysql/tasks/main.yaml

Your repository also contains all the required files from the previous labs.

Your repository **does not contain** Ansible Vault master password.

Your deployment customizations: MySQL host, database name, user and password --
are variables and are stored in `group_vars/all.yaml` file.

Web application that uses MySQL backend, and the MySQL server itself are
installed and configured, each on a separate machine, by running this command
once:

	ansible-playbook lab04_web_app.yaml

Running the same command again does not make any changes to any of the managed
hosts.

AGAMA web application is available on
[your public URL](http://193.40.156.86/vms.html) -- obviously, only on this host
that you set up as a web server, not on the other one.
