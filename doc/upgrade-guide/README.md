# Upgrade guide

The purpose of this guide is to detail the changes made by the successive versions of the DataStax Node.js Driver that 
are relevant to for an upgrade from prior versions.

If you have any questions or comments, you can [post them on the mailing list][mailing-list].

## 4.4

### New default load balancing policy

The driver uses the new `DefaultLoadBalancingPolicy` implementation as default load balancing policy. The new policy
attempts to fairly distribute the load based on the amount of in-flight request per hosts. The
local replicas are initially shuffled and [between the first two nodes in the shuffled list, the one with fewer
in-flight requests is selected as coordinator](https://www.eecs.harvard.edu/~michaelm/postscripts/mythesis.pdf).

### Upgrade guide for DSE driver users

The DSE driver and the Apache Cassandra driver have been merged into a single package. There's a dedicated [guide for
 DSE driver users that plan to migrate to the `cassandra-driver`](upgrade-from-dse-driver).

---

## 4.2

### Tuple constructor with one parameter

The `Tuple` constructor had an undocumented behaviour when invoked with a single parameter which was an `Array`,
the driver used the `Array` instance as `Tuple` elements. We removed this behaviour that was used internally.
`Tuple.fromArray()` method should be used to build a `Tuple` from an `Array` of elements. 

## 4.0

The following is a list of changes made in version 4.0 of the driver that are relevant when upgrading from version 3.x.

### localDataCenter is now a required Client option

When using `DCAwareRoundRobinPolicy`, which is used by default,  a local data center must now be provided to the
`Client` options parameter as `localDataCenter`.  This is necessary to prevent routing requests to nodes in remote
data centers.

### Selection of contact points is now evaluated in random order

The list of contact points provided as a Client option is now shuffled before selecting a node to connect to as part
of initialization.  This change was made for instances where configuration is shared between many clients.  In this
case, it is better to distribute initial connections to different nodes in the cluster instead of choosing the
same node each time as the initial connection makes a number of queries to discover cluster topology and schema.

### Changes to the retry and load-balancing policies

`ExecutionOptions` is introduced as a wrapper around the `QueryOptions`.
The `ExecutionOptions` contains getter methods to obtain the values of each option, defaulting to the execution profile
options or the ones defined in the `ClientOptions`. Previously, a shallow copy of the provided `QueryOptions` was 
used, resulting in unnecessary allocations and evaluations.

The `LoadBalancingPolicy` and `RetryPolicy` base classes changed method signatures to take `ExecutionOptions` instances 
as argument instead of `QueryOptions`.

Note that no breaking change was introduced for execution methods such as `Client#execute()`, `Client#batch()`, 
`Client#eachRow()` and `Client#stream()`. This change only affects custom implementations of the policies.

### Query idempotency and retries

The configured `RetryPolicy` is not engaged when a query errors with a `WriteTimeoutException` or request error and 
the query was not idempotent.

In order to control the possibility of retrying when an timeout/error is encountered, you must mark the query as 
idempotent. You can define it at `QueryOptions` level when calling the execution methods.

```javascript
client.execute(query, params, { prepare: true, isIdempotent: true })
```

Additionally, you can define the default idempotence for all executions when creating the `Client` instance:

```javascript
const client = new Client({
  contactPoints,
  localDataCenter,
  queryOptions: { isIdempotent: true }
});
```

Previously, a similar behaviour was available using `IdempotenceAwareRetryPolicy`, that is now marked as deprecated.

### Removed `retryOnTimeout` property of `QueryOptions`

`retryOnTimeout`, the property that controlled whether a request should be tried when a response wasn't obtained 
after a period of time is no longer available. 

The behaviour should be now controlled using `onRequestError()` method on the `RetryPolicy`  for idempotent 
queries.

### Changes on `OperationInfo` of the retry module 

The retry policy methods takes [`OperationInfo`][op-info] as a parameter. Some `OperationInfo` properties changes or 
were removed.

- Deprecated properties `handler`, `request` and `retryOnTimeout` were removed.
- `options` property was replaced by `executionOptions` which is an instance of `ExecutionOptions`.

### Removed `meta` property from `ResultSet`

On earlier versions of the driver, the `ResultSet` exposed the property `meta` which contained the raw result metadata.
This property was removed in the latest version.

### Removed `DCAwareRoundRobinPolicy` `usedHostsPerRemoteDC` constructor parameter

`DCAwareRoundRobinPolicy` no longer supports routing queries to hosts in remote data centers. Because of this
`usedHostsPerRemoteDC` has been removed as a constructor parameter.  This change was made because handling
data center outages is better suited at a service level rather than within an application client.

---

## 3.0

### Changes in CQL aggregates metadata

The `initCondition` property of `Aggregate`, the class that represents the metadata information of a CQL 
aggregate, changes from `Object` to `String`.

---

## 2.0

The following is a list of changes made in version 2.0 of the driver that are relevant when upgrading from version 1.x.

### API Changes

1. `uuid` and `timeuuid` values are decoded as [`Uuid`](../features/datatypes/uuids) and
[`TimeUuid`](../features/datatypes/uuids) instances.

1. `decimal` values are decoded as [`BigDecimal`](../features/datatypes/numerical) instances.

1. `varint` values are decoded as [`Integer`](../features/datatypes/numerical) instances.

1. `inet` values are decoded as `InetAddress` instances.


[mailing-list]: https://groups.google.com/a/lists.datastax.com/forum/#!forum/nodejs-driver-user
[op-info]: https://docs.datastax.com/en/developer/nodejs-driver/latest/api/module.policies/module.retry/type.OperationInfo/