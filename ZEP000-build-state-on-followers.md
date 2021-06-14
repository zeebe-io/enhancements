---
title: Build state on followers
authors:
- Christopher Zell (@zelldon)
  reviewers: []
  approvers: []
  editor: TBD
  creation-date: 2020-04-28
  last-updated: 2021-06-14
  status: provisional
  see-also: []
  replaces: []
  superseded-by: []
---

# Summary
[summary]: #summary

We should build state on followers to be more reliable, future proven and enable fast fail-over.

# Motivation
[motivation]: #motivation

Generally we want to reduce the failover time. Especially we want to minimize the impact on process execution latency during failover on a partition. This can be achieved with building state on followers.

Furthermore, there are other issues we can solve with building state on followers, which are related to our current snapshot replication strategy. Followers can only delete data if they received snapshots and if they are valid. This means our current snapshot and replication mechanism needs to be reliable. In the past we had plenty bugs in the snapshot replication, which caused that followers where not able to compact their log. Additionally, there is currently two different snapshot replication strategies, we need to maintain. All of that can be removed when we build state on followers.

Another interesting side effect is that not periodically sending the snapshots over the wire would bring us nearer to the ability to support large states, which is currently not possible.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

      <!--
         Explain the proposal as if it was already included in the product and you were teaching it to another user/contributor. That generally means:
         
         - Introducing new named concepts.
         - Explaining the feature largely in terms of examples.
         - Explaining how our users/contributors should *think* about the feature, and how it should impact the way they use our product. It should explain the impact as concretely as possible.
         - If applicable, provide sample error messages, deprecation warnings, or migration guidance.
         - If applicable, describe the differences between teaching this to existing users/contributors and new users/contributors.
         
         For user facing ZEPs, this section should describe the benefits or changes the users will experience, from the point of view of the user.
         
         For maintenance/non-user facing ZEPs, this section should focus on how other contributors should reason about the changes, and give concrete examples of its impact, both short term and long term.
         
         For organizational ZEPs, this section should provide an example-driven introduction to the new policy or process, and explain its impact on the development process in concrete terms.
      -->

In this section I will describe and discuss the conceptional idea.

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

Unfortunately our processing state machine produces new events which need to be applied by the state machine again. This means that we can't just run the same processing state machine on all nodes, since only the leader should be able to create new records.

## State on Followers

Building state on followers and how we react on role changes is summarized in the following picture.

![stateOnFollower](images/stateOnFollower.png)

On bootstrapping a Zeebe Partition we install all needed services, like StreamProcessor, SnapshotDirector etc. After installing our services we go over to a state we call "Stream Processing", which is our real business logic and heart of each partition. Based on the RAFT role we perform different actions on the "Stream Processing". We can distinguish four cases which we will explain more in depth in the following sections:

  * Bootstrapping Leader Partition, no previously running partition
  * Bootstrapping Follower Partition, no previously running partition
  * Switch to Leader Partition, from a previously running Follower partition
  * Switch to Follower Partition, from a previously running Leader Partition.

