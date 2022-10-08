[TOC]

# Redshift常用SQL



## 1. 用户权限相关



创建用户

```
CREATE USER username WITH PASSWORD 'password';
```

[create user]:https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/r_CREATE_USER.html

创建组

```
CREATE GROUP group_name WI username;
```

查看用户

```
select * from pg_user
select * from SVL_USER_INFO
```
Tips: https://docs.aws.amazon.com/redshift/latest/dg/r_SVL_USER_INFO.html

再向新的schema对象授权时，因先授予USAGE权限，再授予操作权限。alter default privileges则是针对schema后续创建的对象设置默认权限
```
grant usage on schema test to [group] username;
grant select on ALL TABLES IN SCHEMA test to [group] username;
alter default privileges for username in schema test grant select on tables TO [group] name;
```
Tips: https://docs.aws.amazon.com/redshift/latest/dg/r_GRANT.html

查看用户权限
```
SELECT * 
FROM 
    (
    SELECT 
        schemaname
        ,objectname
        ,usename
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'select') AND has_schema_privilege(usrs.usename, schemaname, 'usage')  AS sel
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'insert') AND has_schema_privilege(usrs.usename, schemaname, 'usage')  AS ins
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'update') AND has_schema_privilege(usrs.usename, schemaname, 'usage')  AS upd
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'delete') AND has_schema_privilege(usrs.usename, schemaname, 'usage')  AS del
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'references') AND has_schema_privilege(usrs.usename, schemaname, 'usage')  AS ref
    FROM
        (
        SELECT schemaname, 't' AS obj_type, tablename AS objectname, schemaname + '.' + tablename AS fullobj FROM pg_tables
        WHERE schemaname not in ('pg_internal')
        UNION
        SELECT schemaname, 'v' AS obj_type, viewname AS objectname, schemaname + '.' + viewname AS fullobj FROM pg_views
        WHERE schemaname not in ('pg_internal')
        ) AS objs
        ,(SELECT * FROM pg_user) AS usrs
    ORDER BY fullobj
    )
WHERE (sel = true or ins = true or upd = true or del = true or ref = true)
and schemaname='schema'
and usename = 'username';
```

查看组权限
```
select namespace AS schemaname,item AS OBJECT,pu.groname AS groupname,
 decode(charindex ('r',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2),'/',1)),0,0,1) AS SELECT,
 decode(charindex ('w',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2),'/',1)),0,0,1) AS UPDATE,
 decode(charindex ('a',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2),'/',1)),0,0,1) AS INSERT,
 decode(charindex ('d',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2),'/',1)),0,0,1) AS delete  
from (select use.usename AS subject,
		nsp.nspname AS NAMESPACE,
		C.relname AS item,
		C.relkind AS TYPE,
		use2.usename AS OWNER,
		C.relacl from pg_user use 
  cross JOIN pg_class c 
  left JOIN pg_namespace nsp ON (C.relnamespace = nsp.oid)
	LEFT JOIN pg_user use2 ON (C.relowner = use2.usesysid)
	WHERE C.relowner = use.usesysid and nsp.nspname NOT IN ('pg_catalog','pg_toast','information_schema'))
JOIN pg_group pu ON array_to_string(relacl, '|') LIKE '%' || pu.groname || '%'
WHERE pu.groname = 'credit_payment_group'
```



## 2. 会话操作相关

查看当前活动的会话

```
SELECT s.process AS pid
       ,date_Trunc ('second',s.starttime) AS "会话启动时间"
       ,datediff(minutes,s.starttime,getdate ()) AS "会话持续时间"
       ,trim(s.user_name) AS "用户"
       ,trim(s.db_name) AS "数据库"
       ,date_trunc ('second',i.starttime) AS "查询启动时间"
       ,trim(i.query) AS "查询sql"
FROM stv_sessions s
  LEFT JOIN stv_recents i
         ON s.process = i.pid
        AND i.status = 'Running'
WHERE s.user_name <> 'rdsdb'
ORDER BY 1
```

查看TOP SQL

```
SELECT
	TRIM (DATABASE) AS DB,
	COUNT (query) AS n_qry,
	MAX (SUBSTRING(qrytext, 1, 80)) AS qrytext,
	MIN (run_seconds) AS "min",
	MAX (run_seconds) AS "max",
	AVG (run_seconds) AS "avg",
	SUM (run_seconds) AS total,
	MAX (query) AS max_query_id,
	MAX (starttime) :: DATE AS last_run,
	aborted,
	listagg (event, ', ') WITHIN GROUP (ORDER BY query) AS events
FROM
	(
		SELECT
			userid,
			label,
			stl_query.query,
			TRIM (DATABASE) AS DATABASE,
			TRIM (querytxt) AS qrytext,
			md5(TRIM(querytxt)) AS qry_md5,
			starttime,
			endtime,
			datediff (seconds, starttime, endtime) :: NUMERIC (12, 2) AS run_seconds,
			aborted,
			decode(
				alrt.event,
				'Very selective query filter',
				'Filter',
				'Scanned a large number of deleted rows',
				'Deleted',
				'Nested Loop Join in the query plan',
				'Nested Loop',
				'Distributed a large number of rows across the network',
				'Distributed',
				'Broadcasted a large number of rows across the network',
				'Broadcast',
				'Missing query planner statistics',
				'Stats',
				alrt.event
			) AS event
		FROM
			stl_query
		LEFT OUTER JOIN (
			SELECT
				query,
				TRIM (split_part(event, ':', 1)) AS event
			FROM
				STL_ALERT_EVENT_LOG
			WHERE
				event_time >= dateadd (DAY, - 7, CURRENT_DATE)
			GROUP BY
				query,
				TRIM (split_part(event, ':', 1))
		) AS alrt ON alrt.query = stl_query.query
		WHERE
			userid <> 1 
			and (querytxt like 'SELECT%' or querytxt like 'select%' ) 
			and database = 'report_mysql_ng'
		AND starttime >= dateadd (DAY, - 7, CURRENT_DATE)
	)
GROUP BY
	DATABASE,
	label,
	qry_md5,
	aborted
ORDER BY
	total DESC
LIMIT 50;
```

