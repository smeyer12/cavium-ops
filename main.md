Cavium ThunderX Operations Manual
=================================

Cluster Overview
----------------

### Hardware

20 compute nodes (Gigabyte [model number]):
* 2x 48 core Cavium ThunderX processors (96 cores)
* 500 GiB RAM
* 12x 8 TB Hard drives
* 40 GbE network connection

3 head nodes: [TODO: fill in details]

1 login node: [TODO: fill in details]

Further, each compute node has its first hard drive (at `/dev/sda`) partitioned
in the following manner: [TODO: fill in details]

Head nodes partition info: [TODO: fill in details]

Each of the remaining drives in the compute nodes have one xfs formatted
partition completely dedicated to HDFS.  Additionally, each compute node is
configured to yield 90 cores and 450 GiB of memory to YARN for computation.
This gives the cluster the following specifiations:

*Cores*: 1800
*Memory*: 8.79 TiB
*HDFS*: ~1.7 PB

### Software

* Distributed storage: HDFS
* Resource manager: YARN
* Client applications
    * MapReduce
    * Spark (PySpark, SparkR)
    * Hive (Hive-on-MapReduce)

The software stack is build using [Apache Bigtop](http://bigtop.apache.org/).
We have our HDFS configured to use HA with two NameNodes (active and standby).
As such, all three head nodes run the following:

* ZooKeeper server daemon
* Hadoop JournalNode daemon

Of the three head nodes, two are designated as Hadoop NameNodes, and each of
those runs the following:

* Hadoop NameNode daemon
* ZooKeeper failover controller daemon (ZKFC)

The third head node is designated as our resource manager server and runs the
following:

* YARN ResourceManager daemon

It is also designated to be our "true" head node.  This entails two things:

* It is considered the source of truth for user management purposes (i.e. the
  `/etc/passwd` file is the master copy)
* It can write to all Cavium-specific mounts as `root`

All of the compute nodes run the following:

* Hadoop DataNode daemon
* YARN NodeManager daemon

All software installation and configuration is done via
[Ansible](https://www.ansible.com) and is version controlled.

Daily Tasks
-----------

Common Tasks
------------

Common tasks often include configuration changes via Ansible.  All steps will
assume that you have cloned the ARC-TS Ansible repo to a local space.  If you
have not done this, you can do so via the following command on your local
machine:  
``` 
$ git clone git@bitbucket.org:umarcts/ansible.git
```

All paths assume that your current working directory is `ansible` (i.e. your
locally cloned Ansible repo).

Lastly, it is suggested that all pushses of configuration changes be as yourself
using sudo from `/etc/ansible` on flux-admin09.

### Add users

To add a new user to the cluster, `ssh` to cavium-rm01 (the designated head
node) and run the following script:  
```
$ /usr/arcts/systems/scripts/create-cavium-hadoop-user.sh -u $uniqname -g $grp
```

For the `$grp`, I (pattonmj) usually try to add them to the group based on their
school.  If they have a Flux account, you can get the primary group from there.
If all else fails, add them to the `workshop` group.

Once this is done, add the user to the [group] MCommunity group so that they
can get emails. 

### Add mounts

If a user requests to have a MiStorage, Turbo, or Locker mount added to the
cluster:

1. Create a hotfix branch named after the request.  If the request came in
   ticket, include the ticket number in the branch name.  
```
$ git checkout -b hotfix/cavium-autofs-fp12345
```

2. Edit `group_vars/caviumHadoop/autofs`.  Use the current entries to determine
   how the file is formatted and which mount options to use.
    a. If the mount is MiStorage, add it to the `auto.nfs` content section.
    b. If the mount is Turbo, add it to the `auto.nfs.turbo` content section.
    c. If the mount is Locker, add it to the `auto.nfs.locker` content section.

3. Commit the changes.

4. Push the hotfix branch to origin.  
```
$ git push origin hotfix/cavium-autofs-fp12345
```

5. Create a pull request (PR) on bitbucket.org.

6. Once the PR has been approved, merged, and pulled to flux-admin09, push the
   change to the cluster.
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop -t 'autofsMountUpdate' utilityPlaybook/autofs.yml
```

### Python libraries

If a user requires a Python library, our current policy is to provide them
instructions to install it into their home directory.  

### Add new queues

YARN queues are defined in (and distributed via) Ansible.  Currently, there are
queues for each major research group that we support.  You can find the queue
definitions in `dsi-prod/group_vars/caviumHadoop/queues`  There is actually only
a single dictionary defined there calles `queues`.  The keys of the dictionary
are the names of the queues, and the values are the queues' configurations.
Using the `default` queue as an example, we have:

```
queues:
  default:
    capacity: 5
    max_capacity: 5
    state: "RUNNING"
    users:
      - "*"
    admin_users:
      - pattonmj
      - smeyer
```

The following are the only values that are _required_:

* `capacity`: expressed as a percentage of the cluster (float)
* `state`: should be either `"RUNNING"` or `"STOPPED"`, including the quotes
  (string)
* `users`: list of uniqnames in YAML list syntax, even if there is only one user
  (list)
* `admin_users`: list of uniqnames in YAML list syntax, even if there is only
  one user (list)

Other configuration values currently supported are:

* `max_capacity`: for queue elasticity; allows a queue to use more than its
  configured capacity.  Expressed as a percentage of the cluster (float).
* `user_limit_factor`: how much of the queue a single user can consume,
  expressed as a multiplier (float)
* `subqueues`: list of queues in YAML list syntax that split the current queue's
  resources, even if there is only one (list)

Note that subqueues need to be defined in the same way that regular queues are.
The only difference is that their names should be formatted as such:
`$parent_queue.$subqueue`.  For example, if we wanted to add a subqueue called
`special` to the `default` queue, we would define it under `queues` in this way:  

```
queues:
.
.
.
  default.special:
    capacity: 20
    state: "RUNNING"
    users:
      - pattonmj
    admin_users:
      - pattonmj
      - smeyer
```

Finally, it is important to note that all of the `capacity` values for queues
*_at the same level_* should add to 100 (i.e. 100%).  The main consequence of
this is that, when adding a new queue, the capacities of all other queues *_at
the same level_* need to be recalculated.  If this is not done, refreshing the
queue configuration will fail.

Please see the Apache documentation for the [YARN Capacity
Scheduler](https://hadoop.apache.org/docs/r2.8.4/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html)
if more detail is necessary.

The full steps to add a new queue follow:

1. Edit `dsi-prod/group_vars/caviumHadoop/queues` using the configuration
   guidelines described above.

2. See steps 3-5 of the 'Add mounts' section.

3. Once the PR has been approved, merged, and pulled to flux-admin09, push the
   change to the cluster.  
```
$ sudo ansible-playbook -i dsi-prod/ -l 'caviumHadoop' -t 'update-queues' hadoopUtilityPlaybooks/yarn-queues.yml
```

### Add users to queues

Everyone with a Cavium ThunderX account has access to the default queue.  If a
user needs to be added to a certain YARN queue:

1. Edit `dsi-prod/group_vars/caviumHadoop/queues`.  Add the user's uniqname to
   the appropriate queue under the queue's `users` key.

2. See steps 3-5 of the 'Add mounts' section.

3. Once the PR has been approved, merged, and pulled to flux-admin09, push the
   change to the cluster.  
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop -t 'update-queues' hadoopUtilityPlaybooks/yarn-queues.yml
```

### Decommission nodes

The node exclusion/inclusion lists for Cavium are shared between HDFS and YARN.
The files found on the ResourceManager and NameNodes are symlinks to the actual
files in NFS.  The symlinks are found at
`/etc/hadoop/conf/nodes.{include,exclude}`.  The actual files can be found at
`/usr/arcts/systems/cavium-hadoop/{dev,prod}/etc/nodes.{include,exclude}`.  For
clarity and simplicity, we will assume that we are only
commissioning/decommissioning production nodes.  For dev nodes, just replace the
relevant part of the file path (i.e. replace `prod` with `dev`).  

To decommission nodes:

1. Add the node that you want to decom to the `nodes.exclude` file, **but leave
   it in the `nodes.include` file**.

2. Refresh the node lists for the service that you are removing the node from.  
```
$ hdfs dfsadmin -refreshNodes  # on cavium-nn01 or cavium-nn02 for HDFS

OR

$ yarn rmadmin -refreshNodes  # on cavium-rm01 for YARN
```

You can run both commands if you are removing the node from both services.
After this is done, you will need to wait for HDFS to rebalance the cluster.
That is, HDFS will move the blocks on the decommissioned node to other
nodes.  You can monitor this in the HDFS UI.  

For [cavium-nn01:](http://cavium-nn01.arc-ts.umich.edu)

For [cavium-nn02:](http://cavium-nn02.arc-ts.umich.edu)

Either will work, but a good improvement would be to automate this (at least
find a way to send an alert when the node's data has been redistributed).  Once
the cluster is rebalanced, you can remove the node from the `nodes.include`
file and re-run the commands to refresh nodes.

As a side note, the node exclusion/inclusion lists are in git control.  It is
only a local repo, it has been used as record keeping mechanism.  In the git log
you will find which nodes have been decommissioned and why.  I suggest keeping
this going.

Maintenance
-----------

During maintenance, one of the most common tasks we perform is to update the OS.
This can readily be done using Ansible.  There are a number of steps involved,
however, some of which should be done prior to the maintenance window.  

### Use `reposync` to get OS updates (pre-maintenace)

The first task should be to pull down the newest packages.  Using CentOS, only
the `updates` repo needs to be pulled.  The following instructions assume you
are on flux-admin09 as root.

1. Create a directory for the version that you are pulling.  
```
$ mkdir /usr/arcts/repos/dist/CentOS/$arch/$version
```

For the Cavium ThunderX cluster, `$arch` will always be `aarch64`.  (It could be
`x86_64` if working on other systems.)

The ARC-TS convention for versioning is to use the OS major and minor versions,
but then the date of when you are pulling the repo.  Thus the format for a
version is `$major.$minor.$yymm`.  For example, if you are just updating CentOS
7.6 but you are doing it in August 2019, your `$version` would be `7.6.1908`.

2. Sync the `updates` repo.  
```
$ reposync -t -a aarch64 --repoid=updates \
  -c /usr/arcts/repos/reposyncFiles/CentOS-7_aarch64/local_yum.conf \
  -p /usr/arcts/repos/dist/CentOS/aarch64/$version/
```

It is also possible to use the `-n` flag of `reposync` to get only the newest
packages.

3. Create symlinks inside of the new `$version` directory to point to older
   repos.  
```
$ ln -s /usr/arcts/repos/dist/CentOS/aarch64/$prev_version/os/ \
  /usr/arcts/repos/dist/CentOS/aarch64/$version/os

$ ln -s /usr/arcts/repos/dist/CentOS/aarch64/$prev_version/extras/ \
  /usr/arcts/repos/dist/CentOS/aarch64/$version/extras
```

Here, `$prev_version` is the next newest version that is available.  For
example, if `$version` is `7.6.1908`, `$prev_version` _might_ be `7.6.1901` if
the last update was performed in January 2019.  It is necessary to create these
symlinks because Ansible will deploy repo files based on what you use as
`$version`.  If those links are not present (which they won't be if you only
sync the updates repo), `yum` transactions will fail because they can't find the
specified repos.

### Update the version variable in Ansible (pre-maintenance)

The following assumes that you have cloned the ARC-TS Ansible repo to
`~/ansible` and it is your current working directory.  

Create a new hotfix branch as normal.  Change the `centosVersion` variable found
in the file `dsi-prod/group_vars/caviumHadoop/Global`.  It should be updated to
match your `$version` from the previous section.  Commit the change and submit
the PR as normal.  

With the PR approved, you can roll out the change with the following command
(on fa09, preferrably as yourself using `sudo`):  
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop utilityPlaybook/yum_repo.yml
```

### Patching (during maintenance)

To patch the OS, use the following instructions (assuming you are on fa09,
preferrably as yourself using `sudo`):  

1. Shutdown all Hadoop services.  There is a playbook for this.  
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop hadoopUtilityPlaybooks/stop-hadoop.yml
```

2. Roll out updates.  
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop utilityPlaybook/runYumUpdate.yml
```

3. Reboot.  
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop utilityPlaybook/reboot.yml
```

4. When all the machines are back up, start Hadoop services.  
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop hadoopUtilityPlaybooks/start-hadoop.yml
```

The process should be done at this point.  You can perform a few manual checks
(like `hdfs dfs -ls` as yourself and starting a pyspark job).  This would also
be the point where you would run QA, if desired.

Winter 2019 Maintenance Changes
-------------------------------

* Update to CentOS 7.6
* Kerberization completed
* Increased available resources for compute nodes
    * Increased cores available to YARN from 80 to 90
    * Increased memory available to YARN from 400 GiB to 450 GiB

### Operational Changes

We must now obtain Kerberos tickets to run administrative `hdfs` and `yarn`
commands.  There are keytabs in place for this.  For hdfs (as the `hdfs` user):  
```
$ kinit -kt /etc/security/keytabs/hdfs.service.keytab hdfs-cavium1
```

For YARN (as the `yarn` user):  
```
$ kinit -kt /etc/security/keytabs/rm.service.keytab rm/cavium-rm01.arc-ts.umich.edu
```

Also, if you wish to run jobs as yourself, you have two options:  

1. Log directly into the login server from your machine.  All things should work
   as normal.

2. `ssh` to the login server as `root` from flux-admin09, then:
    a. `su` to yourself
    b. run `kinit` (with no arguments)
    c. enter your UMICH password

Summer 2019 Maintenance Changes
-------------------------------

* Update OS
    * New `centosVersion` is `7.6.1908`
