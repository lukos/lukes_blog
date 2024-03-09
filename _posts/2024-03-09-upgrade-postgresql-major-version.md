---
layout: post
title: PostgreSQL Major Cluster Upgrade - Nearly Zero Downtime
date: '2024-03-09T12:28:00.000+00:00'
author: Luke Briner
---

Upgrading the major version of a cluster in `postgresql` cannot be done automatically, although an in-place upgrade can be performed requiring the database server to be stopped, the upgrade to be performed, hoping it all works OK and then starting back up. When this is applied cluster wide, there are a number of failure scenarios, which could add up to a long amount of downtime if it doesn't go well or if the database is large.

To be fair, this is the case with any database system. A major version is likely to include a number of breaking changes, at least at implementation level, which makes it non-trivial to upgrade.

At SmartSurvey, we are 2 major versions of Postgresql behind and although this might not be considered "ancient", it also requires us to look into the not-too-distant future when we might need to perform an upgrade more quickly. In other words, we need to plan now, hopefully script or document the upgrade process so that we can either upgrade now, or in the future when we need to.

The two options you have to upgrade a major version of Postgresql to avoid downtime, would be `pg_dumpall` which is like taking a backup which can then be restored onto a newer major version, or, logical replication, which can be setup from older version to newer version and once the new cluster is setup and tested in-parallel, you can then perform a quick changeover, changing back if there are problems.

`pg_dumpall` has the issue that the backup is not being kept up-to-date, so you would need some plan to get any new data that is added after the backup is taken or you might either stop the database or otherwise possibly block any writes if that is feasible. These sorts of approaches are risky. If they work, they can be problem-free but if they don't, you could end up in a big mess quite quickly.

