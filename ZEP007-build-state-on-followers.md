---
title: Build state on followers
authors:
- Christopher Zell (@zelldon)
reviewers: ["Deepthi Akkoorath (@deepthidevaki)", "Nicolas Pepin-Perreault (@npepinpe)", "Philipp Ossler (@saig0)", "Nico Korthout (@korthout)"]  
approvers: ["Deepthi Akkoorath (@deepthidevaki)", "Nicolas Pepin-Perreault (@npepinpe)", "Philipp Ossler (@saig0)", "Nico Korthout (@korthout)"]
editor: TBD
creation-date: 2020-04-28
last-updated: 2021-07-06
status: implementable
see-also: []
replaces: []
superseded-by: []
---

# Summary
[summary]: #summary

We propose that followers continuously replay the log and build their own state, including taking snapshots and compacting the log without coordinating with the leader.

# Motivation
[motivation]: #motivation

Generally we want to reduce the failover time. With that we mean: the time between no leader is available, processing has stopped, and a new leader has been chosen and processing has started again. Especially we want to minimize the impact on process execution latency during failover on a partition. With this ZEP we want to decrease the time from when a node becomes Leader until it starts to process new commands. This can be achieved with building state on followers.

![processing-drop](ZEP007/drop-closer-general-small.png)

In the screenshot above you can see the current impact of a leader change (fail-over), we want to reduce this gap. In this example it took ~ 2 minutes (from 16:00 to 16:02) to start processing again, which means after 2 minutes we can continue with the real process execution again.

There are other issues we can solve with building state on followers, which are related to our current snapshot replication strategy. Followers can only delete data if they received snapshots and if they are valid. This means our current snapshot and replication mechanism needs to be reliable. In the past we had plenty bugs in the snapshot replication, which caused that followers where not able to compact their log. Additionally, there is currently two different snapshot replication strategies, we need to maintain. All of that can be removed when we build state on followers.

Another interesting side effect is that not periodically sending the snapshots over the wire would bring us nearer to the ability to support large states, which is currently not possible.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
      
[comment]: <> (         Explain the proposal as if it was already included in the product and you were teaching it to another user/contributor. That generally means:)
[comment]: <> (         - Introducing new named concepts.)
[comment]: <> (         - Explaining the feature largely in terms of examples.)
[comment]: <> (         - Explaining how our users/contributors should *think* about the feature, and how it should impact the way they use our product. It should explain the impact as concretely as possible.)
[comment]: <> (         - If applicable, provide sample error messages, deprecation warnings, or migration guidance.)
[comment]: <> (         - If applicable, describe the differences between teaching this to existing users/contributors and new users/contributors.)
[comment]: <> (         For user facing ZEPs, this section should describe the benefits or changes the users will experience, from the point of view of the user.)
[comment]: <> (         For maintenance/non-user facing ZEPs, this section should focus on how other contributors should reason about the changes, and give concrete examples of its impact, both short term and long term.)
[comment]: <> (         For organizational ZEPs, this section should provide an example-driven introduction to the new policy or process, and explain its impact on the development process in concrete terms.)

Building state on follower's means, that since the follower already has the data on the replicated log, he consumes it similar to the Leader. This is done instead of replicating snapshots from the leader to the follower's. In this section I will describe and discuss the conceptional idea.

## Analogy

Imagine an office with three office workers working in it. There is one worker which is the Leader and two others can be seen as his backup.

![worker-group](ZEP007/worker-group.png)

They all have access to the records (documents), and the Leader worker is building a state based on the already processed records. He will regularly share the state of his already processed documents with the others.

![stateDocuments](ZEP007/stateDocuments.png)

If now the Leader is on vacation or sick someone else needs to take over. This might imply several problems.

![backupWorkers](ZEP007/backupWorkers.png)

If the Leader has forgotten to send the state before he left, one of the office worker, which takes over, might have a quite old state, and he needs to do a lot of rework. If the state is not readable he has also a problem to take over or there are documents missing etc.

In order to prevent such situations the idea is that each office worker, since they already have access to the documents (records), are building their own state. The Leader is only in charge of sending letters, invoices out etc. If now the Leader is not available another worker can take immediately over. To prevent not answering letters or answering twice, after take over, the Leader will mark documents as processed. The other workers will only build their state on already processed documents.

## Distinction to normal raft

