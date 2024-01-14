+++
title = 'Mysql 8.0 Notes'
date = 2024-01-14T15:31:46-08:00
draft = false
+++

## CONFIGURATION

- innodb_buffer_pool_size
   - Most important parameter for caching data and indexes in memory.
   - Can set as high as dataset. Otherwise, 80% of available RAM is a good starting point.

- innodb_buffer_pool_instance
   - Can divide buffer pool into separate regions to improve concurrency.
   - If buffer pool is 64 GB and instances is set to 32, then the buffer is split into 32 regions of 2 GB each.
   - If the buffer pool size is more than 16 GB, set the instances so that each region gets at least 1 GB of space.

- innodb_log_file_size
   - Redo log used to replay committed transactions in case of a database crash.
   - Set to 1 or 2 GB to start.
   - Requires a restart but no longer need to delete log files in MySql 8.

## TRANSACTIONS

- Atomicity: all statements in a transaction should succeed. Otherwise, they should all fail.
- Consistency: a transaction should affect data only in allowed ways. If you try to update an account that doesn't exist, the whole transaction should be rolled back.
- Isolation: No transaction should affect the existence of any other transaction. Suppose Bank A executes two transactions to transfer out funds, each transaction should check for funds available.
- Durability: data should persist on disk despite a system failure.
- Use START TRANSACTION; or BEGIN;
- Use SHOW WARNINGS; to check success.
- Use COMMIT; if statements are successful.
- Use ROLLBACK if not.
- 'autocommit' is ON by default. If 'SET autocommit=0;', you need to explicitly issue a COMMIT for every statement.
- DDL statements cannot be rolled back.

**Savepoints**
- BEGIN; <statements>; SAVEPOINT transfer_to_b;
- ROLLBACK TO transfer_to_b;

## BINARY LOGGING

- To enable 'binlog', set 'log_bin = /data/mysql/binlogs/server1' and 'server_id = 100' under [mysqld] in my.cnf then restart MySql. Check with SHOW VARIABLES LIKE 'log_bin%';
- The binary log contains all data and structure changes, not SELECT or SHOW statements that do not modify the data.
- The binary log is crash safe and only complete transactions are logged.
- Used in:
   - Replication: the changes are streamed to the replicas using the binary log.
   - Point-in-time-recovery: suppose you take a backup at 00:00 and the database crashed at 8:00. You can install the backup and replay the binary logs to recover till 8:00.
- The server creates a new binlog file when max_binlog_size is reached.
- SHOW BINARY LOGS; to display all binary logs.
- SHOW MASTER STATUS; to get the current binary log position.
- Use FLUSH LOGS; to close the current binary log and open a new one.
- Set 'binlog_expire_logs_seconds' or 'expire_logs_days' to clean logs to prevent filling up the disk. Set both to 0 to disable automatic expiry.
- You can also set dynamically using SET @@global.expire_logs_days=2.
- To delete all binary logs and start from the beginning, run RESET MASTER.
- Assuming you have a backup with an offset of 471, you can extract a log between offsets into a file:
'sudo mysqlbinlog /data/mysql/binlogs/server1.000001 --start-position=471 --stop-position=1793 > binlog_extract'
- Relocate binary logs using 'mysqlbinlogmove':
   1. Stop MySql.
   2. sudo mysqlbinlogmove --bin-log-base name=server1 --binlog-dir=/data/mysql/binlogs /binlogs
   3. Edit my.cnf:
   [mysqld]
   log_bin=/binlogs
   4. Start MySql.
- You can use the --server option to move all binary logs except the ones currently in use so you only have to move the last binary log when you stop MySql.
   - sudo mysqlbinlogmove --server=root:pass@host1:3306 /new/location

## BACKUPS

- Logical backups generate SQL statements (mysqldump, mysqlpump, mydumper).
- Physical backups contain all the files for the system (XtraBackup, flat file backup).
- For point-in-time recovery and consistent backups, the backup should be able to provide the binary log position up to when the backup was taken.

**mysqldump**

