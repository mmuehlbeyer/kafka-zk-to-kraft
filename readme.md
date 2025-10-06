# Kafka ZK to KRaft 

### Prerequisites

* docker
* confluent platform or kafka binaries installed


### Overview

The setup is made of the following

* 3 ZK nodes 

* 3 Brokers

* 3 KRaft controllers

we start with a "classical" setup, 3 Zookeeper nodes and 3 Brokers
we will then perform the migration step by step while adapting the configs and restarting the brokers.

the steps are taken from the confluent documentation here
https://docs.confluent.io/platform/7.9/installation/migrate-zk-kraft.html


### Starting the environment

#### start zookeeper & broker

```bash
docker-compose -f zk-compose.yml up -d 
```

wait for zookeeper to start succesfully and then start the brokers
broker data gets persisted to ```data```subdir if you need to start over make sure to cleanup

```bash
docker-compose -f broker.yml up -d 
```

check the broker logs if everything is fine & running



#### get the cluster id

as the data is persisted you should be able to get the cluster id from the meta.properties file

```bash
cat data/broker01/meta.properties


broker.id=10
version=0
cluster.id=BACUEAnrS8O3qk8bBJPaBw

```

note down the clusterid 


#### adapt controller.yml 

edit controller.yml and replace CLUSTER_ID value with the value you've obtained above.


#### start the kraft controller
let's start the kraft controllers

```bash
docker-compose -f controller.yml up -d

```
wait a bit and check the controller logs
one of the controller container should log something like

```
[2025-10-02 14:36:09,446] INFO [KRaftMigrationDriver id=300] No brokers are known to KRaft, waiting for brokers to register. (org.apache.kafka.metadata.migration.KRaftMigrationDriver)
```

let's go on with the broker config

#### migrate broker metada from zk to kraft

to achieve this we need to make some modifications in the brokers config
to config for this step is already available in the kafka-extend.yml
 basically remove the comments in line 19-21, 40-42, 60-62 in the broker.yml to include the kafka-extend.yml


file should look like the following after that:


```yml
[...]
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_NUM_PARTITIONS: 3      
    extends:
       file: kafka-extend.yml
       service: kafka      
    volumes:
       - ./data/broker02:/var/lib/kafka/data  
[...]
```

adapt the CLUSTER_ID value in the kafka-extend.yml to match the CLUSTER_ID value from the controller.yml file.
everything else stays the same.


let's start the migration
the following approach was the best working approach for me, it might be necessary to slightly adapt the way you restart each broker.



restart all brokers with

```bash
docker-compose -f broker.yml up -d
```

if you get a "node exists" error just restart the failing again by the above command


wait a bit and then check the kraft controller logs ,you might need to check one after to get the information, in my case it was
controller03

grep the controller logs and check for migration

```bash
docker logs controller03 | grep migration
``
you should hopefully see something like

```txt
[2025-10-02 15:14:43,926] INFO [KRaftMigrationDriver id=300] Completed migration of metadata from ZooKeeper to KRaft
```

####  migrate brokers to use kraft only

at the moment the brokers are running in zookeeper mode
if everything is running fine and the migration has been successuful we can continue and switch to KRaft

therefore we need to make some adoptions to our configs:

remove/comment the following in broker.yml
there are some template files available in templates [subdirectory](./subdirectory) (final files after finishing migration) 

```yml
      #KAFKA_ZOOKEEPER_CONNECT: zookeeper01:2181,zookeeper02:2181,zookeeper03:2181
      #KAFKA_KRAFT_MODE: "false"
```

remove/comment the following in kafka-extend.yml

```yml
      #KAFKA_ZOOKEEPER_METADATA_MIGRATION_ENABLE: true
      #KAFKA_INTER_BROKER_PROTOCOL_VERSION: 3.9
```
add/uncomment the following in kafka-extend.yml

```yml
      KAFKA_KRAFT_MODE: "true"
      KAFKA_PROCESS_ROLES: broker
```




bounce the brokers
```bash
docker-compose -f broker.yml up -d
```


check if everything comes up and is running fine

####  take KRaft controller out of migration mode
remove/comment zookeeper stuff

      # KAFKA_ZOOKEEPER_METADATA_MIGRATION_ENABLE: true
      # KAFKA_ZOOKEEPER_CONNECT: zookeeper01:2181, zookeeper02:2181,zookeeper03:2181

bounce the controller

```bash
docker-compose -f controller.yml up -d
```


check if everything is still running and check broker and controller logs for any errors.

finally stop zookeeper

```bash
docker-compose -f zk-compose.yml down
```

check if everything is still running and check broker logs for errors

finally we've made it and migrated from zookeeper to KRaft! :-)





