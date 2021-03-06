# Databases

## Postgres

### Articles
+ [Postgresql performance](http://www.revsys.com/writings/postgresql-performance.html)
+ [Speedup postgres with partial indixes](https://blog.heapanalytics.com/speeding-up-postgresql-queries-with-partial-indexes/)

Egghead
- https://egghead.io/courses/get-started-with-postgresql
- https://egghead.io/courses/using-postgres-window-functions
- https://egghead.io/courses/understand-joins-and-unions-in-postgres


***
### Objects

Writings from DBA1-08

#### Tables
System tables where can be found more info about this object

```
pg_class where relkind='r'
pg_tables
pg_attribute
pg_attrdef
```

- Non-Journal
    no write to WAL
    table can't be restored if there was server fault

- Temporaly
    - Lives in session or transaction
    - is non-journal
    - not available in shared buffer cache

#### Types

```
pg_type
pg_enum
\dt
```

- Integers
- Symbolic
- Double
- Dates
- Logic (__!important__ true, false and __null__)
- Sequences
- Geomerical
- Arrays
- Records
- Ranges
- XML, JSON et c.

#### Constrains

```
pg_constraint
```

- Primary Key
- Unique
- Foreign Key
- Not null
- Check (row level)
- Exclude (all rows)

Constrain is evaluated in the beginning of the transaction or in the very end.

#### Sequences

```
pg_class where relkind='S'
\d
\ds
```

- Generationg sequnces of numbers
    - guaranted uniquines
    - can be asorted

Internaly it is an onepage table with numbers

#### Indexes

```
pg_class where relkind='i'
pg_index
\di
```

Types:
- B-tree (balanced tree)
- GiST
- SP-GiST
- GIN (inverted list), mainly fulltext search operations

#### User defined functions

```
pg_language
pg_proc
\df
```

- Languages
    - sql
    - C
    - PL/pgSQL, PL/Python

- Overdriven function
- Aggregations and Window functions

#### Triggers

```
pg_trigger
```

#### Event Triggers

```
pg_event_trigger
pg_proc
\dy
```
- on DLL commands (All languages except SQL)
- on (Alterm Create, Drop) before or after command

### Locks
Show blocking Postgres processes and kill them

```sql
CREATE VIEW blocking_procs AS
SELECT 
    kl.pid as blocking_pid,
    ka.usename as blocking_user,
    ka.current_query as blocking_query,
    bl.pid as blocked_pid,
    a.usename as blocked_user, 
    a.current_query as blocked_query, 
    to_char(age(now(), a.query_start),'HH24h:MIm:SSs') as age
FROM pg_catalog.pg_locks bl
    JOIN pg_catalog.pg_stat_activity a 
        ON bl.pid = a.procpid
    JOIN pg_catalog.pg_locks kl 
        ON bl.locktype = kl.locktype
        and bl.database is not distinct from kl.database
        and bl.relation is not distinct from kl.relation
        and bl.page is not distinct from kl.page
        and bl.tuple is not distinct from kl.tuple
        and bl.virtualxid is not distinct from kl.virtualxid
        and bl.transactionid is not distinct from kl.transactionid
        and bl.classid is not distinct from kl.classid
        and bl.objid is not distinct from kl.objid
        and bl.objsubid is not distinct from kl.objsubid
        and bl.pid <> kl.pid 
    JOIN pg_catalog.pg_stat_activity ka 
        ON kl.pid = ka.procpid
WHERE kl.granted and not bl.granted
ORDER BY a.query_start;
```

How to test the query on a testing server (not your production DB server)

Connect to your database open a transaction and manually lock a table:

```sql
LOCK your_table;
```

Leave the transaction and connection open.

Open another client that accesses that data:

```sql
SELECT count(*) from your_table;
```

It now should be blocked.

View the currently held locks with a third client:

```sql
SELECT * FROM blocking_procs;
```

The output should look similar

```
blocking_pid   | 25842
blocking_user  | postgres
blocking_query | in transaction
blocked_pid    | 25844
blocked_user   | postgres
blocked_query  | SELECT COUNT(*) FROM "your_table"
age            | 00h:00m:23s
```

It's now possible to kill the offending process holding the lock using:

```sql
SELECT pg_terminate_backend(25842);
```

This will kill the connection where you've set the lock and the open transaction is rolled back but it seems to leave everything else intact. The second client should now get the response from the server.

[src](http://ghostwritten-insomnia.blogspot.ru/2013/04/show-blocking-postgres-processes-and.html)

### postgres.conf

#### Location
* `/etc/postgresql/9.6/main`
* `/etc/lib/pgsql/pg_data`

#### Sane defaults
```ini
# enable logging
logging_collector = on
log_directory = pg_log
# log queries that take more then 1s
log_min_duration_statement = 1000
```

### Indexing

#### Create simple index
```sql
CREATE INDEX books_idx1 on books(author);
```

#### Create partial index
```sql
CREATE INDEX emplyee_idx2 on employee(active) WHERE active='t';
```

#### Find existing indexes
* Use EXPLAIN, search for __Squental Scans__
* Table `pg_stat_user_tables` also shows sequental scans
