+++
title = 'Mysql Replication Guide With Mysqldump'
date = 2023-09-02T21:26:59-07:00
draft = false
+++

My step by step guide for setting up a replication server for MySql 5.6 and up (with GTID enabled) using mysqldump.

1. Create replication user in MySql, if not already done.

`mysql> CREATE USER 'repl'@'192.168.0.%' IDENTIFIED BY 'repl_password';`
 
2. Configure the production MySql server to accept connections from the replica server, if not already done.

`mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT, SUPER ON *.* TO 'repl'@'192.168.0.%' IDENTIFIED BY 'some_password';`
 
3. Backup leader server data to local directory.

`$ mysqldump --all-databases --hex-blob --single-transaction --add-drop-database --triggers --routines --events --user=root --password > snapshot.sql`

**Notes**

`--hex-blob`: Dump binary columns using hexadecimal notation (for example, ‘abc’ becomes 0x616263). The affected data types are BINARY, VARBINARY, the BLOB types, and BIT.

`--single-transaction`: Useful for exporting InnoDB tables since it does not block transactions during the export.
 

**If GTIDs are disabled**

If not using GTIDs, use either the `--master-data=2` option to create a replica for the server you are creating this mysqldump for or `--dump-slave=2` if mysqldump is being run on a replica and you plan on having a new replica use the same leader. The "2" option writes the `CHANGE MASTER` command in the dump file as a comment while the "1" option runs the command when the dump file is imported.

1. Copy backup to remote slave server.

`$ scp -P 25 -i ~/.ssh/yourkey snapshot.sql david@remoteserver:/home/david`

**OR**

`$ rsync -avzhe ssh –-progress snapshot.sql david@remoteserver:/home/david`
 
2. Make sure to modify or replace my.cnf before initiating restore.

`$ vim /etc/mysql/my.cnf`

3. Initiate restore from sql file.

`$ mysql -u root -p < snapshot.sql`

4. Assuming GTID is enabled on the leader, reset the leader.

`replica> RESET MASTER;`

5. Get `GTID_PURGED` value from the sql file.

`$ grep 'GTID_PURGED' -m 1 snapshot.sql`

6. Set `GTID_PURGED` in MySql prompt.

`replica> SET GLOBAL gtid_purged="c777888a-b6df-11e2-a604-080027635ef5:1-4";`

7. Run the `CHANGE MASTER` command to associate the replica with the leader then start the replica.

**Change master with SSL enabled**

`replica> CHANGE MASTER TO MASTER_HOST='host', MASTER_PORT='port', MASTER_USER='repl', MASTER_PASSWORD='repl_password', MASTER_AUTO_POSITION=1, MASTER_SSL=1, MASTER_SSL_CA='/var/lib/mysql/ca.pem', MASTER_SSL_CAPATH='/var/lib/mysql', MASTER_SSL_CERT='/var/lib/mysql/client-cert.pem', MASTER_SSL_KEY='/var/lib/mysql/client-key.pem';`

**Change master without SSL**

`replica> CHANGE MASTER TO MASTER_HOST='host', MASTER_PORT='port', MASTER_USER='repl', MASTER_PASSWORD='repl_password', MASTER_AUTO_POSITION=1;`

8. Start the replica.

`replica> START SLAVE;`
 

## Promoting a replica to leader

Execute on the replica to be made into leader
```
mysql> STOP SLAVE;
mysql> RESET SLAVE ALL;
```

To reset `gtid_executed` and prevent multiple `gtid_executed` values.

`mysql> RESET MASTER;`