- mysqldump --all-databases --routines --events --triggers --single-transaction --master-data > dump.sql
- To get point-in-time recovery, specify --single-transaction (useful only for transactional tables like InnoDB to get a consistent state) and --master-data (prints the binary log coordinates of the server to the dump file).
- If taking the backup on a replica, use the --dump-slave option instead of --master-data to get the binary log coordinates of the master.
- To only dump the schema with no data, use the --no-data option.

**mysqlpump**

- mysqlpump --default-parallelism=8 > full_backup.sql
- To add more threads to a large table:
mysqlpump -u root --password --parallel-schemas=4:big_table --default-parallelism=2 > full_backup.sql
- Backup MySql users:
mysqlpump --exclude-databases=% --users > users_backup.sql
- To compress backups, you can use lz4 or zlib.
   - mysqlpump --compress-output=lz4 > dump.lz4
- To decompress:
   - lz4_decompress dump.lz4 dump.sql

**mydumper**

- Benefits are parallelism, consistency, easier to manage output (separate files for tables whereas mysqlpump writes everything to one file).
- shell > mydumper -u root --password=<password> -t 8 --trx-consistency-only --compress --outputdir /backups
- -t option specifies number of CPU threads to use.
- --trx-consistency-only is like --single-transaction in mysqldump but include a binlog position.
- Use the --kill-long-queries option to prevent the backup from failing from long running queries or use the --long-query-guard option to set a higher value.

**XtraBackup**

- Copies flat files without shutting down the server and uses the redo log to avoid inconsistencies by performing a crash recovery.

## RESTORING DATA

- For mysqldump and mysqlpump backup files:
   - shell> mysql -u <user> -p -f < /backups/full_backup.sql
   - During the restore, backup statements are recorded in the binary log which slows down the restoration process. You can disable this just for that session using: mysql> SET SQL_LOG_BIN=0; SOURCE full_backup.sql
   - -f option forces the restoration to continue in the event of a failure.
- For mydumper, myloader comes installed: shell> myloader --directory=/backups --user=<user> --password=<password> --queries-per-transaction=5000 --threads=8 --compress-protocol --overwrite-tables

**Point-in-time recovery**

- mysqldump or mysqlpump
- The binary log information is found in the CHANGE MASTER TO statement found in the resulting backup file from either mysqldump or mysqlpump.
- If --master-data was used for the backup, use the binary logs of the slave.
   - shell> head -30 /backups/dump.sql
     CHANGE MASTER TO MASTER_LOG_FILE='server1.000008',MASTER_LOG_POS=154;
   - shell> mysqlbinlog --start-position=154 --disable-log-bin /backups/binlogs/server1.000008 | mysql -u<user> -p -h <host> -f
- If --dump-slave was used for the backup, use the binary logs of the master.
   - CHANGE MASTER TO MASTER_LOG_FILE='centos7-bin.000001',MASTER_LOG_POS=463;
   - shell> mysqlbinlog --start-position=463 --disable-log-bin /backups/binlogs/centos7-bin.000001 | mysql -u<user> -p -h <host> -f

**mydumper**

- shell> sudo cat /backups/metadata
- If the backup is from the replica, start the restore from the SHOW MASTER STATUS section.
   - shell> mysqlbinlog --start-position=154 --disable-log-bin /backups/binlogs/server1.000012 | mysql -u<user> -p -h <host> -f
- If the backup is from the master, start the restore from SHOW SLAVE STATUS.
   - shell> mysqlbinlog --start-position=463 --disable-log-bin /backups/binlogs/centos7-bin.000001 | mysql -u<user> -p -h <host> -f

## REPLICATION

- Traditional replication: Single master, multiple slaves.
- Chain replication: One server replicates from another, which in turn replicates from another. The middle server is called the relay master. This is mainly used for replication between data centers.
- Master-master replication: Both masters accept writes and replicates between each other.
- Multi-source replication: A replica will replicate from multiple masters. Often used to consolidate from multiple shards into a single database and is useful for reporting.

**Setting up master-master replication**