As you can read in the [raft paper](https://raft.github.io/raft.pdf) chapter 7, compaction and taking a snapshot should be done on each server.

> This snapshotting approach departs from Raft’s strong
leader principle, since followers can take snapshots without the knowledge of the leader. However, we think this
departure is justified. While having a leader helps avoid
conflicting decisions in reaching consensus, consensus
has already been reached when snapshotting, so no decisions conflict. Data still only flows from leaders to fol
lowers, just followers can now reorganize their data.
>
> We considered an alternative leader-based approach in
which only the leader would create a snapshot, then it
would send this snapshot to each of its followers. However, this has two disadvantages. First, sending the snapshot to each follower would waste network bandwidth and
slow the snapshotting process. Each follower already has
the information needed to produce its own snapshots, and
it is typically much cheaper for a server to produce a snapshot from its local state than it is to send and receive one
over the network. Second, the leader’s implementation
would be more complex. For example, the leader would
need to send snapshots to followers in parallel with replicating new log entries to them, so as not to block new
client requests.

As the paper states it makes totally sense to build state and take snapshots of it on all servers unrelated of the raft state.

The normal raft approach would mean to run the same state machine on the Leader and Follower, in order to build the state and take snapshot's of it. Unfortunately, our processing state machine produces new commands, which need to be applied to the state machine again. Writing new Commands to the Log, is only allowed to the Leader. Single Writer Principle. In order to overcome this and build state on followers, we need a separate state machine for the Followers.

## State on Followers

Instead of running the same processing state machine on the followers, we replay the events from the replicated log. For that we have a ReplayStateMachine, which allows to continuously replay events and apply them to the state. We call it replay, since they have been already applied on the leader side and have been produced by him. Furthermore, we will reuse this state machine on bootstrapping of a leader partition such that the term "replay" makes sense here as well.

Raft ROLEs are transient states, which means it is likely that a Leader change happens. How the system reacts on these role changes can be seen in the following picture. It shows a collapsed version of the process, we minimize the time for the Follower-to-Leader transition, which impacts the process execution latency. More details can be seen in the following sub-sections.

![collapsedStateOnFollower](ZEP007/collapsedStateOnFollower.png)

On bootstrapping of a Zeebe Partition we first install all needed services, like the LogStream, the SnapshotDirector etc. We don't want to go deeper into this, the bootstrap should be straight forward. Here we want to focus more on the transition between the roles, and the reaction based on that. 

After bootstrapping our base services we go over to a state we call "Stream Processing", which is our real business logic and the heart of each partition. Based on the RAFT role we perform different actions on the "Stream Processing". We can distinguish five cases which we will explain more in depth in the following sections:

  * Bootstrapping Leader Partition, no already running partition
  * Bootstrapping Follower Partition, no already running partition
  * Switch to Leader Partition, from an already running Follower partition
  * Switch to Follower Partition, from an already running Leader Partition.
  * Re-init Follower Partition, from an already running Follower Partition.

We can also represent that as the following state machine.

![statemachine](ZEP007/statemachine.png)

In the following section we will use the term processing and replay heavily, please read the [ZEP004](https://github.com/zeebe-io/enhancements/blob/master/ZEP004-wf-stream-processing.md) if you want to know more about it.

### Bootstrap Leader Partition

![stateOnFollower-BootstrapLeader](ZEP007/stateOnFollower-BootstrapLeader.png)

In this scenario we have no already running Zeebe partition and installed it completely new. During installing our services we restore the state, via copying the latest snapshot into the runtime folder. After installing all services, we start with the "Stream Processing", which is on the Leader separated in two parts. The first part is the processing, which consist of the following steps:

  * **Replay until end**, here we replay all existing events on the Log, which were not part of the Snapshot. This is necessary to get the latest state, before processing.
  * **Reset Dispatcher position**, here we set the Dispatcher position which generates the position for our follow-up events, and wipe all existing data.
  * **Allow Command API**, this means we're now accepting User commands being written to the Dispatcher, which will then later be processed.
  * **Process Commands**, we process commands in a loop and generate new follow-up commands and events. There is of course more behind that state, but as high level this should be enough.

The second part is the exporting, which consist of the following steps:

  * **Restore last exported Position**, we restore the last exported position from the state
  * **Export Records**, we continuously export all kind of records. Exporters, normally send data to an external system. 

In order to not run exporters on the followers, to avoid overhead, we communicate the last successful exported position for each exporter to the followers regularly. This makes it possible for the followers to rebuild the exporter state, such that after fail-over they can take over the exporting without re-exporting too much data. This is of course also necessary to be able to decide until which record we can compact our log, on the follower side.

Furthermore, as part of the general "Stream Processing" a Timer is scheduled which takes periodically a snapshot from the state.

### Bootstrap Follower Partition

![stateOnFollower-BootstrapFollower](ZEP007/stateOnFollower-BootstrapFollower.png)

In this scenario we have no already running Zeebe partition and installed it completely new. During installing our services we restore the state, via copying the latest snapshot into the runtime folder. After installing all services, we start with the "Stream Processing", which is on the Follower quite simple. 

Part of the Follower "Stream Processing" is the continuously applying of events, which is called **Replay Events**, since they already applied on the Leader. This avoids generating follow-up events on the follower. Only the Leader is allowed to write records, Single Writer Principle (SWP). Additionally, we can make sure that the follower is not faster than the leader, and we can easily switch to the Leader processing as we can see in the next section.

In distinction to the Leader, the Follower will not execute any exporters. It will receive periodically the last successful exported positions, which he stores in his state.

It can happen, if the follower is slow, or was not available for longer time that it receives an "InstallRequest" from the Leader. Such an "InstallRequest" contains the latest snapshot from the Leader. This is done to reduce the time the follower needs to catch up, or if the leader already has compacted his log. This scenario is described later [here](#re-init-follower-partition).

One of an edge case topic is the "Blacklisting" of process instances. This done on the Leader side, when during processing an error occurs. The related process instance, where the command corresponds to and caused this error, will be "Blacklisted". To persist that, an error event is written to the log. This kind of error events are applied on the follower replay mode to restore the blacklisted process instances. This is necessary such that we ignore on normal processing/applying of events related commands/events, otherwise we might end in an unexpected state. This restoring of the "Blacklist", is also done on the Leader replay mode.

Furthermore, as part of the general "Stream Processing" a Timer is scheduled which takes periodically a snapshot from the state.

### Switch to Leader Partition

![stateOnFollower-SwitchToLeader](ZEP007/stateOnFollower-SwitchToLeader.png)

In this scenario we have an already running Zeebe Partition, and we were Follower before. This means all our services have been already installed, and the Follower has executed the Replay mode. After switching to the Leader, we need to stop the replay and start with the Leader "Stream Processing". It is the same we have described [above](#bootstrap-leader-partition).

Interesting is the point that we need to replay all events until the end of the log in both cases, on bootstrap but also on switching to Leader. This is necessary to make sure that we applied all events from the **old** Leader, before we start with the general command processing.

As written above on the followers no exporters are running, but they get the latest exported position for each exporter from the leader send over the wire periodically. Based on this values the exporters can be re-stored on the new Leader. Be aware that we expect exporters to be able to handle duplicated records, they should be idempotent.

### Switch to Follower Partition

![stateOnFollower-SwitchToLeader](ZEP007/stateOnFollower-SwitchToFollower.png)

In this scenario we have an already running Zeebe Partition, and we were Leader before. This means all our services have been already installed, and the Leader has executed the command processing and exporting.

After switching to the Follower, we need to stop the processing, exporting and regularly sending of last exported positions. It might happen that we still write follow-up events to the log, because the RAFT role transition is already done, but the Zeebe Partition is notified asynchronous. This will be handled on the RAFT side, since it doesn't allow writing to the log if the node is not the leader. Still we should disallow new incoming user commands, via disabling the Command API. We could also think about having it always on, which would simplify it but this is out of scope.

One important part before we start with the Follower "Replay Mode" is that we need to restore the current state from the latest snapshot. This is necessary because when we process commands we immediately apply the follow-up events to our state, which we write to the log later. We can't be sure what we have already applied is committed, and to not apply events twice it is easier to just restore from the latest snapshot, KISS principle. Other than that it is similar to described [above](#bootstrap-follower-partition).

### Re-init Follower Partition

This can be seen as a special case and can also be called as Follower-To-Follower transition. In RAFT it can happen that InstallRequests are send to an slow Follower or to one which is lagging behind. In order to get the Follower back to the latest state, the Leader will send the Follower his latest snapshot.

The Follower will reset his log, such that no gaps in the log exist. The Leader will send log entries after the snapshot index to the follower only. In order to replay with the latest state and on the correct index and position it is necessary to completely reset the follower. The easiest way is to trigger a new follower transition, which is similar to [Bootstrap the Follower partition](#bootstrap-follower-partition). We just need to make sure that the received snapshot is copied into the runtime folder.

  
[comment]: <> (          We have several challenges we need to overcome to solve/implement build state on followers. Let's take a look of the following causality chain:)
[comment]: <> (           * To have a working and long living system, we need to be able to compact our log no matter the role. )
[comment]: <> (           * To compact our log, we need to be able to take snapshots from the state.)
[comment]: <> (             * The state need to contain a dense reflection of the log, last exported and processed position, from which we can recover.)
[comment]: <> (           * To take a snasphot from the state, we need to build the state on all participants &#40;leader and follower&#41; in order to not replicate it.)
[comment]: <> (           * To build state on followers, we need to avoid creating follow-up events by the state machine, Single Writer Principle. )
[comment]: <> (           * To avoid follow-up events, we need to either replay only or use a NoopWriter.)
[comment]: <> (In the following sections we will go in more depth in some these mentioned parts of the chain.)

### Compacting the log

In order to avoid an ever-growing log we need to compact it from time to time. This means we remove the head of the log until a certain index. 

Compaction can be done after we took a snapshot from the state. A snapshot allows us to start from a specific point, such that we can compact our log. By building state on followers each node is responsible for taking their own snapshots and compacting their log.

Part of the snapshot should be the last processed and last exported position for each exporter. The smallest number indicates until which position we can compact our log, because records needs to be processed and exported by all exporters before we can delete them. In order to avoid running the exporters on all nodes and reduce the network load (since they export normally to an external system), the Leader has to periodically sync the last exported position to the followers. This is done via netty.

The last processed position corresponds to the last processed command on the leader **or** to the source position of last applied event on the follower.

#### Snapshotting

![LeaderFollowerProcessing](ZEP007/LeaderFollowerProcessing.png)

In order to take valid snapshots on the Leader we need to wait until the last written position of the stream processor is committed. This is necessary, because when we process a command we produce follow-up events, which correspond to our state changes. These state changes are immediately applied during processing, which means they will be part of the snapshot. In order to restore the same state on restart and not process or apply events twice we have to wait until the events, which we have written are committed.

On Follower's it is a bit different. The Follower applies events only. The last processed position here corresponds to the source position of the last applied event, which means the last written position corresponds here to the last applied event position. In general the follower doesn't need to wait until the last written position is committed, since he can only read committed entries anyway.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

[comment]: <> (       This is the technical portion of the ZEP. After reading it, a contributor should understand/know the following:)
[comment]: <> (       - [ ] The impact of the changes on other features is clear.)
[comment]: <> (       - [ ] The implementation is delineated)
[comment]: <> (       - [ ] Known corner cases are listed and addressed)
[comment]: <> (       The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.)
[comment]: <> (       - [ ] Does the ZEP affect the official Zeebe distribution &#40;e.g. configuration, logging&#41;?)
[comment]: <> (       - [ ] Does the ZEP require coordination with the platform team?)
[comment]: <> (       - [ ] Does the ZEP require coordination with the Operate team?)


In this section we will describe the proposed implementation in more detail. It is expected to have a fundamental understanding of the proposed concept, which was described [previously](#guide-level-explanation) and also in the current Zeebe design and implementation. We did a POC which covers this concept pretty well, you can find the summary [here](https://github.com/camunda-cloud/zeebe/issues/7328#issuecomment-868503971) and the related branch [here](https://github.com/camunda-cloud/zeebe/tree/zell-7328-poc-state-on-followers).

## Replay State Machine

In this section we will go into more detail how the so called `ReplayStateMachine` should look like.
We will describe the functionality and how it works based on the following process model. The model is a simplified version. For example, the "Read Next Event" covers some details like filtering out records which are not events etc.

![replayStateMachine](ZEP007/replayStateMachine.png)

When starting the state machine we will first seek to the snapshot position (or start from begin if there is none). After that we try to read the next event, filtering out other types of records. If there is no event, which can be applied, then we will check whether we are in a continuous replay mode or not.

On a Leader we just want to replay until the end of the log, to replay all remaining events to build up the latest state and then end the replay. Afterwards the Leader will go into the processing mode. He starts processing after the last processed position, which is part of the state. It corresponds to the last source position, from the last applied event.

The continuous replay is happening on the followers. If they reach the end of the log they are waiting for new events to be available, after this has happened the `ReplayStateMachine` will be triggered again. In order to achieve this kind of triggering `NewRecordListeners` are used, see next sub-section for more details.

If there exist an event on `Read next Event`, then this will be applied to the current state. After applying the state changes, the transaction will be committed. On both stages errors can happen, to be deterministic an endless retry strategy is used. We expect that errors should only be temporary. Otherwise, the event wouldn't be written to the log in the first place, the leader was able to apply that event to his state before. There are several possibilities to improve this approach, but this is out of scope.

Error events are replayed similar to other events, which means we will rebuild the blacklist on replay as well.

### New Record Listeners 

As mentioned before, we need the `NewRecordListeners` on the follower side to trigger our continuous replay. The same strategy we use on the Leader side for our [ProcessingStateMachine](https://github.com/camunda-cloud/zeebe/blob/develop/engine/src/main/java/io/camunda/zeebe/engine/processing/streamprocessor/ProcessingStateMachine.java). This replaces the old commit listeners from the `LogStream`. 

The listeners are registered on the `LogStream` abstraction. A simple version of the interface could
look like the following:

```java
public interface LogStream {

  /**
   * Registers a runnable on the LogStream, which should be called if a new record is available to be read.
   * 
   * @param runnable the Runnable which should be called
   */
  void notifyOnNewRecord(Runnable runnable);
}
```

Internally the `LogStream` implementation will use a raft commit listener to react on new committed entries. If a new entry is committed it is quite likely that a new application entry can be read from the log, this means the runnable will be called, such that the consumer (e.g. Processor- or ReplayStateMachine) can read the possible next record from the stream.

### CommittedEntryListeners

In order to enforce the separation of concerns we remove the commit listeners from the LogStream and replaced them with the `NewRecordListeners`. There is one use case where we need to listen for a specific position to be committed, on taking snapshots on the Leader side. We await that a certain position (lastWrittenPosition) is committed until we mark a snapshot as valid. See related [section](#snapshotting).

Note that Followers don't need to await the commit of a certain position, since they only replay committed data and don't produce new data on the log. For the Followers we have a simplified version of the SnapshotDirector, without awaiting the commit position.

To take snapshots in a performant way we introduce a new `CommittedEntryListener`, which is only called if RAFT is running in the Leader role. This approach avoids the need of having a mapping between index and position or using a separate reader to resolve the committed entry. 

```java

/**
 * This listener will only be called by the Leader, when it commits an entry. If RAFT is currently
 * running in a follower role, it will not call this listener.
 */
@FunctionalInterface
public interface RaftCommittedEntryListener {

  /** @param indexedRaftLogEntry the new committed entry */
  void onCommit(IndexedRaftLogEntry indexedRaftLogEntry);
}
```

As we can see the listener accepts an `IndexedRaftLogEntry`. The SnapshotDirector, expect entries especially positions to be committed, but internally in RAFT we commit indexes. This is the reason why we need some glue code to translate indexes to position's.

In order to explain that more in detail, we first need a basic understanding how a RaftEntry looks like and what an index is, for that we can take a look at the following picture.

![index](ZEP007/IndexedRaftEntry.png)

RAFT entries are written to the log. Each entry contains an index, which can be seen as the identifier of such an entry. Indexes are incremented by one. There are multiple types of raft entries, here we will just concentrate on a raft entry which contain application entries. The `IndexedRaftEntry` can contain multiple application entries. The application entries are sorted in a raft entry. The raft entry knows the lowest and highest position, which references the first and last application entry.

If the Leader commits an index it can immediately call the RAFT commit listeners with the corresponding `IndexedRaftEntry`. This is necessary, since we need to translate the index of the committed entry into a committed position. We can do this in the `AsyncSnapshotDirector` via asking for the highest position, if the entry contains application entries. With the highest position of the committed entry we know the recent commit position. Based on this knowledge we can determine whether the current snapshot is valid or not.

## Stream Processor

In order to run the continuous replay on the follower, the StreamProcessor will support two modes. The first one is the normal mode, which means replay until the end and then start normal processing of commands. This mode is used on the Leader side. The second mode is replay continuously which is used on the follower side. The mode can be set on building the StreamProcessor. 

## Exporter

Exporters are running on Leader's only. In order to make sure that we get exporter updates on the followers, we need to distribute that information to them. 

We do that via a SBE message, which is sent by the Leader `ExporterDirector` regularly. The SBE message will look like the following: 

```xml
  <sbe:message id="1" name="ExporterPositions">
    <group id="0" name="positions">
      <field id="0" name="position" type="uint64" semanticType="long"/>
      <data id="1" name="exporterId" type="varDataEncoding" semanticType="string"/>
    </group>
  </sbe:message>
```

The message contain exporterId and position tuples. The position is the last updated export position for that exporter, identified by his id. The Leader will publish this message on a certain topic (`exporter-$PARTITION_ID`), and the followers will subscribe to that topic. 

If the Follower receive such a message, he will store the content into his state. On the next snapshot and compaction the updated exporter positions can be taken into account.

## Zeebe Partition

As described in [State on Followers](#state-on-followers) sub-section, of the Guide Level explanation, we install certain services or components as a base on bootstrap. We showed that as the following process.

![stateOnFollower](ZEP007/stateOnFollower.png)

In our [POC](https://github.com/camunda-cloud/zeebe/blob/zell-7328-poc-state-on-followers/broker/src/main/java/io/camunda/zeebe/broker/Broker.java) we had the following bootstrap steps:

```java
  private static final List<PartitionStep> BOOTSTRAP_STEPS =
      List.of(
          new AtomixLogStoragePartitionStep(), // the bridge between the RaftLog and the Logstream
          new RaftLogReaderPartitionStep(), // opens an raft reader, which is reused by the state controller
          new LogDeletionPartitionStep(), // service which is in charge of compacting the log
          new StateControllerPartitionStep(), // controls the state managment, recovery, snapshotting etc
          new ZeebeDbPartitionStep(), // recovers the ZeebeDB
          new LogStreamPartitionStep() // this is just an abstraction around the real journal
          );
```

In order to transition to the different states we had the following transitional steps:

```java
  private static final List<PartitionStep> LEADER_STEPS =
      List.of(
          new LogStreamPartitionStep(), // for simplicity; we just recreate the logstream to reset
          // dispatcher and stuff - ideally we should close the old
          // one
          new StreamProcessorPartitionStep(), // installs the stream processor
          new SnapshotDirectorPartitionStep(), // snapshot director which takes regulary snapshots
          new RocksDbMetricExporterPartitionStep(),
          new ExporterDirectorPartitionStep()); // the exporter director which run the exporters and
              // transmit the exported positions to the follower

  private static final List<PartitionStep> FOLLOWER_STEPS =
      List.of(
          new LogStreamPartitionStep(), // we need to do it here as well since from Leader to
          // follower transition will cause closing the log stream
          new ZeebeDbPartitionStep(), // for simplicity; we need to recover the last snapshot from
          // Leader-To-Follower, on bootstrap we would do it twice now,
          // but this makes it still simpler to implement we dont want
          // to do it from Follower-To-Leader switch
          new StreamProcessorPartitionStep(ReplayMode.CONTINUOUSLY), // runs the StreamProcessor in replay only mode
          new SnapshotDirectorPartitionStep(), // snapshot director which takes regulary snapshots
          new RocksDbMetricExporterPartitionStep(),
          new ExporterSatellitePartitionStep() // in order to receive exporter updates
          );
```

This should be seen as an example, how it is actually done in the end is implementation detail. The key point is that we need to install some components once, and need to restore/reset some services on transitioning to a new role. We need to make sure that services which depend on each other have the latest reference. Like the SnapshotDirector, should take a snapshot of the actual database object etc. We need to make sure that all services are closed correctly. The ZeebePartition need to support bootstrap steps, which are installed only once. It has to take them done, if the partition is closed.

### Follower Install Request Handling

In order to keep it simple we handle new `InstallRequests` as a new Follower transition, which will cause to recreate all dependent services/components. Why an `InstallRequest` can happen, you can read about it [here](#re-init-follower-partition). How the recreation look like, is again more of an implementation detail. It can be done by calling the `RoleTransitionListeners` again on the Raft level. This approach was used in our [POC](https://github.com/camunda-cloud/zeebe/issues/7328#issuecomment-867683162) and seems to be the simplest one.

       
## Compatibility

We see no issues with this proposal.

### Rolling updates

We can have various starting scenarios, like followers have the same log, one follower is lagging behind, all followers are lagging behind. When thinking about rolling updates we need to consider how these different scenarios affect the procedure and outcome. 

#### Same state on all nodes

If we have the same state on all nodes and we start a rolling update we could start on a Leader or a Follower.

If we update first the Leader, a Follower will become next Leader, and the old Leader becomes Follower. The new Leader will replicate old entries, which should the Follower with the newest version be able to read in respect to backwards compatibility.

If we update first a Follower, it is similar to above.

#### Follower is lagging behind

If one follower is lagging behind it is likely to happen that if the other Follower or Leader is updated that one of them becomes Leader again. This means we have an old Follower and maybe a Leader with a newer version. In our processing and replay state machine we have event filters, which verify the version of the read entry. Newer entries are ignored by the Follower with the old version. This means for a short period of time the Follower, with the old version, will not replay. This should be fine, since we expect that the rolling update is in a limited time frame and this follower wouldn't take over anyway (since it is lagging behind). 

Interesting is the case where it might get an `InstallRequest` with a snapshot from a newer version. This could cause issue on restoring that state, but this will not affect RAFT itself. Only the Zeebe Partition of the Follower will be affected, and it is time limited until the Follower is updated.

The same states for the case were we have multiple followers lagging behind.

[comment]: <> (         This section should also list incompatible changes of Zeebe's public APIs, and make it explicit should there be any breaking changes.)
[comment]: <> (         Should there be any breaking changes, it should explicitly describe the migration path. Should there be no possible migration paths, it should instead explain why it is not possible, and why we decided that the benefits are worth breaking compatibility.)
[comment]: <> (         After reading this section, a contributor should know the following:)
[comment]: <> (         - [ ] Will it be possible to upgrade a Zeebe cluster?)
[comment]: <> (         - [ ] If applicable, what is the upgrade procedure? Is it automated?)
[comment]: <> (         - [ ] Does the ZEP break compatibility in the Go client?)
[comment]: <> (         - [ ] Does the ZEP break compatibility in `zeebe-client`?)
[comment]: <> (         - [ ] Does the ZEP break compatibility in `zeebe-bpmn-model`?)
[comment]: <> (         - [ ] Does the ZEP break compatibility in `zeebe-exporter-api`?)
[comment]: <> (         - [ ] Does the ZEP break compatibility in `zeebe-protocol`?)
[comment]: <> (         - [ ] Does the ZEP break compatibility in `zeebe-gateway-protocol`?)
[comment]: <> (         - [ ] Does the ZEP break compatibility in `zeebe-test`?)


## Testing


[comment]: <> (        You should describe what is the overall functionality that should be tested.)
[comment]: <> (        If you are omitting tests, explain why, and explain the impact if it fails, specifically the worst case scenario.)
[comment]: <> (        In each of the sections below, we should already list known cases that need to be tested in the final implementation, and at which level. The initial version here should be a best of effort: it is perfectly acceptable and expected that this section will be amended during implementation.)
        
### Unit

### Integration

  * We need to verify that the state is build on Leaders **AND** follower's. 
  * We need to verify that the state is the SAME on Leaders **AND** follower's.
    * Simple way would be to create an instance on the Leader, causing a Leader change and complete such an instance.
    * We could run our randomized property based tests and verify the states.
  * We need to make sure that the followers take snapshots and can compact their log.
    * Verify the disk usage and whether snapshots have been taken, we already have QA tests for that.
  * We need to verify that the blacklisting is build up correctly on the follower side and that if it becomes leader it ignores related commands.
    * Either causing a blacklisting of an instance, triggering a Leader change and verify whether the instance is ignored or we take a look at the states.  
  * In general, we should verify that fail-over works as expected and that the followers are building there state correctly, such that a new Leader can use it.
     * Similar to above.
  * After fail-over the new Leader should be able to export after the last lowest exporter position.
  * After fail-over the new Leader should be able to process after the last processed command.
  * InstallRequest's should be properly handled by followers, and they should continue with applying events after restoring the state.
    * Maybe we verify whether new snapshots are taken, which is a good indication that it still makes progress.

### E2E

 In the E2E test we should verify similar things:

 * We should verify that fail-over work as expected and that the followers are building there state correctly. The leaders should be able to continue where the followers left of and there should be no issue on exporting records from the new leader.

### Benchmarks

Verify the improvements in process execution latency during fail-over and also the latency from Follower-To-Leader processing transition, which was the main goal.

The new metric we should add here is the time between receiving the LEADER Role event (in the ZeebePartition) and starting to process (in StreamProcessor), because this is really the time we can decrease with the ZEP.

![leaderTransition](ZEP007/leaderTransition.png)

Furthermore, it would be interesting to see what kind of impact the snapshot interval has now. Smaller and larger snapshot interval, then the default of 5 minutes.

# Drawbacks
[drawbacks]: #drawbacks

 * Followers need to do more work. Previously, a follower just received a new snapshot and applied the received events to the log. Now, a follower replays all events to build the state. This might have an impact on performance.
   * In the POC we saw no performance regression
 * Generally the nodes will consume more resources.
   * We any way request to provide as much resources as a broker would need to be leader for all partitions

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

[comment]: <> (      - Why is this design the best in the space of possible designs?)
[comment]: <> (      - What other designs have been considered and what is the rationale for not choosing them?)
[comment]: <> (      - What is the impact of not doing this?)

## NoopWriter

As an alternative, I thought about using an NoopWriter which felt less natural and more complex than creating a ReplayStateMachine.

This means we have a new writer type, which contains a strategy to write given records. Based on the RAFT role we can switch this strategy.

**On being leader:** It will write to the dispatcher, get the position and return it to the caller. If, the dispatcher is full it returns -1, then the caller will retry.

**On being follower:** We will have a separate reader (maybe it contains it), which will fill a map with <source to written> position entries. This map is consumed by the writer to get the right writtenPosition, which will be returned to the caller. The entry itself is ignored, not written.  If there is no entry for a position, it returns -1. The caller (ProcessingStateMachine) will retry. It is filled, when the Leader has committed the corresponding follow-up event. This makes it possible that the follower will not process more than the leaders.

### On becoming Leader

On fail-over the new Leader need to drain the map, before he can switch to normal mode. Furthermore, the dispatcher needs to be reset. This allows to continue with the current state, without reinstalling any dependencies.

### On becoming Follower

If, the node was previously Leader, then the writer strategy will be replaced to no longer really write the entries. Possible race conditions can be handled on the RAFT level, since it detects whether a follower tries to write an entry. The new follower continuous processing, with the next command (it skips events) and fills with the reader the corresponding map.

## Exporter Records

Instead of sending the exported positions periodically over the wire, we could write the current state to the log. This can then be applied on the follower side to rebuild the exporter state, see related [issue](https://github.com/camunda-cloud/zeebe/issues/7088). This solution might be much more complex, since we need to handle here replay as well and this needs to be done in the `ExporterDirector` to keep the abstraction between the engine and exporters, which are part of the Broker.

## Commit Listeners

On the follower side we could also build a mapping of indexes and entries or positions. This solution would be a bit more complex and need more time, which is why we for now go with the separate reader approach on the follower side.

Furthermore, the idea came up whether we really need the knowledge about the last commit positions, it might be enough to know whether something new is visible to read. Like having event listeners on the LogStream, which notifies when a new record can be read. The issue with that is that currently we need the last committed position on the processing side for the `ErrorEvents` and on taking snapshots (on Leader side). If we find a solution for both of this we can think about replacing the commit listeners.

# Prior art
[prior-art]: #prior-art
     
[comment]: <> (      Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:)
[comment]: <> (      - For language, library, tools, and UI proposals: Does this feature exist in other tools/products and what experience have their community had?)
[comment]: <> (      - For community proposals: Is this done by some other community and what were their experiences with it?)
[comment]: <> (      - For other teams: What lessons can we learn from what other communities have done here?)
[comment]: <> (      - Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.)
[comment]: <> (      This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your ZEP with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.)
[comment]: <> (      Note that while precedent set by other products is some motivation, it does not on its own motivate a ZEP.)

There are several raft implementations out there and all of them doing it in the same way, the state machine is running on all nodes and build the state and take snapshot and compact the log independently.

Other implementations:

  * Hashicorp raft https://github.com/hashicorp/raft
  * etcd raft https://github.com/etcd-io/etcd/tree/master/raft
  * dragonboat https://github.com/lni/dragonboat

We also saw [above](#distinction-to-normal-raft) the Raft paper, which mentions that this is the way how we should implement it.

# Out of scope
[out-of-scope]: #out-of-scope

[comment]: <> (Call out anything which is explicitly not part of this ZEP.)

* Building and supporting large states was not covered by this ZEP.
* Improve error handling on replay 
* Performance improvement of apply event and commit transaction handling, maybe commit every 10 events etc.
* Requesting an `InstallRequest` by a follower and how to handle that is out of scope of this ZEP
* It might be an issue that the Leader compacts to early, after taking an snapshot, which causes immediately sending of `InstallRequest` to the followers. It was decided to not cover this in the ZEP, since it is not 100% clear how big the impact is.
* How we can improve our update test strategy was out of scope of this ZEP, and is not covered.

# Unresolved questions
[unresolved-questions]: #unresolved-questions
    
[comment]: <> (      - What parts of the design do you expect to resolve through the ZEP process before this gets merged?)
[comment]: <> (      - What parts of the design do you expect to resolve through the implementation of this feature before stabilization?)
[comment]: <> (      - What related issues do you consider out of scope for this ZEP that could be addressed in the future independently of the solution that comes out of this ZEP?)

 * What will the performance characteristics be?
   * We are pretty sure that the state building on followers will have a negative impact on the performance, but how bad we are not sure.
   * In our [POC we saw no performance regression](https://github.com/camunda-cloud/zeebe/issues/7328#issuecomment-868503971).
 * Can the proposed handling of install requests on Followers lead to concurrency issues?
   * When raft follower commits a snapshot, it immediately deletes all log segments while a stream processor reader is concurrently reading from it. 
   * It is possible that this case is already handled in the journal when we fixed the concurrency issues last quarter. We need to verify if it is a problem during the implementation.
 * How to handle concurrent snapshots on followers?
   * Receiving an InstallRequest, during taking a snapshot from the current state?
   * Theoretically the InstallRequest is newer. We should take a look during implement how we can solve that.

# Future possibilities
[future-possibilities]: #future-possibilities

[comment]: <> (      Think about what the natural extension and evolution of your proposal would be and how it would affect the language and project as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.)
[comment]: <> (      This is also a good place to "dump ideas", if they are out of scope for the ZEP you are writing but otherwise related.)
[comment]: <> (      If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.)
[comment]: <> (      Note that having something written down in the future-possibilities section is not a reason to accept the current or a future ZEP; such notes should be in the section on motivation or rationale in this or subsequent ZEPs. The section merely provides additional information.)
   
 * Investigate whether it is an issue to have a separate Reader on the Follower side for the committed raft entries
 * Investigate whether we can improve the performance on apply events in batches, before committing a transaction.
 * Investigate whether we need to switch from distributing exporter events over wire to writing ExporterRecords.
 * Investigate further how we can improve the switch between Follower-To-Leader and make it more performant.
