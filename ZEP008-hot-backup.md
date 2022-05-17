---
title: Take backup of Zeebe brokers without downtime
authors:
  - Deepthi Akkoorath
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: 2022-05-16
last-updated: yyyy-mm-dd
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
see-also:
replaces:
superseded-by:
---

# Summary
[summary]: #summary

<!--
One paragraph summary of the feature
-->

# Motivation
[motivation]: #motivation

<!--
- [ ] Why are we doing this?
- [ ] What problem are we solving?
- [ ] What is the expected outcome?
-->

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

For users:
How the API could look like. This should explain how a user/operator can manually take a backup and/or automate it.

Client send takeBackup command
- What happens when client get a success response?
- What happens when client get a failure response?
- What happens when the request timeout and client did not get a response?

What can fail?
How to monitor backups?
What should the user do if a backup failed?
How/when to retry?

Impact of taking a backup
 - Is too frequent backups a problem?
 - best practices?


For developers:
High level overview of backup process
- Coordinator send request to each partition
- Each partition takes checkpoint independently
- checkpoint command, checkpointId, checkpointPosition
- checkpoint induced by remote message
Highlevel overview of restore process
- During restore find the checkpointPosition and delete all entries before that.
-------------------

### What happens behind the screen

Backup of a zeebe cluster consists of backup of all partitions. A backup of a partition consists of a snapshot and the logStream containing the commands and events after the snapshot position. A partition should be able to restart from this backup the same way it restores its state after a failover or after a normal restart.

For the backup process, we introduce the following concepts:
-  Checkpoint command : This is a command which triggers the backup process with in a partition. A checkpoint can be triggered in two ways.
    1. By a new record `checkpoint`.
    2. When a command send by another partition is received by the partition. These commands are the ones send between partitions by the StreamProcessor. They are the commands related to deployment distribution and message correlation.
- `checkpointId` : This is the id of the backup triggered by the checkpoint command. `checkpointId` must be included in the checkpoint command. CheckpointId is ordered. A backup must have a checkpointId higher than all previous backups.
- `checkpointPosition` : This is the position of the command in the `logStream` which triggered the backup.
- `restore x`: A new command to identify the restored position. This is only used when zeebe is restored from a backup.

#### Highlevel overview of backup process

This is a high level view of the backup process. The detailed process is described in the Reference-level explanation.

##### Coordinator
Coordinator could be a gateway. Coordinator receives a request to take backup from the user and sends a response back to the user regarding the status of the backup.

On receiving a backup request:
- Sends a command `checkpoint Id` to all partitions
- When coordinator receives a response from all partitions, it sends the response back to the user.

##### In each partition
###### StreamProcessor (During processing):  
On reading a command:
1. if state.checkpointId < command.checkpointId
    - trigger a checkpoint
        - The checkpoint must include the current snapshot and the log until this record.
    - state.checkpointId = command.checkpointId
    - state.checkpointPosition = command.checkpointPosition
    - write follow up event, `checkpointTaken checkpointId`
2. Process command
    - If an engine command, engine will process it.
    - If it is `checkpoint` record, then send a response to the coordinator.
    - If it is `restore X` record, then
        - update state.checkpointId=X
        - update state.checkpointPosition=command.position
        - write followup event `restored X`

###### StreamProcessor (During Replay):
- On replaying `checkpointTaken X` event,
    - update state.checkpointId=X
    - update state.checkpointPosition=event.sourcePosition
- On replaying `restored X`
    - update state.checkpointId=X
    - update state.checkpointPosition=event.sourcePosition

###### Inter-partition communication
- Any command that is sent from a StreamProcessor to another partition's StreamProcessor will contain `state.checkpointId` at the time the command was created by the sender. If the command has to be resent, it uses the same checkpointId used in the first attempt.

##### Restore
Restore from backup is a special process that should happen before any normal operations of a zeebe cluster is started.
On Restore from a backup X:
- Delete all records with position >= checkpointPosition of backup X
- Write a new command `restore X` at checkpointPosition

After restoring, when StreamProcessor process the command `restore X`, it just updates `state.checkpointId = X` and `state.checkpointPosition =` command.position.

##### Example

