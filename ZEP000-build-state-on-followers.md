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

We have several challenges we need to overcome to solve/implement build state on followers. Let's take a look of the following causality chain:

 * To have a working and long living system, we need to be able to compact our log. 
 * To compact our log, we need to be able to take snapshots from the state.
   * The state need to contain a dense reflection of the log, last exported and processed position, from which we can recover.
 * To take a snasphot from the state, we need to build the state on all participants (leader and follower) in order to not replicate it.
 * To build state on followers, we need to avoid creating follow-up events by the state machine, Single Writer Principle. 
 * To avoid follow-up events, we need to either replay only or use a NoopWriter.

In the following sections we will go in more depth in some these mentioned parts of the chain.

### Avoid follow-up events on Followers

I have two approaches in my mind (there are probably more), which I would like to show 

#### NoopWriter

We have a new writer type, which contains a strategy to write given records. Based on the RAFT role we can switch this strategy.

**On being leader:** It will write to the dispatcher, get the position and return it to the caller. If, the dispatcher is full it returns -1, then the caller will retry.
**On being follower:** We will have a separate reader (maybe it contains it), which will fill a map with <source to written> position entries. This map is consumed by the writer to get the right writtenPosition, which will be returned to the caller. The entry itself is ignored, not written.  If there is no entry for a position, it returns -1. The caller (ProcessingStateMachine) will retry. It is filled, when the Leader has committed the corresponding follow-up event. This makes it possible that the follower will not process more than the leaders.

##### On becoming Leader

On fail-over the new Leader need to drain the map, before he can switch to normal mode. Furthermore, the dispatcher needs to be reset to the right position. This allows to continue with the current state, without reinstalling any dependencies.

##### On becoming Follower

If, the node was previously Leader, then the writer strategy will be replaced to no longer really write the entries. Possible race conditions can be handled on the RAFT level, since it detects whether a follower tries to write an entry. The new follower continuous processing, with the next command (it skips events) and fills with the reader the corresponding map.

#### Follower Replay

On the leader we run our normal processing, as we currently do. On the followers we run the replay mode only.  

We can't reuse our current ReprocessingStateMachine, since it is written in a way such it doesn't support continuously replay. Currently, it is a one time thing. It scans the log for the last source position to determine, when it will stop with the replay. Furthermore, it collects error events to make sure the records are failing on reprocessing as well. Should be fixed with https://github.com/camunda-cloud/zeebe/issues/7270. We need a new FollowerReplayStateMachine, which only consumes events continuously. The leader is the only one who is allowed to write commands and follow-up events, which ensures that the follower will not be faster than the leader.

If we create that new FollowerReplayStateMachine we could support different strategies, what should happen if the end of the log is reached. For leader replay, it means stop. For follower replay, it means continuing.

##### On becoming Leader

On fail-over the new Leader need to drain the log, means he needs to consume all events before he can switch to the normal processing. We can achieve this via switching the strategy for reaching the end of the log. The dispatcher needs to be reset, such that it produces the correct next written positions.

##### On becoming Follower

If this was due to a restart, then we start the follower with the continuous replay mode.

If the new follower, was previously a leader it needs to stop the processing and starts the replay again. In RAFT we will immediately detect whether we are follower and want to write a new record, it will decline it. Since we only process committed data, we can be sure that nothing of the previously processed commands is gone. The replay needs to start from the last written position, since we have this event already applied.


[comment]: <> (------------------------------------------------------)

[comment]: <> (**OLD**)

[comment]: <> (---)

[comment]: <> (title: Build state on followers)

[comment]: <> (authors:)

