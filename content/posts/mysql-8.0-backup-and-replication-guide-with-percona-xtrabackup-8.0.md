+++
title = 'Mysql 8.0 Backup and Replication Guide With Percona Xtrabackup 8.0'
date = 2023-09-03T00:23:19-07:00
draft = false
+++

Percona's Xtrabackup utility contains a few major changes in MySql 8.0 the most notable of which is the replacement of the innobackupex command with xtrabackup. Below is my no-explanation (reference the flags in Percona's docs) step-by-step guide for setting up replication for MySql updated for version 8.0.

1. Create replication user in MySql, if one does not exist.

`mysql> CREATE USER 'replicauser'@'192.168.%' IDENTIFIED BY 'replicauserpassword';`

2. Create backup user in MySql, if one does not exist.

`mysql> CREATE USER 'backupuser'@'localhost' IDENTIFIED BY 'backupuserpass';`

3. Configure the leader MySql server to accept connections from the replicas, if not already done.

`mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT, SUPER ON *.* TO 'replicauser'@'192.168.%';`

4. Grant backup privileges to your backup user.

`mysql> GRANT RELOAD, LOCK TABLES, BACKUP_ADMIN, REPLICATION CLIENT, CREATE TABLESPACE, PROCESS, SUPER, CREATE, INSERT, SELECT, SHOW DATABASES ON *.* TO 'backupuser'@'localhost';`

5. Backup leader server data to local directory (use `--safe-slave-backup` option if backing up from a replica).

**Create an uncompressed backup**

`$ xtrabackup --user="backupuser" --password="backupuserpass" --backup --slave-info --safe-slave-backup --parallel=2 --ftwrl-wait-timeout=60 --check-privileges --target-dir=$LOCAL_BACKUP_DIR`

**Create a compressed backup**

`$ xtrabackup --user="backupuser" --password="backupuserpass" --backup --slave-info --safe-slave-backup --parallel=2 --ftwrl-wait-timeout=60 --check-privileges --compress --compress-threads=2 --target-dir=$LOCAL_BACKUP_COMPRESSED_DIR`

6. Copy certs, if using SSL for connections to MySql.
```
$ cp /var/lib/mysql/ca-key.pem $LOCAL_BACKUP_DIR
$ cp /var/lib/mysql/ca.pem $LOCAL_BACKUP_DIR
$ cp /var/lib/mysql/client-cert.pem $LOCAL_BACKUP_DIR
$ cp /var/lib/mysql/client-key.pem $LOCAL_BACKUP_DIR
$ cp /var/lib/mysql/server-cert.pem $LOCAL_BACKUP_DIR
$ cp /var/lib/mysql/server-key.pem $LOCAL_BACKUP_DIR
```

7. Copy backup to remote replica server.

`$ scp -P 25 -i ~/.ssh/yourkey -r $LOCAL_BACKUP_COMPRESSED_DIR david@remoteserver:~`

**OR**

`$ rsync -avzhe ssh --progress $LOCAL_BACKUP_COMPRESSED_DIR david@remoteserver~`

8. Uncompress backup in remote server, if necessary.

`$ xtrabackup --decompress --remove-original --parallel=2 --target-dir=$LOCAL_BACKUP_COMPRESSED_DIR`

9. Prepare the backup.

`$ xtrabackup --prepare --use-memory=2G --target-dir=$LOCAL_BACKUP_DIR`

10. Stop MySql, rename or remove current mysql directory, make new mysql directory, copy backup to mysql directory, then change ownership of mysql directory to *mysql*.
```
$ systemctl stop mysql
$ mv /var/lib/mysql /var/lib/mysql_old
$ mkdir /var/lib/mysql
$ xtrabackup --copy-back --parallel=2 --target-dir=$LOCAL_BACKUP_DIR
$ chown -R mysql:mysql /var/lib/mysql
```

11. Make sure to modify or replace *my.cnf* before starting MySql. It is essential to set `skip-slave-start = 1` to prevent prematurely starting the replication process and causing a failure. In MySql 8.0, you're better off adding your configurations in */etc/mysql/conf.d/* and/or in */etc/mysql/mysql.conf.d/* as separate configuration files.

