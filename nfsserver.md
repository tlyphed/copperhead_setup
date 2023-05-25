# NFS Server
## ZFS / Quotas
Quotas are currently set for 'normal' users. As of now you have to do this by hand during user creation.

Check quotas:
```shell
zfs userspace MYPOOL
```

Find disk hogs:
```shell
zfs userspace MYPOOL | sort -h -k4
```

Set quota of testuser for zpool 'interim':
```shell
zfs set userquota@testuser=100G MYPOOL
```

* Currently, openzfs can not set a 'defaultuserquota' as it is possible with Sun variant.
* ZFS quick command reference: https://www.unixarena.com/2012/07/zfs-quick-command-reference-with.html/

## NFS Server
nfs.cluster is NFS Server for nfs.cluster (ldap needs the homes to write configuration during user creation) and nodes.
```text

```