- Set master2 to replicate from master1.
- Make master2 read-only: mysql> SET @@GLOBAL.READ_ONLY=ON;
- On master2, check the current binary log coordinate.
   mysql> SHOW MASTER STATUS;
   File_Executed_Gtid_Set Position
   server1.000017 473
- Execute CHANGE MASTER TO command on master1 log_file_name is server1.000017 and position is 473:
   mysql> CHANGE MASTER TO MASTER_HOST='<master2_host>', MASTER_USER='binlog_user', MASTER_PASSWORD='<password>', MASTER_LOG_FILE='<log_file_name>', MASTER_LOG_POS=<position>;
- Start the slave on master1: mysql> START SLAVE;
- Set master2 back to read-write: mysql> SET @@GLOBAL.READ_ONLY=OFF;

**Setting up multi-source replication**

- Take a backup from server1 and server2 and restore on server3.
- On server3, change the replication repositories to TABLE from FILE.
   mysql> STOP REPLICA;
   mysql> SET GLOBAL master_info_repository = 'TABLE';
   mysql> SET GLOBAL relay_log_info_repository = 'TABLE';
- Modify the configuration file:
   shell> sudo vim /etc/my.cnf
   [mysqld]
   master-info-repository=TABLE
   relay-log-info-repository=TABLE
- On server3, execute the CHANGE MASTER TO command to make it a slave of server1 over a channel named master-1 (or any name you want).
   mysql> CHANGE MASTER TO MASTER_HOST='server1', MASTER_USER='<user>', MASTER_PORT=3306, MASTER_PASSWORD='<password>', MASTER_LOG_FILE='server1.000017', MASTER_LOG_POS=788 FOR CHANNEL 'master-1';
- On server3, do the same for server2 over channel master-2.
   mysql> CHANGE MASTER TO MASTER_HOST='server2', MASTER_USER='<user>', MASTER_PORT=3306, MASTER_PASSWORD='<password>', MASTER_LOG_FILE='server2.000014', MASTER_LOG_POS=75438 FOR CHANNEL 'master-2';
- Execute the START REPLICA FOR CHANNEL statement for each channel:
mysql> START REPLICA FOR CHANNEL 'master-1';
mysql> START REPLICA FOR CHANNEL 'master-2';
- SHOW REPLICA STATUS\G will show two rows for both replication channels. You can show the status for a single channel with SHOW REPLICA STATUS FOR CHANNEL 'master-1'\G
- Other commands for the replica:
   mysql> STOP REPLICA FOR CHANNEL 'master-1';
   mysql> RESET REPLICA FOR CHANNEL 'master-2';

**Setting up delayed replication**

- mysql> STOP REPLICA;
- mysql> CHANGE MASTER TO MASTER_DELAY = 3600;
- mysql> START REPLICA;

**Setting up semi-synchronous replication**

- The master waits until at least one replica has received the writes.
- By default, rpl_semi_sync_master_wait_point is set to AFTER_SYNC which means that it is enough that the write has reached the relay log. To ensure that the replica has committed the write, set rpl_semi_sync_master_wait_point to AFTER_COMMIT.
- To set the replica count to more than 1, set rpl_semi_sync_master_wait_for_slave_count.
- Fully synchronous replication waits until all replicas have committed the transaction though Galera Cluster must be used.
- On the master, install rpl_semi_sync_master plugin:
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_mster.so';
- Verify the installation:
mysql> SELECT PLUGIN_NAME, PLUGIN_STATUS FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME LIKE '%semi%';
- On the master, enable semi-synchronous replication and adjust the timeout to 1 second.
mysql> SET @@GLOBAL.rpl_semi_sync_master_enabled=1;
mysql> SHOW VARIABLES LIKE 'rpl_semi_sync_master_enabled';
mysql> SET @@GLOBAL.rpl_semi_sync_master_timeout=1000;
mysql> SHOW VARIABLES LIKE 'rpl_semi_sync_master_timeout';
- On the replica, enable semi-synchronous replication and restart the slave IO thread.
mysql> SET GLOBAL rpl_semi_sync_slave_enabled = 1;
mysql> STOP SLAVE IO_THREAD;
mysql> START SLAVE IO_THREAD;
- To find the number of clients connected as semi-sync: on master, execute:
mysql> SHOW STATUS LIKE 'Rpl_semi_sync_master_clients';
- To check what type of replication the master is using (on means semi-sync, off means async):
mysql> SHOW STATUS LIKE 'Rpl_semi_sync_master_status';

