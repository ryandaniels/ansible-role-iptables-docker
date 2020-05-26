# Ansible Role: iptables for Docker (and Docker Swarm)

Add firewall rules to server via iptables, for Docker, and Docker Swarm. This will actually protect your Docker containers!  
This Ansible Role exists because firewalld and Docker (and Docker Swarm) do not get along.  

Problem being solved: When starting a container in Docker with a "published" port, you have no control and the port is exposed through your server's firewall. Even if you were using iptables, or another firewall on your server. Docker opens that "published" port to everyone, and bypasses your firewall.  

Use case for this solution: Allow trusted IPs to connect to Docker containers (and Docker Swarm containers), along with other open OS ports. With an option to expose specified ports publicly (Docker/Docker Swarm and OS). The trusted IPs might not be in the same network IP range, or even the same network subnet.  

This was suppose to be simple. Secure Docker with a firewall. But unfortuanately it is not. I've tried to keep this as simple as possible.  

There could be unknown problems with this.. use at your own risk!  

See also: <https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/>  
And about Docker's use of the INPUT chain: <https://ryandaniels.ca/blog/docker-iptables-input-chain/>

Currently tested and working on:

* CentOS/RHEL 7
* Ubuntu 18.04
* Ubuntu 20.04

## Features

* Works with Docker, and Docker Swarm (aka Docker SwarmKit).
* Secure by default. Once configured, only Docker IPs can access all containers, and all other OS processes that have open ports on the server.
* Simple as possible. The less iptables rules, the faster performance will be (in theory at least).
* Automatic. No manually adding ports to the firewall config (if you use a trusted set of IPs)
* Add "trusted" IPs that are allowed to communicate with all Docker containers, and all other OS processes that have open ports on the server.
* Open specified Docker container ports, or the server's OS ports to the public (everyone) through the firewall, like SSH.
* Interfaces can also be specified. By default all interfaces are filtered. You could filter specific network interface(s) and allow all other interfaces (only specify an untrusted interface).
* Everything done in "offline" mode. So there should be no issues with Docker when iptables rules are activated.
* You don't need to be an expert with iptables to use this.
* Works with Docker Swarm's undocumented use of iptables and encrypted overlay networks. (iptables rules are appeneded to the INPUT chain).

This solution is using `iptables` as the firewall, and `ipset` to allow iptables to have a list of IPs that are allowed. `ipset` also allows you to use a non-continueous range of IPs.  

iptables chains used, and how:  
**INPUT**, not flushed. Rule inserted at top to jump to custom chain for OS related rules.  
**DOCKER-USER**, flushed. All Docker (and Docker Swarm) related rules are here to block containers from being exposed to everyone by default. By default only the Docker server IPs are allowed. Other IPs and container ports can be added by user.  
**FILTERS**, flushed. Custom chain for server's processes (that aren't Docker). By default only the Docker server IPs are allowed. Other IPs and container ports can be added by user.  

iptables manual: <http://ipset.netfilter.org/iptables.man.html>  

## Warnings

Don't lock yourself out of your server. This is modifying your firewall. Always have another way to get in, like a "console".  

**Note about IPs**: This is for IPv4 only. IPv6 has not been tested. It is safer if you disable IPv6 on your servers.  

**Other security consideration**:  
If using non-Swarm (normal Docker), consider also binding a port to an internal IP for better security.
If using Swarm, consider using specific IPs for Docker Swarm communication.  
Eg. `docker swarm init --advertise-addr 192.168.100.100 --listen-addr=192.168.100.100 --data-path-addr=192.168.100.100`

**Important Note**: Docker and firewalld do not get along. This Ansible Role has a check enabled to fail this role if the firewalld service is running or enabled.  
For more information about firewalld and Docker:  
<https://success.docker.com/article/why-am-i-having-network-problems-after-firewalld-is-restarted>  
<https://www.tripwire.com/state-of-security/devops/psa-beware-exposing-ports-docker/>  
<https://docs.docker.com/network/iptables/>  

**SELinux Bug**:  
Currently there's a bug with SELinux that prevents saving the iptables rules to the iptables.save file.  
Impact: Saving the iptables rules a 2nd time will silently fail.  
Workaround has been added so SELinux allows chmod to interact with the iptables.save file.  
Alternatively you could disable SELinux, but that's not recommended.  
Bug report: <https://bugs.centos.org/view.php?id=12648>  
See below for more details about manually performing the workaround.  

**WARNING**:  
Make sure you test in non-production first, I cannot make any guarantees or held responsible.  
Be careful, this will remove and add iptables rules on the OS. Use with caution.  
Existing iptables rules could be removed! Confirm what you have setup before running this.  

There could be unknown problems with this.. use at your own risk!  

## Docker versions tested