`$ vim /etc/mysql/my.cnf`

12. Restart MySql

`$ systemctl start mysql`

13. Set the GTID position of the replica. When the XtraBackup process was run, there should have been a file called *xtrabackup_binlog_info* in the top level of that directory which contains both binary log coordinates and GTID information (if GTID is enabled).

*EXAMPLE*

`$ cat xtrabackup_binlog_info`
```
mysql-bin.000002 1232 c777888a-b6df-11e2-a604-080027635ef5:1-4
```

Start the new replica from that GTID position.

`replica> SET GLOBAL gtid_purged="c777888a-b6df-11e2-a604-080027635ef5:1-4";`

---

**Troubleshooting**

If you get the error *GTID_PURGED can only be set when GTID_EXECUTED is empty*, run:

`replica> RESET MASTER;`

then continue on with setting `gtid_purged`.

`replica> SET GLOBAL gtid_purged="c777888a-b6df-11e2-a604-080027635ef5:1-4";`

---

14. Set the new leader for each replica.

**Change leader with SSL enabled (ensure SSL certs are updated)**

`replica> CHANGE MASTER TO MASTER_HOST='host', MASTER_PORT=3306, MASTER_USER='replicauser', MASTER_PASSWORD='replicauserpassword', MASTER_AUTO_POSITION=1, MASTER_SSL=1, MASTER_SSL_CA='/var/lib/mysql/ca.pem', MASTER_SSL_CAPATH='/var/lib/mysql', MASTER_SSL_CERT='/var/lib/mysql/client-cert.pem', MASTER_SSL_KEY='/var/lib/mysql/client-key.pem';`

**Change leader without SSL**

`replica> CHANGE MASTER TO MASTER_HOST='host', MASTER_PORT=3306, MASTER_USER='replicauser', MASTER_PASSWORD='replicauserpassword', MASTER_AUTO_POSITION=1;`

**Change leader without using MASTER_AUTO_POSITION**

When the XtraBackup process was run, there should have been a file called `xtrabackup_slave_info` generated in the top level of that directory if the `--slave-info` flag was used. Take note of the values found in that file to use in the statement below.

`$ replica> CHANGE MASTER TO MASTER_HOST='192.168.0.70', MASTER_USER='replicauser', MASTER_PASSWORD='replicauserpassword', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=24938;`

15. Start replica.

`replica> START SLAVE;`


16. Confirm that the replica is replicating data properly.

`$ replica> SHOW SLAVE STATUS \G`

Then confirm that these values are enabled in the status.
```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0
```

`Seconds_Behind_Master` is likely to be greater than 0 when first starting your replica but this should decrease over time. If your replica is lagging and is unable to catch up to leader, try setting `innodb_flush_method = O_DIRECT` and `innodb_flush_log_at_trx_commit = 2` in your replica's MySql configuration but keep `innodb_flush_log_at_trx_commit = 1` in your leader to increase ACID compliance.

## Promoting a replica to leader

Execute these on the replica to be made into the leader.
```
mysql> STOP SLAVE;
mysql> RESET SLAVE ALL;
```

To reset `gtid_executed` and prevent multiple `gtid_executed` values, run:

`mysql> RESET MASTER;`

Run `CHANGE MASTER` command in the other replicas to point to the new leader.

To prevent replication issues:

1. Set `skip-slave-start = 1` in *my.cnf* to ensure that the replication process doesn't start prematurely when the replicas are restarted.
2. Close application access to the old leader database to prevent further writes.
3. Ensure that all replicas have caught up to the leader by confirming `Seconds_Behind_Master: 0` and that `Retrieved_Gtid_Set` and `Executed_Gtid_Set` are the same across all replicas.
4. Run `CHANGE MASTER` command in the other replicas to point to the new leader.
5. Restart replication of all replicas to the new leader (`START SLAVE`) prior to bringing the old leader down and putting the application back up.

---

**Troubleshooting**

If replication fails with error code 2026, verify that your SSL certificates match across all servers.

---