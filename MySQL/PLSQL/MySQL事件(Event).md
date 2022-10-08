#MySQL #Event
# MySQL事件(Event)

Event为事件调度，类似于Linux Crontab，基于时间触发指定操作。创建Event前，需要确定参数`event_scheduler`已启用，否则事件将不会被触发。
```sql
mysql> show variables like 'event%';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| event_scheduler | ON    |
+-----------------+-------+
```
Event创建语法如下：
```sql
CREATE
    [DEFINER = user]
    EVENT
    [IF NOT EXISTS]
    event_name
    ON SCHEDULE schedule
    [ON COMPLETION [NOT] PRESERVE]
    [ENABLE | DISABLE | DISABLE ON SLAVE]
    [COMMENT 'string']
    DO event_body;
    
schedule: {
    AT timestamp [+ INTERVAL interval] ...
  | EVERY interval
    [STARTS timestamp [+ INTERVAL interval] ...]
    [ENDS timestamp [+ INTERVAL interval] ...]
}

interval:
    quantity {YEAR | QUARTER | MONTH | DAY | HOUR | MINUTE |
              WEEK | SECOND | YEAR_MONTH | DAY_HOUR | DAY_MINUTE |
              DAY_SECOND | HOUR_MINUTE | HOUR_SECOND | MINUTE_SECOND}
```
- ON SCHEDULE schedule：定义执行时间和间隔
- ON COMPLETION [NOT] PRESERVE：可选项。默认是ON COMPLETION NOT PRESERVE 即计划任务执行完毕后自动删除该事件；ON COMPLETION PRESERVE则不会删除。如果事件每隔一段事件就会执行，即使设置的是ON COMPLETION NOT PRESERVE，计划任务执行完毕后也不会自动删除。
- ENABLE | DISABLE | DISABLE ON SLAVE：定义事件创建完成之后是否启用事件，从节点同步时会自动加上DISABLE ON SLAVE

## 最佳实践

由于Event定时后台调度，我们无法观测历史的执行情况，不利于事件监控和问题排查。因此可以考虑将Event执行记录到特定的日志表中。

创建日志表
```sql
CREATE TABLE `mysql`.`t_event_log_history` (
`id` int(10) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增ID',
`event_gid` varchar(36) NOT NULL COMMENT 'EVENT ID',
`db_name` varchar(128) NOT NULL DEFAULT '' COMMENT '数据库名称',
`event_name` varchar(128) NOT NULL DEFAULT '' COMMENT 'EVENT NAME',
`start_time` datetime(3) DEFAULT NULL COMMENT '开始时间',
`end_time` datetime(3) DEFAULT NULL COMMENT '结束时间',
`is_success` tinyint(4) DEFAULT 0 COMMENT '是否成功',
`duration` decimal(15,3) DEFAULT NULL COMMENT '执行时间',
`error_msg` varchar(512) DEFAULT NULL COMMENT '错误信息',
PRIMARY KEY (`id`),
UNIQUE KEY `idx_event_git` (`event_gid`),
KEY `idx_db_event_name` (`db_name`,`event_name`),
KEY `idx_s_e_time` (`start_time`,`end_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

创建event

```sql
DELIMITER $$
CREATE EVENT `event_test1`
ON SCHEDULE EVERY 1 MINUTE STARTS '2021-11-01 00:00:00'
ON COMPLETION PRESERVE ENABLE DO
BEGIN  
    DECLARE r_code CHAR(5) DEFAULT '00000';  
    DECLARE r_msg TEXT;  
    DECLARE v_error INT;  
    DECLARE v_start_time DATETIME(3) DEFAULT NOW(3);
    DECLARE v_event_gid VARCHAR(36) DEFAULT UPPER(REPLACE(UUID(),'-',''));  
    -- 插入事件信息到日志表
    INSERT INTO mysql.t_event_log_history (db_name, event_name, start_time, event_gid) VALUES(DATABASE(), 'event_test1', v_start_time, v_event_gid);    
    BEGIN    
        DECLARE CONTINUE HANDLER FOR SQLEXCEPTION    
        BEGIN  
            SET  v_error = 1;  
            GET DIAGNOSTICS CONDITION 1 r_code = RETURNED_SQLSTATE, r_msg = MESSAGE_TEXT;  
        END;  
        -- 调用相关函数程序
        select current_timestamp();
    END;
    -- 更新日志表信息
    UPDATE mysql.t_event_log_history SET end_time = NOW(3), is_success = ISNULL(v_error), duration = TIMESTAMPDIFF(microsecond,start_time, NOW(3)) / 1000000, error_msg = CONCAT('error = ', r_code,', message = ', r_msg) WHERE event_gid = v_event_gid;      
END$$  
DELIMITER ;
```

查看日志
```sql
mysql> select * from mysql.t_event_log_history;
+----+----------------------------------+---------+-------------+-------------------------+-------------------------+------------+----------+-----------+
| id | event_gid                        | db_name | event_name  | start_time              | end_time                | is_success | duration | error_msg |
+----+----------------------------------+---------+-------------+-------------------------+-------------------------+------------+----------+-----------+
|  1 | DB535A751C8E11EDAC1C525400078CBA | test    | event_test1 | 2022-08-15 19:39:00.895 | 2022-08-15 19:39:00.905 |          1 |    0.010 | NULL      |
+----+----------------------------------+---------+-------------+-------------------------+-------------------------+------------+----------+-----------+
```
