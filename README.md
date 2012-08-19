**Current status**

Just a spec for now. I'm still kicking about ideas. The rest is still to come :-)

In the meantime, please give feedback. If you've found this, then you've probably got a specific use case in mind. Let me know what you intend to do and feel free to suggest features/improvements. Message me on GitHub or create a pull request with your suggestions.

# Marshal

Connection grouping, state monitoring and command routing management for one or more Redis servers.

### Groups
A connection group is effectively a replica set of Redis servers comprising a master and any number of slaves replicating that master. Connections to Redis servers can be classified by master, slave or sentinel roles, and added to a group. A single Redis sentinel role can be used to auto-configure a group using sentinel discovery.

### Command routing
The usual Redis commands are issued to the group. Commands will be routed to particular servers using rules:

**Master only**

* Read & Write commands to be sent to the master.
* Optional readonly mode: If master is down, send reads to one or many slaves and fail writes. This way we can still read from slaves even if master is unavailable.

**Read / Write segregation**

* Send writes to the master only.
* Send reads to one or many slaves with weighted round-robin.

### Connection events

Exposes a set of events to inform the app of state transitions:


#### Definitions
**Down** - Redis server is unavailable due to a failure or disconnect.

**Up** - Redis server is up for the first time, or back up after a disconnect or failure.

#### Events

`masterDown`

`masterUp`

`slaveDown`

`slaveUp`

`allSlavesDown`

`allSlavesUp`

### Clusters

Clusters are collections of groups. In much the same way as groups, commands can be sent to a cluster and be routed to groups using rules:

* marshal0: Consistent hashing (striping). Using a hash algo, assign a key to a group and route commands against the key to its assigned group.
* marshal1: Replica (mirroring). Replicate all commands issued to one group to at least one other group.

### Farms
Farms are a collection of clusters.

* marshal10: marshal0 + marshal1. Mirror a marshal0 cluster.

### Usage
````
var marshal = require('marshal');

var master = {port: 6379, host: 'localhost'};
var slave1 = {port: 6380, host: 'localhost'};
var slave2 = {port: 6381, host: 'localhost'};
var sentinel1 = {port: 26379, host: 'localhost'};
var sentinel2 = {port: 26379, host: 'localhost'};

var group = marshal.newGroup({power: 1});
group.setMaster(master)
	.addSlave(slave1, {power: 1})
	.addSlave(slave2, {power: 2})
	.addSentinel(sentinel1);
	
group.use(marshal.mutableRouting);

// Set a key on master
group.set('foo', 'Set me on Master please');

// Get a key from a slave
group.get('bar');
	
// Set up a cluster
var cluster = marshal.newCluster();
cluster.addGroup(group);

// Create a new group using sentinel auto-discovery
var group = marshal.newGroup({power: 2}).addSentinel(sentinel2);
cluster.addGroup(group)

// Split the keyspace across the two groups using marshal0 routing
cluster.use(marshal.stripe);

cluster.set('test', function(err, res){

});

````