Docker Engine - Community Edition version:

* 19.03.8
* 19.03.9

Tested in normal Docker mode, and with a 3 node Docker Swarm cluster.  

## Distros tested

* CentOS: 7.7, 7.8
* Ubuntu 18.04
* Ubuntu 20.04

## Dependencies

* iptables & iptables-services

Tested with v1.4.21 (Latest available in CentOS 7)  

* ipset & ipset-service

Tested with v7.1 (Latest available in CentOS 7)  

## Default Settings

* Enable debug

```yaml
debug_enabled_default: false
```

* Proxy (Needed when installing required packages if behind a proxy)

```yaml
proxy_env: []
```

* Role disabled by default. Change to true in group_vars or playbook etc

```yaml
iptables_docker_managed: false
```

* Check if (Docker) service is running or enabled, and fail the role

```yaml
iptables_docker_check_problem_service_managed: true
```

* Services to check, and fail the role

```yaml
iptables_docker_check_problem_service:
  - docker.service
```

* Show configuration from variables

```yaml
iptables_docker_show_config: true
```

* Start iptables service

```yaml
iptables_docker_start: true
```

* Install iptables package

```yaml
iptables_docker_managed_pkg: true
iptables_docker_packages:
  - iptables
  - iptables-services
  - ipset
  - ipset-service
  - policycoreutils-python #required for semodule
```

* Force copy of ipset file to trigger ipset reload

```yaml
iptables_docker_copy_ipset_force: false
```

* Force copy of iptables file to trigger iptables reload

```yaml
iptables_docker_copy_iptables_force: false
```

* iptables saved configuration location

```yaml
iptables_docker_iptables_config_save: /etc/sysconfig/iptables
```

* ipset saved configuration location

```yaml
iptables_docker_ipset_config_dir: /etc/sysconfig/ipset.d
```

* ipset maximum elements (IPs in the allow list)

    If changed after first creation, must be deleted and re-created manually. 64k IPs should be enough.

```yaml
iptables_docker_ipset_maxelem: 65536
```

## User Settings

* Override Docker server IPs (Optional)

    Optionally specify the Docker server IPs. If not set, IPs will be determined from docker_hosts group in Ansible inventory.

```yaml
# iptables_docker_server_ip_allow_set:
#   - 192.168.100.100
#   - 192.168.100.101
#   - 192.168.100.102
```

* IPs allowed to use all Docker container's exposed ports and all server's processes' exposed ports.

```yaml
# iptables_docker_ip_allow_set: []
iptables_docker_ip_allow_set:
  - 192.168.100.1
  - 192.168.101.0/24
  - 192.168.102.0/24
```

* Network adapter to restrict for OS rules

    Only listed adapters will be blocked. Others will be allowed through. Defaults to block all (with '+').  
    If you want to restrict only specific network interface use exact name.  
    If you want to restrict all interfaces of the same type, use "interface+" to match every interface, since + is the wildcard for iptables.  
    Eg. To restrict the ethX interfaces, use "eth+". "eth+" is a wildcard for anything starting with eth.  
    DO NOT use "*". This is not a wildcard and matches nothing!  
    The less here the better. Safer to block all ('+') but if cannot, add network adapters with high traffic first.  
    local (lo) is not needed here.  

```yaml
iptables_docker_external_network_adapter:
  - "+" #Wildcard for everything
  # - "eth+"
  # - "enp0s+"
  # - "wlp1s+"
```

* OS tcp ports open to public

    Ports to allow everyone to connect (will be publicly accessible). Ports here will allow all tcp traffic to these ports from iptables level.  
    Only for ports on OS, not for Docker containers.

```yaml
iptables_docker_global_ports_allow_tcp:
  - 22                   # SSH
```

* OS udp ports open to public

    Ports to allow everyone to connect (will be publicly accessible). Ports here will allow all udp traffic to these ports from iptables level.  
    Only for ports on OS, not for Docker containers.

```yaml
iptables_docker_global_ports_allow_udp: []
```

* Network adapter to restrict for Docker rules

    Defaults to use the same setup as the network adapter for the OS.

```yaml
iptables_docker_swarm_network_adapter: "{{ iptables_docker_external_network_adapter }}"
# iptables_docker_swarm_network_adapter:
#   - "+" #Wildcard for everything
#   # - "eth+"
```

* Docker tcp ports open to public

    Add Docker container tcp ports you want open to everyone. For Docker and Docker Swarm.
    Docker Swarm ports aren't needed here.

```yaml
iptables_docker_swarm_ports_allow_tcp: []
# iptables_docker_swarm_ports_allow_tcp:
#   - 9000
```

* Docker udp ports open to public

    Add Docker container udp ports you want open to everyone. For Docker and Docker Swarm.
    Docker Swarm ports aren't needed here.

