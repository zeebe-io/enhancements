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

 * We have two different snapshot replication implementations and we need to maintain them separately.
 * Snapshot replication needs to be reliable, we need to use checksum's and send them via network plus the actual data.
 * Followers can only delete data if they received snapshots and if they are valid
 * periodically snapshot replication consumes bandwith and cpu, which could be used by several other components
 * large state causes problems on periodically snapshot replication  

We would avoid unnecessary snapshot replications, which is not only a problem on bigger state. Furthermore this would enable us fast fail-over. We don't need to maintain multiple snapshot replication implementations. The implementation would be easier to understand since it works in most parts the same as on the leader. 

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

This means the normal processing would run on all nodes unrelated of the raft state. The writing of follow up events is only done on the leader side, which means on followers we just use NoopWriters. Exporters could run on all nodes as well, since we expect them to be idempotent, but to reduce network traffic e.g. for ElasticExporter we could say that we only run exporters on leaders and share the last common exported position periodically via gossip.

If followers build there own state and take snapshot of it, this also means compaction is done independently on all nodes.

**Slow Follower**
If a follower is slower then the leader with processing and becomes later leader, this is not really a problem. It has indeed a snapshot with lower process position, but this is the same as I would increase the snapshot interval. Our current reprocessing mechanism should handle that.

If the follower is to slow to receive events, then normally atomix sends snapshots from the leader. Then we need to halt our processing actors. Install the snapshot and recover from it.

**Fast Follower**
We might think it is a problem that if the follower is faster to process then the leader, because it stands on higher position and no event was created for that. 

This is also not really a problem, because we will not take a snapshot or make it valid until the last written event position is reached.
This means if this fast follower becomes leader only valid snapshots are used on recovery, this position have been reached by the old leader as well otherwise the snapshot can't be valid.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The follower and leader should both run the `StreamProcessor`. On the follower side we use a `NoopWriter`to not write follow up events. 

See also above #guide-level-explanation for more information.

## Compatibility

This should not have any impact of the compatibility.

## Testing

We should test that snapshots are take on all servers.

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
