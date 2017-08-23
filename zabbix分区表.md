# zabbix分区表

**环境描述：**

​	因为zabbix监控的量比较大，数据库中的history、history_uint、trends、trends_uint表比较大，最大的history_uint表已经达到62G，因此把这几个大表做成分区表。

​	按照时间戳字段进行分区，每七天为一个分区。

​	通过搭建一台从库，对从库上的表进行分区操作，然后在切换从库为主库，这样来减小修改表结构对生产的影响

**参考：**

​	[zabbix官网文档](https://www.zabbix.org/wiki/Docs/howto/mysql_partition)

​	[MySQL分区表文档](https://dev.mysql.com/doc/refman/5.7/en/partitioning.html)

**步骤：**

​	1.连接mysql，在zabbix库下创建四个存储过程（不需要修改）：

```
DELIMITER $$
CREATE PROCEDURE `partition_create`(SCHEMANAME varchar(64), TABLENAME varchar(64), PARTITIONNAME varchar(64), CLOCK int)
BEGIN
        /*
           SCHEMANAME = The DB schema in which to make changes
           TABLENAME = The table with partitions to potentially delete
           PARTITIONNAME = The name of the partition to create
        */
        /*
           Verify that the partition does not already exist
        */

        DECLARE RETROWS INT;
        SELECT COUNT(1) INTO RETROWS
        FROM information_schema.partitions
        WHERE table_schema = SCHEMANAME AND table_name = TABLENAME AND partition_description >= CLOCK;

        IF RETROWS = 0 THEN
                /*
                   1. Print a message indicating that a partition was created.
                   2. Create the SQL to create the partition.
                   3. Execute the SQL from #2.
                */
                SELECT CONCAT( "partition_create(", SCHEMANAME, ",", TABLENAME, ",", PARTITIONNAME, ",", CLOCK, ")" ) AS msg;
                SET @sql = CONCAT( 'ALTER TABLE ', SCHEMANAME, '.', TABLENAME, ' ADD PARTITION (PARTITION ', PARTITIONNAME, ' VALUES LESS THAN (', CLOCK, '));' );
                PREPARE STMT FROM @sql;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
        END IF;
END$$
DELIMITER ;
```

```
DELIMITER $$
CREATE PROCEDURE `partition_drop`(SCHEMANAME VARCHAR(64), TABLENAME VARCHAR(64), DELETE_BELOW_PARTITION_DATE BIGINT)
BEGIN
        /*
           SCHEMANAME = The DB schema in which to make changes
           TABLENAME = The table with partitions to potentially delete
           DELETE_BELOW_PARTITION_DATE = Delete any partitions with names that are dates older than this one (yyyy-mm-dd)
        */
        DECLARE done INT DEFAULT FALSE;
        DECLARE drop_part_name VARCHAR(16);

        /*
           Get a list of all the partitions that are older than the date
           in DELETE_BELOW_PARTITION_DATE.  All partitions are prefixed with
           a "p", so use SUBSTRING TO get rid of that character.
        */
        DECLARE myCursor CURSOR FOR
                SELECT partition_name
                FROM information_schema.partitions
                WHERE table_schema = SCHEMANAME AND table_name = TABLENAME AND CAST(SUBSTRING(partition_name FROM 2) AS UNSIGNED) < DELETE_BELOW_PARTITION_DATE;
        DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

        /*
           Create the basics for when we need to drop the partition.  Also, create
           @drop_partitions to hold a comma-delimited list of all partitions that
           should be deleted.
        */
        SET @alter_header = CONCAT("ALTER TABLE ", SCHEMANAME, ".", TABLENAME, " DROP PARTITION ");
        SET @drop_partitions = "";

        /*
           Start looping through all the partitions that are too old.
        */
        OPEN myCursor;
        read_loop: LOOP
                FETCH myCursor INTO drop_part_name;
                IF done THEN
                        LEAVE read_loop;
                END IF;
                SET @drop_partitions = IF(@drop_partitions = "", drop_part_name, CONCAT(@drop_partitions, ",", drop_part_name));
        END LOOP;
        IF @drop_partitions != "" THEN
                /*
                   1. Build the SQL to drop all the necessary partitions.
                   2. Run the SQL to drop the partitions.
                   3. Print out the table partitions that were deleted.
                */
                SET @full_sql = CONCAT(@alter_header, @drop_partitions, ";");
                PREPARE STMT FROM @full_sql;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;

                SELECT CONCAT(SCHEMANAME, ".", TABLENAME) AS `table`, @drop_partitions AS `partitions_deleted`;
        ELSE
                /*
                   No partitions are being deleted, so print out "N/A" (Not applicable) to indicate
                   that no changes were made.
                */
                SELECT CONCAT(SCHEMANAME, ".", TABLENAME) AS `table`, "N/A" AS `partitions_deleted`;
        END IF;
END$$
DELIMITER ;

DELIMITER $$
CREATE PROCEDURE `partition_maintenance`(SCHEMA_NAME VARCHAR(32), TABLE_NAME VARCHAR(32), KEEP_DATA_DAYS INT, HOURLY_INTERVAL INT, CREATE_NEXT_INTERVALS INT)
BEGIN
        DECLARE OLDER_THAN_PARTITION_DATE VARCHAR(16);
        DECLARE PARTITION_NAME VARCHAR(16);
        DECLARE OLD_PARTITION_NAME VARCHAR(16);
        DECLARE LESS_THAN_TIMESTAMP INT;
        DECLARE CUR_TIME INT;

        CALL partition_verify(SCHEMA_NAME, TABLE_NAME, HOURLY_INTERVAL);
        SET CUR_TIME = UNIX_TIMESTAMP(DATE_FORMAT(NOW(), '%Y-%m-%d 00:00:00'));

        SET @__interval = 1;
        create_loop: LOOP
                IF @__interval > CREATE_NEXT_INTERVALS THEN
                        LEAVE create_loop;
                END IF;

                SET LESS_THAN_TIMESTAMP = CUR_TIME + (HOURLY_INTERVAL * @__interval * 3600);
                SET PARTITION_NAME = FROM_UNIXTIME(CUR_TIME + HOURLY_INTERVAL * (@__interval - 1) * 3600, 'p%Y%m%d%H00');
                IF(PARTITION_NAME != OLD_PARTITION_NAME) THEN
                        CALL partition_create(SCHEMA_NAME, TABLE_NAME, PARTITION_NAME, LESS_THAN_TIMESTAMP);
                END IF;
                SET @__interval=@__interval+1;
                SET OLD_PARTITION_NAME = PARTITION_NAME;
        END LOOP;

        SET OLDER_THAN_PARTITION_DATE=DATE_FORMAT(DATE_SUB(NOW(), INTERVAL KEEP_DATA_DAYS DAY), '%Y%m%d0000');
        CALL partition_drop(SCHEMA_NAME, TABLE_NAME, OLDER_THAN_PARTITION_DATE);

END$$
DELIMITER ;

DELIMITER $$
CREATE PROCEDURE `partition_verify`(SCHEMANAME VARCHAR(64), TABLENAME VARCHAR(64), HOURLYINTERVAL INT(11))
BEGIN
        DECLARE PARTITION_NAME VARCHAR(16);
        DECLARE RETROWS INT(11);
        DECLARE FUTURE_TIMESTAMP TIMESTAMP;

        /*
         * Check if any partitions exist for the given SCHEMANAME.TABLENAME.
         */
        SELECT COUNT(1) INTO RETROWS
        FROM information_schema.partitions
        WHERE table_schema = SCHEMANAME AND table_name = TABLENAME AND partition_name IS NULL;

        /*
         * If partitions do not exist, go ahead and partition the table
         */
        IF RETROWS = 1 THEN
                /*
                 * Take the current date at 00:00:00 and add HOURLYINTERVAL to it.  This is the timestamp below which we will store values.
                 * We begin partitioning based on the beginning of a day.  This is because we don't want to generate a random partition
                 * that won't necessarily fall in line with the desired partition naming (ie: if the hour interval is 24 hours, we could
                 * end up creating a partition now named "p201403270600" when all other partitions will be like "p201403280000").
                 */
                SET FUTURE_TIMESTAMP = TIMESTAMPADD(HOUR, HOURLYINTERVAL, CONCAT(CURDATE(), " ", '00:00:00'));
                SET PARTITION_NAME = DATE_FORMAT(CURDATE(), 'p%Y%m%d%H00');

                -- Create the partitioning query
                SET @__PARTITION_SQL = CONCAT("ALTER TABLE ", SCHEMANAME, ".", TABLENAME, " PARTITION BY RANGE(`clock`)");
                SET @__PARTITION_SQL = CONCAT(@__PARTITION_SQL, "(PARTITION ", PARTITION_NAME, " VALUES LESS THAN (", UNIX_TIMESTAMP(FUTURE_TIMESTAMP), "));");

                -- Run the partitioning query
                PREPARE STMT FROM @__PARTITION_SQL;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
        END IF;
END$$
DELIMITER ;
```

​	2.创建总调用的存储过程

```
DELIMITER $$
CREATE PROCEDURE `partition_maintenance_all`(SCHEMA_NAME VARCHAR(32))
BEGIN
        CALL partition_maintenance(SCHEMA_NAME, 'history', 365, 168, 4);
		CALL partition_maintenance(SCHEMA_NAME, 'history_uint', 365, 168, 4);
		CALL partition_maintenance(SCHEMA_NAME, 'trends', 365, 168, 4);
		CALL partition_maintenance(SCHEMA_NAME, 'trends_uint', 365, 168, 4);
END$$
DELIMITER ;
```

​	参数说明：

​		SCHEMA_NAME：要分区的表所在的数据库

​		history：要分区的表名

​		365：分区表保留天数。365天之前的分区会被删除

​		168：每个分区中的数据时间间隔，单位：小时。这里就是每七天一个分区

​		4：每次调用存储过程，预创建几个分区。

​		根据实际情况设置参数

​	3.搭建一台mysql从库。和主库在同一台服务器，方便数据文件转移，同时不用修改zabbix连接数据库的配置

​		1.配置主库

​			shell>cat /etc/my.cnf

​			log_bin=mysql-bin

​			binlog_format=row

​			server_id=101

​			gtid_mode=ON

​			enforce_gtid_consistency=ON

​			shell>service mysqld restart

​		2.配置从库（指定不同的数据目录、socket文件、pid-file、port、server_id等）

​			shell>cat /var/zabbix/mysql_slave/my.cnf

​			datadir=/var/zabbix/mysql_slave/data

​			socket=/var/zabbix/mysql_slave/mysql.sock

​			pid_file=/var/zabbix/mysql_slave/mysqld.pid

​			port=3307

​			log_bin=mysql-bin

​			binlog_format=row

​			server_id=102

​			gtid_mode=ON

​			enforce_gtid_consistency=ON

​		3.主库备份

​			创建备份用户：

​				GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO innobackup@localhost identified by '123456';

​			备份：

​				innobackupex --defaults-file=/etc/my.cnf --user=innobackup --password=123456 /var/zabbix/backup/

​				innobackupex --apply-log --user-memory=1G /var/zabbix/backup/

​		4.从库恢复

​			innobackupex --defaults-file=/var/zabbix/mysql_slave/my.cnf --move-back /var/zabbix/backup/

​			chown -R mysql:mysql /var/zabbix/mysql_slave/data 

​			mysqld --defaults-file=/var/zabbix/mysql_slave/my.cnf &

​		至此，新建的库已经启动，但是先不要指向主库进行复制。首先对表进行分区。

​	4.对新搭的库做分区表

​		这里通过导出数据，然后清空表，对表进行修改，然后再导入数据的方式。也可以直接使用alter table语句进行修改。

​		1.导出表数据

​			mysqldump -uroot -p -t -S /var/zabbix/mysql_slave/mysql.sock zabbix history history_uint trends trends_uint > zabbix.sql

​		2.修改表结构

​			truncate table zabbix.history;

​			truncate table zabbix.history_uint;

​			truncate table zabbix.trends;

​			truncate table zabbix.trends_uint;

​			下面是创建分区的语句，每次只要换表名即可。需要注意，创建的分区要可以存放未来几天的数据，否则会插入失败。

​			ALTER TABLE zabbix.history PARTITION BY RANGE (`clock`) 
(

    PARTITION p201704240000 VALUES LESS THAN (1493568000) ENGINE = InnoDB,
    PARTITION p201705010000 VALUES LESS THAN (1494172800) ENGINE = InnoDB,
    PARTITION p201705080000 VALUES LESS THAN (1494777600) ENGINE = InnoDB,
    PARTITION p201705150000 VALUES LESS THAN (1495382400) ENGINE = InnoDB,
    PARTITION p201705220000 VALUES LESS THAN (1495987200) ENGINE = InnoDB,
    PARTITION p201705290000 VALUES LESS THAN (1496592000) ENGINE = InnoDB,
    PARTITION p201706050000 VALUES LESS THAN (1497196800) ENGINE = InnoDB,
    PARTITION p201706120000 VALUES LESS THAN (1497801600) ENGINE = InnoDB,
    PARTITION p201706190000 VALUES LESS THAN (1498406400) ENGINE = InnoDB,
    PARTITION p201706260000 VALUES LESS THAN (1499011200) ENGINE = InnoDB,
    PARTITION p201707030000 VALUES LESS THAN (1499616000) ENGINE = InnoDB,
    PARTITION p201707100000 VALUES LESS THAN (1500220800) ENGINE = InnoDB,
    PARTITION p201707170000 VALUES LESS THAN (1500825600) ENGINE = InnoDB,
    PARTITION p201707240000 VALUES LESS THAN (1501430400) ENGINE = InnoDB,
    PARTITION p201707310000 VALUES LESS THAN (1502035200) ENGINE = InnoDB,
    PARTITION p201708070000 VALUES LESS THAN (1502640000) ENGINE = InnoDB
);

​		3.导入数据

​			mysql -uroot -p zabbix <zabbix.sql

​	5.搭建主从

​		主库创建用户：

​			GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' identified by '123456';

​		从库指向主库

​			reset master;

​			set @@global.gtid_purged='***';（在xtrabackup_binlog_info文件中获取位置）

​			change master to

​			master_host='主库IP',

​			master_port=3306,

​			master_user='repl',

​			master_password='123456',

​			master_auto_position=1;

​			start slave;

​	6.主从一致后，进行数据库切换

​		关闭主库和从库：

​			shutdown

​		将从库数据文件移到主库数据文件位置

​			mv /var/zabbix/mysql /var/zabbix/mysql_bak

​			mv /var/zabbix/mysql_slave/data /var/zabbix/mysql

​	7.重启主库

​		service mysqld start

​	8.创建计划任务，每周一创建分区

​		crontab -e

​		0 1 * * 1 mysql -uroot -p123456 -e "call zabbix.partition_maintenance_all('zabbix')"

完成！