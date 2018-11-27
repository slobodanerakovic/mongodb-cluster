# mongodb-cluster : 2015

Configuration of mongodb simple cluster

## Three instance mongodb cluster

Here we are going to provide configuration, for mongodb cluster, consisting of 3 mongodb instances. <br /> Depending on your needs, this can be 5, 7, or else...<br />
I am also going to explain potential issues and mitigation regarding particular configuration, in regard to your applicaiton itself.<br />
I will try to be detailed as needed, and stress pontential issue which could araise during the setup, just as solution how to mitigate them.<br />
******************
<br />

1. Follow up-to date instructions on
http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/

- (If there is an issue during installation about locale language, do :
**sudo nano /etc/default/locale**

and:
LANG="en_US.UTF-8"
LANGUAGE="en_US:en"
LC_ALL="en_US.UTF-8"
)

2. Start mongodb as a service, and create all users (at least root one):<br />
**mongo --port 27017**

<br />Create Root User with your own credentials, which is going to be used for database configuration, from mongodb console<br />
--- Accessible from console, but **NOT** from application ---
```
use admin;
db.createUser(
    {
      user: "code_hunter",
      pwd: "c88b72cda7bc",
      roles: [ "root" ]
    }
)
// This depends on the mongodb version. Prior 3.2 should ask for this.
db.grantRolesToUser ( "root", [ { role: "__system", db: "admin" } ] )
```
3. Exit console and:
Stop service, and update **/etc/mongod.conf** file to use authentication:
**#security:
auth=true**

this would engage authentication during the login into mogo console. We do not want anybody to be able to access our mongo on server via console.

4. Start mongo and you can access via console database using authentication:<br />
```
mongo --port 27017 -u code_hunter -p c88b72cda7bc --authenticationDatabase admin
```
5. Create rest of users, each one with specific permissions, depending on the user's role and needs.
```
use admin;
db.createUser(
    {
      user: "app_hunter",
      pwd: "yxb8cawu9c08",
      roles: [ { role: "readWrite", db: "code_hunter" } ]
    }
)

// More permissions
db.createUser({
    user: "cto_hunter",
    pwd: "w6tcrb7964r5",
    roles: [ "readAnyDatabase" ]
})
```
6. Create all necessary directories for each instance of mongodb(for mongo db. **/var/lib/mongodb** should already be created, so here we create additional two structures since we have a cluster of 3):<br />
mkdir -p **/var/lib/mongodb2**
mkdir -p **/var/lib/mongodb3**

## Authentication on Replica set
Before we start out mongodb cluster, we must point to following thing.<br />
We are going to create specific **replicaSet**, containing three mongo instances. However, we need to setup authentication mechanism, between these three instance, so communication between them (replication, checking heartbeats, etc) can go smoothly and secured.<br /><br />
To achieve that, we need to create a keyfile (The content of the keyfile must be between 6 and 1024 characters long and must be the same for all members of the replica set).
<br /><br />
Use **--keyFile** option from command line, specifying the previously created key file:<br />
```
openssl rand -base64 741 > /srv/mongo/mongodb-keyfile
chmod 600 /srv/mongo/mongodb-keyfile
```
on each instance start. <br />
**keyFile** line from **mongod.conf** file is used for mutual authentication of replica set members, in between!
```
mongodb.conf: keyFile = /home/j2ee/mongodb-keyfile
```
Once started, authentication of client is mandatory during database connection.

