---
title: "Restoring MySQL replica from source"
seoTitle: "Syncing replica and source mysql"
datePublished: Mon Mar 24 2025 18:53:50 GMT+0000 (Coordinated Universal Time)
cuid: cm8nfdtng000e08l8fo7r8qvz
slug: storing-mysql-replica-from-source
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/uTFiFYeQhlI/upload/1ead4694e4ac8282eff0627d40d391c8.jpeg
tags: mysql, relationship, relational-database, master-slave

---

In this article, I’m going to dig into, how you can restore your replica from the source. The reason can vary from huge lag to data discrepancy etc…

The overview of steps would look like:

1. Note the master LOG\_POS and LOG\_FILE
    
2. Take snapshot of data volume of the source and attach to replica
    
3. Stop the MySQL and update config on replica
    
4. RESET SLAVE CONFIG
    

1. First, we need to note down the position of master, later we’ll need it to reset the replica  
    

```bash
SHOW MASTER STATUS \G;
>>>*************************** 1. row ***************************
File: mysql-bin.1084550
Position: 240645285
Binlog_Do_DB: test,lol,hehe
Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```

You need to note down the Filename and Position.

2. After, noting down the File name and Log position we can take a snapshot of the data-volume/disk, I’m assuming you’re on cloud ecosystem. For cloud provider like DO, this step would look like:
    

```bash
# Take snapshot
doctl compute volume snapshot <vol_id> --snapshot-name <snapshot_name>

# Create Volume from that snapshot
doctl compute volume create <vol_name> --snapshot <snapshot_id> --size <size in Tb>

# Attach that volume to the replica
doctl compute volume-action attach <vol_id> <droplet_id>

# Create fs and attach block device
sudo mkdir -p /new_replica_data_dir

sudo mount /dev/sdb /mnt/new_replica_data_dir
```

3. Stop the MySQL and update config
    

```bash
systemctl stop mysql

# change data-dir  in mysql conf
datadir         = /mnt/new_replica_data_dir

# remove the auto.cnf file, this file will get regenerated on restart
rm /mnt/wsdbhr_mysql/mysql/auto.cnf

# start the DB
systemctl start mysql

# monitor error logs
tail -f /var/log/mysql/error.log
```

4. RESET SLAVE CONFIG
    

```bash
STOP SLAVE;

RESET SLAVE;

CHANGE MASTER TO MASTER_HOST='x.x.x.x', MASTER_USER='replica_user', MASTER_PASSWORD='...', MASTER_LOG_FILE='mysql-bin.1084550', MASTER_LOG_POS=240645285;

START SLAVE;

# verify if all correct and there are no IO and SQL errors
SHOW SLAVE STATUS \G;
```

After these steps, there will be some replication lag directly related to amount of time spent on whole process, but if there is no error, your replica will catchup and lag should be gone along with any data discrepancy.

Note: Some of you might need to update apparmor and fstab configs, if it was previously setup.