---
title: To get couchdb installed on debian 12 and build a 3 node x 2 zone cluster
layout: default
parent: Community Guides
---

## Comment out CDROM from source list for apt
``` 
nano /etc/apt/sources.list 
#deb cdrom:[Debian GNU/Linux 12.7.0 _Bookworm_ - Official amd64 DVD Binary-1 with firmware 20240831-10:40]/ bookworm contrib main non-free-firmware
```


### update and install necessary packages for convenience 
```
apt update && apt install -y curl apt-transport-https gnupg jq wget net-tools nmap tcpdump
```

### add gpg keys for couch repo
```
curl https://couchdb.apache.org/repo/keys.asc | gpg --dearmor | tee /usr/share/keyrings/couchdb-archive-keyring.gpg >/dev/null 2>&1
```
### Lets configure the couch repo to use that key and set urls
```
source /etc/os-release
echo "deb [signed-by=/usr/share/keyrings/couchdb-archive-keyring.gpg] https://apache.jfrog.io/artifactory/couchdb-deb/ ${VERSION_CODENAME} main" \
    | tee /etc/apt/sources.list.d/couchdb.list >/dev/null
```
### fix /etc/default/couchdb file to comment out erl line
```
nano /etc/default/couchdb  
#ERL_EPMD_ADDRESS=127.0.0.1
```
### edit the local.ini file to set bind address and set a password in the admins section
```
nano /opt/couchdb/etc/local.ini     
{[httpd]}
;port = 5984
;bind_address = 127.0.0.1
uncomment and set bind address to either 0.0.0.0 or whatever interface ip you want it to listen on.

[admins]
;admin = admin2024
uncomment admin line and set password to something reasonably secure.
```

## Generate erlang cookie, update vm.args to set name, cookie, and comment out -kernel line
```
erlangCookie=$(dd if=/dev/urandom bs=30 count=1 | base64)
cat $erlangCookie
DHeNUhLSO6eL9+VKoWkHJpSLbV0fxyYP2noMT0Am

nano /opt/couchdb/etc/vm.args       
-name couchdb@c1.zone1.example.com
-setcookie ='DHeNUhLSO6eL9+VKoWkHJpSLbV0fxyYP2noMT0Am'  
comment out -kernel inet_dist_use_interface line
#-kernel inet_dist_use_interface {127,0,0,1}
```

## create cluster config for zones
```
nano /opt/couchdb/etc/local.d/10-cluster-zones.ini   
add: 
[cluster]
placement = zone1:2,zone2:2  

the placement setting may need to be adjusted depending on your performance desires and risk tolerance.
For low risk with good performance, something like zone1:3,zone2:1 or zone1:3:zone2:2 might be desired. 
Or for higher performance and higher risk, maybe 2:1. Or for the paranoid, 3:3
```
## make sure all hosts are reachable via name, best to use /etc/hosts as dns lookups are slower
```
nano /etc/hosts and add entry for each and every node in each zone, use your ip addresses obviously, and make sure they are all pingable
1.0.0.1 c1.zone1.example.com c1.zone1
1.0.0.2 c2.zone1.example.com c2.zone1
1.0.0.3 c3.zone1.example.com c3.zone1
2.0.0.1 c1.zone2.example.com c1.zone2
2.0.0.2 c2.zone2.example.com c2.zone2
2.0.0.3 c3.zone2.example.com c3.zone2
```

## restart couchdb 
```
systemctl restart couchdb
```
## check that the node you are on is running, and is shown in all_nodes and cluster_nodes
```
curl -u "admin:admin2024" http://localhost:5984/_membership | jq .       <- check that its returning itself in all_nodes and cluster_nodes and then continue
```
If that is correct then continue.

## add the rest of the nodes to the cluster and into their associated zones
```
curl -u "admin:admin2024" http://localhost:5984/_node/_local/_nodes/couchdb@c2.zone1.example.com -X PUT -d '{"zone":"zone1"}'
curl -u "admin:admin2024" http://localhost:5984/_node/_local/_nodes/couchdb@c3.zone1.example.com -X PUT -d '{"zone":"zone1"}'
curl -u "admin:admin2024" http://localhost:5984/_node/_local/_nodes/couchdb@c1.zone2.example.com -X PUT -d '{"zone":"zone2"}' 
curl -u "admin:admin2024" http://localhost:5984/_node/_local/_nodes/couchdb@c2.zone2.example.com -X PUT -d '{"zone":"zone2"}' 
curl -u "admin:admin2024" http://localhost:5984/_node/_local/_nodes/couchdb@c3.zone2.example.com -X PUT -d '{"zone":"zone2"}' 
```

## Now lets fix the primary nodes zone placement and get the document rev number for later
```
curl -u "admin:admin2024" -s http://localhost:5984/_node/_local/_nodes/couchdb@c1.zone1.example.com  <- must get the rev number so we can move this node into the proper zone
```

## Now that we have the rev number lets complete the zone config
```
curl -u "admin:admin2024" -s http://localhost:5984/_node/_local/_nodes/couchdb@c1.zone1.example.com?rev=rev_number_above -d '{"zone":"zone1"}' -X PUT
```

## Create the users and replicator databases
```
curl -u "admin:admin2024" -X PUT http://localhost:5984/_users
curl -u "admin:admin2024" -X PUT http://localhost:5984/_replicator
```

## create a database and document to test database replication
```
curl -u "admin:admin2024" -X PUT http://localhost:5984/testdb/ -d "{"mydata":"test"}"
copy docid and paste below

curl -u "admin:admin2024" http://localhost:5984/testdb/docid

make sure it returns "{"mydata":"test"}" , and check all nodes to ensure that database has replicated.

curl -u "admin:admin2024" http://c1.zone2.example.com:5984/testdb/docid
curl -u "admin:admin2024" http://c2.zone2.example.com:5984/testdb/docid
curl -u "admin:admin2024" http://c1.zone2.example.com:5984/testdb/docid
curl -u "admin:admin2024" http://c2.zone1.example.com:5984/testdb/docid
curl -u "admin:admin2024" http://c3.zone1.example.com:5984/testdb/docid
```
If all that returns the same your databases are being replicated and you can continue with your system installation.
