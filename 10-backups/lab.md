# Lab 10

In this lab we will set up automatic backups for your infrastructure and improve
the documentation from [the previous lab](../09-backups/lab.md) accordingly.

There are multiple ways how to do backups; on this lab we'll use probably the
simplest approach:
 1. Gather data to be backed up from all services to a single location on this
    machine
 2. Upload all the backups to the backup server
 3. Document the service restore process -- no need to automate it here

Note that we keep the data gathering and backup upload steps separate. This is
done to minimize the impact of a backup process to the running service.


## Task 1

Ensure user `backup` can access the data to backup.

Solution for this task pretty much depends on the backup strategy you've chosen
and documented in the previous lab, namely what services you decided to backup
and what to skip. And of course feel free to update the documentation if you
realize that chosen backup strategy doesn't quite work with your service.

Here are some example commands; these need to be run as user `backup` and should
work without errors:

	# Check if user can copy the needed file
	cp /path/to/needed/file /dev/null

	# Check if user can dump needed MySQL database
	mysqldump <database-name> >/dev/null

Below are some common problems you may face, and solutions.

### File access permissions

Getting error like this one?

	cp: cannot open '/path/to/some/file' for reading: Permission denied

You will need to allow Linux user `backup` to access this file. Modifying the
file permissions is generally a bad idea because this will affect the service
itself that usually expects some certain file permissions. Instead, user
`backup` could be added to the proper Linux group so that it can read the file.

Example for Grafana database:

	$ ls -la /var/lib/grafana/grafana.db
	-rw-r-----  1  grafana grafana  512000  Nov 1 18:12  /var/lib/grafana/grafana.db

Note that the file can only be read by the `grafana` user and members of the
`grafana` group.

	$ id
	uid=34(backup) gid=34(backup) groups=34(backup)

