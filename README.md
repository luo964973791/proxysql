### 登录mgr PRIMARY节点，设置用户，并授权.
```JavaScript
CREATE USER 'monitor'@'%' IDENTIFIED BY "Test@123";
GRANT ALL PRIVILEGES ON *.* TO 'monitor'@'%' ;
CREATE USER 'proxysql'@'%' IDENTIFIED BY "Test@123";
GRANT SELECT, INSERT, UPDATE, DELETE, RELOAD, CREATE, ALTER, INDEX, CREATE TEMPORARY TABLES, LOCK TABLES ON *.* TO 'proxysql'@'%';



USE sys;

DELIMITER $$

CREATE FUNCTION my_id() RETURNS TEXT(36) DETERMINISTIC NO SQL RETURN (SELECT @@global.server_uuid as my_id);$$

-- new function, contribution from Bruce DeFrang
CREATE FUNCTION gr_member_in_primary_partition()
    RETURNS VARCHAR(3)
    DETERMINISTIC
    BEGIN
      RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
    performance_schema.replication_group_members WHERE MEMBER_STATE NOT IN ('ONLINE', 'RECOVERING')) >=
    ((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
    'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
    performance_schema.replication_group_member_stats USING(member_id) where member_id=my_id());
END$$

CREATE VIEW gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
Count_Transactions_Remote_In_Applier_Queue as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert'
from performance_schema.replication_group_member_stats where member_id=my_id();$$

DELIMITER ;
```

### 修改/etc/proxysql.cnf里面的用户密码,
```JavaScript
[ -f "/var/lib/proxysql/proxysql.db" ] && rm -rf /var/lib/proxysql/proxysql.db    #如果第一次安装proxysql，这个proxysql.db文件还没生成，就不用执行.
cat /etc/proxysql.cnf | grep -E "admin_credentials|monitor_password="
	admin_credentials="admin:admin"
	monitor_password="Test@123"
systemctl restart proxysql
```

### 设置插入mgr集群信息.
```JavaScript
mysql -uadmin -p -h127.0.0.1 -P6032 -p'admin'
insert into mysql_servers(hostgroup_id,hostname,port) values (10,'172.27.0.6',3306);
insert into mysql_servers(hostgroup_id,hostname,port) values (10,'172.27.0.7',3306);
insert into mysql_users(username,password,active,default_hostgroup,transaction_persistent) values('proxysql','Test@123',1,10,1);
insert into mysql_group_replication_hostgroups (writer_hostgroup,backup_writer_hostgroup,reader_hostgroup,offline_hostgroup,active,max_writers,writer_is_also_reader,max_transactions_behind) values (10,20,30,40,1,1,0,100);
insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply) VALUES (1,1,'^SELECT.*FOR UPDATE$',10,1), (2,1,'^SELECT',30,1);
load mysql servers to runtime;
save mysql servers to disk;
load mysql users to runtime;
save mysql users to disk;
load mysql variables to runtime;
save mysql variables to disk; 
```

### 常用排查命令,用mysql -uadmin -p -h127.0.0.1 -P6032 -p'admin'进入proxysql查看.
```JavaScript
select hostgroup_id, hostname, port,status from runtime_mysql_servers;
select hostname,port,viable_candidate,read_only,transactions_behind,error from mysql_server_group_replication_log order by time_start_us desc limit 6;
select * from mysql_servers;
select * from mysql_users;
select * from runtime_mysql_servers;
SELECT digest,SUBSTR(digest_text,0,25),count_star,sum_time FROM stats_mysql_query_digest WHERE digest_text LIKE 'SELECT%' ORDER BY sum_time DESC LIMIT 5;
SELECT digest,SUBSTR(digest_text,0,25),count_star,sum_time FROM stats_mysql_query_digest WHERE digest_text LIKE 'SELECT%' ORDER BY count_star DESC LIMIT 5;
select rule_id,active,match_digest,destination_hostgroup,apply from mysql_query_rules;
```


### 测试.
```JavaScript
#测试
mysql -uproxysql -p'Test@123' -h172.27.0.6 -P6033
select @@hostname;
select @@hostname for update;

CREATE DATABASE IF NOT EXISTS demo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
use demo;
CREATE TABLE `person` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `name` TEXT COLLATE utf8mb4_unicode_ci,
  `age` INT DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
INSERT INTO person (name, age) VALUES ('zhangsan', 30), ('lisi', 29);
```
