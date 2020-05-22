---
title: Size based snapshots
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

We should take snasphots size based not time based, to prevent out of disk space better.

# Motivation
[motivation]: #motivation

We currently take snapshots on a fixed time interval, default every 15 minutes. This is problematic,
because:

 * we can't react on high disk usage
 * we might take snapshot unnecessary, on small load
 * it might happen a lot during this interval, which increases recovery time
 * we might not take any snapshot when we have periodical fail-overs before the snapshot interval, we have seen this before
 
Since we take snapshots time based this also means we compact time based, not load based.
The problems above can be solved if we would trigger the snapshotting directly in RAFT based on the
load and disk usage.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In the raft paper (_[Raft paper p. 13](https://raft.github.io/raft.pdf)_) this issue is highlighted as well
and it is proposed to do snapshots based on the current disk usage.

```
There are two more issues that impact snapshotting performance. First, servers must decide when to snapshot. If
a server snapshots too often, it wastes disk bandwidth and
energy; if it snapshots too infrequently, it risks exhausting its storage capacity, and it increases the time required
to replay the log during restarts. One simple strategy is
to take a snapshot when the log reaches a fixed size in
bytes. If this size is set to be significantly larger than the
expected size of a snapshot, then the disk bandwidth overhead for snapshotting will be small.
```

This makes also sense for our use case. With this approach we reduce some factors where we going out of disk space.
The reprocessing time becomes deterministic and journal recovery is more predictable and takes less time, since the journal doesn't grow so big as it does with the time based approach.
We will always delete after the same amount. Furthermore we don't need logic to check whether we have processed something or not to take a new snapshot.

Currently the leader partition looks like this:

![leaderPartition](images/leaderPartitions.png)

The `AsyncSnapshotDirector` takes periodically snapshots. It requests information from the stream processor to mark the snapshot valid etc. The exporter state is used later to determine until we can compact.

With the load based approach the trigger comes from the consensus module.

![leaderPartition](images/triggerSnapshot.png)

We still have the dependency to the stream processor and exporter, but no longer periodically trigger from
the `AsyncSnapshotDirector`. This would mean on higher load, we would take more often snapshots. This also means
 we replicate more often the related snapshots. If we build state on Follower's see #PR this problem would be mitigated.

If we have periodically fail-over we would still be able to trigger snapshot, when the disk usage is already high.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


**TODO**

What is the limit when to take snapshots? Maybe 30% disk usage? Probably needs to be evaluated and needs
to be configurable, also for better testing.

## Compatibility

This should not have any impact of the compatibility.

## Testing

We should test that snapshots are taken after a specific disk usage.

### Unit Test

Test configuration.
 
### Integration & E2E

Test that snapshots are take after given load is reached and log is compacted afterwards.

# Drawbacks
[drawbacks]: #drawbacks

Replication is done more often. Might be an issue on bigger state.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We keep it like it is, but we are not able to handle high load scenarios properly. We will still
have the issues which are pointed out above.

# Prior art
[prior-art]: #prior-art

There are several raft implementation out there and all of them doing it in the same way, since it 
is also described in that way in the raft paper see above.

Other implementations

 * Hashicorp raft https://github.com/hashicorp/raft
    * https://github.com/hashicorp/raft/blob/master/snapshot.go#L116 
 * etcd raft https://github.com/etcd-io/etcd/tree/master/raft
 * dragonboat https://github.com/lni/dragonboat
 
# Out of scope
[out-of-scope]: #out-of-scope

# Unresolved questions
[unresolved-questions]: #unresolved-questions

 - What are possible problems and what is the impact which we currently not see?
 
 - What is a good X for triggering size based snapshotting?

# Future possibilities
[future-possibilities]: #future-possibilities

We might manage to replace the current raft implementation with something which is more bullet proven then our current one.