Consider a system with two partitions. Coordinator sends `checkpoint` command to both partitions. There is also inter-partitions communication around the same time.

###### Case 1
Two partitions receive checkpoint command around the same time. The following is a representation of the logStream. Each row represents a record (a command or an event).

position | command/event | checkpointId | other data |
---|---|---|---|
11 | .. | X-1 |  .. |
12 | checkpoint | X |  |
13 | Deployment:Create | .. |  |
14 | checkpoint taken | X | follow up of 12 |
15 | Deployment:Distribute | X | followup of 13 |

position | command/event | checkpointId | info |
---|---|---|---|
11 | .. | X-1 |  .. |
12 | checkpoint | X |  |
13 | Deployment:Create | X | command received from partition 1 |
14 | checkpoint taken | X | follow up of 12 |
15 | Deployment:Created | X | followup of 13 |


In this case, both partition takes the checkpoint when the `checkpoint` command is received (i.e. processed). Only after taking the checkpoint, they receive the remote command. Since the checkpoint is already taken, the remote command will not force a new checkpoint. In this case, `checkpointPosition` for both partitions is 12.

On restore, we have to trim the log after the checkpointPosition. So after the restore operation, the logs of each partition will be:

position | command/event | checkpointId | other data |
---|---|---|---|
11 | .. | X-1 |  .. |
12 | restore | X | written by restore operation |

position | command/event | checkpointId | info |
---|---|---|---|
11 | .. | X-1 |  .. |
12 | restore | X | written by restore operation |


###### Case 2
Two partitions receive checkpoint command. But one partition receives the remote command before the checkpoint.

position | command/event | checkpointId | other data |
---|---|---|---|
11 | .. | X-1 |  .. |
12 | checkpoint | X |  |
13 | Deployment:Create | .. |  |
14 | checkpoint taken | X | follow up of 12 |
15 | Deployment:Distribute | X | followup of 13 |

position | command/event | checkpointId | info |
---|---|---|---|
11 | .. | X-1 |  .. |
12 | Deployment:Create | X | command received from partition 1 |
13 | checkpoint | X |  |
14 | checkpoint taken | X | follow up of 12 |
15 | Deployment:Created | X | followup of 12 |
16 | checkpoint ignored | X | follow up of 13 |

In this case, partition 2 receives the remote command before the `checkpoint` command. The remote command forces partition 2 to take a checkpoint. When `checkpoint` command is received, the checkpoint is already taken. Hence, new checkpoint won't be taken. In this case, `checkpointPosition` for partition 2 is also 12, even though the `checkpoint` command is at position 13.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

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

Explain the backup process in detail
- Why is it correct.
- How race conditions are handled
- How edge cases are handled
- How failures are handled

Explain restore process in detail
- Why is it correct
- Edge cases


### What is a consistent backup

Consistent cut

### Algorithm to find a consistent cut
BCS algorithm


#### Correctness of high-level process

How the process is aligned with BCS algorithm


### Backup Process in Zeebe

Challenges:
Taking backup of a partition can take a long time. We do not want to block the StreamProcessor during this period. To prevent blocking, we should allow the backup to be taken concurrently to the StreamProcessor.
When the backup is taken concurrently, it leads to other challenges. Snapshot is also taken concurrently and a concurrent snapshot can lead to log compaction. So before the backup is taken, a new snapshot might have been take resulting in deleting the old snapshot and compacting the log. The new snapshot might be already at a position further to the checkpointPosition rendering the backup to be invalid.
The challenge here is to allow asynchronous backup while preventing concurrent snapshots and compaction from making the backup invalid.

We extend the above high-level process to allow asynchronous backup as follows.

The components involved in Backup.
* A Coordinator
* BackupStore
In a partition:
	- StreamProcessor
	- SnapshotDirector
	- BackupActor
	- SnapshotStoreActor

###### Coordinator
A backup coordinator is external to all partitions. The gateway could act as a coordinator.

To take a backup `backupId`, coordinator send the command `checkpoint backupId` to all partitions.
The coordinator receives an acknowledgment from the partitions when the partitions have started taking the backup (when the StreamProcessor has processed the command). Note that it does not wait until the backup has completed, because it can take a long time.

