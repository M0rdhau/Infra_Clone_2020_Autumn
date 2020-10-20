# Lab 8

In this lab we will setup centralized logging.

## Task 1: Install InfluxDB

Install InfluxDB from Ubuntu package repository.

In addition install also "influxdb-client" package. That will help you to troubleshoot InfluxDB issues, if any.

## Task 2: Create pinger service on one of VMs

Find bash script "pinger.sh" in 08-demo.

Place this script to /usr/local/bin/pinger.

Create a service "pinger" that runs from user "pinger". Check previous lab how to create Systemd service units.

Pinger script requires config file:

    /etc/pinger/pinger.conf

Example can be found in 08-demo as well.

## Task 3: Add latency monitoring to main Grafana dashboard

Add new Grafana datasource: influxdb/latency.

Use this datasource for new panel in dashboard from previous lab.

No ip addresses allowed. Use influxdb.\<your_domain\>.

## Task 4: Setup Telegraf

Install Telegraf v1.15.2 on the same VM where InfluxDB is located. Docs: https://portal.influxdata.com/downloads/.

Installation steps can be done in "influxdb" role.

Configure Telegraph for only Syslog input and only influxdb output. Hint:

    telegraf --help

UDP transport is more preferable.

## Task 5: Setup rsyslog

Configure rsyslog on all VMs to send all logs to Telegraf. Docs: https://github.com/influxdata/telegraf/blob/release-1.15/plugins/inputs/syslog/README.md

UDP transport is more preferable.

## Task 6: Create logging dashboard in Grafana

Add one more datasource: influxdb/telegraf

Import Grafana dashboard for Syslog: https://grafana.com/grafana/dashboards/12433

In logs panel query change "host" to "hostname". Seems to be a typo there.

## Expected result

Your repository contains these files and directories:

    ansible.cfg
    group_vars/all.yaml
    hosts
    lab08_logging.yaml
    roles/pinger/tasks/main.yaml
    roles/rsyslog/tasks/main.yaml
    roles/influxdb/tasks/main.yaml
    grafana_dashboard.json
    grafana_dashboard.(jpg|png)

Your repository also contains all the required files from the previous labs.

Your repository **does not contain** Ansible Vault master password.

Everything is installed and configured with this command:

	ansible-playbook lab08_logging.yaml

Running the same command again does not make any changes to any of the managed
hosts.

After playbook execution you should be able to see all logs in one Grafana dashboard.