Note that user `backup` is not member of the `grafana` group. Check the
[Ansible module user documentation](https://docs.ansible.com/ansible/2.9/modules/user_module.html)
for details how to add the user to certain groups. Once done, log out the user
`backup` and log in again. It should now be able to read the file:

	$ id
	uid=34(backup) gid=34(backup) groups=34(backup),117(grafana)

	$ cp /var/lib/grafana/grafana.db /dev/null
	(no more errors in the output)

Make sure that you get the task order right: add the user to the group **after**
this group was created. It is usually a good idea to set up the backup user as
one of the last steps, after the 'main' services are set up already.

### MySQL dump

Getting error like this one?

	mysqldump: Got error: 1045: Access denied for user 'ubuntu'@'localhost' (using password: NO)

You will need to authenticate the user `backup` in the MySQL server; for that
you'll need another MySQL user. Your `mysql` role should have the example on
how to add users. Use the same permissions for both MySQL users `agama` and
`backup` if unsure.

Run this command on the MySQL host to check if the authentication works:

	mysqldump -u backup -p --no-tablespaces agama

Next, store this password and some convenience flags for MySQL dump in the
configuration file so that you don't have to type them every time. Example of
such file:

	[client]
	user = backup
	password = <password-of-mysql-user-named-backup>

	[mysqldump]
	no_tablespaces

This file should be placed to `/home/backup/.my.cnf`, be readable only by user
`backup` (noone else) and of course be managed with Ansible -- not manually.

MySQL client configuration reference:
https://dev.mysql.com/doc/refman/5.7/en/option-files.html


## Task 2

Prepare backup scripts for your services you decide to back up, one script per
service. Each of this these scripts should extract the data to be backed up
and/or copy the needed files to backup location **on this server**. 

Backup uploading to the backup server will be handled separately. Don't bother
about it yet.

These scripts will be different per service and heavily depend on the service
specifics. Below are some general recommendations:

### 1: One script should solve only one problem -- but do it well

One script -- one service -- one task.

Don't attempt to write a script that backs up all your services on this machine.

Don't put too much functionality into the script. It should just extract the
data, if needed, and copy the data to the backup location -- and that's it.

### 2: Use separate backup and restore locations

Create two additional directories in the home directory of the `backup` user:

	/home/backup/backup
	/home/backup/restore

Use the former to gather the data for a backup, and later upload it to the
backup server.

Use the latter to download the data from the backup server for restoration
purposes.

### 3: Make scripts non-interactive

Backups scripts should run fully unattended, without asking any questions or
blocking on passwords prompts -- otherwise it won't be possible to run them
automatically on schedule.

### Example: File backup script

Copies entire service working directory to the backup location:

	cp -a /var/lib/prometheus /home/backup/backup/

### Example: MySQL dump script

Dumps one MySQL database:

	mysqldump agama > /home/backup/backup/agama.sql

### Self-check

Run every script manually as the `backup` user on the managed host and make sure
it copies all the required data to the backup location.

Try restoring every service from the created backup.


## Task 3

Update Ansible role `backup` to schedule backup scripts you have prepared in the
previous task to run periodically.

For that add a file `/etc/cron.d/backup` on every managed host. This file should
have Cron schedules for backup jobs that should be run by user `backup`. Example
Cron jobs:

	11 1 * * *  backup  cp -a /var/lib/influxdb /home/backup/backup/
	12 1 * * *  backup  cp -a /var/lib/prometheus /home/backup/backup/


Refer to the backup SLA you've documented in the lab 9 for the exact backup
schedule for every service. Feel free to update the backup SLA if needed.

At this point it makes sense to delete all the files from the local backup
directory so you can check if scheduled jobs re-created them automatically.


## Task 4

Update Ansible role `backup` to install
[Duplicity](http://duplicity.nongnu.org/); the package is available in the
Ubuntu APT repository.

Schedule Duplicity to upload the local backups to the backup server as described
in your backup SLA. If needed, update the SLA accordingly.

It is okay to skip the backup encryption **for this lab**. But keep in mind --
**backups should be encrypted in the production systems!**

Uploads can be scheduled in Cron. For that modify the `/etc/cron.d/backup` file
from the previous task and add the upload job. Example for user `elvis`,
Duplicity running over Rsync, weekly full and daily incremental backups -- your
schedules will probably be different:

	42 1 * * * 0  backup  duplicity --no-encryption full /home/backup/backup/ rsync://elvis@backup.x.y//home/elvis/
	42 1 * * * 1-6  backup  duplicity --no-encryption incremental /home/backup/backup/ rsync://elvis@backup.x.y//home/elvis/

Make sure that whichever schedule you choose the first created backup is full --
not incremental. You may run the first backup command manually to be sure.

Make sure that Duplicity is scheduled to run after all the local backup scripts
have finished!

Once Duplicity creates its first backup make sure you can resotre the service
from it. For that, download the backup to the managed host and try restoring the
service. Example commands run on the managed host as user `root`:

	sudo -u backup duplicity --no-encryption restore rsync://elvis@backup.x.y//home/elvis/ /home/backup/restore/
	service prometheus stop
	rm -rf /var/lib/prometheus/*
	cp -a /home/backup/restore/prometheus/* /var/lib/prometheus/
	service prometheus start

Make sure that service is up and running and contains all the data from the
backup.

Should you accidentally damage the service beyond all repairs -- uninstall the
package manually and re-run the Ansible to restore the service:

	# On the managed host
	apt purge prometheus
	
	# On your Ansible host
	ansible-playbook <playbook-name>.yaml


# Task 5

Add a free-text file named `backup_restore.md` that documents the restoration
process for **every** service you deployed so far.

There should be Ansible installation instructions for every service, for
example:

	DNS
	---

	Install and configure with Ansible:

		ansible-playbook <playbook-name>.yaml

Some services have backups set up, so backup restore steps should also be
listed, for example:

	AGAMA
	-----

	Install and configure with Ansible:

		ansible-playbook <playbook-name>.yaml

	Restore the data from the backup:

		su - backup duplicity restore <args>
		<another-command>
		<yet another-command>

Target audience for this document is someone who has root access to the managed
host but who is **not** familiar with your service setup. In real life it would
be your collegue who will be restoring the service from the backup if you are
not available to do it.

For this lab you can treat the teachers as target audience -- we should be able
to restore your service having only your GitHub repository (that also contains
backup restore document) and the backups, without asking you a single question.

Make sure to verify these instructions, i. e. each service should be restorable
by running the commands you documented.


## Expected result

Your repository contains these files and directories:

	backup_sla.md  # (updated if needed)
	backup_restore.md
	lab10_backups.yaml
	roles/backup/tasks/main.yaml

Your repository also contains all the required files from the previous labs.

Every service you've deployed so far can be restored by following the
instructions in the `backup_restore.md`.