## TABLE MAINTENANCE

- Install Percona Toolkit.
- To alter tables, especially large ones, use pt-online-schema-change which will create an empty copy of the table, alter it, copy the rows, delete the old table, and renames the new one to the previous name.
- To determine if a DDL operation is performed in-place or uses a table copy, look at the 'rows affected' value after the command finishes.
   - Changing the default value of a column is very fast and affects 0 rows.
   - Adding a column is slower but since 0 rows are affected, it is done in-place.
   - Adding an index takes time but 0 rows are affected so it is done in-place.
   - Changing the data type of a column takes significant time and does require rebuilding all the rows of the table (except VARCHAR size).
- shell> pt-online-schema-change D=employees, t=employees, h=localhost -u root --ask-pass --alter="MODIFY COLUMN address VARCHAR(100)" --alter-foreign-keys-method=auto --execute
   - --alter-foreign-keys-method updates foreign keys to refer to the new table after the schema change is complete.
- If altering a table with triggers, specify the --preserve-triggers option
   - shell> pt-online-schema-change D=employees, t=salaries, h=localhost -u user --ask-pass --alter="MODIFY COLUMN salary int" --alter-foreign-keys-method=auto --execute --no-drop-old-table --preserve-triggers
- If the server has replicas, the change can create significant lag. To avoid this, you can specify --check-slave-lag (enabled by default) and set --max-leg (1 second by default).
   - shell> pt-online-schema-change D=employees, t=employees, h=localhost -u user --ask-pass --alter="MODIFY COLUMN address VARCHAR(100)" --alter-foreign-keys-method=auto --execute --preserve-triggers --max-lag=10

**Archiving tables**

- shell> pt-archiver --source h=localhost, D=employees, t=employees -u <user> -p<pass> --where="hire_date<DATE_ADD(NOW(), INTERVAL -30 YEAR)" --no-check-charset --limit 10000 --commit-each

**Archiving data**

- If you want to save the rows after deletion into a separate table or file, you can specify the --dest option.
- shell> pt-archiver --source h=localhost, D=employees, t=employees --dest h=localhost, D=employees_archive -u <user> -p<pass> --where="1=1" --no-check-charset --limit 10000 --commit-each
   - --where="1=1" copies all the rows.

**Copying data**

- You can copy data from one table to another also using pt-archiver:
   - shell> pt-archiver --source h=localhost, D=employees, t=employees --dest h=localhost, D=employees_archive -u <user> -p<pass> --where="1=1" --no-check-charset --limit 10000 --commit-each --no-delete

**Cloning tables**

- One option is to use the INSERT INTO SELECT statement (will not work with generated columns):
   mysql> CREATE TABLE employees_clone LIKE employees;
   mysql> INSERT INTO employees_clone SELECT * FROM employees;
- However, the INSERT INTO SELECT statement is slow and dangerous on big tables. If the statement fails, to restore the table state, InnoDB saves all the rows in UNDO logs, which can make the table inaccessible.
- You can also use pt-archiver with the --no-delete option to copy the desired rows to the destination table.

## MANAGING LOGS

**Error log**

- Configuring the error log in my.cnf:
   - [mysqld]
     log-error=/var/log/mysql/mysqld.log
   - shell> sudo systemctl restart mysql
   - mysql> SHOW VARIABLES LIKE 'log_error';
- To adjust the verbosity, set log_error_verbosity: 1 (errors only), 2 (errors and warnings), 3 (errors, warnings, notes). It is recommended to keep the default value of 3.
   - mysql> SET @@GLOBAL.log_error_verbosity=2;
   - mysql> SELECT @@GLOBAL.log_error_verbosity;

**Rotating the error log**

- shell> sudo mv /var/log/mysql/mysqld.log /var/log/mysql/mysqld.log.0
- shell> mysqladmin -u root -p<password> flush-logs

