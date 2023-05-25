# Cluster Housekeeping
## sinfo - show info on cluster and nodes
```shell
$ sinfo
$ sinfo -s $ sinfo -Nel
```

## squeue - show info on running jobs:

## scancel - cancel a running job
```shell
$ scancel 314
```

## sacct - show info on finished jobs (JobID '314' in this example):
```shell
$ sacct -j 314 --format="User,JobID,JobName%30,Partition,Account,AllocCPUS,State,ExitCode"
$ sacct -j 314 --format=User,JobID,Jobname,partition,state,time,start,end,elapsed,MaxRss,MaxVMSize,nnodes,ncpus,nodelist
```

## Take node offline:
```shell
$ scontrol update NodeName=nodeXXX State=DRAIN Reason=”Maintenance”
```

## Take node online:
```shell
$ scontrol update NodeName=nodeXXX State=RESUME
```

## After config changes to cgroups.conf, gres.conf and/or slurm.conf distribute config files via:
```shell
$ ansible-playbook /etc/ansible/slurm_maint/do_slurmd_config_sync.yml
# and send reconfigure to all nodes: 
$ scontrol reconfigure
```
