# Cassandra
Cassandra is massively linearly scalable NoSQL database.

  - Fully distributed, with no single point of failure
  - Free and open source
  - Highly performant, with near-linear horizontal scaling in proper use cases

### Cassandra Query Language(CQL)
Cassandra models data using [CQL](https://docs.datastax.com/en/cql/3.0/cql/cql_reference/cqlCommandsTOC.html).  
CQL provides a familiar, row-column, SQL-like approach.
  * CREATE, ALTER, DROP
  * SELECT, INSERT, UPDATE, DELETE

```SQL
CREATE TABLE Performer (
  name VARCHAR,
  country VARCHAR,
  style VARCHAR,
  born INT,
  died INT,
  PRIMARY KEY (name)
);
```

## Internal architecture

### Cluster
Cluster is a peer to peer (no master, no slaves) set of nodes.
  * **Node** - one Cassandra instance
  * **Rack** - a logical set of of physically related nodes (availability zone)
  * **Datacenter** - a logical set of racks
  * **Cluster** - the full set of nodes which map to a single complete token ring

![alt text](/img/cassandra-cluster.jpg "Cassandra cluster")

### Coordinator

Coordinator is the node chosen by the client to receive a particular read or write request to its cluster.  
**Any** node can coordinate **any** request.

  * The coordinator manages the Replication Factor
  * The coordinator applies the Consistency Level

![alt text](/img/cassandra-coordinator.jpg "Cassandra coordinator")

### Consistent hashing

Data is stored on nodes in partitions, each identified by a unique token.
  * **Partition** - a storage location on a node (analogous to a "table row")
  * **Token** - integer value generated by a hashing algorithm, identifying a partition's location within a cluster

#### What is the partitioner?
Imagine a 0 to 100 token range (instead of -2^63 to +2^63).

  * Each node is assigned a token, just like each of its partitions

A node's partitioner hashes a token from the partition key value of a write request.

Ex:
```SQL
CREATE TABLE Users (
    firstname text,
    lastname text,
    level text,
    PRIMARY KEY ((lastname, firstname))
);
```
Here ```(lastname, firstname)``` is *partition key*
```SQL
INSERT INTO Users (firstname, lastname, level)
    VALUES ('Oscar', 'Orange', 42);
```
What happens:
![alt text](/img/cassandra-partitioner.jpg "Cassandra partitioner")

### Data replication
**Replication factor** - how many replicas(copies of the data) to make of each partition.  
Replication factor is configured when a keyspace is created.
  * **SimpleStrategy** - one factor for entire cluster
    
    ```SQL
    CREATE KEYSPACE simple-demo
    WITH REPLICATION =
        {'class':'SimpleStrategy', 'replication_factor':2}
    ```
  * **NetworkTopologyStrategy**  - separate factor for each data center in cluster
    
    ```SQL
    CREATE KEYSPACE simple-demo
    WITH REPLICATION =
        {'class':'NetworkTopologyStrategy', 'dc-east':2, 'dc-west':3}
    ```

### Consistency

  * **Consistency Level** - sets how many of the nodes to be sent a given request must *acknowledge* that request, for a response to be returned to the client
  * **Write request** - how many nodes must acknowledge they received and wrote the write request
  * **Read request** - how many nodes must acknowledge by sending their most recent copy of the data
  * **Immediate Consistency** - reads always return the most recent data
  * **Eventual Consistency** - reads may return stale data

In Cassandra you can set up *Consistency Level (CL)* for every request.
```Java
if (nodes_written + nodes_read) > RF { immediate consistency }
```
Some of the available *CL*

| Name | Description |
| --- | --- |
| ANY (writes only) | Write to any node. |
| ALL | Check all nodes. Fail if any is down. |
| ONE (TWO, THREE) | Check closest node to coordinator. |
| QUORUM | Check quorum (RF/2 + 1) of available nodes. |

In Cassandra clock synchronization across nodes is critical because
  * Every write to any column includes column name, column value, and timestamp since epoch (1/1/70)
  * The most recently written data is returned to the client

### Tools

  1. **Nodetool** - a command-line cluster management utility

    [Nodetool](https://docs.datastax.com/en/cassandra/2.1/cassandra/tools/toolsNodetool_r.html) supports over 60 commands, including

    * status - display cluster state, load, host ID, and token
    * info - display node memory use, disk load, uptime, and similar data
    * ring - display node status and cluster ring state
  
  2. [**cqlsh**](https://docs.datastax.com/en/cql/3.0/cql/cql_reference/cqlshCommandsTOC.html) - an interactive, command-line CQL utility
  
  3. Cassandra Cluster Manager ([CCM](https://github.com/pcmanus/ccm/blob/master/README.md))
  
    Creates and manages multi-node clusters on a local machine.  
    It's useful for configuring development and test clusters

[Here](https://docs.datastax.com/en/cassandra/2.1/cassandra/tools/toolsTOC.html) are all the available Cassandra tools.

## Demo

**Requirements**
  * Installed Cassandra. [How to install and run](https://wiki.apache.org/cassandra/GettingStarted).
  * Installed ccm. [How to install](https://github.com/pcmanus/ccm/blob/master/README.md).

You can download prepared VM with Cassandra and CCM [here](https://academy.datastax.com/courses/ds201-cassandra-core-concepts). You need to be logged in.

Create a cluster with cassandra version `2.1.2`  
```
ccm create demo -v 2.1.2
```  
Populate the cluster with 4 nodes  
```
ccm populate -n 4
```  
Start all the nodes  
```
ccm start
```  
See cluster status  
```
ccm status
```  
```
Cluster: 'demo'
---------------
node1: UP
node3: UP
node2: UP
node4: UP
```
Connect to node1 with cqlsh
```
ccm node1 cqlsh
```
Now node1 is coordinator.

Create a keyspace with RF 2
```
cqlsh> CREATE KEYSPACE demo WITH REPLICATION = {'class':'SimpleStrategy', 'replication_factor':2};
```
Use `demo` keyspace
```
cqlsh> use demo;
```
Crete `employees` table in `demo` keyspece
```SQL
cqlsh:demo> CREATE TABLE employees (
                status text,
                job text,
                id int,
                name text,
                hired timestamp,
                PRIMARY KEY ((status, job), id));
```
Here *PRIMARY KEY* contains `status`, `job` and `id` columns.  
*PARTITION KEY* is `(status, job)`.  
*CLUSTERING KEY* is `id`.  
[Here](http://stackoverflow.com/questions/24949676/difference-between-partition-key-composite-key-and-clustering-key-in-cassandra#answer-24953331) is described difference between *PRIMARY KEY*, *PARTITION KEY* and *CLUSTERING KEY*.

Insert a record
```
cqlsh:demo> INSERT INTO employees(status, job, id, name, hired) 
                VALUES('active', 'developer', 123, 'John', '2012-10-30');
```
```
cqlsh:demo> SELECT * FROM employees;
```
```
 status | job       | id  | hired                    | name
--------+-----------+-----+--------------------------+------
 active | developer | 123 | 2012-10-30 00:00:00-0700 | John
```
Let's find on which nodes partition `active:developer` is located
```
ccm node1 nodetool ring
```
```
Datacenter: datacenter1
==========
Address    Rack        Status State   Load            Owns                Token                                       
                                                                          4611686018427387904                         
127.0.0.1  rack1       Up     Normal  71 KB           ?                   -9223372036854775808                        
127.0.0.2  rack1       Up     Normal  71.03 KB        ?                   -4611686018427387904                        
127.0.0.3  rack1       Up     Normal  71 KB           ?                   0                                           
127.0.0.4  rack1       Up     Normal  71 KB           ?                   4611686018427387904    
```
| node | from | to |
| --- | --- | --- |
| node1 | 4611686018427387904 | -9223372036854775808 |
| node2 | -9223372036854775808 | -4611686018427387904 |
| node3 | -4611686018427387904 | 0 |
| node4 | 0 | 4611686018427387904 |


```
cqlsh:demo> SELECT token(status, job), status, job, id, hired, name FROM employees;
```
```
 token(status, job)   | status | job       | id  | hired                    | name
----------------------+--------+-----------+-----+--------------------------+------
 -1776950073870789224 | active | developer | 123 | 2012-10-30 00:00:00-0700 | John
```
So the partiotion `active:developer` has token `-1776950073870789224`.  
First replica of the partition is located on *node3* and the second one on *node4*.

Stop *node3* using ccm
```
ccm node3 stop
```
Let's check the CL
```
cqlsh:demo> CONSISTENCY
```
```
Current consistency level is 1.
```
```
cqlsh:demo> SELECT * FROM employees WHERE status='active' AND job='developer';
```
```
 status | job       | id  | hired                    | name
--------+-----------+-----+--------------------------+------
 active | developer | 123 | 2012-10-30 00:00:00-0700 | John
```
Let's change the CL to ALL
```
cqlsh:demo> CONSISTENCY all
```
```
Consistency level set to ALL.
```
```
cqlsh:demo> SELECT * FROM employees WHERE status='active' AND job='developer';
```
We get error with message `"Cannot achieve consistency level ALL" info={'required_replicas': 2, 'alive_replicas': 1, 'consistency': 5}`. To perform this request with `CL = ALL` all the replicas(2) have to be alive.

Let's change the CL back to ONE
```
cqlsh:demo> CONSISTENCY one
```
```
Consistency level set to ONE.
```
Stop *node4*
```
ccm node4 stop
```
```
cqlsh:demo> SELECT * FROM employees WHERE status='active' AND job='developer';
```
We get error with message `"Cannot achieve consistency level ONE" info={'required_replicas': 1, 'alive_replicas': 0, 'consistency': 1}`. To perform this request with `CL = ONE` at least one replica has to be alive.

Start *node4*
```
ccm node4 start
```
Insert another record
```
cqlsh:demo> INSERT INTO employees(status, job, id, name, hired) 
                VALUES('active', 'developer', 1234, 'Dan', '2013-11-15');
```
If a node is down coordinator stores all mutations locally in `system.hints`
```
cqlsh:demo> SELECT * FROM system.hints;
```
When node is up all the mutations are sent to it.
```
ccm node3 start
ccm node4 stop
```
```
cqlsh:demo> SELECT * FROM system.hints;
```
```
cqlsh:demo> SELECT * FROM employees WHERE status='active' AND job='developer';
```
```
 status | job       | id   | hired                    | name
--------+-----------+------+--------------------------+------
 active | developer |  123 | 2012-10-30 00:00:00-0700 | John
 active | developer | 1234 | 2013-11-15 00:00:00-0800 |  Dan
```

### Check out Datastax cources
  1. [Cassandra Core Concepts](https://academy.datastax.com/courses/ds201-cassandra-core-concepts)
  2. [Data Modeling](https://academy.datastax.com/courses/ds220-data-modeling)

\* You need to be logged in to see the cources (it's free).