**Using the system log for logging**

- mysql> INSTALL COMPONENT 'file://component_log_sink_syseventlog';
- mysql> SET PERSIST log_error_services = 'log_filter_internal;log_sink_syseventlog';
- mysql> SHOW VARIABLES LIKE 'log_error_services';
- To check the logs: shell> sudo grep mysqld /var/log/syslog | tail
- If you have multiple mysqld processes running, you can tag your instance with log_syslog_tag.
   - mysql> SELECT @@GLOBAL.log_syslog_tag='instance1';
   - shell> sudo systemctl restart mysql
- If you switch back to original logging, you can set log_error_services back to 'log_filter_internal;log_sink_internal'.

**Error logging in JSON format**

- mysql> INSTALL COMPONENT 'file://component_log_sink_json';
- mysql> SET PERSIST log_error_services = 'log_filter_internal;log_sink_json';

**General query log**

- mysql> SET @@GLOBAL.general_log_file='/var/log/mysql/general_query_log';
- mysql> SET GLOBAL general_log = 'ON';
- Be very cautious enabling this on a production server since it drastically affects the server's performance.

**Slow query log**

- Set long_query_time to 1 second (default is 10).
   - mysql> SET @@GLOBAL.LONG_QUERY_TIME=1;
   - Verify: mysql> SELECT @@GLOBAL.LONG_QUERY_TIME;
- mysql> SET @@GLOBAL.slow_query_log_file='/var/log/mysql/mysql_slow.log';
   - Verify: SELECT @@GLOBAL.slow_query_log_file;
- Enable the slow query log:
   - mysql> SET @@GLOBAL.slow_query_log=1;
   - Verify: SELECT @@GLOBAL.slow_query_log;
- Verify with a slow query: mysql> SELECT SLEEP(2);
   - shell> sudo less /var/log/mysql/mysql_slow.log

**Managing binary logs**

- Using the PURGE BINARY LOGS command and the expire_logs_days variable is unsafe in a replication environment because any one of the slaves may not have fully consumed the binary logs and go out of sync. The safe way to delete the binary logs is to check which binary logs have been read by each replica then delete them. Use mysqlbinlogpurge for this.
- shell> mysqlbinlogpurge --master=dbadmin:<pass>@master:3306 --slaves=dbadmin:<pass>@slave1:3306,dbadmin:<pass>@slave2:3306

\# Latest binlog file replicated by all slaves: master-bin.000022

\# Purging binary logs prior to 'master-bin.000023'

- To discover all replicas automatically, set report_host and report_port on all replicas and restart MySql on each server.
   - shell> sudo vim /etc/my.cnf
     [mysqld]
     report-host = replica1
     report-port = 3306
   - shell> sudo systemctl restart mysql
   - mysql> SHOW VARIABLES LIKE '%report%';
   - Execute mysqlbinlogpurge with the discover-slaves-login option.
   - mysql> SHOW BINARY LOGS;
   - shell> mysqlbinlogpurge --master=dbadmin:<pass>@master --discover-slaves-login=dbadmin:<pass>

## PERFORMANCE TUNING

- The EXPLAIN keyword gives information now how the optimizer is going to execute the query.
- EXPLAIN FORMAT=JSON gives more detailed information.

**Benchmarking queries**

- Since queries are often executed in milliseconds, you can use mysqlslap to run a query multiple times for a longer execution time.
- Do not run this on production and ensure that concurrency is less than max_connections.
- shell> mysqlslap -u <user> -p<pass> --create-schema=employees --query="SELECT e.emp_no, salary FROM salaries s JOIN employees e ON s.emp_no=e.emp_no WHERE (first_name='Adam');" -c 1000 i 100
   - This query was executed with 1,000 concurrencies and 100 iterations and on average, took 3.216 seconds.

**Adding an index**

- mysql> ALTER TABLE employees ADD INDEX name (first_name, last_name);
- Unique indexes: mysql> ALTER TABLE employees ADD UNIQUE INDEX unique_name (last_name, first_name);
- Prefix index: indexes that use only the leading part of column values rather than the full column can be created. The length must be specified.
   - mysql> ALTER TABLE employees ADD INDEX (last_name(10));

