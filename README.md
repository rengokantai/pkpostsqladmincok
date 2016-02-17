#### pkpostsqladmincok
- cp1
```
SELECT inet_server_port();
SELECT current_database();
SELECT current_user;
```

enable remote connection:  

postgresql.conf:
```
listen_addresses = '*'
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
pg_ctl -D data reload
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


