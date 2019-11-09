---
title: "Replicating a MySQL RDS Instance in Another VPC Within the Same Region"
date: 2019-03-05T14:34:40Z
tags: [aws, rds, mysql, vpc, replica]
---

## Challenge

Within the same AWS region, I want to create a MySQL replica in a different
VPC, where the source is the MySQL primary.

I need this to happen without any interruption to the current MySQL primary
instance.

This assumes that both instances will be in private subnets.

Whereas multi-region replicas are available out-of-the-box in RDS, creating
replicas in different VPCs within the same region is not automated.

## Solution

MySQL replication works by copying binary logs from the MySQL primary instance,
and running queries against the replica instance. To set up replication in an
instance with a lot of data, we need to copy the data, and mark the position
where this data ended to mark where we should begin replicating the latest
data.

[Create a new read replica against the primary instance][1]. Read replicas
can be created without downtime.

Connect to the new read replica, and stop replication. We do not have the
MySQL permissions to do this directly, but we can call an RDS specific
stored procedure:

```
CALL mysql.rds_stop_replication;
```

Find the log coordinates at this point:

```
SHOW SLAVE STATUS \G
```

The lines we require are:

```
master_log_file
master_log_pos
```

Make a copy of these values for safe keeping.

[Create a snapshot of the new replica][2].

[Provision a new instance from the snapshot into the VPC you wish to replicate
to][3].

On the primary instance, connect to the MySQL console, and set up a user
that will be used for replication:

```
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO
'repl_user'@'mydomain.com' IDENTIFIED BY '<password>';
```

On the new replica instance in the new VPC, open up the MySQL console and set
up MySQL replication using the details we fetched in a previous step.

To find the primary instance address, ping the primary instance to discover
the private IP.

```
CALL mysql.rds_set_external_master ('<primary ip>', 3306, 'repl_user',
'<password>', '<master_log_file value>', <master_log_pos value>, 0);
```

Start replication:

```
CALL mysql.rds_start_replication;
```

View status with:

```
SHOW SLAVE STATUS \G
```

At this point, replication should be ready, but will not work if VPC peering
has not yet been configured. Please see Amazon documentation on [creating a
VPC peer][4] and [setting up route tables][5].

The read replica that we created to take the snapshot of can be deleted.

If you need to create read replicas from the new instance, backups should be
enabled (they are disabled by default).

Security groups should also be set up to allow communication between the new
replica and the primary instance.

[1]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html
[2]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateSnapshot.html
[3]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RestoreFromSnapshot.html
[4]: https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html
[5]: https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-routing.html
