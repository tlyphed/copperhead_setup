# Services and Dependencies
* nfs.cluster (Physical Hardware): NFS server
  * User disks use ZFS: easy quota handling / stable data / no issues with many or large files
  * Separate between external, students, staff
  * NFS mounts with parameters async,no_subtree_check to avoid hanging nfs servers 
* ldap.cluster (Virtual Machine): LDAP server (needs NFS) 
* sshgate.cluster (Virtual Machine): SSH gate (needs NFS, LDAP) 
* slurmmaster.cluster (Virtual Machine)
  * Services: slurmd, 
* slurmnodeX.cluster (Physical System): compute nodes (needs everything above)

# Additional Tooling
* Ansible for fast deployment of configurations [Link](https://www.ansible.com/)
* Squid as Proxy Server [Link](http://www.squid-cache.org/)
* Apt-cacher-ng as Package Cache [Link](https://help.ubuntu.com/community/Apt-Cacher%20NG)
* Icinga for Monitoring [Link](https://icinga.com/)
* Fusiondirectory as LDAP system [Link](https://www.fusiondirectory.org/)
* 1Password: Store and Share Passwords