查看当前执行的查询

```
select trim(u.usename) as user, s.pid, q.xid,q.query,q.service_class as "q", q.slot_count as slt, date_trunc('second',q.wlm_start_time) as start,decode(trim(q.state), 'Running','Run','QueuedWaiting','Queue','Returning','Return',trim(q.state)) as state, 
q.queue_Time/1000000 as q_sec, q.exec_time/1000000 as exe_sec, m.cpu_time/1000000 cpu_sec, m.blocks_read read_mb, decode(m.blocks_to_disk,-1,null,m.blocks_to_disk) spill_mb , m2.rows as ret_rows, m3.rows as NL_rows,
substring(replace(nvl(qrytext_cur.text,trim(translate(s.text,chr(10)||chr(13)||chr(9) ,''))),'\\n',' '),1,90) as sql,
trim(decode(event&1,1,'SK ','') || decode(event&2,2,'Del ','') || decode(event&4,4,'NL ','') ||  decode(event&8,8,'Dist ','') || decode(event&16,16,'Bcast ','') || decode(event&32,32,'Stats ','')) as Alert
from  stv_wlm_query_state q 
left outer join stl_querytext s on (s.query=q.query and sequence = 0)
left outer join stv_query_metrics m on ( q.query = m.query and m.segment=-1 and m.step=-1 )
left outer join stv_query_metrics m2 on ( q.query = m2.query and m2.step_type = 38 )
left outer join ( select query, sum(rows) as rows from stv_query_metrics m3 where step_type = 15 group by 1) as m3 on ( q.query = m3.query )
left outer join pg_user u on ( s.userid = u.usesysid )
LEFT OUTER JOIN (SELECT ut.xid,'CURSOR ' || TRIM( substring ( TEXT from strpos(upper(TEXT),'SELECT') )) as TEXT
                   FROM stl_utilitytext ut
                   WHERE sequence = 0
                   AND upper(TEXT) like 'DECLARE%'
                   GROUP BY text, ut.xid) qrytext_cur ON (q.xid = qrytext_cur.xid)
left outer join ( select query,sum(decode(trim(split_part(event,':',1)),'Very selective query filter',1,'Scanned a large number of deleted rows',2,'Nested Loop Join in the query plan',4,'Distributed a large number of rows across the network',8,'Broadcasted a large number of rows across the network',16,'Missing query planner statistics',32,0)) as event from STL_ALERT_EVENT_LOG 
     where event_time >=  dateadd(hour, -8, current_Date) group by query  ) as alrt on alrt.query = q.query
order by q.service_class,q.exec_time desc, q.wlm_start_time;
```

## 3. 对象相关

查看表结构

```
SELECT pn.nspname "schema_name",
       pc.relname AS "table_name",
       cast(obj_description (relfilenode,'pg_class') AS VARCHAR) AS "table_description",
       pa.attname AS "column_name",
       col_description(pa.attrelid,pa.attnum) AS "column_description",
       format_type(pa.atttypid,pa.atttypmod) AS "data_type",
       pa.attnotnull AS "allow_null",
       (CASE WHEN (SELECT count(*)
                   FROM pg_constraint
                   WHERE conrelid = pa.attrelid
                   AND   conkey[1] = pa.attnum
                   AND   contype = 'p') > 0 THEN 'Y' ELSE 'N' END) AS "primary_key",
       (CASE WHEN (SELECT count(*)
                   FROM pg_constraint
                   WHERE conrelid = pa.attrelid
                   AND   conkey[1] = pa.attnum
                   AND   contype = 'u') > 0 THEN 'Y' ELSE 'N' END) AS "unique_key",
       (CASE WHEN (SELECT count(*)
                   FROM pg_constraint
                   WHERE conrelid = pa.attrelid
                   AND   conkey[1] = pa.attnum
                   AND   contype = 'f') > 0 THEN 'Y' ELSE 'N' END) AS "foreign_key"
FROM pg_class AS pc,
     pg_attribute AS pa,
     pg_namespace AS pn
WHERE pc.relnamespace = pn.oid
AND   pn.nspname IN ('report_mongo')
AND   pc.relname = 'ng_okcard_user'
AND   pa.attrelid = pc.oid
AND   pc.relkind = 'r'
AND   pa.attnum > 0;
```


查询当前锁信息
```
select a.txn_owner, a.txn_db, a.xid, a.pid, a.txn_start, a.lock_mode, a.relation as table_id,nvl(trim(c."name"),d.relname) as tablename, a.granted,b.pid as blocking_pid ,datediff(s,a.txn_start,getdate())/86400||' days '||datediff(s,a.txn_start,getdate())%86400/3600||' hrs '||datediff(s,a.txn_start,getdate())%3600/60||' mins '||datediff(s,a.txn_start,getdate())%60||' secs' as txn_duration
from svv_transactions a 
left join (select pid,relation,granted from pg_locks group by 1,2,3) b 
on a.relation=b.relation and a.granted='f' and b.granted='t' 
left join (select * from stv_tbl_perm where slice=0) c 
on a.relation=c.id 
left join pg_class d on a.relation=d.oid
where  a.relation is not null and tablename='ng_okcard_apply_log';
```