# node[1-x] Installation
## On the new node
* Do a basic install with ubuntu 22.04
(version is important and should be consistent over all nodes and the controller)
* set the name of the host (e.g. node242)
* Configure node with a cluster-net (10.0.0.x/24) address and set the default gateway to slurmmaster (10.0.0.9) 
* Install opensshd
```shell
apt update; apt install openssh-server -y
```

* Edit GRUB_CMDLINE_LINUX_DEFAULT 
  * /etc/default/grub and add systemd.unified_cgroup_hierarchy=false 
  * systemd.legacy_systemd_cgroup_controller=false 
```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash systemd.unified_cgroup_hierarchy=false systemd.legacy_systemd_cgroup_controller=false "
```
*  run /usr/sbin/update-grub - This is needed for the cgroup integration in slurmctld

* Download following file and unpack as user root in /root/ on the new node
  * Node install: Kickstart.tar (Or a prepared tarball under /etc/ansible/node_install/)
  * Run as root /root/kickstart/do_rootkey_sshd_config.sh
    * Installs sshd_config and the root pubkey (Passwd can be found in 1Passwd) 
    * The run /root/kickstart/do_ansible_account.sh 
      * installs an ansible account with disabled passwd login and 
      * a passwordless ssh-key login for the ansible-controller on slurmmaster
* Now the ansible-controller (aka. slurmmaster) takes over

## On the Controller Node
All commands are to be issued on slurmmaster
* Edit the [install_nodes] section in /etc/ansible/hosts to reflect settings of your install candidate
* Also add the new node to the [nodes] section above - which is the "production node group" This is the 'computername' you chose during node installation !
An example entry looks like this:
```text
[install_nodes]
nodex ansible_host=10.0.0.113 ansible_ssh_port=22
```
* Check if we can reach our install candidate via ansible.
```shell
root@slurmmaster:/etc/ansible/node_install# ansible-playbook reachable.yml
```

* If yes .... continue - You may have to do this multiple times (depends on newly added nodes) an accept the ssh-keys (type 'yes' to continue - in 'blind' mode)
* Add the new host to cluster wide system hosts template file under /etc/slurm/hostsfile 
  * from this file system hostfile entries (/etc/hosts) are created for the master and the nodes
  * Sync the hostsfile across the master and all nodes via
```shell
root@slurmmaster:/etc/ansible/slurm_maint# ansible-playbook sync_hostfiles.yml
```

* Now we get to the main part of the node installation.
* Issue the following commands only once and in this order if you don't know exactly what you are doing.
  * (All playbooks should be foolproof - but check the results immediately if you have to run it twice)
  * Playbooks reside in /etc/ansible/nodeinstall and should be run in numerical order.
```shell
root@slurmmaster:/etc/ansible/node_install# ansible-playbook 123_example.yml
```

We prepared files config files as follows
* 01_set_proxy_for_nodes.yml - sets proxy for apt tp strauss.kr
* 02_do_initial_update.yml - upgrades to latest packages installed and reboots host if neccessary. 
* 03_distribute_hosts.yml - Distributes hosts file (new hosts need to be added in XXXXXXX)
* 04_do_nfs_mounts.yml - Sets up /etc/fstab, mountpoints and mounts central nfs directories (changes must be done in XXXXXXX) 05_do_ldap_config.yml - Installs and configures ldap client systems
* 06_do_slurm_node_config.yml - Installs neccesary slurm components and configurations
  
Since the files depend on internal configurations, we do not share the files publicly. If you need templates, contact us via mail.

These playbooks are intended for the initial installation of a node - use the playbooks under /etc/ansible/slurm_maint for cluster housekeeping !

* Gather the hardware specs of the candidate:
```shell
root@slurmmaster:/etc/ansible/node_install# ansible-playbook showspecs.yml
```

```text
ok: [node9] => { "command_output.stdout_lines": [
"NodeName=node9 CPUs=24 Boards=1 SocketsPerBoard=2 CoresPerSocket=12 ThreadsPerCore=1 RealMemory=257827",
"UpTime=0-20:27:32" ]
}
```

* Now add the new host to /etc/slurm/slurm.conf with the exact specs from above.
I recommend to use one config line per host in favor of the wildcard entries.
slurm.conf is picky as fuck when it comes to wrong values and if you don't have a range of the exact same node models you're better off this way
* Distribute the config file to all nodes:
```shell
root@slurmmaster:/etc/ansible/slurm_maint# ansible-playbook do_slurmd_config_sync.yml
```
* Restart the controller by issuing:
```shell
$ service slurmctld restart
```

* Force controller and nodes to reread their new config by issuing on slurmmaster
```shell
root@slurmmaster:~# scontrol reconfigure
```

* Activate the new node by executing following command on the slurm controller:
```shell
scontrol update NodeName=nodeXXX State=RESUME
```

* Check on your newly added host:
```shell
root@slurmmaster:~# sinfo
PARTITION AVAIL TIMELIMIT NODES STATE NODELIST
ashpool up infinite 1 idle slurmmaster
MYPARTITION* up infinite 4 idle slurmmaster,node[5,9,12]
```
    * If you get something like this you should be good.
    * If you get a node state of 'DRAIN' or 'DOWN' try adding it with the 'RESUME' command from above 
    * If a node is in state 'INVAL' there's something wrong with it's configuration. 
    * (Most likely the node entry in slurm.conf)