**Drop an index**

- mysql> ALTER TABLE employees DROP INDEX last_name;

**Index on generated columns**

- mysql> ALTER TABLE employees ADD hire_date_year YEAR AS (YEAR(hire_date)) VIRTUAL, ADD INDEX (hire_date_year);
- Now you can use hire_date_year with the index since you can't use hire_date when it's in the YEAR() function.

**Invisible index**

- To test the performance impact of dropping an index, you can make it invisible first.
   - mysql> ALTER TABLE employees ALTER INDEX last_name INVISIBLE;
   - mysql> ALTER TABLE employees ALTER INDEX last_name VISIBLE;

**Descending index**

- mysql> ALTER TABLE employees ADD INDEX name_desc(first_name ASC, last_name DESC);

**Analyzing slow queries using pt-query-digest**

- pt-query-digest can be used on the slow query log, general query log, process list, binary log, and TCP dump.
- The slow query log does not log all queries unless long_query_time is 0, the general query log does not include query time, the process list does not state the full query, only writes are analyzed in the binary log, while using the TCP dump causes server degradation.

**Slow log**

- shell> sudo pt-query-digest /var/lib/mysq/mysql_slow.log > slow_query_digest
- The digest report contains queries ranked by the number of query executions multiplied by the query time. You can drill down to the specific query by searching for the query checksum.

**General query log**

- shell> sudo pt-query-digest –type genlog /var/lib/mysql/mysql_general.log > general_query_digest

**Process list**

- shell> pt-query-digest --processlist h=localhost --iterations 10 --run-time 1m -u <user> -p<pass>
   - run-time specifies how long each iteration should run. The tool generates reports every minute for 10 minutes for the above command.

**Binary log**

- Convert the binary log to text format using the mysqlbinlog utility first:
   - shell> sudo mysqlbinlog /var/lib/mysql/binlog.000639 > binlog.00063
   - shell> pt-query-digest --type binlog binlog.000639 > binlog_digest

**TCP dump**

- shell> sudo tcpdump -s 65535 -x -nn -q -tttt -i any -c 1000 port 3306 > mysql.tcp.txt
- shell> pt-query-digest --type tcpdump mysql.tcp.txt > tcpdump_digest

**Removing duplicate and redundant indexes**

- pt-duplicate-key-checker - part of Percona Toolkit.
   - shell> pt-duplicate-key-checker -u <user> -p<pass> --database employees
- mysqlindexcheck - part of MySql utilities. Note that it ignores descending indexes and may treat them as duplicate keys of ascending indexes of the same columns.
   - shell> mysqlindexcheck --server=<user>:<pass>@localhost:3306 employees --show-drops
- Using the sys schema.
- Not all redundant indexes are bad. Make sure you check the EXPLAIN plan.

**Removing unused indexes**

- pt-index-usage takes queries from the slow query log, runs the explain plan for each query, and identifies unused indexes. This is an approximation since the slow query log does not contain all queries.
- shell> sudo pt-index-usage slow -u <user> -p<password> /var/lib/mysql/mysql_slow.log > unused_indexes

## SECURITY

- As soon as an installation is done, run: mysql> mysql_secure_installation
- Be cautious granting a user FILE privileges since a user can write a file anywhere in the filesystem with privileges of the mysqld daemon.
- The MySql port should only be open to the application server.
   - shell> telnet <mysql ip> 3306
   - If telnet hangs, the port is closed.
   - If it states that your host is not allowed to connect, then the port is open but MySql is restricting access.
- When creating users, avoid giving access from anywhere (the % option). Restrict access to an IP range or subdomain.
- Restrict the user to only the database that is needed.
   - mysql> GRANT SELECT ON employee.* TO ‘employee_read_only’@’10.10.%.%’;
- When writing scripts, it’s often difficult to include passwords for a script. You can store credentials in $HOME/.my.cnf:
shell> cat $HOME/.my.cnf
[client]
user=dbadmin
password=$troNgP@$$w0rd
   - However, the password is in cleartext and presents a security issue.
