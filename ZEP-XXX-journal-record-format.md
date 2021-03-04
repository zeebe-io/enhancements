---
title: Journal record format
authors:
  - Deepthi Akkoorath
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: yyyy-mm-dd
last-updated: yyyy-mm-dd
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced

---

# Summary
[summary]: #summary


This document describes the layout of a journal record. A journal record consists of several layers. The outermost layer consists of fields added and required by the journal to read a record and verify if it is a valid one. The second layer belongs to Raft which adds the common information required by the raft protocol. The third layer consists for different types of data that can be written by the Raft leader. This documents stops at the third layer. The inner layers are defined by the zeebe process engine.


# Motivation
[motivation]: #motivation

<!--
- [ ] Why are we doing this?
- [ ] What problem are we solving?
- [ ] What is the expected outcome?
-->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The journal contains one or more segments. Each segment is a file that contains a sequence of records ordered by index.


TODO: JournalSegmentDescriptor - first 64 bytes of a segment


Notations: X<T> indicates the field X is of type T. For example checksum<int64> is a field named checksum of type int64 (long).
X<SBE(Y)> indicates field X is an SBE (Simple Binary Encoding) message defined by the schema Y.


A record in a segment has the following schema.

```
+-----------------------------------------------------------
| Header<byte> |  metadata<SBE(Metadata)> | data<SBE(Data)> |
+-----------------------------------------------------------
```


TBD - Size of Header - 1 byte or 4 bytes??  
Each record starts with a `Header`. The Header can take following values:  
TBD  
0000 = EOF  
0001 = Version 1 ??

If the value of the Header indicates EOF, then the following bytes does not contain a valid JournalRecord. Otherwise a valid record must exists. (TBD - other values may indicate versions.)

A JournalRecord consists of two SBE messages. The first one is Metadata and the second one is Data.
Note that the record itself is not an SBE message.

### Metadata

Metadata consists of metadata of the data. `checksum` represents the CRC32 checksum of the data. The `length` represents the number of bytes in data. A reader reading the journal can use the checksum to verify if data is corrupted, before it is read.

```
+ --------------------------------
| checksum<int64> | length<int32> |
+ --------------------------------
```

### Data

The records in the journal are ordered by the `index`. The `index` of a record must be the index of previous record + 1.
The `asqn` is an application provided sequence number. The `asqn` must be always increasing. The `data` contains the record provided by the raft protocol.

```
+ -----------------------------------------------------
| index<uint64> | asqn<uint64> | data<SBE(RaftRecord)> |
+ -----------------------------------------------------
```

### RaftRecord

TBD

A RaftRecord consists of a term that indicates the term of the leader that created this record. The `entry` can be the SBE serialized bytes of either an ApplicationEntry or ConfigurationEntry or InitialEntry.

```
+ --------------------------------------
| term<uint64> |  entry<SBE(RaftEntry)> |
+ --------------------------------------
```

RaftEntry = ApplicationEntry OR ConfigurationEntry OR InitialEntry

### ApplicationEntry

This is the entry provided by the application (Zeebe process engine.) The ApplicationEntry can have one or more application record which is batched in `applicationRecords`. Each application record contains a position (plus other fields which is defined by the process engine. We don't describe those fields here as it is irrelevant for this document.) The positions of these records are expected to be always increasing by one. The `lowestAsqn` is the position of the first record in `applicationRecords`. The `highestAsqn` is the position of the last record in `applicationRecords`.

```
+ ----------------------------------------------------------------------
| lowestAsqn<uint64> |  highestAsqn<uint64> | applicationRecords<bytes> |
+ ----------------------------------------------------------------------
```

### ConfigurationEntry TBD

```
+ ---------------
| Members... |   |
+ ---------------
```

### InitialEntry TBD

```
+ -------------
| leaderId ??? |
+ -------------
```

## Compatibility


## Testing


# Drawbacks
[drawbacks]: #drawbacks

<!--
Why should we *not* do this?
-->

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO:
- Why do we need a Header? Why is Header + JournalRecord not an SBE message?
- Why do we split JournalRecord into two messages - metadata and data?
- Why we added index and asqn to the record?
- How should versioning work in future?
    - Can we add new fields - at each layer?
    - Can we change the semantics - eg:- change checksum algo from CRC32 to something else?
    - Can we change the serialization format?

- Each layer is serialized independently. Journal does not care if RaftRecord is serialized using SBE or another serializer.


# Prior art
[prior-art]: #prior-art


# Out of scope
[out-of-scope]: #out-of-scope

<!--
Call out anything which is explicitly not part of this ZEP.
-->

# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities
