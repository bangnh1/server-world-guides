# How to setup mongodb with ReplicationSet and sharding

## Table of Contents

1. [Servers](#Servers)
2. [Install mongoDB replicationSet cluster](#Install-mongoDB-replicationSet-cluster)
   - [Disable senlinux](#Disable-senlinux)
   - [Install MongoDB](#Install-mongoDB)
   - [Start MongoDB](#Start-mongoDB)
   - [Initialization mongoDB cluster](#Initialization-mongoDB-cluster)
   - [Check replicationSet](#Check-replicationSet)
   - [Create keyfile in primary server and copy it to secondary servers](#Create-keyfile-in-primary-server-and-copy-it-to-secondary-servers)
   - [Create and populate a new collection](#Create-and-populate-a-new-collection)
3. [Migrate from replicationSet cluster to sharded cluster](#Migrate-from-replicationSet-cluster-to-sharded-cluster)
   - [Restarts secondary and primary members with --shardsvr option](#Restarts-secondary-and-primary-members-with---shardsvr-option)
   - [Deploy mongos](#Deploy-mongos)
4. [Add second shards](#Add-second-shard)

## Servers

1. mongo1: 2 vCPU 4GB memory 100GB SSD, Centos 8, ip address: 10.0.2.138
2. mongo2: 2 vCPU 4GB memory 100GB SSD, Centos 8, ip address: 10.0.2.139
3. mongo3: 2 vCPU 4GB memory 100GB SSD, Centos 8, ip address: 10.0.2.140

## Install mongoDB replicationSet cluster

### Disable senlinux

```
$ sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
$ sestatus
$ systemctl reboot 5
```

### Install MongoDB

```
$ vi /etc/yum.repos.d/mongodb-org-5.0.repo
[mongodb-org-5.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/5.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-5.0.asc

$ yum install -y mongodb-org
$ vi /etc/hosts
10.0.2.138 mongo1
10.0.2.139 mongo2
10.0.2.140 mongo3
```

```
// 10.0.2.138
// 10.0.2.139
// 10.0.2.140
$ vi /etc/mongod.conf
```

```
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# Where and how to store data.
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true
#  engine:
#  wiredTiger:

# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.


#security:

#operationProfiling:

#replication:
replication:
  oplogSizeMB: 1
  replSetName: "mongo_rs"

#sharding:
sharding:
   clusterRole: shardsvr


## Enterprise-Only Options

#auditLog:

#snmp:
```

### Start MongoDB

```
// 10.0.2.138
// 10.0.2.139
// 10.0.2.140
$ systemctl enable mongod
$ systemctl start mongod
```

### Initialization mongoDB cluster

```
// 10.0.2.140
$ mongo
MongoDB shell version: 5.0
> rs.initiate()
> rs.add("mongo1:27017")
> rs.add("mongo2:27017")
> var cfg = rs.conf()
> cfg.members[0].host = "mongo3:27017"
> rs.reconfig(cfg)
mongo_rs:PRIMARY> use people
mongo_rs:PRIMARY> db.createCollection('people')
mongo_rs:PRIMARY> db.getCollection("people").insert({    "firstName": "test",    "lastName": "tech",    "started": NumberInt("2020")})
```

### Check replicationSet

```
// 10.0.2.138
$ mongo
mongo_rs:SECONDARY> rs.secondaryOk()
mongo_rs:SECONDARY> show dbs
mongo_rs:SECONDARY> use people
mongo_rs:SECONDARY> show collections
mongo_rs:SECONDARY> db.people.find()
```

### Create keyfile in primary server and copy it to secondary servers

```
// 10.0.2.140
$ openssl rand -base64 741 > /opt/mongo/mongodb-keyfile
$ chmod 600 /opt/mongo/mongodb-keyfile
$ rsync -avz /opt/mongo/mongodb-keyfile centos@10.0.2.138:/opt/mongo
$ rsync -avz /opt/mongo/mongodb-keyfile centos@10.0.2.139:/opt/mongo
```

```
// 10.0.2.138
// 10.0.2.139
// 10.0.2.140
$ vi /etc/mongodb.conf
security:
  authorization: enabled
  keyFile: /opt/mongo/mongodb-keyfile
$ service mongod restart
```

### Create admin user and super admin user

```
// 10.0.2.140
$ mongo
mongo_rs:PRIMARY> use admin
mongo_rs:PRIMARY> db.createUser({ user: "admin", pwd: "adminpassword", roles: [{ role: "userAdminAnyDatabase", db: "admin" }] })Successfully added user: {        "user" : "admin",        "roles" : [                {                        "role" : "userAdminAnyDatabase",                        "db" : "admin"                }        ]}

// Check if user already
mongo_rs:PRIMARY> db.auth("admin","adminpassword")
mongo_rs:PRIMARY> exit
```

```
// 10.0.2.140
$ mongo --port 27017 -u "admin" -p "adminpassword" --authenticationDatabase "admin"
mongo_rs:PRIMARY> use admin
mongo_rs:PRIMARY> db.createUser({ user: "superadmin", pwd: "superpassword", roles: ["root"] })
Successfully added user: { "user" : "superadmin", "roles" : [ "root" ] }
exit
$ mongo --port 27017 -u "superadmin" -p "superpassword" --authenticationDatabase "admin"
mongo_rs:PRIMARY> rs.status()
```

### Create and populate a new collection

```
use test
var bulk = db.test_collection.initializeUnorderedBulkOp();
people = ["Marc", "Bill", "George", "Eliot", "Matt", "Trey", "Tracy", "Greg", "Steve", "Kristina", "Katie", "Jeff"];
for(var i=0; i<1000000; i++){
   user_id = i;
   name = people[Math.floor(Math.random()*people.length)];
   number = Math.floor(Math.random()*10001);
   bulk.insert( { "user_id":user_id, "name":name, "number":number });
}
bulk.execute();

```

## Migrate from replicationSet cluster to sharded cluster

### Restarts secondary and primary members with --shardsvr option

```
// 10.0.2.138
// 10.0.2.139
$ vi /etc/mongodb.conf
sharding:
   clusterRole: shardsvr
$ service mongod restart
```

```
// 10.0.2.140
$ rs.stepDown()
$ vi /etc/mongodb.conf
sharding:
   clusterRole: shardsvr
$ service mongod restart
```

### Deploy mongos

```
$ mkdir -p /mongos/configdb /mongos/db /var/log/mongodb/mongos /var/run/mongos
$ chown -R mongod:mongod /mongos/configdb /mongos/db /var/log/mongodb/mongos /var/run/mongos
```

```
// 10.0.2.138
// 10.0.2.139
// 10.0.2.140
$ vi /mongos/configdb/mongod.conf
# mongod.conf

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongos/mongod.log

# Where and how to store data.
storage:
  dbPath: /mongos/db
  journal:
    enabled: true

# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongos/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
  port: 27019
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.


#security:
security:
  authorization: enabled
  keyFile: /opt/mongo/mongodb.key

#replication:
replication:
  oplogSizeMB: 1
  replSetName: "rs0"

#sharding:
sharding:
    clusterRole: shardsvr
```

```
// 10.0.2.138
// 10.0.2.139
// 10.0.2.140
$ vi /mongos/configdb/mongos.conf
# mongos.conf

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongos/mongos.log

# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongos/mongos.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
  port: 27020
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.

#security:
security:
  keyFile: /opt/mongo/mongodb.key

#sharding:
sharding:
    configDB: rs0/mongo1:27019,mongo2:27019,mongo3:27019
```

```
// 10.0.2.138
// 10.0.2.139
// 10.0.2.140
$ mongod --config /mongos/configdb/mongod.conf
# mongo --port 27019
> rs.initiate()
> rs.add("mongo1:27019")
> rs.add("mongo2:27019")
> var cfg = rs.conf()
> cfg.members[0].host = "mongo3:27019"
> rs.reconfig(cfg)

```

```
// 10.0.2.138
// 10.0.2.139
// 10.0.2.140
$ mongos --config /mongos/configdb/mongos.conf
```

```
// 10.0.2.140
$ mongo --port 27020
> sh.addShard( "mongo_rs/mongo1:27017,mongo2:27017,mongo3:27017" )
```

## Add second shards

Create a new replicationSet and addShard to the cluster

```
// 10.0.2.140
$ mongo --port 27020
> sh.addShard( "rs1/mongo1:27018,mongo2:27018,mongo3:27018" )
```

```
> sh.status()
--- Sharding Status ---
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("61128ce723a9630763f9563d")
  }
  shards:
        {  "_id" : "mongo_rs",  "host" : "mongo_rs/mongo1:27017,mongo2:27017,mongo3:27017",  "state" : 1,  "topologyTime" : Timestamp(1628610866, 1) }
        {  "_id" : "rs1",  "host" : "rs1/mongo1:27018,mongo2:27018,mongo3:27018",  "state" : 1,  "topologyTime" : Timestamp(1628655108, 5) }
  active mongoses:
        "5.0.2" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled: yes
        Currently running: yes
        Failed balancer rounds in last 5 attempts: 0
        Migration results for the last 24 hours:
                102 : Success
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                mongo_rs	922
                                rs1	102
                        too many chunks to print, use verbose if you want to force print
        {  "_id" : "people",  "primary" : "mongo_rs",  "partitioned" : true,  "version" : {  "uuid" : UUID("5b973967-abef-4f76-9515-de8128300183"),  "lastMod" : 1 } }
        {  "_id" : "test",  "primary" : "mongo_rs",  "partitioned" : true,  "version" : {  "uuid" : UUID("cb0b7f54-5a0b-4579-be5a-c99356d4b162"),  "lastMod" : 1 } }
```
