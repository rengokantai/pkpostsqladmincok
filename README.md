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
######Changing your password securely
use meta command
```
\password
```

######Avoiding hardcoding
.pgpass file. in ~/
```
host:port:dbname:user:password
such as		myhost:5432:postgres:user:pass
```
######Using a connection service file (may need to use with .pgpass)
.pg_service.conf in ~/, or pg_service.conf in /etc/
```
[ke]
host=localhost
port=5432
dbname=postgres
```

#####Chapter 2. Exploring the Db
######What version
```
SELECT version();
psql --version
```
######uptime
```
SELECT date_trunc('second',current_timestamp - pg_postmaster_start_time()) as uptime;
```
######database server's message log
in debian:
```
cd /var/log/postgresql
```
redhat
```
cd /var/lib/pgsql/data/pg_log
```
######database's system identifier
```
cd /usr/lib/postgresql/9.5/bin
```
use
```
./pg_controldata /var/lib/postgresql/9.5/main/ | grep "system identifier"
```
######Listing databases
```
psql -l
\l
select datname from pg_database;
```
or debug mode(vertical output, =\G)
```
\x
select * from pg_database;
```
```
select * from pg_tables;
```
######How many tables
```
SELECT count(*) FROM information_schema.tables WHERE table_schema NOT IN ('information_schema', 'pg_catalog');
```
use psql
```
psql -c "\d" postgres
```
######disk space does a database use
```
SELECT pg_database_size(current_database());
SELECT sum(pg_database_size(datname)) from pg_database;
```
######disk space does a table use
```
select pg_relation_size('test');
```
total size including index
```
select pg_total_relation_size('test');
```
metacommand
```
\dt+ test
```
use pretty
```
SELECT pg_size_pretty(pg_relation_size('test'));
```
######biggest tables
```
SELECT table_name,pg_relation_size(table_schema || '.' || table_name) as size FROM information_schema.tables WHERE table_schema NOT IN ('information_schema', 'pg_catalog') ORDER BY size DESC;
```
estimate number of rows in a (big) table
```
SELECT (CASE WHEN reltuples > 0 THEN pg_relation_size('ke')*reltuples/(8192*relpages) ELSE 0 END)::bigint AS row_count FROM pg_class WHERE oid = 'ke'::regclass;
```
```
postgres=# select reltablespace, relfilenode from pg_class where oid = 'ke'::regclass;
```
(tbc)
######extensions in this database
```
select * from pg_extension;
\dx
```
######
(tbc)
#####Chapter 3. Configuration
######Changing parameters in your programs
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
show more props
```
SELECT name, setting, reset_val, source FROM pg_settings WHERE source = 'session';
```
######Find the current configuration
```
SHOW config_file;
```
###### nondefault settings?

Check which parameters have been changed already or whether our changes have correctly taken effect.
```
SELECT name, source, setting FROM pg_settings WHERE source != 'default' AND source != 'override' ORDER by 2, 1;
```
other columns:
```
boot_val,reset_val
```

######Updating the parameter file
```
ALTER SYSTEM SET shared_buffers = '1GB';
```
This command will not edit postgresql.conf.  
It writes the new setting to another file postgresql.auto.conf.  
use pg_ctlcluster, a usage:
```
pg_ctlcluster 9.5 main restart
```


######server configuration checklist
```
vim  /etc/sysctl.conf
```
edit
```
kernel.shmmax=value
```






######This section is from 1st edition.
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
=====
#####Chapter 4. Server Control
######Starting the database server manually
start pg server. from [here](http://www.postgresql.org/docs/current/static/server-start.html)  

in ubuntu 16.04
```
pg_ctlcluster 9.5 main start
```
redhat:
```
pg_ctl -D /var/lib/pgsql/data start
```
######Stopping the server safely and quickly
```
pg_ctl -D datadir (-m fast) stop    (-m fast) means fast stop
pg_ctlcluster 9.0 main stop --force
```
######Stopping the server in an emergency
redhat:
```
pg_ctl -D datadir stop -m immediate
```
######Reloading the server configuration files
```
pg_ctlcluster 9.5 main reload
```
in psql:
```
select pg_reload_conf(); //show result
```
some important settings: possible context value: sighup , superuser
```
SELECT name, setting, unit,(source = 'default') as is_default FROM pg_settings WHERE context = 'sighup' AND (name like '%delay' or name like '%timeout') AND setting != '0';
```

kill(-SIGHUP = restart)
```
kill -SIGHUP `psql -t -c "select pid from pg_stat_activity limit 1"`
```
######Restarting the server quickly
redhat
```
pg_ctl -D datadir (-m fast) start
```
######Preventing new connections

limit database connection:
```
ALTER DATABASE db CONNECTION LIMIT 0;
ALTER USER user CONNECTION LIMIT 0;
```

(test), create a pg_hba.conf),,backup original pg_hba.conf, add these: (only superusers are allowed to connect)
```
local  all	       postgres			ident
  local  all		all	      			reject
  host   all		all	      0.0.0.0/0	reject
```