[comment]: <> (  - Christopher Zell &#40;@zelldon&#41;)

[comment]: <> (reviewers: [@npepinpe, @menski, @deepthi])

[comment]: <> (approvers: [])

[comment]: <> (editor: TBD)

[comment]: <> (creation-date: 2020-04-28)

[comment]: <> (last-updated: 2020-5-12)

[comment]: <> (status: provisional)

[comment]: <> (see-also: [])

[comment]: <> (replaces: [])

[comment]: <> (superseded-by: [])

[comment]: <> (---)

[comment]: <> (# Summary)

[comment]: <> ([summary]: #summary)

[comment]: <> (We should build state on followers to be more reliable, future prooven and enable fast fail-over.)

[comment]: <> (# Motivation)

[comment]: <> ([motivation]: #motivation)

[comment]: <> (We currently have several issues with our snapshotting and replication strategy, some can be resolved)

[comment]: <> (when we build state on followers. Issues which could be solved by building state on followers are:)

[comment]: <> ( * Followers can only delete data if they received snapshots and if they are valid. In the past we had lot of bugs in snapshot replication which caused that followers where not able to delete.)

[comment]: <> ( * Snapshot replication needs to be reliable, we need to use checksum's and send them via network plus the actual data.)

[comment]: <> ( * We have two different snapshot replication implementations and we need to maintain them separately.)

[comment]: <> ( * Periodically snapshot replication consumes bandwith and cpu, which could be used by several other components.)

[comment]: <> ( * Large state will cause problems on periodically snapshot replication.)

[comment]: <> (Periodically snapshot replication would be unnecessary, which is not only a problem on bigger state. Building state on Follower would enable us fast fail-over. We don't need to maintain multiple snapshot replication implementations. The implementation would be easier to understand, since most parts work in the same way unrelated to the raft role.)

[comment]: <> (# Guide-level explanation)

[comment]: <> ([guide-level-explanation]: #guide-level-explanation)

[comment]: <> (As you can read in the [raft paper]&#40;https://raft.github.io/raft.pdf&#41; chapter 7, compaction and taking a snapshot should)

[comment]: <> (be done on each server.)

[comment]: <> (> This snapshotting approach departs from Raft’s strong)

[comment]: <> (leader principle, since followers can take snapshots without the knowledge of the leader. However, we think this)

[comment]: <> (departure is justified. While having a leader helps avoid)

[comment]: <> (conflicting decisions in reaching consensus, consensus)

[comment]: <> (has already been reached when snapshotting, so no decisions conflict. Data still only flows from leaders to fol)

[comment]: <> (lowers, just followers can now reorganize their data.)

[comment]: <> (>)

[comment]: <> (> We considered an alternative leader-based approach in)

[comment]: <> ( which only the leader would create a snapshot, then it)

[comment]: <> ( would send this snapshot to each of its followers. However, this has two disadvantages. First, sending the snapshot to each follower would waste network bandwidth and)

[comment]: <> ( slow the snapshotting process. Each follower already has)

[comment]: <> ( the information needed to produce its own snapshots, and)

[comment]: <> ( it is typically much cheaper for a server to produce a snapshot from its local state than it is to send and receive one)

[comment]: <> ( over the network. Second, the leader’s implementation)

[comment]: <> ( would be more complex. For example, the leader would)

[comment]: <> ( need to send snapshots to followers in parallel with replicating new log entries to them, so as not to block new)

[comment]: <> ( client requests.)

[comment]: <> (As the paper states it makes totally sense to build state and take snapshots of it on all servers unrelated of the raft state.)

[comment]: <> (To better explain this topic we will use images from the [Zeebe-Overview]&#40;https://drive.google.com/drive/u/1/folders/13Orl4K_z-samdqTt8qHWotR2oPFjSN2a&#41;.)

[comment]: <> (If we take for example a look at the leader and follower partition, which can be modeled in a simplified way like this:)

[comment]: <> (![leaderPartition]&#40;images/leaderPartitions.png&#41;)

[comment]: <> (![followerPartition]&#40;images/followerPartition.png&#41;)

[comment]: <> (We can see that the Leader partition contains most of the logic and needs to replicate the records and also the snapshots, which are taken periodically.)

[comment]: <> (If we would build the state on followers, then the leader partition would look like this:)

[comment]: <> (![leaderPartitionwithoutSnapshotReplication]&#40;images/leaderWithoutSnapshotReplication.png&#41;)

[comment]: <> (We see that we no longer replicate the latest snapshot, but still we need to replicate the last lowest exported position.)

[comment]: <> (The exporter position is used to determine which is the last common position until we can delete the log.)

[comment]: <> (_Alternatively we could run the exporters as well on followers, but this would increase the traffic in the cluster)

[comment]: <> (and load on the external systems._)

[comment]: <> (The follower partition would look like this:)

[comment]: <> (![leaderPartitionwithoutSnapshotReplication]&#40;images/followerBuildsState.png&#41;)

[comment]: <> (It looks now quite similar to the leader partition. As written above the difference is that the exporters are not running on the follower, which means we need to distribute the lowest exporter position. This is done via a new component called `RemoteExporterStation` &#40;names are discussable of course&#41;. On the follower this component just receives the position)

[comment]: <> (and writes this to the state, on the Leader this component distributes the exporter position.)

[comment]: <> (Another difference is that the Appender was replaced by a `NoopAppender`, which only consumes the entries from the RingBuffer and throw them away. The follower is not allowed to write to the corresponding log, but in order to generate the same positions we need to use the ring buffer and writers. )

[comment]: <> (With this changes the follower would be able build the state in the same way as the leader does &#40;via stream processing&#41; and we could remove our current periodically snapshot replication approach. The follower would be able to delete his log based on his)

[comment]: <> (current state instead of relying on the snapshot replication. )

[comment]: <> (# Reference-level explanation)

[comment]: <> ([reference-level-explanation]: #reference-level-explanation)

[comment]: <> (In this reference level explanation we will discuss and explain the planed changes in more technical detail.)

[comment]: <> (## Leader Partition)

[comment]: <> (Currently the Leader partition can be drawn like this:)

[comment]: <> (![leaderPartition]&#40;images/leaderPartitions.png&#41;)

[comment]: <> (We know that the Leader partition contains most of the logic and needs to replicate the records and also the snapshots, which are taken periodically. If we would build state on the followers this would mean we could remove snapshot replication logic, which is currently triggered on the `SnapshotStore` after a new snapshot is taken.)

[comment]: <> (In order to delete the log we need determine the last common position, based on the stream processor position and lowest exporter position. If the followers build there own state, they need to know the current lowest exporter position for this partition.)

[comment]: <> (The probably easiest way would be to distribute this position via gossip. We will introduce a new component called `RemoteExporterStation`, the name is discussable. This component is called by the `ExporterDirector`, when the)

[comment]: <> (export position is updated. This component will then share this information via gossip and store the position also in the corresponding `ExporterState`. )

[comment]: <> (The interface could for example look like this:)

[comment]: <> (```java)

[comment]: <> (public interface RemoteExporterStation extends ClusterMembershipEventListener {)

[comment]: <> (  /**)

[comment]: <> (  * Takes the newest exporter position, distributes it to its counter part and stores in the )

[comment]: <> (  * corresponding exporter state.)

[comment]: <> (  */)

[comment]: <> (  public void onNewExportPosition&#40;long newExporterPosition&#41;; )

[comment]: <> (   /**)

[comment]: <> (   * Triggered by gossip listener implementation to handle exporter position change.)

[comment]: <> (   * Will update the corresponding exporter state.)

[comment]: <> (   */)

[comment]: <> (  public void receiveNewExporterPosition&#40;&#41;;)

[comment]: <> (})

[comment]: <> (``` )

[comment]: <> (After changing the leader partition it would probably look like this:)

[comment]: <> (![leaderPartition]&#40;images/leaderWithoutSnapshotReplication.png&#41;)

[comment]: <> (## Follower Partition)

[comment]: <> (The current follower partition looks like this:)

[comment]: <> (![followerPartition]&#40;images/followerPartition.png&#41;)

[comment]: <> (As already mentioned the most logic and processing is part of the leader partition. When we build state on followers the partition would look quite similar.)

[comment]: <> (They will differ only in two parts: exporters and not writing to the log.)

[comment]: <> (On the `ZeebePartition` where we currently install the different partitions and actors we would still need to make a small difference.)

[comment]: <> (We would install no exporters, but a component called `RemoteExporterStation`, which is used to receive the last exported position, see above for an example interface. When the component receives a new exporter position it will update the exporter state. This can then be use later, by our deletion components to determine what is the lowest common position. This works then in the same way as it does on the leader.)

[comment]: <> (On installing a follower partition we would probably also install not the same Logstream as we currently do. )

[comment]: <> (We would need a logstream, which accepts to open writers and readers, but where the written events are not end in the backed logstorage. We might call it a *read-only* logstream, which contains an `LogStorageAppender`, which just consumes the `Dispatcher` but doesn't append the Blocks to th corresponding `LogStorage`. This is necessary such that the writers produce the same position in the `StreamProcessor`.)

[comment]: <> (If we would have that the stream processing and snapshotting would look the same as on the leader side. The follower partition would then look like this:)

[comment]: <> (![leaderPartitionwithoutSnapshotReplication]&#40;images/followerBuildsState.png&#41;)

[comment]: <> (## Possible Problems)

[comment]: <> (### Slow Follower)

[comment]: <> (If a follower is slower then the leader with processing and becomes later leader, this is not really a problem. It has indeed a snapshot with lower process position, but this is the same as we would increase the snapshot interval were snapshots are taken and replicated later. Our current reprocessing mechanism should handle that.)

[comment]: <> (If the follower is to slow to receive events, then raft is in charge to send snapshots from the leader. Then we need to halt our processing actors. Install the snapshot and recover from it.)

[comment]: <> (### Fast Follower)

[comment]: <> (We might think it is a problem that if the follower is faster to process then the leader, because it stands on higher position and no event was created for that. )

[comment]: <> (This is also not really a problem, because we will not take a snapshot or make it valid until the last written event position is reached.)

[comment]: <> (This means if this fast follower becomes leader only valid snapshots are used on recovery, this position have been reached by the old leader as well otherwise the snapshot can't be valid.)

[comment]: <> (### Non-deterministic Event)

[comment]: <> (When an exception is happening during processing then the current workflow instance will be blacklisted and a corresponding error event is written to the log. The processing continues only after the error event is committed. )

[comment]: <> (If we have such an event/exception on one node, it doesn't matter on follower or leader, then it might not happen on the other.)

[comment]: <> (When the exception happend on the leader we are on the happy path, since the leader is able to write an error event. This can also be consumed by the follower later, and at least at this point the instance can be marked as blacklisted.)

[comment]: <> (When the exception happens on a follower, we can't do much only update the state but we are not able to write an error event.)

[comment]: <> (We have to think about if this is really possible, since we retry recoverable exceptions. All other exceptions will cause blacklisting, which means normally an bug/error in the code. This we expect to happen also on followers.)

[comment]: <> (## Compatibility)

[comment]: <> (This should not have any impact of the compatibility.)

[comment]: <> (## Testing)

[comment]: <> (We should test that snapshots are take on all servers and fail over is still possible. Furthermore I would expect that instances which are started on a leader will continue after fail-over.)

[comment]: <> (### Integration)

[comment]: <> (Snapshots are taken unrelated of the raft states.)

[comment]: <> (### E2E)

[comment]: <> (Test that fail-over is done without problems.)

[comment]: <> (# Drawbacks)

[comment]: <> ([drawbacks]: #drawbacks)

[comment]: <> (It will take time and work, but I see no blocker.)

[comment]: <> (# Rationale and alternatives)

[comment]: <> ([rationale-and-alternatives]: #rationale-and-alternatives)

[comment]: <> (If we not doing it we would still need to fix some of our current issues separately with appropriate solutions.)

[comment]: <> (We could do only some of these things but we always limit ourself with this.)

[comment]: <> (There is no real way around, if we want to support at the end also long running instances it makes no sense to transfer big snapshots periodically over the network.)

[comment]: <> (# Prior art)

[comment]: <> ([prior-art]: #prior-art)

[comment]: <> (There are several raft implementation out there and all of them doing it in the same way)

[comment]: <> (the state machine is running on all nodes and build the state and take snapshot and compact the log independently.)

[comment]: <> (Other implementations)

[comment]: <> ( * Hashicorp raft https://github.com/hashicorp/raft)

[comment]: <> ( * etcd raft https://github.com/etcd-io/etcd/tree/master/raft)

[comment]: <> ( * dragonboat https://github.com/lni/dragonboat)
 
[comment]: <> (As we also saw above the Raft paper which mentions that this is the way how we should implement it.)

[comment]: <> (# Out of scope)

[comment]: <> ([out-of-scope]: #out-of-scope)

[comment]: <> (# Unresolved questions)

[comment]: <> ([unresolved-questions]: #unresolved-questions)

[comment]: <> ( - What are possible problems and what is the impact which we currently not see?)

[comment]: <> ( - What is a good X for triggering size based snapshotting?)

[comment]: <> (# Future possibilities)

[comment]: <> ([future-possibilities]: #future-possibilities)

[comment]: <> (We might manage to replace the current implementation with something which is more bullet proven then our current one.)
