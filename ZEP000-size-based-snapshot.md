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

Instead of triggering time based in the `AsyncSnapshotDirector` we need to trigger load based. 
We should check on each or on a rate of `X` appends the disk usage. This could be done in the `AbstractAppender`.
When the disk usage is above an certain threshold, like above `30%`, then we could trigger snapshotting. I think this was also done
before in atomix in a similar way. Snapshotting can then be triggered quite often until we going under our threshold.

To avoid to take snapshots too often we could have an check in the `AsyncSnapshotDirector` which verifies that the 
processor position or exporter position has changed with an amount of `Y` positions. We probably need this check to avoid
to many snapshots at once, since we currently directly replicate these snapshots. This problem mitigates if we build state on follower.

After taking a snapshot we immediately trigger compaction based on the last processed position and exporter position.
It might happen that the exporter is slow and we are not able to compact yet or not so much. Depending how much we compact we
will probably triggered again, shortly afterwards. This is good, since we then can overcome our disk issue earlier.
After the processor or exporter position was for example updated by `Y` positions we will take a snapshot again and
might be able to compact more then before. If this is not the case then we will be triggered again. But at least we have
taken a snapshot with higher processed position, which means it will be faster on reprocessing.

The threshold should be configurable and a good value needs probably be evaluated.

## Compatibility

This should not have any impact of the compatibility.

## Testing

We should test:
 
 * snapshots are taken after a specific disk usage
 * snapshots are taken again after process or exporter position have changed by `Y`
 * compaction is done

### Unit Test

Test configuration.
 
### Integration & E2E

Test that snapshots are take after given load is reached and log is compacted afterwards.

# Drawbacks
[drawbacks]: #drawbacks

Replication could be done more often. Might be an issue on bigger state.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We keep it like it is, but we are not able to handle high load scenarios properly. We will still
have the issues which are pointed out above.

# Prior art
[prior-art]: #prior-art

There are several raft implementation out there and I think all of them doing it in the same way, since it 
is also described in that way in the raft paper see above. I think it was also done in atomix before.

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