###### BackupStore
BackupStore is a remote storage where each partition can backup it's data. Backup[N] refers to a backup with id N.

Backup[N].status = doesNotExist | ongoing | completed | failed
Backup[N][p][b].files = contains backup N of partition `p` started by broker `b`
Backup[N][p][b].status = doesNotExist | ongoing | completed | failed

###### BackupActor

On receiving the command `takeBackup checkpointId position`
- currentSnapshot = get the current snapshot from SnapshotStore
- If `currentSnapshot.processedPosition`  < `position`
    - then "lock" the snapshot by sending a command to SnapshotStore
    - If successfully locked start taking backup
    - else mark the backup as failed.
- else
   - mark the backup as failed

on Backup Start:
- Initialize Backup[N][p][b].status = ongoing
    - If Backup[N][p][b] already exists and is not completed, delete the backup and re-initialize it.
- Start copying files to Backup[N][b][p].files
	- copy locked snapshot
	- copy all segment files (until atleast the checkpointPosition)
    - store `checkpointPosition` along with the backup

On Backup Completed:
- Atomically Backup[N][p][b].status = completed
- unlock snapshot

On Backup Error:
- Set Backup[N][p][b].status = failed
- Set Backup[N] = failed.
- unlock snapshot

on Replay Completed:
- N = gets the lastCheckpointId from StreamProcessor
- if Backup[N][p][*].status != completed
	then Backup[N][p][*].status = failed
		 Backup[N].status = failed

###### StreamProcessor:
On reading a command:
1. if state.checkpointId < command.checkpointId
    - trigger a checkpoint
        - The checkpoint must include the current snapshot and the log until this record.
    - state.checkpointId = command.checkpointId
    - state.checkpointPosition = command.checkpointPosition
    - write follow up event, `checkpointTaken checkpointId`
2. Process command
    - If an engine command, engine will process it.
    - If it is `checkpoint` record, then send a response to the coordinator.
    - If it is `restore X` record, then
        - update state.checkpointId=X
        - update state.checkpointPosition=command.position
        - write followup event `restored X`

During Replay:
- On replaying `checkpointTaken X` event,
    - update state.checkpointId=X
    - update state.checkpointPosition=event.sourcePosition.
- On replaying `restored X`
    - update state.checkpointId=X
    - update state.checkpointPosition=event.sourcePosition

Inter-partition communication:  
- Any command that is sent from a StreamProcessor to another partition's StreamProcessor will contain `state.checkpointId` at the time the command was created by the sender. If the command has to be resent, it uses the same checkpointId used in the first attempt.


###### SnapshotStoreActor:
- On snapshot lock request
	- Mark the snapshot as locked in memory
	- A locked snapshot should not be deleted until it is unlocked
	- When a snapshot is locked, the corresponding logs should be not compacted as well
- On snapshot unlock request
	- Remove the in-memory lock
	- Delete the snapshot if there is a newer snapshot
	- Compact the logs

###### SnapshotDirector:
Before committing the snapshot check the following:
If `snapshotId.processedPosition < streamProcessor.lastCheckpointPosition < lastWrittenPosition`, then abort the snapshot. This is needed to prevent the scenario where the concurrently taken snapshot has `processedPosition` > `checkpoint position`.


#### On Restore
When restoring from a backup
- Delete all records with position > checkpointPosition (including the command at the checkpointPosition.)
- Write new record `Restore backupId` at the checkpointPosition

Note:- Restore process should be completed before any normal operation of Zeebe including raft leader election. No new records should be written to the log before restore process is completed.

### Edge cases and failure scenarios

Explain known edge cases and expected failure scenarios and describe how the above algorithm handles them.

Concurrent snapshots and compaction

Failure scenarios due to failover

Failure scenarios
Here are some scenarios where the backup can fail or interrupted before it is completed.

Scenario 1:

- StreamProcessor sends "takeBackup" command to BackupActor
- StreamProcess writes "checkpoint taken" record
- Leader change before "checkpoint taken" is committed
- BackupActor did not start taking checkpoint before the leaderchange

This case is fine as it will not lead to any incorrect state. After the leader change, the new leader will process the checkpoint command and triggers the backup again.

