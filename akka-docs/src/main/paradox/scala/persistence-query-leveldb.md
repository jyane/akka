# Persistence Query for LevelDB

This is documentation for the LevelDB implementation of the @ref:[Persistence Query](persistence-query.md) API.
Note that implementations for other journals may have different semantics.

## Dependencies

Akka persistence LevelDB query implementation is bundled in the `akka-persistence-query` artifact.
Make sure that you have the following dependency in your project:

@@@vars
```
"com.typesafe.akka" %% "akka-persistence-query" % "$akka.version$"
```
@@@

## How to get the ReadJournal

The `ReadJournal` is retrieved via the `akka.persistence.query.PersistenceQuery`
extension:

@@snip [LeveldbPersistenceQueryDocSpec.scala]($code$/scala/docs/persistence/query/LeveldbPersistenceQueryDocSpec.scala) { #get-read-journal }

## Supported Queries

### EventsByPersistenceIdQuery and CurrentEventsByPersistenceIdQuery

`eventsByPersistenceId` is used for retrieving events for a specific `PersistentActor` 
identified by `persistenceId`.

@@snip [LeveldbPersistenceQueryDocSpec.scala]($code$/scala/docs/persistence/query/LeveldbPersistenceQueryDocSpec.scala) { #EventsByPersistenceId }

You can retrieve a subset of all events by specifying `fromSequenceNr` and `toSequenceNr`
or use `0L` and `Long.MaxValue` respectively to retrieve all events. Note that
the corresponding sequence number of each event is provided in the `EventEnvelope`, 
which makes it possible to resume the stream at a later point from a given sequence number.

The returned event stream is ordered by sequence number, i.e. the same order as the
`PersistentActor` persisted the events. The same prefix of stream elements (in same order)
are returned for multiple executions of the query, except for when events have been deleted.

The stream is not completed when it reaches the end of the currently stored events,
but it continues to push new events when new events are persisted.
Corresponding query that is completed when it reaches the end of the currently
stored events is provided by `currentEventsByPersistenceId`.

The LevelDB write journal is notifying the query side as soon as events are persisted, but for
efficiency reasons the query side retrieves the events in batches that sometimes can
be delayed up to the configured `refresh-interval` or given `RefreshInterval`
hint.

The stream is completed with failure if there is a failure in executing the query in the
backend journal.

### AllPersistenceIdsQuery and CurrentPersistenceIdsQuery

`allPersistenceIds` is used for retrieving all `persistenceIds` of all persistent actors.

@@snip [LeveldbPersistenceQueryDocSpec.scala]($code$/scala/docs/persistence/query/LeveldbPersistenceQueryDocSpec.scala) { #AllPersistenceIds }

The returned event stream is unordered and you can expect different order for multiple
executions of the query.

The stream is not completed when it reaches the end of the currently used *persistenceIds*,
but it continues to push new *persistenceIds* when new persistent actors are created.
Corresponding query that is completed when it reaches the end of the
currently used *persistenceIds* is provided by `currentPersistenceIds`.

The LevelDB write journal is notifying the query side as soon as new `persistenceIds` are
created and there is no periodic polling or batching involved in this query.

The stream is completed with failure if there is a failure in executing the query in the
backend journal.

### EventsByTag and CurrentEventsByTag

`eventsByTag` is used for retrieving events that were marked with a given tag, e.g. 
all domain events of an Aggregate Root type.

@@snip [LeveldbPersistenceQueryDocSpec.scala]($code$/scala/docs/persistence/query/LeveldbPersistenceQueryDocSpec.scala) { #EventsByTag }

To tag events you create an @ref:[Event Adapters](persistence.md#event-adapters) that wraps the events in a `akka.persistence.journal.Tagged`
with the given `tags`.

@@snip [LeveldbPersistenceQueryDocSpec.scala]($code$/scala/docs/persistence/query/LeveldbPersistenceQueryDocSpec.scala) { #tagger }

You can use `NoOffset` to retrieve all events with a given tag or retrieve a subset of all
events by specifying a `Sequence` `offset`. The `offset` corresponds to an ordered sequence number for
the specific tag. Note that the corresponding offset of each event is provided in the
`EventEnvelope`, which makes it possible to resume the stream at a later point from a given offset.

The `offset` is exclusive, i.e. the event with the exact same sequence number will not be included
in the returned stream. This means that you can use the offset that is returned in `EventEnvelope`
as the `offset` parameter in a subsequent query.

In addition to the `offset` the `EventEnvelope` also provides `persistenceId` and `sequenceNr`
for each event. The `sequenceNr` is the sequence number for the persistent actor with the
`persistenceId` that persisted the event. The `persistenceId` + `sequenceNr` is an unique
identifier for the event.

The returned event stream is ordered by the offset (tag sequence number), which corresponds
to the same order as the write journal stored the events. The same stream elements (in same order)
are returned for multiple executions of the query. Deleted events are not deleted from the
tagged event stream.

@@@ note

Events deleted using `deleteMessages(toSequenceNr)` are not deleted from the "tagged stream".

@@@

The stream is not completed when it reaches the end of the currently stored events,
but it continues to push new events when new events are persisted.
Corresponding query that is completed when it reaches the end of the currently
stored events is provided by `currentEventsByTag`.

The LevelDB write journal is notifying the query side as soon as tagged events are persisted, but for
efficiency reasons the query side retrieves the events in batches that sometimes can
be delayed up to the configured `refresh-interval` or given `RefreshInterval`
hint.

The stream is completed with failure if there is a failure in executing the query in the
backend journal.

## Configuration

Configuration settings can be defined in the configuration section with the
absolute path corresponding to the identifier, which is `"akka.persistence.query.journal.leveldb"`
for the default `LeveldbReadJournal.Identifier`.

It can be configured with the following properties:

@@snip [reference.conf]($akka$/akka-persistence-query/src/main/resources/reference.conf) { #query-leveldb }