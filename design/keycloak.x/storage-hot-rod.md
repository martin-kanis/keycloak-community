# Keycloak X - Storage - Infinispan/Hot Rod persistence layer

* **Status**: Draft
* **JIRA**: https://issues.redhat.com/browse/KEYCLOAK-18997 // TODO: add some epic link

# Goals of this document

This document aims to address the following items:

* Follow up on no-downtime strategy presented in the [the Storage / Persistence layer proposal](storage-persistence.md)
* Present a way how to store and query data using Hot Rod Client and Remote Infinispan server so that we achieve the no-downtime requirements

# Non-Goals of this document

This document aims to _not_ address the following items:

* Provide a way how to replace the existing Infinispan layer
* Discuss design of caching or its impact on performance

# Technologies

- **[Infinispan](https://infinispan.org)** - Infinispan is a distributed in-memory key/value data store with optional schema. It can be used both as an embedded Java library and as a language-independent service accessed remotely over a variety of protocols. In this proposal, we will target Infinispan in the remote (client/server) scenario.
- **[Hot Rod](https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html)** - Hot Rod is a wire protocol that Infinispan clients use to talk to a remote grid. It is a binary, platform-independent protocol that was developed in the open as a part of Infinispan.
- **[Protobuf buffers](https://developers.google.com/protocol-buffers/)** - Protocol Buffers (Protobuf) is a lightweight binary media type for structured data. In our case, it provides interoperability between the client (Keycloak-X store) and the Infinispan server.

# Introduction

[The Storage / Persistence layer proposal](storage-persistence.md) suggests the following strategy to achieve 0-downtime upgrade store:

1. The store can read objects from the oldest ones to the current version
2. The store can read objects from the version following the current one
3. Objects are only updated when written to.
4. Number of schema changes should be kept at absolute minimum
5. Schema changes can be postponed and run at chosen time
   (even if that means running a degraded service)

A simple implementation of the following design can be found in this [pull request](https://github.com/keycloak/keycloak-playground/pull/11) into the keycloak-playground repository. This repository is a playground project that simplifies implementing and testing Keycloak-X-like no-downtime storage implementations.

# Data structure definition

For describing a structure of data that is stored in the Infinispan server there are Protobuf message definitions. Based on the message definition the Hot Rod client knows how to marshal and unmarshal the data that is sent/received to/from server. To make the Infinispan server able to query stored data, it is also necessary to share message definitions with the Infinispan server.

An example of a message definition is following:

```protobuf
package nodowntimeupgrade;

message InfinispanObjectEntity {
    required string entityVersion = 1;
    required string id = 2;
    optional string name = 3;
    optional string clientTemplateId = 4;
}
```

This protobuf definition can be automatically generated from an annotated Java class definition. For setting Protobuf message-specific settings there are `@ProtoField` and `@ProtoDoc` (described in the indexing section) annotations.

```java
public class InfinispanObjectEntity {

    @ProtoField(number = 1, required = true)
    public int entityVersion = ModelVersion.VERSION_1.getVersion();

    @ProtoField(number = 2, required = true)
    public String id;

    @ProtoField(number = 3)
    @ProtoDoc("@Field(index = Index.YES, store = Store.YES, analyze = Analyze.YES, analyzer = @Analyzer(definition = \"keyword\"))")
    public String name;

    @ProtoField(number = 4)
    public String clientTemplateId;

}
```

Each field has its number. These numbers serve for identification during the marshaling/unmarshaling process.

## Changes in object representations

### Addition of new fields

The advantage of Protobuf definitions is that they are flexible. It is possible to add any number of fields on updates. For example, when we want to introduce a new field named `clientScopeId`, we can do it by adding

```java
@ProtoField(number = 5)
public String clientScopeId;
```

to the message definition and updating the definition in the Infinispan server and in the current `SerializationContext`. Then everything should work as before. It is also possible to read objects with new fields from the old storage version. The new field `clientScopeId` is skipped during unmarshaling.

### Changing field name/type

It is possible to change field names or types. However, such changes can introduce backward incompatibility that we need to avoid due to no-downtime requirements. We can avoid breaking backward compatibility by doing updates in the following steps (in this example, we have the field `clientTemplateId` in the first object version (`entityVersion = 1`) and we want to replace it with `clientScopeId`; the migration path is following `clientScopeId = "template-" + clientTemplateId`):

---

**NOTE**

In this proposal, we distinguish between two types of version:

* **Version of storage** - This corresponds to the version of source code and message definitions on the Keycloak server side; in other words an incremented storage version means there was a release of Keycloak. In a world where there is no change in Keycloak codebase other than changes in `InfinispanObjectEntity`, `entityVersion` corresponds to Keycloak version.
* **Version of object** - This refers to the version of object in the storage (Infinispan server); in other words, this says what version of storage originally created the object in the store. The object version can be incremented by reading the old object from the store by newer storage, migrating it, and storing it back into the storage.

---

1. We update the old message defintion with `entityVersion = 2` (incrementing version of storage) so that it contains both `clientTemplateId` and `clientScopeId`
2. Keycloak codebase that is using storage version 2 uses only the new field (`clientScopeId`). The deprecated `clientTemplateId` field is taken into account in three cases:
   1. When writing to the storage and `clientScopeId` value starts with `"template-"` prefix. In such case, `clientTemplateId` is written together with `clientScopeId`. This is to support the reading of objects version 2 by storage version 1. 
   2. When reading an old object from the storage. The `clientTemplateId` field is read from the storage, and migrated to `clientScopeId`. This step is necessary read objects version 1 by storage version 2.
   3. When querying objects, this will be discussed in the section around querying.
3. Then, there can be a new storage version `>= 3` which removes the deprecated `clientTemplateId`. Migrating to that storage version requires that all stored objects are migrated to object version `>= 2`. We will get back to the removal later in the proposal as it gets more complicated with regards to querying.

# Reading older versions of objects

Each object contains `entityVersion` field that represents its object version (version of the storage that created it). To fulfill the requirements, we need to be able to read all older versions of objects. As described in the previous section, unmarshaling of objects is flexible enough to be able to read an older version when message definition contains all fields in unmarshalled stream. Moreover, unknown fields are ignored; though, we are losing the information that was stored there. In some cases, like the migration described in the previous section, we need to migrate one field to another. In this case, we need to check what version the read object was and migrate it when necessary. For example, this is how the migration works in the case of the previous migration path:

```java
public class EntityMigrationToVersion3 extends ObjectEntityDelegate {

    public EntityMigrationToVersion3(InfinispanObjectEntity delegate) {
        super(delegate);
    }

    @Override
    public String getClientScopeId() {
        if (super.getClientScopeId() == null && super.getClientTemplateId() != null) {
            return "template-" + super.getClientTemplateId();
        }

        return super.getClientScopeId();
    }
}
```

* `ObjectEntityDelegate` implements the same interface as `InfinispanObjectEntity`; it is used for overriding only some of the methods from the original object, in this case, the `getClientScopeId` method.
* All layers above this storage layer use only `clientScopeId`; the presence of the `clientTemplateId` field is hidden.
* Delegation object is created when `entityVersion` is lower than the current version.
* Too long chain of delegations can cause performance overhead. It can be solved by the migration of storage objects to a newer version (in the future might be solved by some scheduled tasks functionality)

# Querying objects

Thanks to the fact, that schema is known on the Infinispan side, it is possible to query objects based on field names. It is also possible to create indices to speed up queries, more on that later in a separate section. Infinispan supports queries using [Ickle query language](https://infinispan.org/docs/stable/titles/developing/developing.html#creating_ickle_queries-querying). Here is an example, how the Ickle query looks like:

```java
// Remote Query, using protobuf
QueryFactory qf = org.infinispan.client.hotrod.Search.getQueryFactory(remoteCache);
Query<InfinispanObjectEntity> query = qf.create("from nodowntimeupgrade.InfinispanObjectEntity where name = 'searched-name'");
```

# Message definition changes and Ickle queries

Ickle query with a field name that is unknown to the Infinispan server fails. Therefore, we need to be careful about updates and removals we perform in message definitions. On top of that, we need to make sure that queries are built with definition changes in mind. For this reason, it may be necessary to introduce some intermediate storage versions that provide the migration step on query level. Given the previous example where we replaced `clientTemplateId` with `clientScopeId`, the migration could be done in the following steps:

### Query in storage version 1

- Message definition in this version is the original one without `clientScopeId` field.
- It is possible to search by `clientTemplateId` without any additional query adjustments.
  ```
  FROM nodowntimeupgrade.InfinispanObjectEntity WHERE (clientTemplateId = 'desired-value')
  ```

### Query in storage version 2

- This is the intermediate version that uses both `clientTemplateId` and `clientScopeId`.
- When doing a query it must count with the fact that the storage may contain objects version 1, that does not have `clientScopeId` defined.
- The Keycloak service layer (or whatever layer is above storage layer) is searching only by `clientScopeId`, presence of `clientTemplateId` field is hidden to these layers.
- Store version 1 is able to return newly created objects of version 2 because these are backward compatible and contain `clientTemplateId` as well as `clientScopeId` when appropriate.

For this particular migration path (`clientScopeId = "template-" + clientTemplateId`) there are two scenarios how the resulting query looks like based on the searched value.

**Searched value is _NOT_ prefixed with `"template-"**

No adjustment is needed as no object entity version 1 fulfills this search (`clientScopeId` without `"template-"` prefix is undefined on these objects).

```
FROM nodowntimeupgrade.InfinispanObjectEntity WHERE (clientScopeId = 'desired-value')
```

**Searched value is prefixed with `"template-"`**

In this case, there may be also some object version 1 that fulfills the given criteria; we need to take it into account.

```
FROM nodowntimeupgrade.InfinispanObjectEntity WHERE 
    (entityVersion >= 2 AND entityVersion <= 3 AND clientScopeId = 'template-desired-value') 
    OR
    (entityVersion < 2 AND c.clientTemplateId = 'desired-value')
```

Note the condition `entityVersion <= 3` comes from the no-downtime requirement that each version can read all previous versions, but only one following (`this.version + 1`).

### Query in storage version 3

* The message definition must still contain both `clientTemplateId` and `clientScopeId` (the old field needs to be present because storage version 2 sometimes includes it in Ickle queries).
* There is no migration of Ickle queries anymore, object entities version 1 are not returned from storage when searching by `clientScopeId`.
* If the store detects, on startup, that there are some entities of version 1 in the storage, there is a **WARNING** printed that warns administrator of possible incomplete search results until she/he updates all objects to the version `>= 2`.

We use only `clientScopeId` in queries.

```
FROM nodowntimeupgrade.InfinispanObjectEntity WHERE (clientScopeId == 'desired-value')
```

### Query in storage in subsequent versions

No changes for this particular field in search queries. From the query perspective, it is possible to remove `clientTemplateId` from the message definition. On the other hand, if the administrator has no problem with incomplete results for this field, we can retain the presence of this field so that it is possible to read `clientTemplateId` and migrate it to `clientScopeId`. This can be decided for each migration individually.

# Speeding up queries with indices

Infinispan uses [`Apache Lucene`](http://lucene.apache.org/) technology to index values in caches. Entities that are indexed needs to be specified per cache configuration. Java annotation `@Protodoc` is used for setting which fields, for the particular entity, are indexed and also for specifying some additional indexing attributes.

```java
@ProtoDoc("@Field(index = Index.YES, store = Store.YES, analyze = Analyze.YES, analyzer = @Analyzer(definition = \"keyword\"))")
```

- **index** - To index or not to index, that is the switch.
- **store** - Index is stored in the host filesystem or the heap. Defaulting to the filesystem.
- **analyze** - Configures whether the value is searched as a value or as a full-text search. It is also possible to specify an [analyzer](https://infinispan.org/docs/stable/titles/developing/developing.html#analysis) used. 

# Case-insensitive queries

Often requirement in the Keycloak codebase is to search case-insensitively. For example, for email addresses we do not want to allow two accounts with emails `Admin@keycloak.org` and `admin@keycloak.org`. Even though, there is an operator `LIKE` in Ickle queries, it does not support case-insensitive search. For this reason, we need to have all fields with this requirement analyzed. 

We need to be also cautios about choosing the correct analyzer. The default analyzer supports case-insensitive searches, however treats the words in the text as separate tokens. Therefore, it is not possible to search for strings with space in it. For example, given a group with name `My group name`, a search with string `My gr*` would not find any group with default analyzer. It should be possible to use `keyword` analyzer for searches like this, but it may have some other disadvantages. This needs to be decided for each index individually.

## Queries with analyzed fields

Ickle queries are [different](https://infinispan.org/docs/stable/titles/developing/developing.html#using_full_text_search) for analyzed fields. Operators used for non-analyzed fields do not work for full-text searches. We need to use `Phrase queries` or `Wildcard queries`. For example, the following case-insensitive query:
`FROM UserEntity ue WHERE ue.email ILIKE "%@gmail.com"` is replaced with `FROM UserEntity ue WHERE ue.email : "*@gmail.com"`. For this reason, there are some restrictions about operator usage for analyzed fields. Supported operators are only ILIKE (case-insensitive wildcard search), EQ (phrase query without wildcards) and NE (negated EQ). This needs to be documented somewhere.

# Transactional layer

The goal of the transactional layer is to fulfill the following items:
1. The transaction layer must keep track of all objects returned from the storage; all changes done to tracked objects are propagated to the Infinispan server in the commit phase
2. To save network utilization, the transactional layer can decide whether performed changes were meaningful; non-meaningful changes are not propagated to the Infinispan server.
3. The transactional layer participates in the main transaction. It is possible to rollback when the Infinispan storing process, or any other participant fails.

## Usage of Hot Rod client transaction

The Hot Rod client can participate in [JTA transactions](https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html#hotrod_transactions). Unfortunately, the first two items from our requirements are not achievable by this transaction implementation. Therefore, we need to keep track of changes done to returned objects manually. The good news is that since `RemoteCache` has the same interface as `ConcurrentHashMap`, we should be able to use the same implementation, or maybe just a little bit adjusted implementation of [`ConcurrentHashMapKeycloakTransaction`](https://github.com/keycloak/keycloak/blob/af3b573d196af882dfb25cdccb98361746e85481/model/map/src/main/java/org/keycloak/models/map/storage/chm/ConcurrentHashMapKeycloakTransaction.java) for this.

### `TransactionManager` lookup

The Hot Rod client is able to automatically lookup `TransactionManager` using [`GenericTransactionManagerLookup`](https://docs.jboss.org/infinispan/9.4/apidocs/org/infinispan/transaction/lookup/GenericTransactionManagerLookup.html) when running in Java EE application server. When there is no `TransactionManager` running in the container, [`RemoteTransactionManager`](https://docs.jboss.org/infinispan/10.0/apidocs/org/infinispan/client/hotrod/transaction/manager/RemoteTransactionManager.html) is returned which can be enlisted to [KeycloakTransactionManager](https://github.com/keycloak/keycloak/blob/af3b573d196af882dfb25cdccb98361746e85481/server-spi/src/main/java/org/keycloak/models/KeycloakTransactionManager.java). 

### Hot Rod client transaction mode

The Hot Rod client transaction can work in the following modes:
- **NONE** - non-transactional - when using this setting, we would not be able to achieve point number 3 from our requirements.
- **NON_XA** - in this mode, the `RemoteCache` interacts with the `TransactionManager` via [`Synchronization`](https://docs.oracle.com/javaee/7/api/javax/transaction/Synchronization.html); this means the changes are propagated to the Infinispan storage only when the main transaction successfully commits; however, in the case of a failure during storing data in the Infinispan server the main transaction is not rolled back. 
- **NON_DURABLE_XA** - `RemoteCache` interacts with the `TransactionManager` using [`XAResource`](https://docs.oracle.com/javaee/7/api/javax/transaction/xa/XAResource.html) without possibility of [recovery](https://infinispan.org/docs/stable/titles/developing/developing.html#tx_recovery).
- **FULL_XA** - `RemoteCache` interacts with the `TransactionManager` using [`XAResource`](https://docs.oracle.com/javaee/7/api/javax/transaction/xa/XAResource.html) with the possibility of recovery. Should be avoided due to the [documentation note](https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html#hr_transactions_config_server) about performance degradation.

Based on the above, **NON_DURABLE_XA** seems to be the best option to use given the requirements.

# Infinispan Initialization

## Remote caches

Configuration of the remote caches will be stored declaratively in a xml file. The caches can be created either:
* by a user when starting the Infinispan server using the provided configuration
* by Keycloak which will check if the caches exist on the Infinispan server during the startup and if not, it will create them. However some users may want to tweak the configuration based on their special needs in which case they will create the caches themselves.

Example of a remote cache configuration:
```
<infinispan>
   <cache-container>
       ...
       <distributed-cache name="clients" mode="SYNC">
           <encoding media-type="application/x-protostream"/>
           <locking isolation="REPEATABLE_READ"/>
           <transaction mode="NON_XA"/>
       </distributed-cache>
	...
   </cache-container>
</infinispan>
```

Each map storage type (clients, users, ...) will have a separate remote cache. The exact configuration of caches to be decided based on the testing. 

## Protobuf schema

It’s required to register protobuf schema definition to the Infinispan server for all entities which will be stored in the caches. We consider two options:
* register Protobus schemas automatically during the startup by adding them to `___protobuf_metadata` cache. We need to make sure that an old schema won’t override a newer schema if a pod restarts. This is the preferred option.
* `protostream-processor` dependency processes Java annotations in your classes at compile time to generate Protobuf schemas. These schemas then can be added to the Infinispan server by a user via CLI/REST.

## Changes in cache configuration at runtime

TBD - spin up new Infinispan server with desired cache configuration and migrate data using the [cache store migrator](https://infinispan.org/docs/dev/titles/upgrading/upgrading.html#offline_data_migration)?