In the following section we will use the term processing and replay heavily, please read [ZEP004](https://github.com/zeebe-io/enhancements/blob/master/ZEP004-wf-stream-processing.md) if you want to know more about it.

### Bootstrap Leader Partition

In this scenario we have no running Zeebe partition and installed it completely new. We restore the state, via copying the latest snapshot into the runtime folder. After installing all services, we start with the "Stream Processing", which is on the Leader separated in two parts. The first part is the processing, which consist of the following steps:

  * **Replay until end**, here we replay all existing events on the Log, which were not part of the Snapshot. This is necessary to get the latest state, before processing.
  * **Reset Dispatcher Position**, here we set the Dispatcher position which generates the position for our follow-up events.
  * **Allow Command API**, this means we're now accepting User commands being written to the Dispatcher, which will then later be processed.
  * **Process Commands**, we process commands in a loop and generate new follow-up events. There is of course more behind that state, but as high level this should be enough.

The second part is the exporting, which consist of the following steps:

  * **Restore last exported Position**, we restore the last exported position from the state
  * **Export Records**, we continuously export all kind of records. Exporters, normally sends data to an external system. 

In order to not run exporters on the followers, to avoid overhead, we communicate the last exported position to the followers regularly.

Furthermore, as part of the general "Stream Processing" a Timer is scheduled which takes periodically a snapshot from the state.

### Bootstrap Follower Partition

In this scenario we have no running Zeebe partition and installed it completely new. We restore the state, via copying the latest snapshot into the runtime folder. After installing all services, we start with the "Stream Processing", which is on the Follower quite simple. 

On of the Follower "Stream Processing" is the continuously applying of events, which called **Replay Events**, since they already applied on the Leader. This avoids generating follow up events on the follower. Only the Leader is allowed to write records, Single Writer Principle (SWP). Additionally, we can make sure that the follower is not faster than the leader, and we can easily switch to the Leader processing as we can see in the next section.

Furthermore, as part of the general "Stream Processing" a Timer is scheduled which takes periodically a snapshot from the state.

### Switch to Leader Partition

### Switch to Follower Partition



====
# NOTES:
====


      <!--
          
          We have several challenges we need to overcome to solve/implement build state on followers. Let's take a look of the following causality chain:
          
           * To have a working and long living system, we need to be able to compact our log no matter the role. 
           * To compact our log, we need to be able to take snapshots from the state.
             * The state need to contain a dense reflection of the log, last exported and processed position, from which we can recover.
           * To take a snasphot from the state, we need to build the state on all participants (leader and follower) in order to not replicate it.
           * To build state on followers, we need to avoid creating follow-up events by the state machine, Single Writer Principle. 
           * To avoid follow-up events, we need to either replay only or use a NoopWriter.
      
      -->
In the following sections we will go in more depth in some these mentioned parts of the chain.

### Compacting our log

As written above in order to compact our growing log, we need to take a projection of the state such that we can restore the state after a restart. This means we need to be able to take a snapshot (async) on all nodes.

Part of the snapshot should be the last processed and last exported position. The smaller number indicates until which position we can compact our log. In order to avoid running the exporters on all nodes and reduce the network load (since they export normally to an external system), the Leader has to periodically sync the last exported position to the followers. This is done via SWIM.

The last processed position corresponds to the last processed command on the leader **or** the last applied event on the follower. 

TODO: There is an issue with leaders compacting to early. Ask deepthi or nicolas about it.

### Avoid follow-up events on Followers

In order to build state on followers and avoid creating new follow-up events on the followers, which hurt our Single Writer Principle. We only apply events on the followers, it called "replay", since we apply the event again (they already have been applied on the leader).

#### Follower Replay

On the leader we run our normal command processing. On the followers we run the replay mode, only. For more details what replay and processing of commands means please take a look at the [ZEP004](https://github.com/zeebe-io/enhancements/blob/master/ZEP004-wf-stream-processing.md).

There exist two StateMachines, which are used in the StreamProcessor, depending on the RAFT role. This allows us to "hot-swap" the processing with replay and vice versa. The processing execution latency is kept to a minimum on fail-over. 




We need a new FollowerReplayStateMachine, which only consumes events continuously. The leader is the only one who is allowed to write commands and follow-up events, which ensures that the follower will not be faster than the leader.

```java
 /*
 * +------------------+   +-------------+           +------------------------+
 * |                  |   |             |           |                        |
 * |  startRecover()  |--->  scanLog()  |---------->|  reprocessNextEvent()  |
 * |                  |   |             |           |                        |
 * +------------------+   +---+---------+           +-----^------+-----------+
 *                            |                           |      |
 * +-----------------+        | no source events          |      |
 * |                 |        |                           |      |
 * |  onRecovered()  <--------+                           |      |    +--------------------+
 * |                 |                                    |      |    |                    |
 * +--------^--------+                hasNext             |      +--->|  reprocessEvent()  |
 *          |            +--------------------------------+           |                    |
 *          |            |                                            +----+----------+----+
 *          |            |                                                 |          |
 *   +------+------------+-----+                                           |          |
 *   |                         |               no event processor          |          |
 *   |  onRecordReprocessed()  |<------------------------------------------+          |
 *   |                         |                                                      |
 *   +---------^---------------+                                                      |
 *             |                                                                      |
 *             |      +--------------------------+       +----------------------+     |
 *             |      |                          |       |                      |     |
 *             +------+  updateStateUntilDone()  <-------+  processUntilDone()  |<----+
 *                    |                          |       |                      |
 *                    +------^------------+------+       +---^------------+-----+
 *                           |            |                  |            |
 *                           +------------+                  +------------+
 */
```

If we create that new FollowerReplayStateMachine we could support different strategies, what should happen if the end of the log is reached. For leader replay, it means stop. For follower replay, it means continuing.

##### On becoming Leader

On fail-over the new Leader need to drain the log, means he needs to consume all events before he can switch to the normal processing. We can achieve this via switching the strategy for reaching the end of the log. The dispatcher needs to be reset, such that it produces the correct next written positions.

##### On becoming Follower

If this was due to a restart, then we start the follower with the continuous replay mode.

If the new follower, was previously a leader it needs to stop the processing and starts the replay again. In RAFT we will immediately detect whether we are follower and want to write a new record, it will decline it. Since we only process committed data, we can be sure that nothing of the previously processed commands is gone. The replay needs to start from the last written position, since we have this event already applied.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


We can't reuse our current ReprocessingStateMachine, since it is written in a way such it doesn't support continuously replay. Currently, it is a one time thing. It scans the log for the last source position to determine, when it will stop with the replay. Furthermore, it collects error events to make sure the records are failing on reprocessing as well. Should be fixed with https://github.com/camunda-cloud/zeebe/issues/7270. We need a new FollowerReplayStateMachine, which only consumes events continuously. The leader is the only one who is allowed to write commands and follow-up events, which ensures that the follower will not be faster than the leader.

```java
 /*
 * +------------------+   +-------------+           +------------------------+
 * |                  |   |             |           |                        |
 * |  startRecover()  |--->  scanLog()  |---------->|  reprocessNextEvent()  |
 * |                  |   |             |           |                        |
 * +------------------+   +---+---------+           +-----^------+-----------+
 *                            |                           |      |
 * +-----------------+        | no source events          |      |
 * |                 |        |                           |      |
 * |  onRecovered()  <--------+                           |      |    +--------------------+
 * |                 |                                    |      |    |                    |
 * +--------^--------+                hasNext             |      +--->|  reprocessEvent()  |
 *          |            +--------------------------------+           |                    |
 *          |            |                                            +----+----------+----+
 *          |            |                                                 |          |
 *   +------+------------+-----+                                           |          |
 *   |                         |               no event processor          |          |
 *   |  onRecordReprocessed()  |<------------------------------------------+          |
 *   |                         |                                                      |
 *   +---------^---------------+                                                      |
 *             |                                                                      |
 *             |      +--------------------------+       +----------------------+     |
 *             |      |                          |       |                      |     |
 *             +------+  updateStateUntilDone()  <-------+  processUntilDone()  |<----+
 *                    |                          |       |                      |
 *                    +------^------------+------+       +---^------------+-----+
 *                           |            |                  |            |
 *                           +------------+                  +------------+
 */
```
     <!--
       
       This is the technical portion of the ZEP. After reading it, a contributor should understand/know the following:
       
       - [ ] The impact of the changes on other features is clear.
       - [ ] The implementation is delineated
       - [ ] Known corner cases are listed and addressed
       
       The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
       
       
       - [ ] Does the ZEP affect the official Zeebe distribution (e.g. configuration, logging)?
       - [ ] Does the ZEP require coordination with the platform team?
       - [ ] Does the ZEP require coordination with the Operate team?
       
     -->

## Compatibility

       <!--
         This section should also list incompatible changes of Zeebe's public APIs, and make it explicit should there be any breaking changes.
         
         Should there be any breaking changes, it should explicitly describe the migration path. Should there be no possible migration paths, it should instead explain why it is not possible, and why we decided that the benefits are worth breaking compatibility.
         
         After reading this section, a contributor should know the following:
         
         - [ ] Will it be possible to upgrade a Zeebe cluster?
         - [ ] If applicable, what is the upgrade procedure? Is it automated?
         - [ ] Does the ZEP break compatibility in the Go client?
         - [ ] Does the ZEP break compatibility in `zeebe-client`?
         - [ ] Does the ZEP break compatibility in `zeebe-bpmn-model`?
         - [ ] Does the ZEP break compatibility in `zeebe-exporter-api`?
         - [ ] Does the ZEP break compatibility in `zeebe-protocol`?
         - [ ] Does the ZEP break compatibility in `zeebe-gateway-protocol`?
         - [ ] Does the ZEP break compatibility in `zeebe-test`?
       -->

## Testing

      <!--
        You should describe what is the overall functionality that should be tested.
        
        If you are omitting tests, explain why, and explain the impact if it fails, specifically the worst case scenario.
        
        In each of the sections below, we should already list known cases that need to be tested in the final implementation, and at which level. The initial version here should be a best of effort: it is perfectly acceptable and expected that this section will be amended during implementation.
        
        ### Unit
        ### Integration
        ### E2E
      
      -->

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

    <!--
    
      - Why is this design the best in the space of possible designs?
      - What other designs have been considered and what is the rationale for not choosing them?
      - What is the impact of not doing this?
    
    -->

As an alternative, I thought about using an NoopWriter which felt less natural and more complex than creating a ReplyStateMachine.

#### NoopWriter

We have a new writer type, which contains a strategy to write given records. Based on the RAFT role we can switch this strategy.

**On being leader:** It will write to the dispatcher, get the position and return it to the caller. If, the dispatcher is full it returns -1, then the caller will retry.
**On being follower:** We will have a separate reader (maybe it contains it), which will fill a map with <source to written> position entries. This map is consumed by the writer to get the right writtenPosition, which will be returned to the caller. The entry itself is ignored, not written.  If there is no entry for a position, it returns -1. The caller (ProcessingStateMachine) will retry. It is filled, when the Leader has committed the corresponding follow-up event. This makes it possible that the follower will not process more than the leaders.

##### On becoming Leader

On fail-over the new Leader need to drain the map, before he can switch to normal mode. Furthermore, the dispatcher needs to be reset to the right position. This allows to continue with the current state, without reinstalling any dependencies.

##### On becoming Follower

If, the node was previously Leader, then the writer strategy will be replaced to no longer really write the entries. Possible race conditions can be handled on the RAFT level, since it detects whether a follower tries to write an entry. The new follower continuous processing, with the next command (it skips events) and fills with the reader the corresponding map.


# Prior art
[prior-art]: #prior-art
    <!--
      
      Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:
      
      - For language, library, tools, and UI proposals: Does this feature exist in other tools/products and what experience have their community had?
      - For community proposals: Is this done by some other community and what were their experiences with it?
      - For other teams: What lessons can we learn from what other communities have done here?
      - Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.
      
      This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your ZEP with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.
      
      Note that while precedent set by other products is some motivation, it does not on its own motivate a ZEP.
      
    -->

# Out of scope
[out-of-scope]: #out-of-scope

    <!-- Call out anything which is explicitly not part of this ZEP. -->

# Unresolved questions
[unresolved-questions]: #unresolved-questions
    
    <!--
    
      - What parts of the design do you expect to resolve through the ZEP process before this gets merged?
      - What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
      - What related issues do you consider out of scope for this ZEP that could be addressed in the future independently of the solution that comes out of this ZEP?
    
    -->

# Future possibilities
[future-possibilities]: #future-possibilities

    <!--
    
      Think about what the natural extension and evolution of your proposal would be and how it would affect the language and project as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.
      
      This is also a good place to "dump ideas", if they are out of scope for the ZEP you are writing but otherwise related.
      
      If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.
      
      Note that having something written down in the future-possibilities section is not a reason to accept the current or a future ZEP; such notes should be in the section on motivation or rationale in this or subsequent ZEPs. The section merely provides additional information.
    
    -->      
      
      <!--
      
      ------------------------------------------------------
      
      **OLD**
      
      ---
      
      title: Build state on followers
      
      authors:
      
        - Christopher Zell (@zelldon)
      
      reviewers: [@npepinpe, @menski, @deepthi]
      
      approvers: []
      
      editor: TBD
      
      creation-date: 2020-04-28
      
      last-updated: 2020-5-12
      
      status: provisional
      
      see-also: []
      
      replaces: []
      
      superseded-by: []
      
      ---
      
      # Summary
      
      [summary]: #summary
      
      We should build state on followers to be more reliable, future prooven and enable fast fail-over.
      
      # Motivation
      
      [motivation]: #motivation
      
      We currently have several issues with our snapshotting and replication strategy, some can be resolved
      
      when we build state on followers. Issues which could be solved by building state on followers are:
      
       * Followers can only delete data if they received snapshots and if they are valid. In the past we had lot of bugs in snapshot replication which caused that followers where not able to delete.
      
       * Snapshot replication needs to be reliable, we need to use checksum's and send them via network plus the actual data.
      
       * We have two different snapshot replication implementations and we need to maintain them separately.
      
       * Periodically snapshot replication consumes bandwith and cpu, which could be used by several other components.
      
       * Large state will cause problems on periodically snapshot replication.
      
      Periodically snapshot replication would be unnecessary, which is not only a problem on bigger state. Building state on Follower would enable us fast fail-over. We don't need to maintain multiple snapshot replication implementations. The implementation would be easier to understand, since most parts work in the same way unrelated to the raft role.
      
      # Guide-level explanation
      
      [guide-level-explanation]: #guide-level-explanation
      
      As you can read in the [raft paper](https://raft.github.io/raft.pdf) chapter 7, compaction and taking a snapshot should
      
      be done on each server.
      
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
      
      To better explain this topic we will use images from the [Zeebe-Overview](https://drive.google.com/drive/u/1/folders/13Orl4K_z-samdqTt8qHWotR2oPFjSN2a).
      
      If we take for example a look at the leader and follower partition, which can be modeled in a simplified way like this:
      
      ![leaderPartition](images/leaderPartitions.png)
      
      ![followerPartition](images/followerPartition.png)
      
      We can see that the Leader partition contains most of the logic and needs to replicate the records and also the snapshots, which are taken periodically.
      
      If we would build the state on followers, then the leader partition would look like this:
      
      ![leaderPartitionwithoutSnapshotReplication](images/leaderWithoutSnapshotReplication.png)
      
      We see that we no longer replicate the latest snapshot, but still we need to replicate the last lowest exported position.
      
      The exporter position is used to determine which is the last common position until we can delete the log.
      
      _Alternatively we could run the exporters as well on followers, but this would increase the traffic in the cluster
      
      and load on the external systems._
      
      The follower partition would look like this:
      
      ![leaderPartitionwithoutSnapshotReplication](images/followerBuildsState.png)
      
      It looks now quite similar to the leader partition. As written above the difference is that the exporters are not running on the follower, which means we need to distribute the lowest exporter position. This is done via a new component called `RemoteExporterStation` (names are discussable of course). On the follower this component just receives the position
      
      and writes this to the state, on the Leader this component distributes the exporter position.
      
      Another difference is that the Appender was replaced by a `NoopAppender`, which only consumes the entries from the RingBuffer and throw them away. The follower is not allowed to write to the corresponding log, but in order to generate the same positions we need to use the ring buffer and writers. 
      
      With this changes the follower would be able build the state in the same way as the leader does (via stream processing) and we could remove our current periodically snapshot replication approach. The follower would be able to delete his log based on his
      
      current state instead of relying on the snapshot replication. 
      
      # Reference-level explanation
      
      [reference-level-explanation]: #reference-level-explanation
      
      In this reference level explanation we will discuss and explain the planed changes in more technical detail.
      
      ## Leader Partition
      
      Currently the Leader partition can be drawn like this:
      
      ![leaderPartition](images/leaderPartitions.png)
      
      We know that the Leader partition contains most of the logic and needs to replicate the records and also the snapshots, which are taken periodically. If we would build state on the followers this would mean we could remove snapshot replication logic, which is currently triggered on the `SnapshotStore` after a new snapshot is taken.
      
      In order to delete the log we need determine the last common position, based on the stream processor position and lowest exporter position. If the followers build there own state, they need to know the current lowest exporter position for this partition.
      
      The probably easiest way would be to distribute this position via gossip. We will introduce a new component called `RemoteExporterStation`, the name is discussable. This component is called by the `ExporterDirector`, when the
      
      export position is updated. This component will then share this information via gossip and store the position also in the corresponding `ExporterState`. 
      
      The interface could for example look like this:
      
      ```java
      
      public interface RemoteExporterStation extends ClusterMembershipEventListener {
      
        /**
      
        * Takes the newest exporter position, distributes it to its counter part and stores in the 
      
        * corresponding exporter state.
      
        */
      
        public void onNewExportPosition(long newExporterPosition); 
      
         /**
      
         * Triggered by gossip listener implementation to handle exporter position change.
      
         * Will update the corresponding exporter state.
      
         */
      
        public void receiveNewExporterPosition();
      
      }
      
      ``` 
      
      After changing the leader partition it would probably look like this:
      
      ![leaderPartition](images/leaderWithoutSnapshotReplication.png)
      
      ## Follower Partition
      
      The current follower partition looks like this:
      
      ![followerPartition](images/followerPartition.png)
      
      As already mentioned the most logic and processing is part of the leader partition. When we build state on followers the partition would look quite similar.
      
      They will differ only in two parts: exporters and not writing to the log.
      
      On the `ZeebePartition` where we currently install the different partitions and actors we would still need to make a small difference.
      
      We would install no exporters, but a component called `RemoteExporterStation`, which is used to receive the last exported position, see above for an example interface. When the component receives a new exporter position it will update the exporter state. This can then be use later, by our deletion components to determine what is the lowest common position. This works then in the same way as it does on the leader.
      
      On installing a follower partition we would probably also install not the same Logstream as we currently do. 
      
      We would need a logstream, which accepts to open writers and readers, but where the written events are not end in the backed logstorage. We might call it a *read-only* logstream, which contains an `LogStorageAppender`, which just consumes the `Dispatcher` but doesn't append the Blocks to th corresponding `LogStorage`. This is necessary such that the writers produce the same position in the `StreamProcessor`.
      
      If we would have that the stream processing and snapshotting would look the same as on the leader side. The follower partition would then look like this:
      
      ![leaderPartitionwithoutSnapshotReplication](images/followerBuildsState.png)
      
      ## Possible Problems
      
      ### Slow Follower
      
      If a follower is slower then the leader with processing and becomes later leader, this is not really a problem. It has indeed a snapshot with lower process position, but this is the same as we would increase the snapshot interval were snapshots are taken and replicated later. Our current reprocessing mechanism should handle that.
      
      If the follower is to slow to receive events, then raft is in charge to send snapshots from the leader. Then we need to halt our processing actors. Install the snapshot and recover from it.
      
      ### Fast Follower
      
      We might think it is a problem that if the follower is faster to process then the leader, because it stands on higher position and no event was created for that. 
      
      This is also not really a problem, because we will not take a snapshot or make it valid until the last written event position is reached.
      
      This means if this fast follower becomes leader only valid snapshots are used on recovery, this position have been reached by the old leader as well otherwise the snapshot can't be valid.
      
      ### Non-deterministic Event
      
      When an exception is happening during processing then the current workflow instance will be blacklisted and a corresponding error event is written to the log. The processing continues only after the error event is committed. 
      
      If we have such an event/exception on one node, it doesn't matter on follower or leader, then it might not happen on the other.
      
      When the exception happend on the leader we are on the happy path, since the leader is able to write an error event. This can also be consumed by the follower later, and at least at this point the instance can be marked as blacklisted.
      
      When the exception happens on a follower, we can't do much only update the state but we are not able to write an error event.
      
      We have to think about if this is really possible, since we retry recoverable exceptions. All other exceptions will cause blacklisting, which means normally an bug/error in the code. This we expect to happen also on followers.
      
      ## Compatibility
      
      This should not have any impact of the compatibility.
      
      ## Testing
      
      We should test that snapshots are take on all servers and fail over is still possible. Furthermore I would expect that instances which are started on a leader will continue after fail-over.
      
      ### Integration
      
      Snapshots are taken unrelated of the raft states.
      
      ### E2E
      
      Test that fail-over is done without problems.
      
      # Drawbacks
      
      [drawbacks]: #drawbacks
      
      It will take time and work, but I see no blocker.
      
      # Rationale and alternatives
      
      [rationale-and-alternatives]: #rationale-and-alternatives
      
      If we not doing it we would still need to fix some of our current issues separately with appropriate solutions.
      
      We could do only some of these things but we always limit ourself with this.
      
      There is no real way around, if we want to support at the end also long running instances it makes no sense to transfer big snapshots periodically over the network.
      
      # Prior art
      
      [prior-art]: #prior-art
      
      There are several raft implementation out there and all of them doing it in the same way
      
      the state machine is running on all nodes and build the state and take snapshot and compact the log independently.
      
      Other implementations
      
       * Hashicorp raft https://github.com/hashicorp/raft
      
       * etcd raft https://github.com/etcd-io/etcd/tree/master/raft
      
       * dragonboat https://github.com/lni/dragonboat
       
      As we also saw above the Raft paper which mentions that this is the way how we should implement it.
      
      # Out of scope
      
      [out-of-scope]: #out-of-scope
      
      # Unresolved questions
      
      [unresolved-questions]: #unresolved-questions
      
       - What are possible problems and what is the impact which we currently not see?
      
       - What is a good X for triggering size based snapshotting?
      
      # Future possibilities
      
      [future-possibilities]: #future-possibilities
      
      We might manage to replace the current implementation with something which is more bullet proven then our current one.
      
      --!>
