# crossover

[![License](https://img.shields.io/github/license/EnterpriseVE/eve4pve-barc.svg)](https://www.gnu.org/licenses/gpl-3.0.en.html)

Cross-Pool (live) Replication and near-live migration for Proxmox VE


```text
______
|     |___ ___ ___ ___ ___ _ _ ___ ___ 
|   --|  _| . |_ -|_ -| . | | | -_|  _|
|_____|_| |___|___|___|___|\_/|___|_|  

Cross Pool (live) replication and near-live migration for Proxmox VE  

Usage:
    crossover <COMMAND> [ARGS] [OPTIONS]
    crossover help
    crossover version
 
    crossover mirror     --vmid=<string> --destination=<destionationhost> --pool=<targetpool> --keeplocal=n --keepremote=n
Commands:
    version              Show version program
    help                 Show help program
    mirror               Replicate a stopped VM to another Cluster (full clone)

Options:
    --vmid               The source+target ID of the VM/CT, comma separated (eg. --vmid=100:100,101:101), 
    --destination        'Target PVE Host in target pool. e.g. --destination=pve04
    --pool               'Ceph pool name in target pool. e.g. --pool=data
    --keeplocal          'How many additional Snapshots to keep locally. e.g. --keeplocal=2
    --keepremote         'How many additional Snapshots to keep remote. e.g. --keepremote=2
    --online             'Allow online Copy
    --nolock             'Don't lock source VM on Transfer (mainly for test purposes)
    --keep-slock         'Keep source VM locked on Transfer
    --keep-dlock         'Keep VM locked after transfer on Destination
    --overwrite          'Overwrite Destination
    --protect            'Protect Ceph Snapshots
    --debug              'Show Debug Output

Report bugs to the Github repo at https://github.com/lephisto/crossover/
```

## Introduction

When working with hyperconverges Proxmox HA Clusters you sometimes need to get VMs migrated 
to another cluster, or have a cold-standby copy of a VM ready to start there. Crossover implements
functions that enable you to do the following:

- Transfer a non-running VM to another Cluster
- Transfer a running VM to another Cluster
- Continuously update a previously tranferred VM in another Cluster with incemental snapshots

Currently this only works with Ceph based storage backends, since the incremental logic heavily
relies on Rados block device features.

It'll work according this scheme:

```
.:::::::::.                                                           .:::::::::. 
|Cluster-A|                                                           |Cluster-B|
|         |                                                           |         |
| _______ |  rbd export-diff [..] | ssh pve04 | rbd import-diff [..]  | _______ |
|  pve01 -|-----------------------------------------------------------|->pve04  |
| _______ |                                                           | _______ |
|  pve02  |                                                           |  pve05  |
| _______ |                                                           | _______ |
|  pve03  |                                                           |  pve06  |
| _______ |                                                           | _______ |
|         |                                                           |         |
|:::::::::|                                                           |:::::::::|
```

## Main features

* Currently only for KVM. I might add LXC support when I need to.
* Can keep multiple backup
* Retention policy: (eg. keep x snapshots on the source and y snapshots in the destination cluster)
* Rewrites VM configurations so they match the new VMID and/or poolname on the destination

## Protected / unprotected snapshot

!TBD!
You can protect Ceph Snapshots by the according Ceph/RDB flag, to avoid accidental deletion
and thus damaging your chain. Keep in mind that Proxmox won't let you delete VM's then, because
it's not aware of that paramter.

## Installation of prerequisites

```apt install git

## Install the Script somewhere, eg to /opt

git clone https://github.com/lephisto/crossover/ /opt 

```

## Usage

Mirror VM to another Cluster:

```
root@pve01:~/crossover# ./crossover mirror --vmid=100:10100 --destination=pve04 --pool=data2 --keeplocal=4 --keepremote=8 --overwrite --keep-dlock --online

Start mirror 2022-10-21 18:09:36
Transmitting Config for VM 100 to desination 10100
update VM 100: -lock backup
update VM 10100: -lock backup
VM 100 - Issuing fsfreeze-freeze to 100 on pve01
2
VM 100 - Creating snapshot data/vm-100-disk-0@mirror-20221021180936
Creating snap: 100% complete...done.
VM 100 - Creating snapshot data/vm-100-disk-1@mirror-20221021180936
Creating snap: 100% complete...done.
VM 100 - Issuing fsfreeze-thaw to 100 on pve01
2
Exporting image: 100% complete...done.
Importing image diff: 100% complete...done.
Houskeeping localhost data vm-100-disk-0, keeping previous 4 Snapshots
Removing snap: 100% complete...done.
Houskeeping pve04 data2 vm-10100-disk-0, keeping previous 8 Snapshots
Exporting image: 100% complete...done.
Importing image diff: 100% complete...done.
Houskeeping localhost data vm-100-disk-1, keeping previous 4 Snapshots
Removing snap: 100% complete...done.
Houskeeping pve04 data2 vm-10100-disk-1, keeping previous 8 Snapshots
Unlocking source VM 100
root@pve01:~/crossover#

```
This example creates a mirror of VM 100 (in the source cluster) as VM 10100 (in the destination cluster) using the ceph pool "data2" for storing all attached disks. It will keep 4 Ceph snapshots prior the latest (in total 5) and 8 snapshots on the remote cluster. It will keep the VM on the target Cluster locked to avoid an accidental start (thus causing split brain issues), and will do it even if the source VM is running. 

The use case is that you might want to keep a cold-standby copy of a certain VM on another Cluster. If you need to start it on the target cluster you just have to unlock it with `qm unlock VMID` there.

Another usecase could be that you want to migrate a VM from one cluster to another with the least downtime possible. Real live migration that you are used to inside one cluster is hard to achive cross-cluster, but you can easily make an initial migration while the VM is still running on the source cluster (fully transferring the block devices), shut it down on source, run the mirror process again (which is much faster now because it only needs to transfer the diff since the initial snapshot) and start it up on the target cluster. This way the migration basically takes one boot plus a few seconds for transferring the incremental snapshot.

## Things to check

From Proxmox VE Hosts you want to backup you need to be able to ssh passwordless to all other Cluster hosts, that may hold VM's or Containers. This goes for the source and for the destination Cluster.

This is required for using the free/unfreeze and the lock/unlock function, which has to be called locally from that Host the guest is currently running on. Usually this works out of the box for the source cluster, but you may want to make sure that you can "ssh root@pvehost1...n" from every host to every other host in the cluster.

For the Destination Cluster you need to copy your ssh-key to the first host in the cluster, and login once to every node
in your cluster.


## Some words about Snapshot consistency and what qemu-guest-agent can do for you

Bear in mind, that when taking a snapshot of a running VM, it's basically like if you have a server which gets pulled away from the Power. Often this is not cathastrophic as the next fsck will try to fix Filesystem Issues, but in the worst case this could leave you with a severely damaged Filesystem, or even worse, half written Inodes which were in-flight when the power failed lead to silent data corruption. To overcome these things, we have the qemu-guest-agent to improve the consistency of the Filesystem while taking a snapshot. It won't leave you a clean filesystem, but it sync()'s outstanding writes and halts all i/o until the snapshot is complete. Still, there might me issues on the Application layer. Databases processes might have unwritten data in memory, which is the most common case. Here you have the opportunity to do additional tuning, and use hooks to tell your vital processes things to do prio and post freezes.

First, you want to make sure that your guest has the qemu-guest-agent running and is working properly. Now we use custom hooks to tell your services with volatile data, to flush all unwritten data to disk. On debian based linux systems the hook file can be set in ```/etc/default/qemu-guest-agent``` and could simply contain this line:

```
DAEMON_ARGS="-F/etc/qemu/fsfreeze-hook"
```

Create ```/etc/qemu/fsfreeze-hook``` and make ist look like:

```
#!/bin/sh

# This script is executed when a guest agent receives fsfreeze-freeze and
# fsfreeze-thaw command, if it is specified in --fsfreeze-hook (-F)
# option of qemu-ga or placed in default path (/etc/qemu/fsfreeze-hook).
# When the agent receives fsfreeze-freeze request, this script is issued with
# "freeze" argument before the filesystem is frozen. And for fsfreeze-thaw
# request, it is issued with "thaw" argument after filesystem is thawed.

LOGFILE=/var/log/qga-fsfreeze-hook.log
FSFREEZE_D=$(dirname -- "$0")/fsfreeze-hook.d

# Check whether file $1 is a backup or rpm-generated file and should be ignored
is_ignored_file() {
    case "$1" in
        *~ | *.bak | *.orig | *.rpmnew | *.rpmorig | *.rpmsave | *.sample | *.dpkg-old | *.dpkg-new | *.dpkg-tmp | *.dpkg-dist | 
*.dpkg-bak | *.dpkg-backup | *.dpkg-remove)
            return 0 ;;
    esac
    return 1
}

# Iterate executables in directory "fsfreeze-hook.d" with the specified args
[ ! -d "$FSFREEZE_D" ] && exit 0
for file in "$FSFREEZE_D"/* ; do
    is_ignored_file "$file" && continue
    [ -x "$file" ] || continue
    printf "$(date): execute $file $@\n" >>$LOGFILE
    "$file" "$@" >>$LOGFILE 2>&1
    STATUS=$?
    printf "$(date): $file finished with status=$STATUS\n" >>$LOGFILE
done

exit 0
```

For testing purposes place this into ```/etc/qemu/fsfreeze-hook.d/10-info```:

```
#!/bin/bash
dt=$(date +%s)

case "$1" in
    freeze)
        echo "frozen on $dt" | tee >(cat >/tmp/fsfreeze)
    ;;
    thaw)
        echo "thawed on $dt" | tee >(cat >>/tmp/fsfreeze)
    ;;
esac

```

Now you can place files for different Services in ```/etc/qemu/fsfreeze-hook.d/``` that tell those services what to to prior and post snapshots. A very common example is mysql. Create a file ```/etc/qemu/fsfreeze-hook.d/20-mysql``` containing

```
#!/bin/sh

# Flush MySQL tables to the disk before the filesystem is frozen.
# At the same time, this keeps a read lock in order to avoid write accesses
# from the other clients until the filesystem is thawed.

MYSQL="/usr/bin/mysql"
#MYSQL_OPTS="-uroot" #"-prootpassword"
MYSQL_OPTS="--defaults-extra-file=/etc/mysql/debian.cnf"
FIFO=/var/run/mysql-flush.fifo

# Check mysql is installed and the server running
[ -x "$MYSQL" ] && "$MYSQL" $MYSQL_OPTS < /dev/null || exit 0

flush_and_wait() {
    printf "FLUSH TABLES WITH READ LOCK \\G\n"
    trap 'printf "$(date): $0 is killed\n">&2' HUP INT QUIT ALRM TERM
    read < $FIFO
    printf "UNLOCK TABLES \\G\n"
    rm -f $FIFO
}

case "$1" in
    freeze)
        mkfifo $FIFO || exit 1
        flush_and_wait | "$MYSQL" $MYSQL_OPTS &
        # wait until every block is flushed
        while [ "$(echo 'SHOW STATUS LIKE "Key_blocks_not_flushed"' |\
                 "$MYSQL" $MYSQL_OPTS | tail -1 | cut -f 2)" -gt 0 ]; do
            sleep 1
        done
        # for InnoDB, wait until every log is flushed
        INNODB_STATUS=$(mktemp /tmp/mysql-flush.XXXXXX)
        [ $? -ne 0 ] && exit 2
        trap "rm -f $INNODB_STATUS; exit 1" HUP INT QUIT ALRM TERM
        while :; do
            printf "SHOW ENGINE INNODB STATUS \\G" |\
                "$MYSQL" $MYSQL_OPTS > $INNODB_STATUS
            LOG_CURRENT=$(grep 'Log sequence number' $INNODB_STATUS |\
                          tr -s ' ' | cut -d' ' -f4)
            LOG_FLUSHED=$(grep 'Log flushed up to' $INNODB_STATUS |\
                          tr -s ' ' | cut -d' ' -f5)
            [ "$LOG_CURRENT" = "$LOG_FLUSHED" ] && break
            sleep 1
        done
        rm -f $INNODB_STATUS
        ;;

    thaw)
        [ ! -p $FIFO ] && exit 1
        echo > $FIFO
        ;;

    *)
        exit 1
        ;;
esac

```

## Last remarks

_Test your Backups on a regular Base. Restore them and see if you can mount and/or boot. Snapshots are not meant to be a full replacement for traditional Backups, don't rely on them as the only Source even if it looks very convenient. Follow the n+1 principle and do filebased backups from within your VM's (with Bacula, Borg, rsync, you name it.). If one concept fails for some reason you always have another way to get your Data._

## Useful resources

Ceph Documentation:
[Incremental snapshots with rbd](http://ceph.com/dev-notes/incremental-snapshots-with-rbd/)
[rdb – manage rados block device (rbd) images](http://docs.ceph.com/docs/master/man/8/rbd/)

Proxmox Wiki:
https://pve.proxmox.com/wiki/