# Icinga
* icinga command definitions reside under /usr/share/icinga2/include
```shell
icinga2 object list --type Service --name *node*
```

* dreaded '/var/run/doc/0' problem
* set this in /etc/icinga2/conf.d/services.conf
* See below for a better solution:
```text
apply Service for (disk => config in host.vars.disks) { import "generic-service"
check_command = "disk"
vars.disk_partitions_excluded = ["/run","/run/user","/run/user/0","/run/user/0/doc"] 
# You have do specify the _exact_ location of the directory to be ignored
vars += config }
```

* Declare this in services.conf:
```text
apply Service "disk" { check_command = "disk"

// disk_all is needed by the following regex ignore 
vars.disk_all = true
vars.disk_ignore_ereg_path = ["/run/*"]

// Specify the remote agent as command execution endpoint, fetch the host custom variable 
command_endpoint = host.vars.agent_endpoint
// Only assign where a host is marked as agent endpoint assign where 
host.vars.agent_endpoint
}
```

## Resource Specification and Limits
As mentioned above, the node definition in slurm.conf needs to include what hardware resources the node offers (calling slurm -C on the node outputs the available spec). Importantly, the slurm controller will not start if the config announces more resources (memory, cpu, ...) than available. Especially for main memory this seems to be an issue as a node might report less memory after a restart. Taking this into account, the current node config for a slurmmaster node is as follows.
```text
NodeName=nodeX NodeAddr=10.0.0.1XX CPUs=24 RealMemory=250000 Sockets=2 CoresPerSocket=12 ThreadsPerCore=1 State=UNKNOWN
```
* CPUs=24 indicates that there are 24 cpu cores
* RealMemory=250000 sets the maximal available main memory to 250000MB, which is less than possible but this value is stable and also
leaves some overhead for the OS
* Sockets=2 is the number of cpu sockets and CoresPerSocket=12 defines how many cores each socket has
(CPUs=Sockets*CoresPerSocket)
* ThreadsPerCore=1 sets how many threads can be executed on each core, since we have no hyperthreading this has to be 1

For gpunode1, we have the following configuration.
```text
NodeName=gpunode1 NodeAddr=10.0.0.150 CPUs=4 Boards=1 SocketsPerBoard=1 CoresPerSocket=2 ThreadsPerCore=2 RealMemory=15955
```

In difference to the nodes, here we allow hyperthreading and each physical core has 2 logical cores.


Resource limits are enforced by the task/affinity and the task/cgroups plugins. Jobs are contrained to certain cores, their amount depending on what was requested. Similarly, should a job try to allocate more memory than it requested, it will receive an OOM error. The same holds for the requested time. After the requested time has passed, slurm will send a SIGTERM and, should the job not terminate, a SIGKILL. The time between SIGTERM and SIGKILL can be specified with the parameter KillWait in slurm.conf. The default number of CPUs requested by a job is 1 and there is no way to change this.

Default values for those limits should be specified in the partition config. For MYPARTITION:
```text
PartitionName=MYPARTITION Nodes=node[3-11] Default=YES MaxTime=INFINITE State=UP DefMemPerCPU=8192 DefaultTime=01:00:00 CpuBind=core
```

* (Default=YES sets MYPARTITION as the default partition)
* MaxTime=INFINITE sets the maximal time that a job can request to infinite, we may want to reconsider this at some point DefMemPerCPU=8192 sets the default memory to 8GB
* DefaultTime=01:00:00 sets the default time to 1h
* CpuBind=core defines that when a job requests 1 cpu, it will be assigned 1 core (not 1 socket or logical core)

For opencl the DefMemPerCPU is set to 3GB, since gpunode1 only has 16GB of RAM, and there is no default time.

