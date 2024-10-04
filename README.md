# Slurm Documentation for SCI

These are the docs that compiled for SCI so that the Slurm install is documented. This readme serves as a one stop shop for all things Slurm at SCI.
You can find Slurms main docs here: https://slurm.schedmd.com/documentation.html This is where you can find the nitty gritty about Slurm

#### Purpose
The Wormulon Cluster is meant for testing the functionality and viability of a compute cluster administered by SCI. It consists of a controller node (wormulonctl) and 5 worker nodes (wormulon1-4, dev1). The controller node handles the database, login and delegation of the work to the worker nodes. The worker nodes have no login except for SCI-IT to administer and maintain.

# Install Instructions

## Building the Slurm .deb files

**Note, There will usually be prebuilt .deb files if just adding a compute node so you may not need to run these steps. Please ask before creating new .deb files, thanks :)**

To build the slurm .deb files there are a few prerequisites that you need to take into account for it to work as intended.
For NVIDIA GPU support these need to be built on a GPU system. You can use a gpu enabled Slurm package on non gpu systems and it still works.

Make sure to install all the nvidia packages first (nvidia drivers, nvidia-cuda-toolkit, cuda-toolkit, ect...)
Packages need to be present in order for slurm to build support for them. So we install the needed packages now before we build.

    sudo apt install munge libmunge2 libmunge-dev build-essential fakeroot devscripts git gcc make ruby ruby-dev libpam0g-dev libmysqlclient-dev mariadb-server libssl-dev hwloc equivs

This will build the most generic Slurm install .deb. It includes functionality that we may want in the future.

Now that we have that all installed go ahead and make a temp dir on the machine you are building from.

    sudo mkdir /var/tmp/slurmbuild

    cd /var/tmp/slurmbuild

**NOTE: Current version of Slurm SCI is using is 24.05.1, if it needs to be updated you need to build new pakages above.**

In the slurmbuild directory run `wget https://download.schedmd.com/slurm/slurm-24.05.1.tar.bz2` **Note version and download appropriately**

Unzip the Slurm source `tar -xaf slurm*tar.bz2`

Cd into the source directory

Install the Slurm package dependencies: `mk-build-deps -i debian/control`

Build the Slurm packages with: `debuild -b -uc -us`

Packages will end up in the slurmbuilds directory.

Packages we will need from this folder:

    slurm-smd-client_24.05.1-1_amd64.deb
    slurm-smd-slurmctld_24.05.1-1_amd64.deb
    slurm-smd-slurmd_24.05.1-1_amd64.deb
    slurm-smd-slurmdbd_24.05.1-1_amd64.deb
    slurm-smd-libpam-slurm-adopt_24.05.1-1_amd64.deb

The other packages there are for other compute things such as MPI and the rest api (which we are not using)

##### END OF PACKAGE BUILD SECTION

# Building the cluster

**Important Note: Make sure to name your machines appropriately as doing so now makes it a lot easier later. Keep it simple. Do like 2 letters then a number for the node. This will make the config part MUCH easier later**

## Controller Node Setup

We will start with the controller node as it is the most important node in the cluster it controls the configs and delegates work to the compute nodes.

### Create Munge and Slurm Users

**Verify the GID and UID for these users as they need to be the same on ALL systems**

    export MUNGEUSER=601
    groupadd -g $MUNGEUSER munge
    useradd  -m -c "MUNGE Auth" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge

    export SLURMUSER=600
    groupadd -g $SLURMUSER slurm
    useradd  -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm

### Install Munge packages

    sudo apt install munge libmunge2 libmunge-dev

Set the correct permissions for the Munge user

    sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
    sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
    sudo chmod 0755 /run/munge/
    sudo chmod 0700 /etc/munge/munge.key
    sudo chown -R munge: /etc/munge/munge.key

Take the munge.key and store it someplace where you can easily copy it to other systems but where it is also secure (no user access)

Install the slurm packages needed for operation

    sudo apt install ./slurm-smd_24.05.1-1_amd64.deb
    sudo apt install ./slurm-smd-client_24.05.1-1_amd64.deb
    sudo apt install ./slurm-smd-slurmctld_24.05.1-1_amd64.deb

Head node FS perms (ensure that slurm has needed FS perms)

    chown root:root /var/lib/slurm
    chmod -R 755 /var/lib/slurm

Locate the config files in /sci-it/hosts/wormulonctl/slurmdocs folder.
You will need the cgroup.conf, gres.conf, slurm.conf (slurmdbd.conf if dbd is on controller)

