#### pkpsqladmincok2ed
#####Chapter 1. First Steps
######Connecting to
```
psql postgresql://root:root@localhost:5432/postgres
```
inner vars
```
SELECT inet_server_port();
SELECT current_database();
SELECT current_user;
```

######enable remote connection:  

postgresql.conf:
```
listen_addresses = '*'
``` 
and change pg_hba.conf
######Using the psql query
```
psql –h hostname –p 5432 –d ke –U root
```
-c and -f
```
psql -c "SELECT current_database"
```
interactive
```
psql –f file.sql
```

==
.pgpass file.
```
			host:port:dbname:user:password
such as		myhost:5432:postgres:user:pass
```

- cp2
```
SELECT date_trunc('second',current_timestamp - pg_postmaster_start_time()) as uptime;
```



- cp3
work_mem  
```
SET work_mem = '16MB';   //MB is case sensitive.
```
verify:
```
SELECT name, setting, reset_val, source FROM pg_settings WHERE source = 'session';
```
in current transaction:
```
BEGIN;
SET LOCAL work_mem = '16MB';
```
reset:
```
RESET work_mem;
RESET ALL;
```


enable expanded display:
```
\x
```

show work_mem:
```
SELECT * FROM pg_settings WHERE name='work_mem';
```
Check which parameters have been changed already or whether our changes have correctly taken effect.
```
SELECT name, source, settingFROM pg_settings WHERE source != 'default' AND source != 'override' ORDER by 2, 1;
```
other columns:
```
boot_val,reset_val
```


reload config file: re-read the postgresql.conf
```
pg_ctl -D /usr/local/pgsql/data reload         //data dic
```

alter role, with and set
```
ALTER ROLE ke SET maintenance_work_mem = 100000;
ALTER ROLE ke CREATEROLE CREATEDB;
ALTER ROLE ke IN DATABASE dbname SET client_min_messages = DEBUG;
```

some optimization  
heavy write activity->set wal_buffers to a much higher value.  
heavy write activity and/or large data loads->checkpoint_segments higher value.  
database has many large queries->set work_mem to a value higher.  
autovacuum should be turned on.  
don't touch fsync.  


power save mode
```
autovacuum = off
wal_writer_delay = 10000
bgwriter_delay = 10000
```

In each round, no more than this many buffers will be written by the background writer. 
```
bgwriter_lru_maxpages = 0
```

- cp4
start pg server. from [here](http://www.postgresql.org/docs/current/static/server-start.html)  
```
postgres -D /usr/local/pgsql/data
```

stop:
```
pg_ctl -D /usr/local/pgsql/data -m fast stop   // -m fast means immediately
pg_ctl -D /usr/local/pgsql/data -m immediate stop   //even fast
```
Reloading the server configuration files,(see prev chap)
```
select pg_reload_conf(); //show result
```


some important settings: possible context value: sighup , superuser
```
SELECT name, setting, unit,(source = 'default') as is_default FROM pg_settings WHERE context = 'sighup' AND (name like '%delay' or name like '%timeout') AND setting != '0';
```

kill
```
kill -SIGHUP \`psql -t -c "select pid from pg_stat_activity limit 1"`
```

limit database connection:
```
ALTER DATABASE db CONNECTION LIMIT 0;
ALTER USER user CONNECTION LIMIT 0;
```

(test), create a pg_hba.conf, add these: (only superusers are allowed to connect)
```
local  all	       postgres			ident
  local  all		all	      			reject
  host   all		all	      0.0.0.0/0	reject
```

list connections per user
```
SELECT rolconnlimit FROM pg_roles WHERE rolname = 'ke';
```
check online users
```
SELECT count(*) FROM pg_stat_activity WHERE usename = 'ke';
```


kick user out.  
//list all non-superusers
```
SELECT count(pg_terminate_backend(pid)) FROM pg_stat_activity WHERE usename NOT IN (SELECT usename FROM pg_user WHERE usesuper);
```
  
  
- cp5
```
SELECT * FROM AA; = SELECT * FROM aa; = SELECT * FROM aA;
```
but not SELECT * FROM "AA";
to remove this, use 
```
select quote_ident('AA');
```

show al rows with same id
```
SELECT * FROM data WHERE id IN (SELECT id FROM data GROUP BY id HAVING count(*) > 1);
```


delete exactly duplicate row
```
BEGIN;
LOCK TABLE table IN ROW EXCLUSIVE MODE;
DELETE FROM table WHERE ctid NOT IN (SELECT min(ctid) FROM table WHERE id IN (1) GROUP BY customerid);
COMMIT;
```
Then cleanup
```
VACUUM table;
```

prevent dulpicate row
```
ALTER TABLE table ADD PRIMARY KEY(id);
ALTER TABLE table ADD UNIQUE(id);
CREATE UNIQUE INDEX ON table(id);
```


create partial index:
```
CREATE UNIQUE INDEX ON table(customerid) WHERE column = 'prop';
```


analyze duplicates(finding duplicate key)
```
analyze table;
SELECT attname, n_distinct FROM pg_stats WHERE schemaname = 'public' AND tablename = 'name';
```
-1 means no duplicate.