7. Create /srv/mongo/startupMongo.sh script (following content):
```
echo "Start three mongodb instances..."

echo "MONGODB 1 --dbpath /var/lib/mongodb --port 27017 --logpath /var/log/mongodb_daemon.log STARTED !!!"
mongod --replSet bot_hunter --dbpath /var/lib/mongodb --port 27017 --keyFile /srv/mongo/mongodb-keyfile --fork --logpath /var/log/mongodb/mongod.log

echo "MONGODB 2 --dbpath /var/lib/mongodb2 --port 27018 --logpath /var/log/mongodb2_daemon.log STARTED !!!"
mongod --replSet code_hunter --dbpath /var/lib/mongodb2 --port 27018 --keyFile /srv/mongo/mongodb-keyfile --fork --logpath /var/log/mongodb2/mongod.log

echo "MONGODB 3 --dbpath /var/lib/mongodb3 --port 27019 --logpath /var/log/mongodb3_daemon.log STARTED !!!"
mongod --replSet code_hunter --dbpath /var/lib/mongodb-arbiter --port 27019 --keyFile /srv/mongo/mongodb-keyfile --fork --logpath /var/log/mongodb-arbiter/mongod.log
```
* keep in mind that, all folders and files, accessed within this script, **MUST** be accessible by the user which is going to invoke this script.<br /> If script is called by **root** then no issue should raise, but if with another one (mongo for example), make sure that mongo user has full access to all  folders and files, accessed within this script.<br />
This will start replica set with 3 mongo instances.

8. Go to the database via console, switch to local db, and clear replica set configuration:
```
mongo --port 27017 -u code_hunter -p c88b72cda7bc --authenticationDatabase admin
use local;
db.system.replset.find();
```
9. Behaviour:
1 - PRIMARY (priority - 10)
2- SECONDARY (priority - 9)
3- SECONDARY (priority - 0, hidden = true)
(this can be changed regarding the mongodb version)

10. Initiate replicaSet and configure members:
```
rs.initiate()
rs.add("j2ee:27018")

#if we want arbitter then :
rs.addArb("j2ee:27019")

#If hidden:
rs.add("j2ee:27019")
```
(Also, we can add arbitrary instead of real node which hidden. The arbitrary, does not holds data or memory, but just serve as a purpose of election. Follow up regarding this at the end of document)<br />
Also, editing an ordinary node to become arbiter:<br />
https://docs.mongodb.com/manual/tutorial/convert-secondary-into-arbiter/
https://docs.mongodb.com/manual/core/replica-set-architecture-three-members/

```
c = rs.conf();
c.members[0].priority = 10
c.members[1].priority = 9

#If arbitter is configured, than this is not needed
c.members[2].priority = 0
c.members[2].hidden = true

rs.reconfig(c)
rs.status();
```

11. Behivour of replicaSet:
- Replication is happening to both SECONDARIES
- Both SECONDARY (priority - 9) and SECONDARY (priority - 0, hidden = true) can be read from, and to both of them replication is happening
- When PRIMARY goes down, SECONDARY with priority 9 becomes PRIMARY and receive write request.
- Replication continues on SECONDARY (hidden)
- When former primarily comes up, then he becomes PRIMARY and get synchronized with a current PRIMARY which goes to his first state as SECONDARY

12. Choosing type of replicaSet member (regular / arbiter / hidden)
Depending on your application nature and usage (does it have only main app client to use it, or there is manual use as well (eg. analitics department), you must pay attention on replicaSet **member type**, and **read/write concerns**.

## member type -> read/write concerns discussion
- **Using arbiter**. <br />In this case, pay attention on replica set, since **arbiter has NO data**, and hence leverage system, but it leaves us with two servers with data. <br />Potential issue is (three member replica): One is master, second slave, third arbiter. If we use majority write (**W2**), in this case, everything works fine, **until master die**. When master dies, **arbiter participate** in election, slave is promoted in master. <br />
Then, majority write, **will not work**, since arbiter holds **NO** data, and **majority of two is NOT one**. SOLUTION:<br />
So, here, we need to switch to default write concern (requests acknowledgement only from the primary), instead of majority write, **OR**, use **hidden member** as well, since hidden **holds data** and participate in election project **BUT** it is invisible to **client applications** and cannot become primary. <br />
Use hidden members for dedicated tasks such as reporting and backups, OR use timeout option, everything depending on the needs.
- **Reading preference.**<br />
Exercise care when specifying read preferences: <br />Nodes other than primary may return stale data because with asynchronous replication, data in the secondary may not reflect the most recent write operations.