They will be placed in the /etc/slurm/ folder on the controller. The current way it is setup is to allow only the controller to have the configs and push them out to the nodes so the nodes are configless. It makes editing the configs and pushing updates to them REALLY easy.
The SRV record for the wormulon cluster is: _slurmctld._tcp.sci.utah.edu 3600 IN SRV 0 0 6817 "hostname of controller node"

Make sure to edit the slurm.conf with the nodes you will be using and the partitions that you need as well.

**Enable and start the slurmctld daemon**

    systemctl enable slurmctld
    systemctl restart slurmctld

From there the slurm controller should be setup and ready to get rolling.

##### END OF CONTROLLER SETUP SECTION

## Worker/Compute Node Setup

### Create Munge and Slurm Users

**Verify the GID and UID for these users as they need to be the same on ALL systems**

    export MUNGEUSER=601
    groupadd -g $MUNGEUSER munge
    useradd  -m -c "MUNGE Auth" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge

    export SLURMUSER=600
    groupadd -g $SLURMUSER slurm
    useradd  -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm

### Install Munge packages

    sudo apt install munge libmunge2 libmunge-dev

Set the correct permissions for the Munge user

    sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
    sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
    sudo chmod 0755 /run/munge/
    sudo chmod 0700 /etc/munge/munge.key
    sudo chown -R munge: /etc/munge/munge.key

Run the following again after you copy munge.key into /etc/munge/ on the compute node

    sudo chmod 0700 /etc/munge/munge.key
    sudo chown -R munge: /etc/munge/munge.key

**RESTART MUNGE AFTER KEY CHANGE**

    systemctl enable munge
    systemctl restart munge

Install the slurm packages needed for operation

    sudo apt install ./slurm-smd_24.05.1-1_amd64.deb
    sudo apt install ./slurm-smd-client_24.05.1-1_amd64.deb
    sudo apt install ./slurm-smd-slurmd_24.05.1-1_amd64.deb

Node FS Perms (Slurm will not function if this is not set)

    chown root:root /var/lib/slurm
    chmod -R 755 /var/lib/slurm
    mkdir /var/log/slurm/
    touch /var/log/slurm/slurmd.log
    chown -R slurm:slurm /var/log/slurm/slurmd.log

**Enable and start slurmd**

    systemctl enable slurmd
    systemctl restart slurmd

The worker node should be all setup now and ready to talk to the controller.

#### Repeat for all worker/compute nodes you will need.

##### End of Compute node setup section

## Database Node Setup

### Create Munge and Slurm Users

**Verify the GID and UID for these users as they need to be the same on ALL systems**

    export MUNGEUSER=601
    groupadd -g $MUNGEUSER munge
    useradd  -m -c "MUNGE Auth" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge

    export SLURMUSER=600
    groupadd -g $SLURMUSER slurm
    useradd  -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm

### Install Munge packages

    sudo apt install munge libmunge2 libmunge-dev

Set the correct permissions for the Munge user

    sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
    sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
    sudo chmod 0755 /run/munge/
    sudo chmod 0700 /etc/munge/munge.key
    sudo chown -R munge: /etc/munge/munge.key


Install the slurm packages needed for operation

    sudo apt install ./slurm-smd_24.05.1-1_amd64.deb
    sudo apt install ./slurm-smd-slurmdbd_24.05.1-1_amd64.deb

Install mariadb-server

    sudo apt-get install mariadb-server -y

Grant slurm user full rights to the slurm_acct_db database

    mysql -u root -p
    mysql> grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by 'slurmtestdbpass' with grant option;
    mysql> create database slurm_acct_db;

Create / edit slurm DB conf file under /etc/slurm/slurmdbd.conf (**Needs to be local to dbd server**)

        # Here is an example slurmdbd.conf:
        LogFile=/var/log/slurm/slurmdbd.log
        DbdHost=localhost    # Replace by the slurmdbd server hostname (for example, slurmdbd.my.domain)
        DbdPort=6819    # The default value
        SlurmUser=slurm
        StorageHost=localhost
        StoragePass=slurmtestdbpass    # The above defined database password, change it for your site!
        StorageLoc=slurm_acct_db
        StorageType=accounting_storage/mysql

        PurgeEventAfter=12months
        PurgeJobAfter=12months
        PurgeResvAfter=2months
        PurgeStepAfter=2months
        PurgeSuspendAfter=1month
        PurgeTXNAfter=12months
        PurgeUsageAfter=12months