Scenario 2:

- StreamProcessor sends "takeBackup" command to BackupActor
- StreamProcess writes "checkpoint taken" record
- BackupActor starts taking checkpoint
- Leader change before "checkpoint taken" is committed

In this case, the new leader process the checkpoint command and triggers the backup again. But a partial backup Backup[N][p][old leader] exists. This is ok. The new leader will start taking the backup to a new location Backup[N][p][new leader] and will commit the new one. The partial backup from the old leader will remain there, which can be pruned in later.

Scenario 3: (this case is very less likely)

- StreamProcessor sends "takeBackup" command to BackupActor
- StreamProcess writes "checkpoint taken" record
- BackupActor starts taking checkpoint
- BackupActor completes taking the checkpoint
- Leader change before "checkpoint taken" is committed

This scenario is very rare because usually the committing `checkpoint taken` record will be much faster than taking a backup.
In this case, the new leader process the checkpoint command and triggers the backup again. But a completed backup Backup[N][p][old leader] exists. The new leader will start taking the backup to a new location Backup[N][p][new leader] and will commit the new one. This is a bit tricky. If the new leader commits a backup with the same id, it is semantically equivalent to the backup from the old leader. So there is no harm in allowing two backups for the same partition. However, this could be confusing and consumes extra storage which is unnecessary. But since it doesn't lead to any inconsistent state, and this scenario is very rare, we can allow this to happen to keep it simple.

Scenario 4:

- StreamProcessor sends "takeBackup" command to BackupActor
- StreamProcess writes "checkpoint taken" record
- BackupActor starts taking snapshot
- checkpoint taken record is committed
- Failover happens before backup is completed

In this scenario, the new leader will not process the checkpoint command. It will only replay `checkpoint taken` record. When replaying it updates the checkpointId and position, but will not trigger a backup. As a result, we have a partial backup for this partition initiated by the old leader. Since the new leader does not take a backup, the backup will never be completed. The coordinator or a user monitoring the backup will observe it as "ongoing" for ever. Inorder to prevent that, we have added a cleanup after the failover. After failover, once the replay is completed, the `BackupActor` marks the previous checkpoint as failed if it is still marked as ongoing.

Note:- In some cases, it might be possible for the new leader to take a consistent backup. But to keep it simple we fails the backup.

Scenario 5:

- StreamProcessor sends "takeBackup" command to BackupActor
- StreamProcess writes "checkpoint taken" record
- BackupActor starts taking snapshot
- checkpoint taken record is committed
- New snapshot is taken with processedposition > checkpoint position
- Failover happens before backup is completed
- New leader starts with the new snapshot at processedPosition > checkpoint position

This is similar to scenario 4. The difference is that the checkpoint taken is not replayed. But everything else is similar. In this case, there is no way for the new leader to take a consistent backup because the snapshot is already past the checkpoint record and it is probably already compacted.

Scenario 6:

- StreamProcessor sends "takeBackup" command to BackupActor
- StreamProcess writes "checkpoint taken" record
- checkpoint taken record is committed
- Failover happens before BackupActor starts taking backup

This is similar to scenario 4. The difference is that there is no "ongoing" backup. But the new leader will still mark it as failed because it will not take a new backup.

Scenario 7:

- StreamProcessor sends "takeBackup" command to BackupActor
- StreamProcess writes "checkpoint taken" record
- checkpoint taken record is committed
- New snapshot is taken with processedPosition > checkpoint position
- Failover happens before BackupActor starts taking backup
- New leader starts with the new snapshot at processedPosition > checkpoint position

This is similar to scenario 6.

Scenario 8:

- StreamProcessor sends "takeBackup" command to BackupActor
- StreamProcess writes "checkpoint taken" record
- BackupActor starts taking snapshot
- checkpoint taken record is committed
- Backup fails due to some errors - such as i/o exceptions

This is outside of our control. The backup can fail due to several reasons such as remote storage is not available, or other i/o errors. We can retry it for a few times before marking the backup as failed.

How coordinator can monitor the status of the backup

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

<!--
Why should we *not* do this?
-->

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
-->

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

<!--
Call out anything which is explicitly not part of this ZEP.
-->

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
