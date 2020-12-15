# Lab 13

In this lab we will install Keepalived and HAProxy in front of our Docker containers. Playbook for this lab should be named `lab13_haproxy.yaml`.

## Task 1. Run Agama in Docker containers

Start at least 2 containers with Agama app on different VMs. Reuse role from lab12.

## Task 2. Install Keepalived

Keepalived will assign to pair of your VMs additional virtual IP. That IP will be assigned to one VM at a time, that VM will be MASTER. Second VM will become BACKUP and in case MASTER is dead will promote itself to MASTER and assign that additional virtual IP to its own interface.

Some configuration should be done before. After you install `keepalived` with APT module there won't be any configuration template provided, but in order to start, Keepalived needs non-empty `/etc/keepalived/keepalived.conf`.

Here is an example of `keepalived.conf` with some comments:

    vrrp_script check_haproxy {                 
        script "netstat -ntl | grep -q ':88 '" 
        weight 20                              
        interval 1               
    }
    static_routes {                             
        192.168.100.0/24 dev ens3
    }
    vrrp_instance elvis {             
        interface ens3
        virtual_router_id X
        priority Y
        advert_int 1                            
        virtual_ipaddress {                     
            192.168.100.XX                    
        }
        unicast_peer {                          
            192.168.42.YY
        }
        track_script {
            check_haproxy
        }
    }

Some comments to config example:

`vrrp_script` will add some weight to node priority if it was executed sucessfully. In given example, if node has priority 100 and in `netstat -ntl` output there is a line `:88 `, node priority will become 120. If another node has priority 110 - script execution result will affect which node is MASTER and which node assigns virtual IP to its own interface.

`static_routes` will add static route to VM routing table. Because we have virtual IP from another subnet - VM should have instructions via which interface this subnet is reachable.

`virtual_router_id` should be the same on different VMs.

`priority` should be different of different VMs. Use if-else-endif statements in Jinja2 template.

`virtual_ipaddress` should be the same on different VMs. if `your-name-1` VM has IP 192.168.42.35, virtual IP will be 192.168.100.35, 3rd octet changed from 42 to 100.

`unicast_peer` should contain IP of another VM. Multicast is default message format for VRRP, but it doesn't work in our cloud, you should specify IP of your another VM here that VRRP can start use unicast messages. Use Ansible facts.

If all done correctly, command `ip a` on VM with higher priority will show that there are 2 IPs on ens3 interface. No changes on VM with lower priority.

Hints:

After `service keepalive stop` on MASTER, BACKUP should become a MASTER and `ip a` will show that `192.168.100.X` assigned to another VM.

That's one of the 2 roles, where IPs are allowed in configuration.

## Task 3. Install HAProxy

Can be installed with APT module as easy as Keepalived.

Clear installation will provide you config template in `/etc/haproxy/haproxy.cfg`.

Copy blocks `global` and `default` to your template.

Add section `listen` to template. Example of section:

  listen my_ha_frontend
    bind :88
    server docker1 web-server1:8081 check
    server docker2 web-server2:7785 check

Port should be `88` because our NAT is configured to forward all requests to `192.168.100.X:88`.

88 is not the default HTTP port, but in our labs ports 80 and 8080 already have some services running, so we decided to use 88 to avoid any binding conflicts.

Usage of IPs is not allowed here.

If all done correctly, `Public HA URLs` of vm1 from [here](http://193.40.156.86/vms.html) should show you Agama app. Stopping HAProxy service on Keepalived MASTER should not affect Agama service reachability.

## Task 4. Add HAProxy monitoring

The easiest way is to run exporter in Docker container: https://github.com/prometheus/haproxy_exporter

You should expose HAProxy stats for exporter. Config example:

    listen my_ha_frontend
        stats enable
        stats uri /haproxy?stats

Hint:

`--haproxy.scrape-uri="http://haproxy.example.com/haproxy?stats;csv"` is a command that you should pass to container, might be missleading as it looks like parameter. Example:

    - name: HAProxy exporter
      docker_container:
        name: haproxy_exporter
        image: quay.io/prometheus/haproxy-exporter:v0.9.0
        ports:
          - 9101:9101
        command: --haproxy.scrape-uri="http://haproxy.example.com/haproxy?stats;csv"

## Task 5. Add Keepalived monitoring

There are a few Keepalived exporters available, we propose to use this one: https://github.com/cafebazaar/keepalived-exporter

Download a binary, create systemd unit. Same as for Nginx exporter few labs back.

## Task 6. Grafana dashboard

Add new metrics to your main Grafana dashboard. Should be panels for each node with those metrics:
  
  - haproxy_up (last value)
  - haproxy_server_up (last value for each backend)
  - keepalived_vrrp_state (last value)

Hint:

If you don't see these metrics in Grafana drop-down, make sure you have added HAProxy and Keepalived exporters to Prometheus configuration.

## Expected result

Your repository contains these files:

    lab13_haproxy.yaml
    roles/haproxy/tasks/main.yaml
    roles/keepalived/tasks/main.yaml
    grafana_dashboard.json
    grafana_dashboard.(jpg|png)


Your Agama application is accessible on its public HA URL from
[this list](http://193.40.156.86/vms.html).

Your Grafana and Prometheus are accessible on its public URLs, just public, not HA.