Slurm uses Munge sockets for secure communications.

    # Relevant section from slurm.conf
    # LOGGING AND ACCOUNTING
    AccountingStorageEnforce=limits
    AccountingStorageTRES=gres/gpu,gres/shard
    AccountingStorageHost=wormulonctl
    AccountingStoragePass="/var/run/munge/munge.socket.2="  **ATTENTION TO THIS**
    #AccountingStoragePort=
    AccountingStorageType=accounting_storage/slurmdbd
    AccountingStorageUser=slurm

Start mariadb and slurm dbd

    systemctl start mariadb
    systemctl enable mariadb
    systemctl enable slurmdbd
    systemctl start slurmdbd

##### End of Database Node Section

## Account and cluster accounting

### Initialize the cluster in slurm accounting

    sacctmgr create cluster <Cluster>       EX. <cluster>=wormulon

*sacctmgr* is the main account managing command

#### Default QOS's/Accounts (Everyone will use)
Main for shared resources QOS

    sacctmgr create qos name=<default shared> priority=100000 maxwall=3-00:00:00

Main for Guest on owner nodes

    sacctmgr create account name=owner-guest org=<sci> description=<something>
    sacctmgr create qos name=<cluster-guest> priority=10000 grpwall=3-00:00:00 flags=NoReserve

Associate owner-guest QOS to Account

    sacctmgr modify account name=owner-guest set qos=<cluster-guest>
    sacctmgr modify account name=owner-guest set defaultqos=<cluster-guest>

#### Create Accounts (Not users)

PI Accounts for shared

    sacctmgr create account name=<PI> org=<sci> description=<Something here>

Account for pi on owner machines

    sacctmgr create account name=<PI-Cluster> org=<sci> description=<PI Owner Account> 

EX. for cluster: wormulon (so it would be PI-wormulon, change *Cluster* to fit needs)

QOS for Owner Accounts

        sacctmgr create qos name=<PI-cluster> priority=100000 maxwall=7-00:00:00 preempt=owner-guest 

Associate QOS to PI Accounts

    sacctmgr modify account name=<PI> set qos=<default shared>
    sacctmgr modify account name=<PI> set defaultqos=<default shared>

Associate QOS to PI Owner Accounts

    sacctmgr modify account name=<PI-cluster> set qos=<PI-cluster>
    sacctmgr modify account name=<PI-cluster> set defaultqos=<PI-cluster>


#### Create users

Create user with default pi account (SHOULD INHERIT QOS FROM ACCOUNTS IF DONE PROPERLY)

    sacctmgr create user name=<USER> account=<PI>

Add users to additional Accounts

    sacctmgr add user <USER> account=owner-guest
    sacctmgr add user <USER> account=<PI-Cluster>


#### Restricting Accounts to certain partitions (useful for owner boxes)
See Slurm.conf for in depth config of partitions and nodes

    # Partitions
    PartitionName=wormulon Nodes=wormulon[1-3] Default=YES OverSubscribe=FORCE:4 State=UP
    PartitionName=wormulon-guest Nodes=wormulon[4],dev[1] AllowAccounts=owner-guest DefMemPerCPU=5120 OverSubscribe=FORCE:4 State=UP
    PartitionName=bigler-worm Nodes=wormulon[4],dev[1] AllowAccounts=bigler-worm DefMemPerCPU=5120 OverSubscribe=FORCE:4 State=UP

    # Note the AllowAccounts, add accounts allowed to use this partition here. Note how the general wormulon partition has none, this is so everyone can use it.
    # Also vist the slurm docs for more in depth explanations of things.

##### End of Accounts Section

## Misc Information

### Config Files and their purpose

slurm.conf: Is the main config file that runs all of how slurm acts and how nodes and partitions are handeled. 
(slurm.conf addendum: Here is a config I found online for the kingspeak cluster. It is old so take it with a grain of salt. https://home.chpc.utah.edu/~u0104663/SchedMD/kp.slurm.conf-10-02-17)
slurmdbd.conf: Is the database specific config file that controls how the database handles storage and length of storage.
gres.conf: Is the GPU specific config. By building slurm with Nvidia support we get access to GPU autodetect, but you can also specify GPU's by node here for direct control.
cgroup.conf: Controls jobs and prevents them from getting more than they asked for. Kills jobs that go over their requested memory allocation.

### Why use source-built .deb files vs APT installing slurm.

Well when I was planning to do this initially, I noticed the version of slurm that was being offered was lower then the version on the slurm website. I was fine with this as it meant easier installation. But upon furthur investigation, these apt packages did not provide the support needed to function with Nvidia GPUS. 

This meant that I was able to build from source (not without its struggles) and get a working .deb file for Slurm to function correctly. Another thing that I learned is that when you install from handbuilt source, you need to install .deb files with APT otherwise you end up with dependancies that DPKG cant install. APT can install these depends so it just works. 