still want superuser access.
```
  local  all       postgres                   peer
  local  all       all                      reject
  host   all       all          0.0.0.0/0   reject
```

######Restricting users to only one session
list connections per user
```
ALTER ROLE ke CONNECTION LIMIT 1;
```
```
SELECT rolconnlimit FROM pg_roles WHERE rolname = 'ke';
```
check online users
```
SELECT count(*) FROM pg_stat_activity WHERE usename = 'ke';
```


######Pushing users off the system
//list all non-superusers
```
SELECT count(pg_terminate_backend(pid)) FROM pg_stat_activity WHERE usename NOT IN (SELECT usename FROM pg_user WHERE usesuper);
```
  
Note:  
(before psql 9.4)
pid was called procpid  
query was called current_query  

######Using multiple schemas
Separate groups of tables into their own namespaces, referred to as schemas by PostgreSQL.  
```
CREATE SCHEMA ke;
CREATE TABLE ke.tbname (id integer);
```
```
select current_schema;  --should return public
```

If we want to let only a specific user look at certain sets of tables, we can modify their search_path parameter.
```
ALTER ROLE ke SET search_path = 'ke';
```
revoke/grant
```
REVOKE ALL ON SCHEMA ke FROM public;
GRANT ALL ON SCHEMA ke TO ke;
```
or grant some privileges
```
GRANT USAGE ON SCHEMA ke TO ke;
GRANT CREATE ON SCHEMA ke TO ke;
```
Note you will also need to issue specific grants for objects.
```
GRANT SELECT ON tbname TO public;
```
or set default previleges
```
ALTER DEFAULT PRIVILEGES FOR USER ke IN SCHEMA ke GRANT SELECT ON TABLES TO PUBLIC;
```
######Giving users their own private database
set a private database and revoke:
```
create database a owner=root;
revoke connect on database a from root;
```
revoke from public and grant to user:
```
begin;
revoke connect on database a from public;
grant connect on database a to root;
commit;
```
######Running multiple servers on one system
root user:
```
sudo -u postgres pg_createcluster 9.5 main2
sudo -u postgres pg_ctlcluster 9.4 main2 start
```
in postgres user:
```
psql --cluster 9.5/main2
```

Redhat(not verified)
```
sudo -u postgres initdb -D /var/lib/pgsql/main2
sudo -u postgres pg_ctl -D /var/lib/pgsql/main2 start
```
######Setting up a connection pool
######Accessing multiple servers using the same host 

#####Chapter 5. Tables and Data
######Choosing good names for database objects
######Handling objects with quoted names
```
SELECT * FROM AA; = SELECT * FROM aa; = SELECT * FROM aA;
```
but not SELECT * FROM "AA";
to remove this, use 
```
select quote_ident('AA');
```

######Enforcing the same name and definition for columns
######Identifying and removing duplicates

show al rows with same id
```
CREATE UNLOGGED TABLE dup_cust AS SELECT * FROM data WHERE id IN (SELECT id FROM data GROUP BY id HAVING count(*) > 1);
```
An UNLOGGED table can be created with less I/O because it does not write WAL. 


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

######Preventing dulpicate rows
```
ALTER TABLE table ADD PRIMARY KEY(id);
ALTER TABLE table ADD UNIQUE(id);
CREATE UNIQUE INDEX ON table(id);
```


create partial index:
```
CREATE UNIQUE INDEX ON table(customerid) WHERE column = 'prop';
```

gist type
```
CREATE TABLE boxes (name text, position box);
INSERT INTO boxes VALUES ('First', box '((0,0), (1,1))');
```
then enforce uniqueness
```
ALTER TABLE boxes ADD EXCLUDE USING gist (position WITH &&);
```


analyze duplicates(finding duplicate key)
```
analyze table;
SELECT attname, n_distinct FROM pg_stats WHERE schemaname = 'public' AND tablename = 'name';
```
-1 means no duplicate.

######Generating test data
generate series of number
```
SELECT * FROM generate_series(1,5);
```
generate one week
```
SELECT date(generate_series(now(), now() + '1week', '1 days'));    --  1week=1 week, day=days
```
