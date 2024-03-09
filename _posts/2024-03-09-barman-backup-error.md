---
layout: post
title: Barman backup error
date: '2024-03-09T11:14:00.000+00:00'
author: Luke Briner
---

## The scenario
Setting up a new Postgresql 16 cluster in addition to an existing Postgres 14 cluster and now setting up Barman to be able to backup from both places. Barman is setup and `barman check postgres-16` comes back all "green" so now I attempt to take a backup.

## The error
```
barman$> barman backup postgres-16

ERROR: Backup failed copying files.
DETAILS: data transfer failure on directory '/sqldata/postgres-16/base/20240307T131134/data'
pg_basebackup error:
pg_basebackup: initiating base backup, waiting for checkpoint to complete
WARNING:  aborting backup due to backend exiting before pg_backup_stop was called
pg_basebackup: error: could not initiate base backup: ERROR:  could not open directory "./main": Permission denied
pg_basebackup: removing contents of data directory "/sqldata/postgres-16/base/20240307T131134/data"
```

Why it's a warning and not an error, I am unsure about.

I have not seen this previously despite having setup a number of barman backups. I also can't find any internet hits on the same problem but one issue is always that e.g. "./main" might not be the same directory name as for other people so I can't search for an exact match.

## The problem
There was a stray folder in the *source* database data directory and it was owned by root so was not accessible by postgres. Annoyingly, the message shows a relative path which isn't too helpful since it is not clear where "./main" is relative to. Also, annoyingly, the name "main" is also the same as the name of the cluster.

## Debugging
My main problem is usually not reading the error messages carefully enough but there was definitely nothing I could work out so:

1. I needed to attempt `pg_basebackup` on the source database server since this is what barman was trying to run. This would immediately tell me whether the problem was in the source server or whatever it was barman/network config that was wrong. This also failed with the same error (and also didn't tell me where this folder was relative to). This "reduction" technique to a smaller set of variables is a cornerstone of debugging and something that not everyone thinks to do.

2. I did an `ls -l` on the data directory for the database. I could see that all the files seemed to be correctly owned by the `postgres` user except one! One called "main". I don't know where it came from, possibly from Ansible, which might explain why it was owned by root and not by postgres. This might also explain why it is not a common error on the internet since it is unlikely to normally happen.

The folder was empty and not required but its presence was enough to trip over `pg_basebackup` so all I had to do was delete it and everything worked perfectly!