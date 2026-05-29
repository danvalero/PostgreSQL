# PostgreSQL Streaming replication

**Author**: Daniel Valero

- [1. Setting up the environment](#1-setting-up-the-environment)
- [2. Basic indexing 1 (searching conditions on a single column)](#2-basic-indexing-1-searching-conditions-on-a-single-column)
- [3. Basic indexing 2 (searching conditions on multiple column)](#3-basic-indexing-2-searching-conditions-on-multiple-column)
- [4. Why is PostgreSQL not using existing indexes?](#4-why-is-postgresql-not-using-existing-indexes?)


> This lab uses PostgreSQL v18. The behavior can be diferrent for older version. Check documentations

## 1. Setting up the environment

For this lab, four differente virutal machines running Ubuntu Server 24.04 LTS:

- pgserverA
- pgserverB
- pgserverC
- pgserverD

Name resolution can be set up in any way your prefer (by setting a DNS server, using the */etc/hosts* file or any other methiod you prefer). Also, firewall must be set to allow traffic between the servers. This configuration will be set aside during this lab but it is something you must do before starting

NOTE: I used Azure and used the cli command [here](support_files/azure_env_lab_cli.txt) to create a VNET and VMs for the lab.

At the end of the lab, you will have the enviroment below:

--AGREAGAR IMAGEN

PostgreSQL v18 was installed following  [Linux downloads (Ubuntu)](https://www.postgresql.org/download/linux/ubuntu/#apt). 
For reference, when installing PostgreSQL 18 unsing this method, the important PostgreSQL direcatories are:
- cluster file: /var/lib/postgresql/18/main
- confguration files: /etc/postgresql/18/main
- binnary files: /usr/lib/postgresql/18/bin
- log files: more /var/log/postgresql/

In PostgreSQL on pgserverA, restore the test database [Pagila](https://github.com/devrimgunduz/pagila)

---

## 2. Prepare the primary server for replication

1. Connect to the primary PostgreSQL cluster

1. Set the paratmers required to support streaming replication
   
   NOTE: You can modify the postgresql.conf or you can use ALTER SYSTEM

   ```sql
   ALTER SYSTEM SET wal_level = 'replica';
   ```

   max_wal_senders = 10
   max_replication_slots = 10
   wal_sender_timeout = 60 

1. Create an user the replica server will use to connect and push data

   ```sql
   CREATE USER repuser WITH replication ENCRYPTED PASSWORD 'complexpassword1.';
   ```
 
1. Allow the user *repuser* to connect from the replica server by updating *pg_hba.conf*

   ```
   host    replication     repuser          10.0.2.5/32               scram-sha-256
   host    replication     repuser          10.0.2.6/32               scram-sha-256
   host    replication     repuser          10.0.2.7/32               scram-sha-256
   ```

   In this example the IP addresses are:
   - 10.0.2.5 for pgserverB
   - 10.0.2.6 for pgserverC
   - 10.0.2.7 for pgserverD

   NOTE: modifiy the IP addresses based on you enviroment

1. Create the physical replications slot, one per replica server

   ```sql
   SELECT * FROM pg_create_physical_replication_slot('replica_slotb'); --to be used by server pgserverB
   SELECT * FROM pg_create_physical_replication_slot('replica_slotc'); --to be used by server pgserverC
   SELECT * FROM pg_create_physical_replication_slot('replica_slotd'); --to be used by server pgserverD
   ```

   NOTE: This can be done when running pg_basebackup from the replica using -C -S parameters. We will do it manually this time 

1. Check the replication slots were created

   ```sql
   SELECT slot_name, slot_type, temporary, active, active_pid, restart_lsn, wal_status
   FROM  pg_replication_slots;
   ```

   NOTE: The *active_pid* cloumns shows pid for PostgreSQL walsender process. It is empty as the slots are not in use yet.

---

## 3. Add a replica server

1. Connect to replica server pgserverB

1. Impersonnate the SO postgres user 

   ```bash
   sudo -i -u postgres
   ```

1. Stop postgres

   ```bash
   systemctl stop postgresql@18-main
   ```

1. Delete the directory that holds the data, delete

   ```bash
   rm -r /var/lib/postgresql/18/main/
   ```

1. Take the base backup and XXX

   ```bash
   pg_basebackup -h pgserverA -U repuser -p 5432 -D /var/lib/postgresql/18/main/ -Fp -Xs -P -R -S replica_slotb
   ```

   EXPLCAR LOS PARAMETROS!!!

1. Check that the *standby.signal* was created automaticlly

   ```bash
   ls -ls /var/lib/postgresql/18/main/
   ```

1. Confirm that the *primary_conninfo* and *primary_slot_name* paramters have ben set automaticaly

   ```bash
   more /var/lib/postgresql/18/main/postgresql.auto.conf
   ```

1. Start postgres

   ```bash
   systemctl start postgresql@18-main
   ```

1. Check the error log and confirm the server and replica started correctly

   ```bash
   more /var/log/postgresql/postgresql-18-main.log
   ```

1. Check the walreceiver process by running 

   ```bash
	ps -ef | grep postg
   ```

1. Connect to the primary PostgreSQL cluster server  and check there is now a walsender processes for the first repliac

   ```bash
	ps -ef | grep postg
   ```

1. Connect to the primary PostgreSQL cluster server  and check the *active_pid* columns shows the pid fot the walsender process. Also notice that the column *active* now shows *t* (True).

   ```sql
   SELECT slot_name, slot_type, temporary, active, active_pid, restart_lsn, wal_status
   FROM  pg_replication_slots;
   ```

1. Repeat the steps add pgserverC and pgserverD replicas.

---
## 4. Test and monitor replication

1. Connect to primary server and create a new table with a few records

	```sql
    CREATE TABLE animals (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    species VARCHAR(50),
    age INT,
    habitat VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   INSERT INTO animals (name, species, age, habitat) VALUES
   ('Leo', 'Lion', 5, 'Savanna'),
   ('Milo', 'Cat', 3, 'Domestic'),
   ('Bella', 'Dog', 4, 'Domestic'),
   ('Charlie', 'Elephant', 12, 'Forest'),

   SELECT * FROM animals;
   ```

1. Connect to one or all replica servers and confirm the table and its records has been replicated

   ```sql
   SELECT * FROM animals;
   ```

   > You are able to read because parameter *hot_standby* is set to ON by default. If you want to make a replica server not available for reads, set *hot_standby* to off in its configuration file or by running: "ALTER SYSTEM SET hot_standbyl = 'off';"

1. Check the stats for the wal receiver by running:

   ```sql
   \x
   SELECT * FROM pg_stat_wal_receiver;
   ```

1. You can confirm th replica is recovy mode by running:

   ```sql
   SELECT pg_is_in_recovery();
   ```

1. Connect to the primary postgreSQL cluster and check the status of the replication sessions:

   ```sql
   SELECT pid, usename, application_name, client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn, sync_state 
   FROM pg_stat_replication;
   ```

---

## 5. Set a syncronous replica server

Let's start by setting only one syncronous replica, and see the inmpact it as

1. In the replica server pgserverB, edit */var/lib/postgresql/18/main/postgresql.auto.conf* to inlcude "application_name=pgserverB*

   NOTE: Not the recommended method as postgresql.auto.conf should not be manually edited. 

1. In the primary server, execute 
   
   ```sql
   ALTER SYSTEM SET synchronous_standby_names = 'pgserverB';
   SELECT pg_reload_conf();
   ```

   Confirm that the replica state is now *sync* not async
   ```sql
   SELECT pid, usename, application_name, client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn, sync_state 
   FROM pg_stat_replication;
   ```

1. connect to primary server and create a new table with a few records

	```sql
   INSERT INTO animals (name, species, age, habitat) VALUES
   ('Max', 'Tiger', 6, 'Jungle'),
   ('Luna', 'Wolf', 7, 'Mountain'),
   ('Rocky', 'Bear', 10, 'Forest'),
   ('Daisy', 'Cow', 6, 'Farm'),
   ('Coco', 'Parrot', 2, 'Tropical'),
   ('Nemo', 'Fish', 1, 'Ocean');

   SELECT * FROM animals;
   ```

1. connect to replica server and confirm the records have been replicated

   ```sql
   SELECT * FROM animals;
   ```

1. Stop the replica server 

   ```bash
   systemctl stop postgresql@18-main
   ```

1. Insert a record in the primary server

   ```sql
   INSERT INTO animals (name, species, age, habitat) VALUES
   ('Daisy', 'Cow', 6, 'Farm'),
   ```

   it is frozen... not good, if the syn replica is down, the primary wont be available 

   > TO void the primary server to be "frozen" while the sync replicas are unabailable, there are 2 options
   > - set synchronous_commit = off risk of data lost,  but you primaery is available
   >- add addional syn replicas and set synchronous_standby_names using the FIRST or ANY option. That what we will do here!

1. Start the replica server 

   ```bash
   systemctl start postgresql@18-main
   ```

1. See that "frozen" insert operation has now finished. You can query the table in the server replica and see that row is also there. This show how PostgreSQL protect agins data loss

1. Add additional sync and asyn replicas

   - Do step 5.1 for servers pgserverV and pgserverC
   
   - Set the primary so it requires that any 1 of the sync replicas answer before continuing. This will allow the primary server to continue if on of pgserverB and pgserverC is unavailbel, ensuer one of them will be on sync and there is no data loss.

   - add addional syn replicas and set synchronous_standby_names using the FRISt or NAY

     Review  [synchronous_standby_names](https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-SYNCHRONOUS-STANDBY-NAMES) for additinal detls of all possible options

	 ```sql
	 ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (pgserverB,pgserverC)';
	 SELECT pg_reload_conf();
	 ```

   -  Confirm that the replica state is now *quorum* 

      ```sql
      SELECT pid, usename, application_name, client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn, sync_state 
      FROM pg_stat_replication;
      ```

1. Stop any of the 2 replica servers 

   ```bash
   systemctl stop postgresql@18-main
   ```

1. Insert a record in the primary server

   ```sql
   INSERT INTO animals (name, species, age, habitat) VALUES
   ('Coco', 'Parrot', 2, 'Tropical'),
   ('Nemo', 'Fish', 1, 'Ocean');
   ```

   Now, it is not froze. as it was able to sync with one of the 

   no wait

1. Start the repclia server 

   ```bash
   systemctl start postgresql@18-main
   ```

1. Confirm the row was inserted once it was online

---

## 6. What happens to the WAL files in the primary when a replica is not available?

There are 3 important paramters you need consider to understand how PostgreSQL will behave in this escenario;

- [wal_keep_size](https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-WAL-KEEP-SIZE) Specifies the minimum size of past WAL files kept in the pg_wal directory, in case a standby server needs to fetch them for streaming replication. If a standby server connected to the sending server falls behind by more than wal_keep_size megabytes, the sending server might remove a WAL segment still needed by the standby, in which case the replication connection will be terminated. Downstream connections will also eventually fail as a result. (However, the standby server can recover by fetching the segment from archive, if WAL archiving is in use.)

- Starting PostgreSQL 13, you can use [max_slot_wal_keep_size](https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-MAX-SLOT-WAL-KEEP-SIZE) to limits how much WAL a slot can retain. If a replica falls behind and exceeds the limit:
  - The slot is invalidated
  - WAL can be removed
  - Replica breaks and must be rebuilt

  If *max_slot_wal_keep_size* is -1 (the default), replication slots may retain an unlimited amount of WAL files.

- Starting PostgreSQL 18, you can set [idle_replication_slot_timeout](https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-IDLE-REPLICATION-SLOT-TIMEOUT) to automatically invalidates a slot if it’s inactive too long
Helps avoid permanent WAL retention from dead replicas.  A value of zero (the default) disables the idle timeout invalidation mechanism.