- mysql_config_editor can be used to store the password in encrypted format instead.
   - Create the .mylogin.cnf file using mysql_config_editor:
   shell> mysql_config_editor set --login-path=dbadmin_local --host=localhost --user=dbadmin --password
- You can add multiple hostnames and passwords by changing the login path:
   - shell> mysql_config_editor set --login-path=dbadmin_remote --host=56.363.642.17 --user=dbadmin --password
- To log in as dbadmin_remote:
   - shell> mysql --login-path=dbadmin_remote
- To connect to localhost, just execute mysql or mysql --login-path=dbadmin_local.
- To print all the login paths: shell> mysql_config_editor print --all

**Reset the root password**

- First method is to specify the new root password in an init-file:
   - shell> sudo systemctl stop mysql
   - shell> pgrep mysqld
- Save the SQL codein /var/lib/mysql/mysql-init-password. Make this file readable to MySql only.
   - shell> vim /var/lib/mysql/mysql-init-password
     ALTER USER ‘root’@’localhost’ IDENTIFIED BY ‘New$trongPass4’;
   - shell> sudo chmod 400 /var/lib/mysql/mysql-init-password
   - shell> sudo chown mysql:mysql /var/lib/mysql/mysql-init-password
- Start the MySql server with the --init-file option:
   - shell> sudo -u mysql /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid --user=mysql --init-file=/var/lib/mysql/mysql-init-password
- Verify the error log file: “Execution of init_file ‘var/lib/mysql/mysql-init-password’ ended.
- Verify login: shell> mysql -u root -p’New$trongPass4’
- Remove the file: shell> sudo rm -rf /var/lib/mysql/mysql-init-password

**Setting up encrypted connections using X509**

- If the client and server are in different data centers, it is recommended to use encrypted connections since anyone with access to the network can inspect the data.
- mysql> SHOW STATUS LIKE ‘Ssl_cipher’;
   - If you are not using SSL, Ssl_cipher will be blank.
- The server needs the ca.pem, server-cert.pem, and server-key.pem files.
- The clients use the client-cert.pem and client-key.pem files to connect to the server.
- The .pem files are located in /var/lib/mysql by default.
- shell> sudo vim /etc/my.cnf
[mysqld]
ssl-ca=/var/lib/mysql/ca.pem
ssl-cert=/var/lib/mysql/server-cert.pem
ssl-key=/var/lib/mysql/server-key.pem
- shell> sudo systemctl restart mysql
- mysql> SHOW VARIABLES LIKE ‘%ssl%’;
- Copy the client certs to the client location.
- Connect to the server by passing the ssl options:
   - shell> mysql --ssl-cert=client-cert.pem --ssl-key=client-key.pem -h 64.234.136.178
- Mandate the user to connect only by X509:
   - mysql> ALTER USER ‘dbadmin’@’%’ REQUIRE X509;
- shell> mysql --login-path=dbadmin_remote -h 35.186.158.188 --ssl-cert=client-cert.pem --ssl-key=client-key.pem

**Setting up SSL replication**

- Transfer client-key.pem and client-cert.pem to the replicas then chmod them.
   - shell> sudo chmod 600 /etc/mysql/client-key.pem
   - shell> sudo chmod 644 /etc/mysql/client-cert.pem
- On the replica, execute the CHANGE_MASTER command with the SSL options.
   - mysql> STOP SLAVE;
   - mysql> CHANGE MASTER TO MASTER_SSL=1, MASTER_SSL_CERT=’/etc/mysql/client-cert.pem’, MASTER_SSL_KEY=’/etc/mysql/client-key.pem’;
- mysql> SHOW REPLICA STATUS\G
Master_SSL_Allowed: Yes
Master_SSL_CA_File: /etc/mysql/ca.pem
Master_SSL_Cert: /etc/mysql/client-cert.pem
Master_SSL_Key: /etc/mysql/client-key.pem
- mysql> ALTER USER ‘repl’@’%’ REQUIRE X509;
