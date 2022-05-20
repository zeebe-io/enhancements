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

- [Summary](#summary)
- [Motivation](#motivation)
- [Guide-level explanation](#guide-level-explanation)
    - [API](#api)
    - [Internal Backup Process](#internal-backup-process)
- [Reference-level explanation](#reference-level-explanation)
    - [Backup process](#backup-process-in-zeebe)
    - [Edge cases and failure scenarios](#edge-cases-and-failure-scenarios)
- [Unresolved questions](#unresolved-questions)

# Summary
[summary]: #summary

This document describes a proposal for taking backup of Zeebe cluster without downtime.

In Guide-level explanation we describe:
1. API: How a user can take and manage backups. It doesn't explain a concrete api, but only an overview of how the api could look like. This section is targeted to both users and developers.
2. Overview of backup process in Zeebe. This explains how it works internally. This section is targeted to Zeebe developers.

In reference-level explanation we describe the backup process in detail. Here we explain, how failure scenarios and edge cases are handled.

# Motivation
[motivation]: #motivation

<!--
- [ ] Why are we doing this?
- [ ] What problem are we solving?``
- [ ] What is the expected outcome?
-->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Backup is stored in a remote storage accessible to all brokers. In theory, backup storage could be any remote storage such as a remote file system, cloud storage like Google Cloud Storage or S3 buckets. What types of remote storage are supported will be left to the implementation. For this document, we just assume any one of the remote storage type is available.

A backup of a Zeebe cluster consists of backup of each partition. A partition's backup consists of a snapshot and the compacted log stream. A backup is identified by a `backupId`. `backupId` is a unique integer to identify a backup. To restore from a backup, all partitions must restore from the same `backupId`. There will be only one backup for a partition in a Zeebe backup. That means, even if it is a replicated cluster, we only keep one replica of a backup per partition. All replicas of a partition must restore from this single backup.

### API

This section describes how a user can take and manage backups.
The minimum api we need to allow a user to take backups are:
* Trigger backup
* Monitor backup

In addition, we may also provide
* List backups
* Delete backup

We only discuss the minimum api here.

#### Trigger Backup

A user can trigger a backup by sending a `TriggerBackup` request to the coordinator.
The users must also provide a backupId.
A `backupId` is a unique integer that identifies a backup.
`backupId` is ordered. That means, a backup must have an id that is greater than all previous backups.
If two backups are triggered with the same backup id, only one backup will be taken.

The coordinator will acknowledge the request after all partitions have started taking the backup. The acknowledgment will be sent before the backup is completed.
Hence, it is important to have the monitor api to keep track of the backup.

In each partition, the backup consists of state until the backup is started. The backup will not contain any new data written after the request is acknowledged.

#### Monitor Backups

A user can monitor the status of the backup by sending monitor request to the coordinator.
The coordinator responds with a status - `doesNotExist | ongoing | completed | failed`.

- `doesNotExist` : TriggerBackup command with this `backupId` was never send.
- `ongoing` : Backup is currently being taken. Monitor again for the status.
- `completed`: Backup is taken successfully. The cluster will be able to successfully restore from this backup.
- `failed`: Backup has failed. A new backup with the same id cannot be taken. When retrying a new `backupId` must be used.

#### Restore

The user must specify the id of the backup from which Zeebe should restore from. Copying the data and restoring the state from it is done either by Zeebe or an accompanying helper application. Users shouldn't have to do anything manually.

### Internal backup process

Backup of a Zeebe cluster consists of backup of all partitions. A backup of a partition consists of a snapshot and the logStream containing the commands and events after the snapshot position. A partition should be able to restart from this backup the same way it restores its state after a failover or after a normal restart.

For the backup process, we introduce the following concepts:
-  Checkpoint command : This is a command which triggers the backup process with in a partition. A checkpoint can be triggered in two ways.
    1. By a new record `checkpoint`.
    2. When a command send by another partition is received by the partition. These commands are the ones send between partitions by the StreamProcessor. They are the commands related to deployment distribution and message correlation.
- `checkpointId` : This is the id of the backup triggered by the checkpoint command. `checkpointId` must be included in the checkpoint command. CheckpointId is ordered. A backup must have a checkpointId higher than all previous backups.
- `checkpointPosition` : This is the position of the command in the `logStream` which triggered the backup.
- `restore x`: A new command to identify the restored position. This is only used when Zeebe is restored from a backup.

#### Highlevel overview of backup process

This is a high level view of the backup process. The detailed process is described in the Reference-level explanation.

##### Coordinator
Coordinator could be a gateway. Coordinator receives a request to take backup from the user and sends a response back to the user regarding the status of the backup.

On receiving a `TriggerBackup backupId` request:
- Sends a command `checkpoint backupId` to all partitions
- When coordinator receives a response from all partitions, it sends the response back to the user.

##### In each partition
###### StreamProcessor (During processing):
```
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
```

###### StreamProcessor (During Replay):
```
- On replaying `checkpointTaken X` event,
    - update state.checkpointId=X
    - update state.checkpointPosition=event.sourcePosition
- On replaying `restored X`
    - update state.checkpointId=X
    - update state.checkpointPosition=event.sourcePosition
```

###### Inter-partition communication
- In any command that is sent from a StreamProcessor to another partition's StreamProcessor, `command.checkpointId` will be `state.checkpointId` at the time the command was created by the sender. If the command has to be resent, it uses the same `checkpointId` used in the first attempt.

##### Restore
Restore from backup is a special process that should happen before any normal operations of a Zeebe cluster is started.
On Restore from a backup X:
- Delete all records with position >= checkpointPosition of backup X
- Write a new command `restore X` at checkpointPosition

After restoring, when StreamProcessor process the command `restore X`, it just updates `state.checkpointId = X` and `state.checkpointPosition =` command.position.

##### Example

Consider a system with two partitions. Coordinator sends `checkpoint` command to both partitions. There is also inter-partition communication around the same time.

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

### What is a consistent backup

A global checkpoint (backup) in a distributed systems consists of a set of local checkpoints from the participating nodes. A *consistent* global checkpoint is a set of local checkpoints from which the nodes can recover without any inconsistencies in the state of each node in relation to the state of any other node.

In a few words a global checkpoint is consistent if it is a consistent cut.

To understand a consistent cut, let us introduce the concept of happened before relation.

An event $e_i^h$ occurred in process i at logical time h. A happened before ($\rightarrow$) relation between two events is defined as follows:  
An event $e_i^h \rightarrow e_j^k$  if and only if
 $(i = j \wedge k = h + 1) \vee$
 $(e_i^h = send(m) \wedge e_j^k = receive(m)) \vee$
 $(\exists e_p^n : (e_i^h \rightarrow e_p^n) \wedge ( e_p^n \rightarrow) e_j^k))$

That means an event e1 happens before e2 iff:
- both events happened in the same process and e1 occurred before e2
- e1 sends a message and e2 receives it
- e1 happened before e2 transitively

A cut is a subset of events in the state.
A cut C is consistent if : $\forall e_i, e_j : e_j \in C \wedge e_i \rightarrow e_j \implies e_i \in C$.
That means in consistent cut C, if an event exists in C then all events *happened before* it must also exist in C.


![consistent-cut](ZEP008/consistent-cut.png)

In the above image, the horizontal line depicts the time in each process. The vertical slanted arrow shows communication from one process to another. Dotted lines represent a cut. That means - we take a checkpoint where the dotted line meets the horizontal timeline. The green ones are a consistent cut. The red lines are cuts, but they are not consistent cuts. The red line is not consistent because e2 is in the cut, but e1 is not. e1 has happened before e2 because it is a "send-receive" pair.

### Algorithm to find a consistent cut

[BCS algorithm as proposed in this paper](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.149.5565&rep=rep1&type=pdf) is a way to take coordinated consistent checkpoints across multiple processes. (An easy to understand version is [here](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.78.4041&rep=rep1&type=pdf)). This is a communication induced checkpointing and relies on lazy coordination among the participating processes.

Here is how the algorithm works.

$C_i^{sn_i}$ is a local checkpoint of process $i$, and its checkpoint id is ${sn_i}$. When a process sends a message, it also embeds it current checkpoint id ${sn_i}$ with it.

A process takes it local checkpoint when
1. A basic checkpoint is triggered. A basic checkpoint is triggered either by an external trigger or it could be something that is triggered periodically.
2. A forced checkpoint is triggered. A forced checkpoint happens when a process received a message from another process, and it is forced to take a checkpoint to guarantee the consistency.

The algorithm is as follows:

*On basic checkpoint:*  
&emsp; $sn_i = sn_i + 1$  
&emsp; take checkpoint $C_i^{sn_i}$

*Upon the receipt of a message m:*  
&emsp; if $sn_i < m.sn$ then  
&emsp; &emsp; $sn_i = m.sn$  
&emsp; &emsp; take checkpoint $C_i^{sn_i}$
&emsp; process message $m$

It has been proven that if we follow the above algorithm, a global checkpoint $C = \{ C_i^{sn_i}, C_j^{sn_k}, C_l^{sn_l}, ...\}$ is a consistent cut if all local checkpoints in $C$ has the same id. $sn_i = sn_j = sn_l = ...$.

#### Correctness of high-level process

The high level process explained in [Guide-level explanation](#internal-backup-process) follows implements the BCS algorithm.

A record in the logstream represents an event in the system. The position of the event can be considered as the logical time when the event occurred.

An event is written to the log at one time, but it is actually processed later. But we consider the event "occurs" when the record is written to the log. So any records that exists before this record has happened before this event.
When a remote command is received by the partition, it is written to the log first. However, we trigger the checkpoint later when it is processed by the StreamProcessor. This is correct because when the record is processed by the StreamProcessor, the state also reflects the events that have happened before this record. The checkpoint consists of all records until this remote command.

##### Message Reordering

The commands send between partitions are not send over a reliable channel. Hence, StreamProcessor has an inbuilt re-try mechanism to handle message losses. As a result, it is possible that the messages are delivered out of order. However, this will not break the backup algorithm. Consider the following case:

![message reordering](ZEP008/message-reordering.png)

In this scenario, reordered messages does not lead to an inconsistent checkpoint.
In $CP-2$, neither of the messages are correlated.

![message reordering](ZEP008/message-reordering-2.png)

Here, in $CP-2$, message 2 is published and correlated, but message 1 is not correlated. This is ok, because this can also happen in a normal execution. When the cluster restore from this state, message 1 correlation command will be re-send.

### Detailed Backup Process in Zeebe

Taking backup of a partition can take a long time. We do not want to block the StreamProcessor during this period. To prevent blocking, we should allow the backup to be taken concurrently to the StreamProcessor.
When the backup is taken concurrently, it leads to other challenges. Snapshot is also taken concurrently, and a concurrent snapshot can lead to log compaction. So before the backup is taken, a new snapshot might have been taken resulting in deleting the old snapshot and compacting the log. The new snapshot might be already at a position further to the checkpointPosition rendering the backup to be invalid.
The challenge here is to allow asynchronous backup while preventing concurrent snapshots and compaction from making the backup invalid.

We extend the high-level process described in [Guide-level explanation](internal-bakcup-process) to allow asynchronous backup as follows.

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

To monitor the status of a backup, the coordinator sends a request to a partition.
The partition can then check the status of its backup and sends a response.
The coordinator then aggregates the status from all partitions.

###### BackupStore
BackupStore is a remote storage where each partition can backup it's data. Backup[N] refers to a backup with id N.

Backup[N][p][b].files = contains backup N of partition `p` started by broker `b`
Backup[N][p][b].status = doesNotExist | ongoing | completed | failed

Backup[N].status = completed if for all partitions p, Backup[N][p].status= completed.
Backup[N].status= failed if for atleast one partition p, Backup[N][p].status=failed.
Backup[N].status = ongoing if for atleast one partition p, Backup[N][p].status=ongoing and Backup[N].status != failed.

###### BackupActor

On receiving the command `takeBackup checkpointId position` :
```
currentSnapshot = get the current snapshot from SnapshotStore
if currentSnapshot.processedPosition  < position
    "lock" the snapshot by sending a command to SnapshotStore
    if successfully locked start taking backup
    else mark the backup as failed.
else
    mark the backup as failed
```
on Backup Start:
```
Initialize Backup[N][p][b].status = ongoing
     - If Backup[N][p][b] already exists and is not completed, delete the backup and re-initialize it.
Start copying files to Backup[N][b][p].files
	- copy locked snapshot
	- copy all segment files (until atleast the checkpointPosition)
    - store `checkpointPosition` along with the backup
```
On Backup Completed:
```
Atomically Backup[N][p][b].status = completed
unlock snapshot
```
On Backup Error:
```
Backup[N][p][b].status = failed
unlock snapshot
```
on Replay Completed:
```
N = gets the lastCheckpointId from StreamProcessor
if Backup[N][p][*].status != completed
	then Backup[N][p][*].status = failed
```
###### StreamProcessor:
On reading a command:
```
if state.checkpointId < command.checkpointId
    send a command `takeBackup checkpointId command.position` to BackupActor        
    state.checkpointId = command.checkpointId
    state.checkpointPosition = command.checkpointPosition
    write follow up event, `checkpointTaken checkpointId`
Process command
    If an engine command, engine will process it.
    If it is `checkpoint` record, then send a response to the coordinator.
    If it is `restore X` record, then
        update state.checkpointId=X
        update state.checkpointPosition=command.position
        write followup event `restored X`
```
During Replay:
```
On replaying `checkpointTaken X` event,
    update state.checkpointId=X
    update state.checkpointPosition=event.sourcePosition.
On replaying `restored X`
    update state.checkpointId=X
    update state.checkpointPosition=event.sourcePosition
```
Inter-partition communication:  
- Any command that is sent from a StreamProcessor to another partition's StreamProcessor will contain `state.checkpointId` at the time the command was created by the sender. If the command has to be resent, it uses the same checkpointId used in the first attempt.


###### SnapshotStoreActor:
```
On snapshot lock request
	Mark the snapshot as locked in memory
	A locked snapshot should not be deleted until it is unlocked
	When a snapshot is locked, the corresponding logs should be not compacted as well
On snapshot unlock request
	Remove the in-memory lock
	Delete the snapshot if there is a newer snapshot
	Compact the logs
```
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

##### Edge cases due to concurrent snapshots and compaction

While taking the backup, the available snapshot has `processedPosition >= checkpointPosition`. This would mean that we cannot take a valid backup, because there is not way to retrieve a state that represents the checkpoint until the checkpoint command.

In this there are two scenarios that can happen:

1. The available snapshot is after the `checkpointPosition`. That s`napshotId.processedPosition > checkpointPosition`.
2. The available snapshot was started at a position before `checkpointPosition`, but was taken in parallel to the backup process. In this case `snapshotId.processedPosition < checkpointPosition`. But since the snapshot is taken concurrently, the actual `processedPosition` in the state is > `checkpointPosition`.

To prevent case 1, we fail the backup if `snapshotId.processedPosition >= checkpointPosition`.

In case 2, it is difficult to find the actual `processedPosition` in the snapshot without opening the database. Hence to be safe, we abort the snapshot if  `snapshotId.processedPosition <= checkpointPosition < lastWrittenPosition`. This means that we abort any snapshot that is taken in parallel to a backup. This is not ideal, but it is a simple solution to prevent inconsistencies.

##### Failure scenarios

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

##### Failure scenarios in coordinator to Zeebe broker communication

- Backup request from coordinator to a Zeebe broker is lost.
- Acknowledgment from Zeebe broker to the coordinator is lost.

In this case, the coordinator can resend the backup request. It is ok if a partition receives the request twice. Only the first request triggers the backup. The second request will be processed by the streamProcessor, but it will not trigger a backup. Instead it can just send an acknowledgment back to the coordinator indicating that the backup is already ongoing.

What happens if a backup fails?
If a backup failed, the coordinator can see the status via the monitoring query. If a backup fails, there is no use in re-sending the backup request with the same backup id. If a backup failed, then the users must retry with a new backup id.

##### Rationale for restore process
The backup should contain the snapshot and the log until the checkpoint position. However while taking the backup, raft will be writing new entries to the log concurrently. To simplify taking the backup, we include extra entries in the backup. Instead, during restore we will fix it by deleting all entries that should not be in the backup.

However, after deleting the entries, according to StreamProcessor lastCheckpointId = backupId - 1. This is not correct, as it may trigger another backup with the same backupId. To prevent this, we write a new command restore backupId. This way, StreamProcessor on processing this command can just update the state and set lastcheckpointId = backupId.

## Compatibility


## Testing


# Drawbacks
[drawbacks]: #drawbacks

- Backup logic is coupled with StreamProcessor. StreamProcessor must know how to process the newly introduced checkpoint records. The snapshotting process is also involved in the backup process to ensure that the snapshot required by the backup is consistent and not deleted. It would be good if we can make the backup as loosely coupled as possible.

- Backup can fail due to failovers. New leader will not complete a backup that was started by the old leader. Instead, it will mark the backups as failed. If there are frequent leader changes, it would be difficult to take a backup.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

#### Backup using an exporter

In the contest of backup of camunda 8 cluster, we had discussed one idea where we export all records to an external storage. This external storage will server as our backup storage. The backup will be all records from the beginning. To restore to a particular backup, Zeebe can replay records until a specific position. This is a good idea to decouple backup process from the stream processor. However, this poses new challenges like how to prune the logs from the backup storage. Not pruning the logs will lead the backup to grow unlimited. Also restoring from it can take too long depending on how far in the past we have to start replaying from. To handle this we have to also take snapshots and prune the logs from the backup, which would be either re-running a copy of Zeebe on the backup or we have some way to copy the snapshots from zeebe brokers to the backup and prune the logs. Finding the correct snapshot to include in the backup poses similar challenges in the solution proposed in this document. The challenges such as - how to identify the checkpointPosition to ensure consistency, how to ensure the required snapshot is not deleted before it is copied to the backup etc.


Pros:
1. Less coupled with StreamProcessor.
2. It uses the power of logs as a single source of truth.
3. There is a possibility to extend this idea to other components in C8.

Cons:
1. There are still a lot of challenges to resolve
    - How to ensure that the backup is consistent. For this we can apply the solution proposed in this document. That is using the checkpoint command and embedding checkpointId in remote commands. We can use this to identify the checkpointPosition until which to restore from.
    - How to prune the logs. This poses similar challenges as in the proposed solution such as how to ensure that there is a snapshot at the expected position. Ensuring that snapshot + logs is a valid backup and so on.
    - How can Zeebe replay from records in the external storage etc.
    - This would require a database to which we can export the records. While the proposed solution can be applied to a simple remote file system.
2. Due to the above challenges, I feel that this idea will be equally or more challenging than the proposed solution.
3. It is an active process. The backup is a continuous process rather than
 something controlled by the user.

# Prior art
[prior-art]: #prior-art



# Out of scope
[out-of-scope]: #out-of-scope

<!--
Call out anything which is explicitly not part of this ZEP.
-->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### What parts of the design do you expect to resolve through the ZEP process before this gets merged?

- All failure cases must be acknowledged. ZEP must describe how the failure cases are handled.
- Known edge cases are listed, and the solution must handle them.
- Algorithm/design must be correct.

### What parts of the design do you expect to resolve through the implementation of this feature before stabilization?

###### Who implements restore operation and the restore api.
I see several ways to achieve this without changing the fundamental concept.
1. Zeebe implements the restore operation. We can start Zeebe with a special flag, which executes the restore operation before the normal bootstrap. Or Zeebe can initiate a restore operation while it is still running.
2. We can have a helper application that can be run in each broker before they are started. In k8s environment this could be an Init container for example.

###### Record format and backward compatibility
We have to introduce new records and new fields in existing records. What would it look like, and how to do it in a backward compatible way is left to the implementation phase.

###### Remote storage
What types of remote storage will be available and how to configure Zeebe to use a specific storage for backup is left to implementation.

### What related issues do you consider out of scope for this ZEP that could be addressed in the future independently of the solution that comes out of this ZEP?


# Future possibilities
[future-possibilities]: #future-possibilities
