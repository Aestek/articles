# Scaling consul

Criteo has a pretty uncommon service landcape for Consul: we have a lot of services ?get_num, but more importantly we have services with large number of instances.

This is where our highest load spikes comes from: a service with ~800 instances watching a service with ~600 instances. The watcher, with its many instances, creates a steady stream of requests to the Consul servers, and each request is costly due to the amount of data that need to be returned.

Additionnaly a change on a single instance will cause Consul to notify every watcher.

## Consul watches

Consul's interface for being notified of service changes is blocking queries (long polling) :

1. The client makes a first request for the service it wishes to watch
2. Consul replies with the service instances, and an index
3. The client makes the same request but this time passes the index it received. The request now blocks until a change happens to the service.
4. Consul replies with the service instances, and the new index
5. GOTO step 3

### Under the hood

Consul uses an immutable radix tree to store its data. Nodes from the tree can be watched, this is the basis for blocking queries.

On the server side, the following happen during a blocking query :

1. First `WatchSet` object is created, this will allow to block until change happen.
2. Then the tree is queried for the nescessary data, and nodes from the tree are added the the watchset. The index for the service is also computed.
3. If the index is greater than the one the client gave us, we return the data and the new index. If no change happened we use the watchset to block until a change happens on any node from the watchset
4. GOTO step 1

## 682

Two weeks ?check before last year's black friday, we have a Consul incident. In our two biggest datacenters, Consul servers have trouble responding to requests. Their load is abnormally high and clients cannot registering their services to Consul. The timeouts increase and soon the servers fail to ping each others, the clusters lose their leaders and are too loaded to elect a new one.

We have lost Consul.

After a night of intervention we finally put the clusters back up by stopping service registrations and slowly ramping the servers back up. Consul is running again but the load is sill high.

A few days later ?check it happens again. While we apply the same method as before to try and restore service we profile the servers using Golang pprof.
Most of the CPU time is spent in point 2. (querying the tree). We dive into the code, looking for optimizations we can make to reduce the load and notice something...

### A bit more on watch sets

Watching nodes from the radix makes use of Golang's [goroutines](https://tour.golang.org/concurrency/1) and [channels](https://tour.golang.org/concurrency/2). Each node has a channnel assiociated with it, and when the node or its children change the channel is closed, waking up any goroutine blocking on it ([example](https://play.golang.org/p/NTrB2ILQ9N2)).

For health query, the nodes added to the watchset include: one node per service instance, one node per instance check and one node for the server the instances runs on.

While Consul has an optimization that allows one goroutine to watch 32 channels, this can still result in a large number of goroutines being spawned, especially since this happens for each client request.

There is however a mechanism to limit the number of goroutines: `WatchSet` has a `WatchWithLimit()` function that falls back on a more coarse grained node (remember that a watch on a node will also trigger if any of its chilren are modified) if watching each individual node would result in too much goroutines being spawned.

### Back to our issue

We notice that for health queries, the "fallback nodes" described above are the root of services, checks and nodes. This means that when reaching the watch limit, a health query no longer blocks on change from a single service, but on changes from **any** service, check, or node. This is not visible from clients however because the service index does not change, so the tree is queried over and over until a change finally happens.

The limit is 2048. Could we have reached it ? This does not seem to match: our biggest service in affected datacenters only has 690 instances ?check. Remember a health query watches the service instance, the checks, and the node ? This means that the limit is not 2048, but *682* (2048 / 3) !

We check and *indeed* new servers were provisioned in preparation for upcomming backfriday, and we just crossed it. Having depleted our other options we take a shot in the dark: we shutdown enough servers from our biggest service to get back under the limit. After a few minutes of looking anxiously at grafana dashboards the load gets back to normal and service is completely restored. Success !

### What next

We obvisouly cannot stay with a reduced number of instances for one of our most important service, black friday is coming. Having confirmed with more tests that the load problem was indeed caused by this 2048 / 682 limit, we patch consul to allow configuring it, make a PR upstream and deploy this new version in production, with a value of 8192. We ramp back up the servers we shut down, everything stays green, and we pass the back friady without a bleep.