```yaml
iptables_docker_swarm_ports_allow_udp: []
```

* Docker bridge network name (docker0), and IP range (for DOCKER-USER iptables source allow)

```yaml
iptables_docker_bridge_name: docker0
iptables_docker_bridge_ips: 172.17.0.0/16
```

* Docker Swarm bridge network IP range (docker_gwbridge) (for DOCKER-USER iptables source allow)

```yaml
iptables_docker_swarm_bridge_name: docker_gwbridge
iptables_docker_swarm_bridge_ips: 172.18.0.0/16
```

## Example config file (inventories/dev-env/group_vars/all.yml)

From the example below:  
IPs will be added to the trusted list:

* `192.168.100.1`
* `192.168.101.0/24`

All network interfaces will be restricted since using wildcard '+' for iptables_docker_external_network_adapter.  

Port 22 will be open publicly.

```yaml
---
iptables_docker_ip_allow_set:
  - 192.168.100.1
  - 192.168.101.0/24

iptables_docker_external_network_adapter:
  - "+" #Wildcard for everything

iptables_docker_global_ports_allow_tcp:
  - 22                   # SSH
```

## Example inventory file

```ini
[docker_hosts]
centoslead1 ansible_host=192.168.100.100
centoswork1 ansible_host=192.168.100.101
centoswork2 ansible_host=192.168.100.102
```

## Example Playbook iptables_docker.yml

```yaml
---
- hosts: '{{ inventory }}'
  become: yes
  vars:
    # Use this role
    iptables_docker_managed: true
  roles:
  - ryandaniels.iptables_docker
```

## Usage

Before running make sure you check if you are already using iptables! Nothing should be overwritten/removed, unless you are using the same iptables chains as this.  

By default no tasks will run unless you set `iptables_docker_managed=true`. This is by design to prevent accidents by people who don't RTFM.

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true" -i hosts-dev
```

Skip installing packages (if known already there - speeds up task)

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true" -i hosts --skip-tags=iptables_docker_pkg_install
```

Show more verbose output (debug info)

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true debug_enabled_default=true" -i hosts-dev
```

Do not start iptables service or add rules for iptables

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true iptables_docker_start=false" -i hosts-dev
```

Force ipset and iptables to update

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true iptables_docker_copy_ipset_force=true iptables_docker_copy_iptables_force=true" -i hosts-dev
```

Only show configuration (from variables)

```bash
ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7 iptables_docker_managed=true iptables_docker_show_config=true" -i hosts --tags "iptables_docker_show_config"
```

## Note about ipset size limit

Important: Make note of the size of "Number of entries". If that number is close to the maxelem size (65536), then you need to delete the ipset "ip_allow" and re-create it with a larger max size.  
64K ought to be enough for anyone.  

File is in: `templates/ip_allow.set.j2`

```text
create -exist ip_allow hash:ip family inet hashsize 1024 maxelem 65536
```

Check size of ipset list:

```bash
ipset list |grep "Number of entries"
```

Important output:

```bash
Number of entries: 3
```

## SELinux manual workaround for iptables and chmod

Bug details: <https://bugs.centos.org/view.php?id=12648>

The problem is when saving iptables a 2nd time, SELinux blocks it since chmodhas a problem with the iptables.save file.
Use below as workaround to allow chmod to modify iptables.save file if not using the Ansible role.  
To reproduce, restart iptables service after setting iptables config to save after restart/stop,

```bash
ausearch -m AVC,USER_AVC,SELINUX_ERR,USER_SELINUX_ERR -i|tail -55
grep "iptables.save" /var/log/audit/audit.log|tail | audit2allow -M iptables_save_chmod
#or ausearch -c 'chmod' --raw | audit2allow -M iptables_save_chmod
semodule -i iptables_save_chmod.pp
```

## iptables Command Reference

More commands can be found in iptables documentation: <http://ipset.netfilter.org/iptables.man.html>

List iptables that are active:

```bash
iptables -nvL --line-numbers
```

Misc CentOS/RHEL useful commands:

```bash
cat /etc/sysconfig/ipset.d/ip_allow.set
systemctl restart ipset
ipset list | head

iptables -F DOCKER-USER
iptables -F FILTERS
iptables-restore -n < ansible_iptables_docker-iptables

grep -v "^#" ansible_iptables_docker-iptables
iptables -S INPUT
iptables -S DOCKER-USER
iptables -S FILTERS
```

Misc Ubuntu useful commands:

```bash
vi /etc/iptables/ipsets
#Manually add 'flush' before add, if removing IPs manually.

/usr/sbin/netfilter-persistent reload

cat /etc/iptables/ipsets