## Logical Replication
Thanks to [Knock](https://knock.app/blog/zero-downtime-postgres-upgrades) who helped me to both understand logical replication and also to just get confidence that there weren't too many variables and it was all going to be fine! At SmartSurvey, we currently have 1 planned downtime per year on Christmas Eve for 4 hours to re-build major indexes etc. (on SQL Server) so we ideally didn't want any downtime and it is possible with logical replication.

Logical replication is less efficient than the physical replication that is used for streaming, essentially because it isn't just replicating the physical contents of the log files for replication but instead a logical set of data, imagine something like SQL statements, that would be larger and need to contain more information that is not needed when copying physical files between two servers of the same major version.

There are a couple of [restrictions](https://www.postgresql.org/docs/current/logical-replication-restrictions.html) on this approach but they are workable still:
1. The current index of sequences (auto increment columns) are not copied during replication - these need to be managed separately
1. The schema of the databases is not copied, this needs to be done manually before the logical replication is setup
1. Large objects are not supported
1. `TRUNCATE` requires some care to replicate if you use this
1. You cannot replicate views etc. only actual tables
1. If you use partitioning, you must have matching partitions on the subscriber

For us, the only thing of any great consequence concerns sequences. 
> Essentially, the sequence problem means that if you have replicated e.g. 10 rows and the source database sequence therefore has a current value of 11 for the next row, the *destination* database, if not modified, will still have a current value of 1. If you subsequently insert a row into the destination after you changeover to the new cluster, it will fail since the sequence number will already exist in the replicated table. If you can, you can simply reseed these after all the data has been copied from the source (if the source is not being actively written to) or otherwise you can reseed the destination to e.g. current source value + 1000 so that the source can still increment its sequence and still hopefully avoid a collision with the destination. Obviously, the 1000 will depend on how much data is added to the source table. In high traffic scenarios, you might be adding 1,000,000 for example.

## The Approach
This will depend on how static your databases are but basically:
1. Plan, think, understand. I cannot over-emphasise that 1 person needs oversight who understands all the moving parts. There might be moving parts that some people do not know so if they are all made visible to a single person, it is up to them to plan and agree the changeover plan
2. Setup a new major cluster alongside the existing
3. Setup logical replication of the old databases to the new cluster
4. Test that the apps can connect to the new cluster and work properly
5. Stop the replication and changeover at the same time

*You might not need to stop the replication first if you have left room in tables to replicate old data during the changeover.

As much as possible should be done before even considering switchover. For example, setting up at least one new VM to start with, setting up the new databases, configuring backups and monitoring etc. You can also check that the replication continues to work in real-time i.e. it didn't just work once.

You will need to list all of the apps/systems that depend on these databases and work out for each of them how their connection strings work and how you can quickly change over from old to new. If you are fortunate, you might have some kind of load balancer between all apps and the databases, which can be quickly swapped in and out when needed without changing the apps but if not, you might choose to use a feature gate that all systems will use to decide whether to hit the old or the new databases. Alternatively, just change them one-at-a-time after you have tested them on a staging environment.

## Reusing the Current Replica VMs
Although it might seem financially prudent to reuse the existing replica VMs and since postgresql can theoretically run multiple versions alongside each other, there are a number of problems with this approach. Firstly, if you are installing from e.g. Ubuntu packages, they will have a layout that might not necessarily be compatible with side-by-side running. Secondly, to install a newer version of `postgresql` on an older distribution, you are likely to need to add additional repositories to `sources.list`, which will then likely have updates for the older version which might or might not be compatible with something that is already running. Thirdly, almost certainly you will need to stop and/or restart the older version during the upgrades so you could easily end up with a broken VM or at least downtime.

The best option is simply to add a new VM completely to setup the initial primary node, this could be as well as or in place of an existing replica depending on how many replicas you need to achieve your desired HA. Honestly, for the price of most VMs, you could have them for a month for the cost of one SRE for 1 day that might need if you break stuff.

## Scripting the VM installation
One concern about starting from a fresh VM is that it can take time and you might not remember all the steps you took to build the original replicas that are currently working. This is 100% a reason to script the install as much as possible, documenting the rest and we use `Ansible`. I don't know how it compares to `Puppet` or `Chef` but it does the job for us.

When you use Ansible properly, because it is idempotent, you should be able to create an initial script and keep running it against your target, adding in the various extra pieces you need like some support libraries, repmgr, `pg_hba.conf` and the like. Most of these will already be available in your old replica to copy from and you can and should be constantly testing as you go along that the cluster is working as expected. It is perfectly possible to have it working, then to break it with a change that you didn't think was risky (maybe a typo in a config file).

Our script installs `postgresql`, ajusts the data directory to point to an external disk that we add on Azure, which just gives us some performance benefit (by not running it on the OS disk). We add a tweaked `postgresql.conf` which adjusts some of the memory allocations, which are all quite low by default e.g. `shared_buffers`. We also add a `postgresql` Prometheus exporter and a Node exporter for the VM itself for monitoring.

The only thing we need to do up-front, after creating the VM, is to check/configure the external disk (in one of our scale sets, it automatically gets the correctly mounted disk), add a user for ansible with a specific public key and then update our ansible inventory to include the new host. I also ssh from our build agents to the new VM to get past the fingerprint warning message but I suspect I could do something better for that.

## Preparing and implementing logical replication
Once the target VMs have been setup with the newer version of `postgresql` and the logs checked to make sure it is running OK. We can prepare for logical replication.

### Big tables
The logical replication will do an initial data sync after which it will replicate any changes. If your database/certain tables are large, then this initial sync could be a significant network or disk hit on your production system so you might want to test it on a smaller table and see what is happening before deciding how to do it. You can choose only a subset of tables for replication initially and then add the others in bit by bit to reduce this risk but for us, the largest DB is only about 2GB so small enough to manage over a data centre network.

### wal_level
Firstly, the old cluster nodes need to have their `wal_level` changed to `logical` if not already. Previous to v16, you can only logically replicate from the primary cluster so potentially this change only needs to be made there but I like to keep the configurations the same so I will change it on all 3 of mine. They need restarting, which is a small amount of downtime so I did this late at night when usage on our cluster was very low. You should do this first although I think that as long as you do it before you start the replication, it should be fine since I think the initial load of data does not use the logs, just the subsequent updates.

"logical" is the highest level of logging and will take up the most space since it includes "logical" information that allows a newer version to read the logs. However, since we will eventually be decomissioning the old server, we shouldn't need to worry, although if you are tight on disk space you will need to be careful and might want to take a backup and cleanup all old logs after changing over a database.

### schema export/import
I then used `pg_dump` to get the schemas for all of the old databases to run onto the new database. This isn't done by logical replication so needs to be done manually. You will also need to do the roles, which you can do in `pgadmin` if you prefer by right-clicking the objects and "CREATE Script", which you can then run against the new database server. These will be needed if your databases are owned by a specific user other than "postgres" (they should be!) before you create the databases. `pg_dump --schema-only -U postgres -h 127.0.0.1 -d mydatabase > mydatabase.sql` is the command I used. Note that because of the way my auth is setup, I need to use 127.0.0.1 which allows logins whereas without that, it would attempt to use the unix socket which isn't setup for local auth and will fail. Your options might be slightly different. Once this is done, uploading the scripts to the new database can also be done from the command line on the new server `psql -U postgres -h 127.0.0.1 -d mydatabase -f mydatabase.sql`

### replica identity
When using logical replication, the system needs to know how to identify a source row in the target database. The obvious answer would be "by primary key" but, of course, not all tables have a primary key. If your table does NOT have a primary key, you will need to change the Replica Identity setting for that table in the source database to something that will work for your setup. See [REPLICA IDENTITY](https://www.postgresql.org/docs/current/sql-altertable.html#SQL-ALTERTABLE-REPLICA-IDENTITY) for details. If you do not do this, updates won't work during replication.

### Create publication
Setting up the source end is simply enough `CREATE PUBLICATION mydatabase_pub FOR ALL TABLES;` but see [CREATE PUBLICATION](https://www.postgresql.org/docs/16/sql-createpublication.html) if you need to only initially replicate a subset of tables.

### Create subscription
This is also quite simple `CREATE SUBSCRIPTION mydatabase_sub CONNECTION 'host=postgres1 user=repmgr port=5432 dbname=mydatabase' PUBLICATION mydatabase_pub;`. If you don't have the correct permissions with the specified user, you might need to choose a different user, create a new one or modify `pg_hba.conf` to allow the user to connect. Fortunately, `postgresql` will tell you if the problem is e.g. `pg_hba.conf` rather than the more generic "an error ocurred" leaving you guessing!

In my case, the `repmgr` user only had "replication" access, which is not enough to start the subscription so I needed to add a `pg_hba.conf` entry to allow repmgr access to all databases. The old server will eventually go so I am happy about this since it is locked down to a private subnet anyway.

Start with a single small database for testing purposes. If you don't have one, create one temporarily. It is much better to get things working on small tests end-to-end before committing to the larger/more complex databases.

Insert a new row into the old database to ensure it is also replicated across. Of course, there is no reason why it shouldn't but it is best to check, especially if you have been fiddling and might have somehow disabled or broken the replication process.

## Application testing
The drivers for communicating with `postgresql` probably don't change much between major versions but even if you have the latest drivers (npgsql for .Net for us), you will obviously need to make sure there are no issues connecting to the new database. If there are, you might need some updates, tweaks to code or whatever.

Another thing that might be easy to forget is that sometimes older features get removed. These are usually subtle or niche features and for us, fortunately, I know that we don't use anything weird that would be removed. For you, however, you might use all kinds of tips and tricks. I am not sure if there is an easy way to test whether your old database uses any removed features. I guess you might spot them during schema creation or automated tests.

For me, it is a simple case of pointing a local instance of a microservice to the new server and attempting a call that accesses the database. Just to make sure I wasn't missing anything, I queried something that didn't exist and then added a new row to the new database to make sure I was definitely pointing at the correct thing. In software, I believe the saying, "measure twice, cut once" is especially true. Something like a stray hosts file entry or a change you forgot you made to DNS could be the difference between success and failure (or at least a very long time spent debugging)!

## Sequence setting
Remember, you might want to update the sequence values in the new database to match the old, unless the old one is still have inserts applied, in which case you might want to wait until closer to switchover or simply reset them to a value like the old one + some offset.

To get the current value from the old DB, simply run:

 ```
select schemaname as schema, 
       sequencename as sequence, 
       last_value
from pg_sequences
```
 
 To update it in the new database `alter sequence seq_name RESTART WITH new_value;`. There might be a quick way to do this for lots of sequences but I'm not sure - I just used NimbleText to take the output of the query and build the update statements for me. `pg_sequences` theoretically has the last value but it also returned null for me on some sequences, when the direct select statement didn't so I had to manually replace these with 1 in NimbleText. I couldn't work out how to "add 1" to the value in SQL so I had to manually add 1 to all of the numbers for those databases that are not being updated during changeover.

## Backups
This is another thing that can be setup before any switchover is required and is really important to make sure it is up and running for a while before switching over. We use barman to take backups and currently keep 9 from the primary and 5 from a replica on the old cluster. On the new cluster, initially, since it will be a copy of the primary, we can just keep 1 or 2 once setting it up and leaving it to run for a few days. We don't want surprises like large backup sizes/poor disk space etc. which might happen.

One thing I will need to do is to reset the passwords on the new server for barman and streaming_barman which are used by the backup server to connect. I can't remember whether these were created by the script with default passwords or whether there were scripted as part of the schema copy (I can obviously just try and connect from barman and see if I get an error).

You will also need to install the client package e.g. `postgresql-client-16` for any new version of postgres you are accessing. If this is newer than the distribution has in their main repos, you will need to add another repo that does contain it. Be very careful here. The external repo (the postgresql one in my case) is likely to contain all kinds of other updates that might not be tested on your distribution. In this case, I added the repo, did an update, installed *only* the client package for Postgresql 16 and then commented out the repo again!

> Although `barman` can support multiple versions of postgresql, it seems like it cannot automatically determine this, even though postgresql-client-common theoretically can select different versions of tools like `pg_basebackup` but I don't think the mechanisms it uses are compatible with barman, at least not the version I have installed. In that case you need to set the `path_prefix` parameter in your backup configuration to point to the bin directory for the version of postgresql you are using.

> If you are testing this with a rarely updated database, no logs will be written for a while and `barman check backupname` will show "WAL archive: FAILED" for a while (or indefinitely) so you can run `barman switch-xlog --force --archive backupname` to force a log rotation on the source and get the wal streaming working.

Remember that you should also test that a backup works by restoring to a new server. This is really important because if you document and test what you need to do, you won't feel under lots of pressure if a server does die and you need to create a new instance from a backup. It is also recommended that you schedule in e.g. a quarterly test to ensure it still works (or find out your backups aren't running!). Although you can generally test the backup by creating it on a new server and then just connecting to it without needing to bring it online in the cluster, for example, it is also useful to consider what would happen if you somehow lost an entire cluster and wanted to start from scratch with a backup. You might not have this problem though!

## New cluster replication
This is another thing that you could setup for a single server initially to keep costs down but with the intention of adding more replicas once the older cluster is closer to being switched off. The replicas are around £30/month so not exactly bank breaking.

This will use *physical* replication and is something we have done before using `repmgr`, which allows us to use the barman backups to seed the initial database creation (not that important with our small test database but anyway) and then it will setup streaming replication to the new primary. Again, we then test this by connecting to it and also check that if we add something to the new primary (via the old primary!) then it also appears in the new replica table.

This is also a good chance to test your ansible scripts are up to date and check your documentation is also up-to-date, there are lots of small things that can change over time or that you tweaked manually but forgot to add to the script.

Another test that is easier to make now before we are live, is to attempt to the failover from primary to replica, which I have to do roughly monthly for Linux updates. These failovers do create a small window of dataloss/connection issues and are generally done at night but we can just test our new cluster, that isn't in production yet, without worrying.

## Monitoring
It is a good opportunity now to setup some monitoring for the new cluster. We monitor the number of backups, including failed as well as disk space and replica lag, which is really important to know if the replicas are keeping up or not. This monitoring first alerted Gitlab to a problem, which was caused by deleting a very large account synchronously and the time it was taking to replicate each operation onto the secondary. It could also highlight network problems with your provider etc.

## Swapping over
If you have setup a new cluster alongside the old one, you can switchover in stages and how easy it is depends on whether the source database is actively receiving updates and how many.

The discipline here is to try and do everything in an ordered way, after planning, so that you don't break something if you e.g. switched over an app before you have setup its database!

### Apps with static database data
We have a database that is not actively used by its microservice currently, except for a healthcheck table. The data is static and therefore, it was easy enough to setup the replication, re-seed the sequences manually (add 1 to the current value, I used nimbletext to make this easier), updating on the new database e.g. `ALTER SEQUENCE "mytable_id_seq" RESTART WITH 1147865;` and switch the microservice over to point to the new cluster after testing it on our staging environment. Once this was done, I removed the unneeded subscription `drop subscription auth_sub` which also removes the replication slot on the source database server, which was no longer needed. Effectively, the old database is now dead and could be deleted either now or simply when we delete the entire old database server. Deleting it now helps make it clear that it is not used.

Another database was active but was still static data so was treated in the same way.

### Apps with moving data
Only one of our databases is used a lot - the one that sends emails. Although I was able to stop our batch-sending application, there are lots of other scenarios that can trigger emails at any time. In the evenings, the numbers are low, however. All these do is add a message log entry, which we decided was OK if we lost a couple of these but it did make the changeover a little more thought provoking!

We re-seeded the big tables to be "current sequence + 1000" to give us some space.

We then tested the app could communicate via the new database while replication was still running via our staging environment.

I could have left replication running until the changeover but I was nervous about the sequence numbers and what that might mean if two systems were writing to the same table so I disabled it, deployed the update to production, which was about 10 seconds and then checked the old database. We had only lost 3 log entries (we could have copied them over manually but didn't bother).

### Problem apps
We only had an issue with 1 app. Grafana. Grafana can use postgresql as a backend, which is important when running in a container since the sqlite database it uses by default would either need mounting into a volume or it would lose its data when the container was killed. Also, it obviously wouldn't work across multiple instances.

When grafana starts up, it checks the database and if it thinks it needs to, it performs a set of migrations on the database. You can jump multiple versions and it will simply apply all migrations from start to finish. However, even though the old and new databases were the same database, when I tried to switch over Grafana (a production system but we could lose it for a minute or 2 without big problems), it basically fell over trying to run migrations and seemed to leave the database in a broken state. I tried deploying the latest minor version, the next minor version and even redeploying the same version but no dice.

I am not sure whether I broke something. Whether I remembered to reseed the target, whether I left the replication on for too long or whatever. Fortunately, I can leave it where it is for now, pointing at the old database (the only database still there), and start again. I will delete the target instance of grafana, create the schema again, add logical replication and then stop the replication before re-deploying the same instance. Only at that point will I try updating it to a newer version.

## Conclusion
So this was all very doable and now I have done it once, I am much less worried about doing this the next time we upgrade from v16 to something else. A nice improvement in v16 is that we can now take logical replicas from non-primary database servers, which is a nice little way of reducing the load on the primary.

I will monitor the apps for a few days (I think we will know fairly quickly if there is a major problem) and then I will delete most of the old replicas, apart from one for Grafana (until we can migrate) and increase the number of v16 replicas. We use automation so we can treat them like cattle. Simply delete the old ones completely and setup new ones with v16 instead of v14. This gives them a chance to retire old hardware that might be waiting for us to remove VMs and ensures any "damage" we have done manually gets removed.