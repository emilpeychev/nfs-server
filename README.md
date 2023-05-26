# nfs-server

Setup NFS server for persistent volume storage

## Table of contents

* [Prepare the NFS share](#prepare_the_nfs_share)
* [Setup the NFS server](#setup_the_nfs_server)
* [Install NFS client](#install_nfs_client)
* [Setup AutoFS](#setup_autofs)

### Prepare_the_nfs_share

* This chapter explains how to create NFS server and allow for the persistent volume storage share to be written on instead of using local storages on the worker nodes.
* The first thing we want to do is to prepare the share I am using a raspberry pi 3b+ and a USB 2.0 stick 120GB.

* Format the USB with gdisk or other tool and give it ext4 file system.

* Create mounting point (Example)

```
mkdir -p /mnt/nfs

```

* Create two directories volumes and web-volumes in the nfs dir.
* Mount the stick

```
sudo mount /dev/sda1 /mnt/nfs
```

* Get the block UUID of the mounted USB.

```
sudo blkid /dev/sda1
```

* Edit /etc/fstab and add

```
UUID=76906fdc-xxxx-oooo-xxxx-xxxxxxxxxx /mnt/nfs ext4 defaults,auto,users,rw,nofail,noatime 0 0 
```

*Obviously replace your UUID with your output from the previous command.

* Reboot and ping the server to know when it is ready.

### Setup_the_NFS_server

* Install nfs server

```
 sudo apt install nfs-kernel-server
```

### Create_shared_directories

* Create the directories that you want to share and use as Persistent Volumes.

```
cd /mnt/nfs/ 
sudo mkdir volumes web-volumes devops-tools
```

* Exposed share  network  /mask nfs options

```
/mnt/nfs/volumes *(rw,sync,no_subtree_check,no_root_squash)
/mnt/nfs/web-volumes *(rw,sync,no_subtree_check,no_root_squash)
/mnt/nfs/devops-tools *(rw,sync,no_subtree_check,no_root_squash)
```

* More information (<https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_file_systems/exporting-nfs-shares_managing-file-systems#nfs-server-configuration_exporting-nfs-shares>)
* Last

```
sudo chown -R $(id -u):$(id -g) /mnt/nfs 
```

### Install_nfs_client

* !Important, in case you want to backup etcd from master node you need to create its dir in /mnt/nfs as well and expose it in the /etc/exports ``` /mnt/nfs/etcd-backup 192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash) ```.

### Install the nfs client

* Install the nfs client on worker nodes.

```
sudo apt install nfs-common
```

* Now that you have the client show the exported shares of the nfs server. On the worker node pass the command

```
showmount --exports <NFS-ServerIP>
```

### Setup_AutoFS

* !Important, in case you want to backup etcd from master node you need to install AutoFS on the master-node.
Install AutoFS. We install autofs to handle mount-unmount operation because otherwise we need to mount the share to the /etc/fstab of the worker nodes. There are issues with this approach and namely the nfs can get locked and this can cause major problems to the cluster.

```
sudo apt install autofs
```

* Edit the /etc/auto.master and add the following lines at the end of the file.

```
#settings
/mnt/nfs /etc/auto.nfs --ghost --timeout=60
```

* This tells the autofs to mount the exposed dirs as ghosts and set timeout of 60 sec.
* Edit or Create the /etc/auto.nfs and add the following lines at the end of the file.

```
volumes -fstype=nfs4,rw <NFS-ServerIP>:/mnt/nfs/volumes
web-volumes -fstype=nfs4,rw <NFS-ServerIP>:/mnt/nfs/web-volumes
```

* Respectively if you will be using the master to backup the etcd add

```
etcd-backup -fstype=nfs4,rw <NFS-ServerIP>:/mnt/nfs/etcd-backup
```