cat /etc/iptables/rules.v4
```

## Manual Commands (CentOS/RHEL)

Check what iptables rules you already have. Make note in case they are lost!

```bash
iptables -nvL --line-numbers
```

Install required packages:

```bash
yum install iptables iptables-services ipset ipset-service
```

If using SELinux, also install:

```bash
yum install policycoreutils-python
```

Configure ipset with your server IPs and other trusted IPs:

```bash
mkdir -p /etc/sysconfig/ipset.d
cat > /etc/sysconfig/ipset.d/ip_allow.set  << 'EOF'
create -exist ip_allow hash:ip family inet hashsize 1024 maxelem 65536
add ip_allow 192.168.1.123
add ip_allow 192.168.101.0/24
add ip_allow 192.168.102.0/24
EOF
```

Start, and Enable the ipset service:

```bash
systemctl status ipset
systemctl start ipset
systemctl enable ipset
```

See what ipset has in it's loaded configuration:

```bash
ipset list | head
```

iptables rules being added (by default), and command to append them into iptables rules:

```iptables
cat > ansible_iptables_docker-iptables << 'EOF'
*filter
:DOCKER-USER - [0:0]
:FILTERS - [0:0]
#Can't flush INPUT. wipes out docker swarm encrypted overlay rules
#-F INPUT
#Use ansible or run manually once instead to add -I INPUT -j FILTERS
#-I INPUT -j FILTERS
-A DOCKER-USER -m state --state RELATED,ESTABLISHED -j RETURN
-A DOCKER-USER -i docker_gwbridge -j RETURN
-A DOCKER-USER -s 172.18.0.0/16 -j RETURN
-A DOCKER-USER -i docker0 -j RETURN
-A DOCKER-USER -s 172.17.0.0/16 -j RETURN
#Below Docker ports open to everyone if uncommented
#-A DOCKER-USER -p tcp -m tcp -m multiport --dports 8000,8001 -j RETURN
#-A DOCKER-USER -p udp -m udp -m multiport --dports 9000,9001 -j RETURN
-A DOCKER-USER -m set ! --match-set ip_allow src -j DROP
-A DOCKER-USER -j RETURN
-F FILTERS
#Because Docker Swarm encrypted overlay network just appends rules to INPUT. Has to be at top unfortunately
-A FILTERS -p udp -m policy --dir in --pol ipsec -m udp --dport 4789 -m set --match-set ip_allow src -j RETURN
-A FILTERS -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FILTERS -p icmp -j ACCEPT
-A FILTERS -i lo -j ACCEPT
#Below OS ports open to everyone if uncommented
-A FILTERS -p tcp -m state --state NEW -m tcp -m multiport --dports 22 -j ACCEPT
#-A FILTERS -p udp -m udp -m multiport --dports 53,123 -j ACCEPT
-A FILTERS -m set ! --match-set ip_allow src -j DROP
-A FILTERS -j RETURN
COMMIT

EOF
```

Use iptables-restore to add the above rules into iptables. The very important flag is -n. That makes sure we don't flush the iptables rules if we have rules already in Docker (or Docker Swarm).

```bash
iptables-restore -n < ansible_iptables_docker-iptables
```

Next, add a rule to the INPUT chain, so we start using the new rules in FILTERS. It has to be at the top, and only needs to be added once:

```bash
iptables -I INPUT 1 -j FILTERS
```

Save the iptables rules:

```bash
/usr/libexec/iptables/iptables.init save
```

Start and Enable the iptables service:

```bash
systemctl status iptables
systemctl start iptables
systemctl enable iptables
```

If you want to customize the iptables rules to allow more ports to be open to everyone, just add the port to the appropriate rule in the iptables file (tcp or udp), then re-run the same commands from above:

```bash
iptables-restore -n < iptables-rules.txt
/usr/libexec/iptables/iptables.init save
```

Don't miss the Warnings from above! Especially about SELinux.

## TODO

* [x] Check for firewalld and fail if running or enabled
* [x] Problem with iptables saving Docker rules in iptables rules? Should be fine.
* [x] iptables_docker_ip_allow_set can't be empty. If it is, there's no point to this since nothing is blocked!
* [x] add check in network adapters for * and error
* [x] add automatic list of docker IPs in allowed list (uses IPs from inventory group docker_hosts)
* [x] Change auto Docker server trusted IPs so can override
* [x] confirm "when" and "tags" are ok
* [x] Ubuntu? Ubuntu doesn't have iptables-services or ipset-service. has iptables-persistent and ipset-? No ufw support
* [ ] ipv6?? This is for ipv4 only
* [x] test TCP, UDP Docker container and OS port work
* [x] test outound traffic from Docker containers work
* [ ] add test? Molecule? Single node swarm mode only? how to test connection doesn't work from "untrusted" ip?

## Author

Ryan Daniels
