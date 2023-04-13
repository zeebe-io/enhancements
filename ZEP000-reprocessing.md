---
title: Replication through Reprocessing
authors:
  - Ole Schönburg
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: 2023-04-13
last-updated: yyyy-mm-dd
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
see-also:
  - "ZEP-0004-wf-stream-processing.md"
  - "ZEP-0007-build-state-on-followers.md"
---

# Summary
[summary]: #summary

Use deterministic reprocessing of commands instead of persisting and replicating events.
This reduces disk and network IO throughput.


# Motivation
[motivation]: #motivation

A Zeebe partition leader processes new commands which results in follow-up commands and events.
Followers receive all commands and events but only apply events.
Because events need to be replicated and exported, they also need to be persisted.
Since every command has at least one confirming event, events should account for at least 50% of disk and network IO.

Scaling IO is often more expensive and difficult than scaling CPU.
Additionally, Zeebe is often run on remote storage where throughput and latency is difficult to control.
Removing the need to persist and replicate events will significantly reduce resource consumption.
Reprocessing on followers requires deterministic processing which is helpful in many regards, for example testing.
Reprocessing makes Zeebe more flexible and enables other interesting experiments such as reprocessing on exporters.

This was already considered in [ZEP-007]:
> The normal raft approach would mean to run the same state machine on the Leader and Follower,
> in order to build the state and take snapshot's of it. 
> Unfortunately, our processing state machine produces new commands, which need to be applied to the state machine again.
> Writing new Commands to the Log, is only allowed to the Leader. Single Writer Principle. 
> In order to overcome this and build state on followers, we need a separate state machine for the Followers.

With the Stream-Platform abstraction we now have the tools to *run the same state machine* on followers.
The platform layer can choose to not write commands produced by followers, just like it can decide to not write events.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


## Followers
- run the same processing state machine as leaders
- processing results are discarded: no records written, no responses sent

## Leaders
- processing results are filtered: only commands are written, events are discarded

## Deterministic Processing

To ensure that followers apply the same events as leaders without simple replication,
processing commands must always result in the same events.
One source of non-determinism is time because replicas don't share the same clock.
This is solved by retrieving the time based on record timestamps.
When a leader writes a new record to the log, the record is timestamped.
When processing a record, the clock is set to match that timestamp.
This ensures that all replicas process based on the same logical clock and process deterministically.

## Exporting

See [unresolved-questions].


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Compatibility

See [unresolved-questions], there might be implications for exporters and rolling upgrades.

# Drawbacks
[drawbacks]: #drawbacks

Potentially more CPU usage on followers because they need to process commands.
This has an upside too because followers and leaders would have very similar resource utilization which is great for efficiency.

Any non-deterministic processing can lead to replicas drifting apart such that two replicas are not in the same state.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

# Out of scope
[out-of-scope]: #out-of-scope


# Unresolved questions
[unresolved-questions]: #unresolved-questions


## Exporting
How do we support the existing exporters?
They need to receive events and a leader can only provide them if they are kept in memory or persisted somewhere.
There is a large design space here, ranging from
1. persisting but not replicating events
2. persisting commands and events on secondary local storage, used only for exporting
3. persisting commands and events on external storage such as S3 and drive exporters from there

## Rolling Upgrade
Needs to be considered, could be tricky but hopefully not impossible.

# Future possibilities
[future-possibilities]: #future-possibilities


[ZEP-007]: https://github.com/zeebe-io/enhancements/blob/master/ZEP007-build-state-on-followers.md