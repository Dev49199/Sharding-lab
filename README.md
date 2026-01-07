# MongoDB Sharding -- Hands-on Local Lab

This repository contains a **hands-on MongoDB sharding lab** created to
understand sharding **practically**, not just conceptually.

The goal of this lab is to observe: - how data is distributed across
shards\
- how shard keys affect query routing
- how queries behave in a sharded setup
- how rebalancing works when new shards are added

This exercise was done as a continuation of learning backend
fundamentals after exploring Redis caching.

------------------------------------------------------------------------

## What This Lab Covers

-   MongoDB sharded cluster setup (local, Docker-based)
-   Config server, shard servers, and mongos router
-   Shard key selection and its impact
-   Chunk creation and movement
-   Targeted vs scatter-gather queries
-   Adding a new shard and observing rebalancing

------------------------------------------------------------------------

## Architecture Overview

    Client
      |
    mongos (Query Router)
      |
    ---------------------------
    |            |            |
    Shard 1      Shard 2      Shard 3 (added later)
    (Replica Sets)

------------------------------------------------------------------------

## Prerequisites

-   Docker installed
-   Basic MongoDB familiarity
-   \~4--6 GB free memory

------------------------------------------------------------------------

## Step 1: Create Docker Network

``` bash
docker network create mongo-shard-net
```

------------------------------------------------------------------------

## Step 2: Start Config Server

``` bash
docker run -d --name configsvr \
  --network mongo-shard-net \
  mongo mongod \
  --configsvr \
  --replSet configReplSet \
  --port 27019
```

``` js
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [{ _id: 0, host: "configsvr:27019" }]
})
```

------------------------------------------------------------------------

## Step 3: Start Shard 1

``` bash
docker run -d --name shard1 \
  --network mongo-shard-net \
  mongo mongod \
  --shardsvr \
  --replSet shard1ReplSet \
  --port 27018
```

``` js
rs.initiate({
  _id: "shard1ReplSet",
  members: [{ _id: 0, host: "shard1:27018" }]
})
```

------------------------------------------------------------------------

## Step 4: Start Shard 2

``` bash
docker run -d --name shard2 \
  --network mongo-shard-net \
  mongo mongod \
  --shardsvr \
  --replSet shard2ReplSet \
  --port 27018
```

``` js
rs.initiate({
  _id: "shard2ReplSet",
  members: [{ _id: 0, host: "shard2:27018" }]
})
```

------------------------------------------------------------------------

## Step 5: Start mongos

``` bash
docker run -d --name mongos \
  --network mongo-shard-net \
  -p 27017:27017 \
  mongo mongos \
  --configdb configReplSet/configsvr:27019
```

------------------------------------------------------------------------

## Step 6: Add Shards

``` js
sh.addShard("shard1ReplSet/shard1:27018")
sh.addShard("shard2ReplSet/shard2:27018")
```

------------------------------------------------------------------------

## Step 7: Enable Sharding

``` js
use ecommerce
sh.enableSharding("ecommerce")
```

------------------------------------------------------------------------

## Step 8: Shard Collection

``` js
sh.shardCollection(
  "ecommerce.orders",
  { userId: "hashed" }
)
```

------------------------------------------------------------------------

## Step 9: Insert Data

``` js
for (let i = 1; i <= 20000; i++) {
  db.orders.insertOne({
    userId: i,
    orderId: "ORD-" + i,
    amount: Math.floor(Math.random() * 1000),
    createdAt: new Date()
  })
}
```

------------------------------------------------------------------------

## Step 10: Observe Distribution

``` js
sh.status()
```

------------------------------------------------------------------------

## Step 11: Query Behavior

Targeted query:

``` js
db.orders.find({ userId: 123 }).explain("executionStats")
```

Scatter-gather query:

``` js
db.orders.find({ amount: { $gt: 500 } })
```

------------------------------------------------------------------------

## Step 12: Add New Shard

``` js
sh.addShard("shard3ReplSet/shard3:27018")
```

------------------------------------------------------------------------

## Key Takeaways

-   Sharding is a **data modeling problem**
-   Shard key selection impacts performance
-   Queries without shard keys hit all shards
-   MongoDB rebalances data using chunks
-   Horizontal scaling needs good access patterns

------------------------------------------------------------------------

## Cleanup

``` bash
docker rm -f mongos shard1 shard2 shard3 configsvr
docker network rm mongo-shard-net
```

------------------------------------------------------------------------

## Purpose

This repository exists to learn sharding by doing and build intuition
for distributed databases.
