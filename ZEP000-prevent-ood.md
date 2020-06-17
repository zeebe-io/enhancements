---
title: ZEP Enable brokers to recover from out of disk space due to exporter failure
authors:
  - Deepthi Akkoorath
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: yyyy-mm-dd
last-updated: yyyy-mm-dd
status: implementable

---

# Summary
[summary]: #summary

When exporter is slow or not available, it prevents the broker from taking snapshot and compacting the log. Eventually the broker goes out of disk space. This ZEP proposes a solution to prevent the broker from reaching a non-recoverable state. When the exporter is healthy again, the broker should be able to compact the log eventually.

When the broker's disk space usage goes above a threshold
  * Reject client requests
  * Stop generating internal events
	  * Reject commands from other partitions
	  * Pause stream processor so it doesnot write new follow up events
	  * Pause internal timer command - timer triggers and job timeout triggers

# Motivation
[motivation]: #motivation

Zeebe brokers compacts the event log periodically to free up the disk space. An event from the log can be removed only if it is processed, exported and a snapshot incuding the results of that event is persisted. It can happen that the event is not compacted because it is not exported due to some failures in the exporter such as the exporter is not available for a period or it is too slow that new events are produced much faster than it is exported. This results in increased disk usage and at some point the broker goes out of disk space. This leads to another problem that the broker cannot take snapshot because there is no space for the snapshot and as a result it cannot compact the logs leading to a non-recoverable state.

To prevent the broker from reaching such a non-recoverable state, we have the following proposal.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This feature add new configuration parameters.

- data.lowFreeDiskSpaceBufferWatermark
- data.highFreeDiksSpaceBufferWatermark
- data.diskSpaceMonitorFrequency

When free disk space available on a broker B is <= `data.highFreeDiskSpaceBufferWatermark` then all new user commands directed towards the partitions for which B is the leader will be rejected with a 'RESOURCE_EXHAUSTED' error.

When free disk space available on a broker B is <= `data.lowFreeDiskSpaceBufferWatermark` then the followers reject replication of events.

`diskSpaceMonitorFrequency` determines the frequency at which the broker monitors for disk space usage.

`lowFreeDiskSpaceBufferWatermark` and `highFreeDiksSpaceBufferWatermark` should be large enough to store a snapshot.

An example configuration can be :
lowFreeDiskSpaceBufferWatermark = 3GB
highFreeDiksSpaceBufferWatermark = 4GB
diskSpaceMonitorFrequency = 10s

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Broker starts a `DiskSpaceUsageMonitor` which peridiocally checks for disk space usage. The frequency at which it monitors is configured by `diskSpaceMonitorFrequency`. It monitors every `diskSpaceMonitorFrequency` seconds. When free disk space available goes below highFreeDiksSpaceBufferWatermark and when it again goes above the watermark it notifies `DiskSpaceUsageListener`s.

On disk space not available `CommandApiService` goes to "reject all requests" mode until the listener notifies that the disk space available again.
When disk space is not available `ZeebePartition` pauses the `StreamProcessor` which in turn notifies the `DueDateTimerChecker` and `JobTimeoutTrigger` to stop triggering timers. Streamprocessor stops processing more records, but it is still running. When the disk space becomes available again, `ZeebePartition` resumes the `StreamProcessor`.
Similarly the actors that receives commands (`LeaderManagementRequestHandler`) from other partition should also react to disk space and stop writing more events to the logstream.

`lowFreeDiskSpaceBufferWatermark` is used by the Journal. When the available space is < lowFreeDiskSpaceBufferWatermark, it does not create new segments and throws `OutOfDiskSpace` exception. We recommend lowFreeDiskSpaceBufferWatermark < highFreeDiksSpaceBufferWatermark to make sure leader can commit all events that are written by the streamprocessor before rejecting the requests. Otherwise, it can happen that we cannot commit a snapshot because it is waiting for an event to be committed.

Note that we don't trigger fail over when a leader goes out of disk space, because if the follower is also out of disk space it will lead to a situation where we can never recover. On the other hand, if the leader waits until the exporter failure is resolved, it can eventually take a snapshot and compact.


- Does the ZEP affect the official Zeebe distribution (e.g. configuration, logging)?
	- It add new configuration parameters
- Does the ZEP require coordination with the platform team? - May be - to find an optimal value for the parameters
- Does the ZEP require coordination with the Operate team? - No

## Compatibility

There are no compatability breaking changes.

## Testing

You should describe what is the overall functionality that should be tested.

If you are omitting tests, explain why, and explain the impact if it fails, specifically the worst case scenario.

In each of the sections below, we should already list known cases that need to be tested in the final implementation, and at which level. The initial version here should be a best of effort: it is perfectly acceptable and expected that this section will be amended during implementation.

### Unit
### Integration
### E2E

Automated test seems difficult. One way to test is as follows
* Start the cluster with replication and benchmark clients, also configure small disk size
* Simulate exporter failure by making the elastic index read-only
* Wait until the brokers starts rejecting all requests
* Revert read-only elastic index
* Wait until brokers start exporting and eventually reclaims the disk space.
* Verify that brokers starts accepting the requests and the througput goes back to normal.

# Drawbacks
[drawbacks]: #drawbacks

The solution is not complete. There are cases where we cannot recover from out of disk space. For example, the leader has enough space but both followers are out of disk space. In order for the followers to compact, the leader should send a snapshot. However the leader cannot take a snapshot because it cannot commit the follow up events.

The solution also assumes that the configured freeDiskSpaceWatermarks are large enough to take a snapshot. It is not easy to estimate how large the snapshot will be.

The solution also does not help if some external entity is responsible for filling up the disk.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The proposed design is simple to implement and helps to recover from out of disk space when the main reason is exporter failure.

In the cloud it is possible to increase the size of pvc. That would help to recover in many cases. However it requires manual intervention.

# Prior art
[prior-art]: #prior-art

ElasticSearch uses similar parameters to transition to a read-only mode when the disk usage crosses a threshold.

# Out of scope
[out-of-scope]: #out-of-scope

* Preventing out of disk space due to internal load. If there are lot of timers, it is possible that there are lot of timer events in the log, but the stream processor is incapable of processing them at the same rate as they are written. This can also lead to Out of disk space as the stream processor is way behind and we cannot compact. To prevent this case we need a solution where we stop writing these internal events, at the same time let the stream processor continue so that eventually a snapshot is taken and the log is compacted.

# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities

* Use heuristics to calculate `highDiskSpaceUsageWatermark`. We can estimate the snapshot size from the size of runtime. Ensure that there is enough disk space to take the snapshot.
* Leader also knows about the disk space use of the followers. If the followers are going out of disk space, leader should stop writing new events.
* Also if the leader is going out of diskspace but follower has enough disk space, a step down would be ideal.
