# Lab 11

In this lab we will set up MySQL in highly available mode, namely configure a
replication with two MySQL servers.

A lot of things can go wrong with MySQL in this lab. If you feel that you are
stuck and MySQL is broken beyond repairs -- delete it and start from scratch:

	# On a managed host
	apt-get purge mysql-*
	rm -rf /etc/mysql

	# On the Ansible host
	ansible-playbook <playbook-name>

Answer 'yes' is asked about deleting all the databases. We have backups set up,
right? :)


## Task 1

Create a playbook named `lab11_mysql_ha.yaml`. Add plays that
 - Set up Bind DNS server 
 - Set up MySQL server and MySQL exporter for Prometheus
 - Set up Agama
 - Set up backup user and backups
 - Set up Prometheus
 - Set up Grafana

Add other plays and roles as you feel needed.

Most (if not all) of these plays should be available in previous playbooks
already, so it shouldn't take too long to achieve.


## Task 2

Add another dahsboard named 'MySQL' to Grafana (if you haven't done it yet).

Add a few more graphs for every MySQL server -- we have one so far but will have
more soon.

Add a text field showing MySQL server id (Prometheus metric
`mysql_global_variables_server_id`).

Add graphs should show the historical data for:
 - MySQL server being up on this host (Prometheus metric `mysql_up`)
 - MySQL server read only status (Prometheus metric
   `mysql_global_variables_read_only`)

Save your updated Grafana dashboard as `grafana_dashboard.json` (same as in
[lab 7](../07-grafana/lab)).


## Task 3

Modify your `mysql` role from previous labs and add another MySQL user named
`replication`.

This user should use a password to log in and should be able to access this
MySQL server from any host in our network -- similar configuration as `agama`
user.

This user, however, should have different permissions: `REPLICATION SLAVE` on
every database and table (`*.*`). Check Ansible module `mysql_user`
[documentation](https://docs.ansible.com/ansible/2.9/modules/mysql_user_module.html)
for details on how to achieve this.

Run this command on the managed host to verify that the user can log in:

	mysql -u replication p


## Task 4

Update your Ansible inventory file and add another host to the `db_servers`
group. This group should contain two hosts now.

Update the MySQL configuration file (`override.cnf` discussed in detail in
[lab 4](../04-troubleshooting/lab.md)) and add the following attribute to the
`mysqld` section:

	server-id = {{ ... }}


`server-id` can be some number (2 or larger) set explicitly in `group_vars`, or
the last octet of the machine IP address computed as
`{{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'].split('.')[-1] }}`
-- you decide which you like more, but make sure that different servers get
different ids!

Run the playbook again. It should install the MySQL server on the second host,
and restart the MySQL server on the first host as server configuration was
changed.

Your Grafana dahsboard should show that the new MySQL server is up.

Also Grafana should show both server ids as you've set in the MySQL
configuration file.


## Task 5

Configure MySQL replication. Add these to the same MySQL configuration file, to
`mysqld` section:

	log-bin = /var/log/mysql/mysql-bin.log
	relay-log = /var/log/mysql/mysql-relay.log
	replicate-do-db = {{ ... }}

`replicate-do-db` only limits the replication to one database -- our application
database. This is needed to skip replication of MySQL own database named `mysql`
that contains user and permission info -- these are managed by Ansible in our
case, and may break the replication.

Settings could be the same for both MySQL master and slave for now. We will
update them later.

Run the playbook to apply the changes.

Now it's time to test if replication actually works. On the managed host open
the MySQL shell by running `mysql` as user root, and then:

	STOP SLAVE;
	CHANGE MASTER TO MASTER_HOST='<your-mysql-master-host>',
	                 MASTER_USER='replication',
	                 MASTER_PASSWORD='<your-replication-password>'
	RESET SLAVE;
	START SLAVE;
	SHOW SLAVE STATUS\G

The output of the last command should show contain no errors. Fileds to check
are `Last_IO_Error` and `Last_SQL_Error`.

But Even if you have done everything correctly the replication may work, or may
not work at this point. It depends on various factors. If the output of the last
command shows something similar to

	Last_SQL_Error: Error executing row event: 'Table 'agama.item' doesn't exist'

It means that the MySQL slave cannot pick up some entries from the replication
log on master (creating the database), and fails to proceed with the next
entries (adding rows). This may happen if you have created and populated the
database _before_ setting up the replication.

This is easy to fix:
 1. Create a new backup on the MySQL master host and upload it to the backups
    server (see [lab 10](../10-backups/lab.md)).
 2. Download the backup to the MySQL slave server
 3. Restore the database from the backup on the slave server:
    `mysql agama < /path/to/backup.sql` -- you may need to run it as root if
    the MySQL is running in the read only mode already
 4. Run the SQL commands mentioned above again

If you are getting a clearly different error, please contact the teachers.

Finally, if there are no more replication errors left, try adding, deleting or
modifying some items via Agama web UI. Make sure that changes reach both MySQL
master and replica databases; this can be checked by running this command on
every MySQL server:

	mysql -e "SELECT * FROM agama.item"

Congratulations! You now know how to set up the simple MySQL replication.


## Expected result

Your repository contains these files and directories:

	grafana_dashboard.json
	lab11_mysql_ha.yaml
	roles/mysql/tasks/main.yaml

Every service you've deployed so far can be restored by following the
instructions in the `backup_restore.md`.
