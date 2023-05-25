# slurmmaster.cluster Installation
## NAT/router
* is NAT-router for client requests to the ldap server.
* set net.ipv4.ip_forward=1 in /etc/sysctl.conf and reload config sysctl -p
* enable NAT for all traffic coming from our private net
```shell
 iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE
# (HINT: to check settings add type NAT - otherwise you won't see the POSTROUTING chain 
# iptables -t nat -L -v)
```
* make configuration persistent (i.e. survive reboot)
```shell
apt update -y && apt install iptables-persistent
```
    * It will asks you to save the current ipv4 and ipv6 iptables rules, answer yes to save.
    * Run this command if you made changes afterwards iptables-save > /etc/iptables/rules.v
    * Restores settings via iptables-persistent on every reboot.

## Munge
* Required for fast authentication on the hosts
* ssh is too slow for many extremely short running jobs
* See: [https://github.com/dun/munge/wiki/Installation-Guide](https://github.com/dun/munge/wiki/Installation-Guide)

## Ansible Controller
```shell
apt install ansible
```

This installs ansible and a bunch of python packages which are needed by it.

```shell
$ ansible-config init --disabled > ansible.cfg
$ ansible-config init --disabled -t all > ansible.cfg
Useful commands:
ansible-inventory --list -y
-y flag == output as yaml instead of JSON
ansible all -m ping -u ansible
ansible node12 -m ping -u ansible ansible all -a "ls -la" -u ansible ansible-playbook rds_prod.yml --syntax-check
```

## SLURM
* install /etc/hosts - do via ansible
* verify ntp/time on master and clients --> done by do_mungenode.sh
* set search domain
* Integrate into ldap
```shell
#Test ldap connectivity
ldapsearch -x -b "dc=cobra,dc=kr,dc=tuwien,dc=ac,dc=at" -H ldap://128.130.121.237
```
* Configure sshd to use only keys, install wrapper --> done by do_sshd_config.sh
* distribute munge key to clients --> done by do_mungenode.sh
  * (user 'munge' is a system account and does not need to have the sam UID/GID across nodes)
* install mailutils so slurm won't complain about missing /bin/mail --> ansible
```shell
 apt install -y mailutils
```
distribute configuration
* install slurm on master apt install -y slurm-wlm slurm-wlm-basic-plugins slurm-wlm-doc slurm-client slurmctld
   slurmd slurmdbd


## mariadb
* install mariadb:
```shell
apt install mariadb-server mariadb-client 
systemctl enable 
mariadb mysql_secure_installation
```
* add to /etc/mysql/mariadb.conf.d/50-server.cnf
```text
innodb_buffer_pool_size = 8G 
innodb_log_file_size=128M 
innodb_lock_wait_timeout=900
```
```shell
systemctl stop mariadb
mv /var/lib/mysql/ib_logfile? /tmp/ 
systemctl start mariadb
```
```shell

$ mysql -u root -p
mysql> grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by 'some_pass' with grant option; 
mysql> create database slurm_acct_db;
```

* Do the same for the jobcomp db needed by the ctld
```shell
$ mysql -u root -p
mysql> grant all on slurm_jobcomp_db.* TO 'slurm'@'localhost' identified by 'some_pass' with grant option; 
mysql> create database slurm_jobcomp_db;
```
* configure slurmdbd
  * make sure that each and every file ist owned by slurm:slurm 
  * create /etc/slurm/slurmdbd.conf - and set it to 600
```shell
systemctl start slurmdbd 
systemctl enable slurmdbd 
systemctl status slurmdbd
```

* correct any errors thrown by status
* Now start in this order, slurmd the slurctld
* Remedy any error it throws. Especially check log/run files directories if they exist and ownership is set to slurm:slurm

## Runtime Checks
On Master: Verify that these daemons are running:
```text
munged, slurmdbd, slurmd, slurmctld
```
If you have to restart them do as in the above order.


## Commands
* To check on the controller use scontrol which takes a lot of arguments.
* If we get something like this we should be good:
```shell
:~# scontrol show job No jobs in the system
```
* Have a go at scontrol show <ENTITY>
```text
<ENTITY> may be "aliases", "assoc_mgr", "bbstat", "burstBuffer", "config", "daemons", "dwstat", "federation", "frontend", "hostlist", "hostlistsorted", "hostnames", "job", "node", "partition", "reservation", "slurmd", "step", or "topology"
```
* to load config changes on all machines (controller & nodes) - however does not restart the daemons
```shell
scontrol reconfigure
```

```shell
scontrol show job
```

```shell
scancel <jobID>
```

```shell
sinfo
```

* Modify the state with scontrol, specifying the node and the new state. You must provide a reason when disabling a node.
  * Disable: scontrol update NodeName=node[02-04] State=DRAIN Reason=”Cloning”
  * Enable: scontrol update NodeName=node[02-04] State=RESUME
```shell
scontrol update NodeName=node5 State=RESUME
```
