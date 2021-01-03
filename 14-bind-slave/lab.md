# Lab 14

In this lab we will install DNS slave to provide DNS high availability. Playbook for this lab should be named `lab14_bind_slave.yaml`.

## Task 1. Change location of DNS zone file

In lab 5 you most probably created zone files in `/etc/bind/` directory. This directory should be managed by root, Bind9 service doesn't have permissions to write there by default.

For files that should be changed by service there is a special directory: `/var/lib/`.

Make sure your zone database files are located inside `/var/lib/bind/`. Configuration files are still in `/etc/bind/`.

Why do we need to do that? Check task 5, Bind9 will update database for us, we're going to stop managing database files via Ansible directly.

## Task 2. Generate DNS key

You need to generate DNS key that will be used to authenticate slave on master.

Keys are generated with command on DNS server or any other server where `tsig-keygen` exists. Example:

  tsig-keygen example.key

Generated output should be added to Bind9 config.

Key name should be `transfer.key`.

Obviously, key secret is secret data, use Ansible Vault to store it in your repo.

For more detailed explanation check section `4.5 TSIG` in the docs: https://downloads.isc.org/isc/bind9/cur/9.11/doc/arm/Bv9ARM.pdf.

## Task 3. Create DNS slave

Install Bind9 on second VM.

Configuration file with global options will be the same for master and slave, file with zone configuration will be different. Check docs how to configure zone as slave: https://downloads.isc.org/isc/bind9/cur/9.11/doc/arm/Bv9ARM.pdf. Section `3.1 Sample configurations`.

On DNS master allow zone transfer only for those who have `transfer.key` set. Section `4.5 TSIG` in the docs.

On DNS slave configure to use `transfer.key` when sending requests to master.  Section `4.5 TSIG` in the docs.

After this step slave should be able to resolve all your internal FQDNs. Command for checking:

  dig name.domain @slave_ip

Hint #1: Bind9 master and slave should be configured in one role. Role should be applied to group `dns_servers`, which includes groups `dns_slaves` and `dns_masters`. When you want to apply some task only to slave, use conditions:

  when: "inventory_hostname in groups['dns_slaves']"

Hint #2: If you don't like hint #1, you can create host variable `dns_role` and execute tasks based on this value. For example:

  when: "dns_role == 'slave'"

## Task 4. Update /etc/resolv.conf

/etc/resolv.conf now should contain IPs of both DNS servers.

Good idea to include `search` option there as well. In that case you don't need to specify your full domain every time. Template example:

    search {{ your_domain }}     // will be added to short names
    nameserver {{ dns_master }}  // can be a loop over all masters
    nameserver {{ dns_slave }}   // can be a loop over all slaves

## Task 5. Rewrite Ansible bind role

Change the way how you create zone file and records.

Initial zone file since now should contain only minimum required set: SOA record, 2xNS records and A record for each NS record. Example can be found in [demo](./demo/) folder.

All other records Bind9 will add there by itself. Problem that might happen with this approach: next Ansible run will overwrite the database file and delete all the records created by Bind9. Solution: database file should be uploaded by Ansible only if it's missing. If file already there, Ansible should not touch it. Check docs for template module how to achieve this: https://docs.ansible.com/ansible/2.9/modules/template_module.html

In Bind9 configuration allow zone updates only for those who have `nsupdate.key` set. Generate it same way as `transfer.key` from task 2. Example configuration:

  zone example.com {
    type master;
    file ...;
    allow-update { key nsupdate.key; };
  }
  
All other records should be added with Ansible nsupdate module. Docs: https://docs.ansible.com/ansible/2.9/modules/nsupdate_module.html

Use update.key and IP of your DNS master server, take IP from Ansible facts.

## Task 6. Grafana dashboard

Make sure that DNS slave metrics are gathered by Prometheus.

Add DNS slave graphs to your Grafana dashboard (same as for DNS master).

## Expected result

Your repository contains these files:

    lab14_bind_slave.yaml
    roles/bind/tasks/main.yaml
    grafana_dashboard.json
    grafana_dashboard.(jpg|png)
