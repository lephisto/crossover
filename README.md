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
 
    crossover mirror     --vmid=<string> --destination=<destionationhost> --pool=<targetpool> --keeplocal=[n][d|s] --keepremote=[n][d|s]
Commands:
    version              Show version program
    help                 Show help program
    mirror               Replicate a stopped VM to another Cluster (full clone)

Options:
    --vmid               The source+target ID of the VM, comma separated (eg. --vmid=100:100,101:101)
                         (The possibility to specify a different Target VMID is to not interfere with VMIDs on the
                         target cluster, or mark mirrored VMs on the destination)
    --prefixid           Prefix for VMID's on target System [optional]
    --excludevmids       Exclusde VM IDs when using --vmid==all
    --destination        Target PVE Host in target pool. e.g. --destination=pve04
    --pool               Ceph pool name in target pool. e.g. --pool=data
    --keeplocal          How many additional Snapshots to keep locally, specified in seconds or day. e.g. --keeplocal=2d
    --keepremote         How many additional Snapshots to keep remote, specified in seconds or day. e.g. --keepremote=7d
    --rewrite            PCRE Regex to rewrite the Config Files (eg. --rewrite='s/(net0:)(.*)tag=([0-9]+)/\1\2tag=1/g' would
                         change the VLAN tag from 5 to 1 for net0.
    --influxurl          Influx API url (e.g. --influxurl=https://your-influxserver.com/api/)
    --influxtoken        Influx API token with write permission
    --influxbucket       Influx Bucket to write to (e.g. --influxbucket=telegraf/autogen)
Switches:    
    --online             Allow online Copy
    --nolock             Don't lock source VM on Transfer (mainly for test purposes)
    --keep-slock         Keep source VM locked on Transfer
    --keep-dlock         Keep VM locked after transfer on Destination
    --overwrite          Overwrite Destination
    --protect            Protect Ceph Snapshots
    --debug              Show Debug Output

Report bugs to the Github repo at https://github.com/lephisto/crossover/
```

## Introduction

When working with hyperconverged Proxmox HA Clusters you sometimes need to get VMs migrated to another cluster, or have a cold-standby copy of a VM ready to start there in case your main Datacenter goes boom. Crossover implements functionality that enables you to do the following:

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
* Secure an encrypted transfer (SSH), so it's safe to mirror between datacenter without an additional VPN
* Near live-migrate: To move a VM from one Cluster to another, make an initial copy and re-run with --migrate. This will shutdown the VM on the source cluster and start it on the destination cluster.

## Installation of prerequisites

```apt install git pv gawk jq

## Install the Script somewhere, eg to /opt

git clone https://github.com/lephisto/crossover/ /opt 

```

Ensure that you can freely ssh from the Node you plan to mirror _from_ to _all_ nodes in the destination cluster, as well as localhost.

## Continuous replication between Clusters

Example 1: Mirror VM to another Cluster:

```
root@pve01:~/crossover# ./crossover mirror --vmid=all --prefixid=99 --excludevmids=101 --destination=pve04 --pool=data2 --overwrite --online 
ACTION: Onlinemirror
Start mirror 2022-11-01 19:21:44
VM 100 - Starting mirror for testubuntu
VM 100 - Checking for VM 99100 on Destination Host pve04 /etc/pve/nodes/*/qemu-server
VM 100 - Transmitting Config for to destination pve04 VMID 99100
VM 100 - locked 100 [rc:0]
VM 99100 - locked 99100 [rc:0]
VM 100 - Creating snapshot data/vm-100-disk-0@mirror-20221101192144
VM 100 - Creating snapshot data/vm-100-disk-1@mirror-20221101192144
VM 100 - unlocked source VM 100 [rc:0]
VM 100 - I data/vm-100-disk-0@mirror-20221101192144: e:0:00:01 c:[ 227KiB/s] a:[ 227KiB/s]  372KiB
VM 100 - Housekeeping: localhost data/vm-100-disk-0, keeping Snapshots for 0s
VM 100 - Removing Snapshot localhost data/vm-100-disk-0@mirror-20221101192032 (106s) [rc:0]
VM 100 - Housekeeping: pve04 data2/vm-99100-disk-0-data, keeping Snapshots for 0s
VM 100 - Removing Snapshot pve04 data2/vm-99100-disk-0-data@mirror-20221101192032 (108s) [rc:0]
VM 100 - Disk Summary: Took 2 Seconds to transfer 372.89 KiB in a incremental run
VM 100 - I data/vm-100-disk-1@mirror-20221101192144: e:0:00:00 c:[ 346 B/s] a:[ 346 B/s] 74.0 B
VM 100 - Housekeeping: localhost data/vm-100-disk-1, keeping Snapshots for 0s
VM 100 - Removing Snapshot localhost data/vm-100-disk-1@mirror-20221101192032 (114s) [rc:0]
VM 100 - Housekeeping: pve04 data2/vm-99100-disk-1-data, keeping Snapshots for 0s
VM 100 - Removing Snapshot pve04 data2/vm-99100-disk-1-data@mirror-20221101192032 (115s) [rc:0]
VM 100 - Disk Summary: Took 1 Seconds to transfer 372.96 KiB in a incremental run
VM 99100 - Unlocking destination VM 99100
Finnished mirror 2022-11-01 19:22:30
Job Summary: Bytes transferd 2 bytes for 2 Disks on 1 VMs in 00 hours 00 minutes 46 seconds
VM Freeze OK/failed...: 1/0
RBD Snapshot OK/failed: 2/0
Full xmitted..........: 0 byte
Differential Bytes ...: 372.96 KiB

```

This example creates a mirror of VM 100 (in the source cluster) as VM 10100 (in the destination cluster) using the ceph pool "data2" for storing all attached disks. It will keep 4 Ceph snapshots prior the latest (in total 5) and 8 snapshots on the remote cluster. It will keep the VM on the target Cluster locked to avoid an accidental start (thus causing split brain issues), and will do it even if the source VM is running. 

The use case is that you might want to keep a cold-standby copy of a certain VM on another Cluster. If you need to start it on the target cluster you just have to unlock it with `qm unlock VMID` there.

Another usecase could be that you want to migrate a VM from one cluster to another with the least downtime possible. Real live migration that you are used to inside one cluster is hard to achive cross-cluster, but you can easily make an initial migration while the VM is still running on the source cluster (fully transferring the block devices), shut it down on source, run the mirror process again (which is much faster now because it only needs to transfer the diff since the initial snapshot) and start it up on the target cluster. This way the migration basically takes one boot plus a few seconds for transferring the incremental snapshot.

## Near-live Migration 

To minimize downtime and achive a near-live Migration from one Cluster to another it's recommended to do an initial Sync of a VM from the source to the destination cluster. After that, run the job again, and add the --migrate switch. This causes the source VM to be shut down prior snapshot + transfer, and be restarted on the destination cluster as soon as the incremental transfer is complete. Using --migrate will always try to start the VM on the destination cluster.

Example 2: Near-live migrate VM from one cluster to another (Run initial replication first, which works online, then run with --migrate to shutdown on source, incrematally copy and start on destination):

```
root@pve01:~/crossover# ./crossover mirror --jobname=migrate --vmid=100 --destination=pve04 --pool=data2 --online
ACTION: Onlinemirror
Start mirror 2023-04-26 15:02:24
VM 100 - Starting mirror for testubuntu
VM 100 - Checking for VM 100 on destination cluster pve04 /etc/pve/nodes/*/qemu-server
VM 100 - Transmitting Config for to destination pve04 VMID 100
VM 100 - locked 100 [rc:0] on source
VM 100 - locked 100 [rc:0] on destination
VM 100 - Creating snapshot data/vm-100-disk-0@mirror-20230426150224
VM 100 - Creating snapshot data/vm-100-disk-1@mirror-20230426150224
VM 100 - unlocked source VM 100 [rc:0]
VM 100 - F data/vm-100-disk-0@mirror-20230426150224: e:0:09:20 r:            c:[36.6MiB/s] a:[36.6MiB/s] 20.0GiB [===============================>] 100%
VM 100 - created snapshot on 100 [rc:0]
VM 100 - Disk Summary: Took 560 Seconds to transfer 20.00 GiB in a full run
VM 100 - F data/vm-100-disk-1@mirror-20230426150224: e:0:00:40 r:            c:[50.7MiB/s] a:[50.7MiB/s] 2.00GiB [===============================>] 100%
VM 100 - created snapshot on 100 [rc:0]
VM 100 - Disk Summary: Took 40 Seconds to transfer 22.00 GiB in a full run
VM 100 - Unlocking destination VM 100
Finnished mirror 2023-04-26 15:13:47
Job Summary: Bytes transferred 22.00 GiB for 2 Disks on 1 VMs in 00 hours 11 minutes 23 seconds
VM Freeze OK/failed.......: 1/0
RBD Snapshot OK/failed....: 2/0
RBD export-full OK/failed.: 2/0
RBD export-diff OK/failed.: 0/0
Full xmitted..............: 22.00 GiB
Differential Bytes .......: 0 Bytes

root@pve01:~/crossover# ./crossover mirror --jobname=migrate --vmid=100 --destination=pve04 --pool=data2 --online --migrate
ACTION: Onlinemirror
Start mirror 2023-04-26 15:22:35
VM 100 - Starting mirror for testubuntu
VM 100 - Checking for VM 100 on destination cluster pve04 /etc/pve/nodes/*/qemu-server
VM 100 - Migration requested, shutting down VM on pve01
VM 100 - locked 100 [rc:0] on source
VM 100 - locked 100 [rc:0] on destination
VM 100 - Creating snapshot data/vm-100-disk-0@mirror-20230426152235
VM 100 - Creating snapshot data/vm-100-disk-1@mirror-20230426152235
VM 100 - I data/vm-100-disk-0@mirror-20230426152235: e:0:00:03 c:[1.29MiB/s] a:[1.29MiB/s] 4.38MiB
VM 100 - Housekeeping: localhost data/vm-100-disk-0, keeping Snapshots for 0s
VM 100 - Removing Snapshot localhost data/vm-100-disk-0@mirror-20230323162532 (2930293s) [rc:0]
VM 100 - Removing Snapshot localhost data/vm-100-disk-0@mirror-20230426144911 (2076s) [rc:0]
VM 100 - Removing Snapshot localhost data/vm-100-disk-0@mirror-20230426145632 (1637s) [rc:0]
VM 100 - Removing Snapshot localhost data/vm-100-disk-0@mirror-20230426145859 (1492s) [rc:0]
VM 100 - Removing Snapshot localhost data/vm-100-disk-0@mirror-20230426150224 (1290s) [rc:0]
VM 100 - Housekeeping: pve04 data2/vm-100-disk-0-data, keeping Snapshots for 0s
VM 100 - Removing Snapshot pve04 data2/vm-100-disk-0-data@mirror-20230426150224 (1293s) [rc:0]
VM 100 - Disk Summary: Took 4 Seconds to transfer 4.37 MiB in a incremental run
VM 100 - I data/vm-100-disk-1@mirror-20230426152235: e:0:00:00 c:[ 227 B/s] a:[ 227 B/s] 74.0 B
VM 100 - Housekeeping: localhost data/vm-100-disk-1, keeping Snapshots for 0s
VM 100 - Removing Snapshot localhost data/vm-100-disk-1@mirror-20230323162532 (2930315s) [rc:0]
VM 100 - Removing Snapshot localhost data/vm-100-disk-1@mirror-20230426144911 (2098s) [rc:0]
VM 100 - Removing Snapshot localhost data/vm-100-disk-1@mirror-20230426145632 (1659s) [rc:0]
VM 100 - Removing Snapshot localhost data/vm-100-disk-1@mirror-20230426145859 (1513s) [rc:0]
VM 100 - Removing Snapshot localhost data/vm-100-disk-1@mirror-20230426150224 (1310s) [rc:0]
VM 100 - Housekeeping: pve04 data2/vm-100-disk-1-data, keeping Snapshots for 0s
VM 100 - Removing Snapshot pve04 data2/vm-100-disk-1-data@mirror-20230426150224 (1313s) [rc:0]
VM 100 - Disk Summary: Took 2 Seconds to transfer 4.37 MiB in a incremental run
VM 100 - Unlocking destination VM 100
VM 100 - Starting VM on pve01
Finnished mirror 2023-04-26 15:24:25
Job Summary: Bytes transferred 4.37 MiB for 2 Disks on 1 VMs in 00 hours 01 minutes 50 seconds
VM Freeze OK/failed.......: 0/0
RBD Snapshot OK/failed....: 2/0
RBD export-full OK/failed.: 0/0
RBD export-diff OK/failed.: 2/0
Full xmitted..............: 0 Bytes
Differential Bytes .......: 4.37 MiB
```

## Things to check

From Proxmox VE Hosts you want to backup you need to be able to ssh passwordless to all other Cluster hosts, that may hold VM's or Containers. This goes for the source and for the destination Cluster. Doublecheck this.

This is required for using the free/unfreeze and the lock/unlock function, which has to be called locally from that Host the guest is currently running on. Usually this works out of the box for the source cluster, but you may want to make sure that you can "ssh root@pvehost1...n" from every host to every other host in the cluster.

For the Destination Cluster you need to copy your ssh-key to the first host in the cluster, and login once to every node in your cluster.

Currently preflight checks don't include the check for enough resources in the destination cluster. Check beforehand that you don't exceed the maximum safe size of ceph in the destination cluster.

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

