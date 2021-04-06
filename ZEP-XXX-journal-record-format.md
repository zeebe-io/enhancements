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
status: implemented

---

# Summary
[summary]: #summary


This document describes the format of a journal record as defined in the upcoming release Zeebe 1.0.0.

In this document we describe fields added by the raft and the journal to the record.
The fields added by the process engine is out of scope for this document.

Different fields of a journal record is encoded using [SBE](https://github.com/real-logic/simple-binary-encoding). Please see Zeebe repository to find the SBE schema.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


## Journal layout
The journal contains one or more segments. Each segment is a file that contains a sequence of records ordered by index.


TODO: JournalSegmentDescriptor - first 64 bytes of a segment


## Journal record format

Notations: X<T> indicates the field X is of type T. For example checksum<int64> is a field named checksum of type int64 (long).
X<SBE(Y)> indicates field X is an SBE (Simple Binary Encoding) message defined by the schema Y.


A journal record in a segment has the following format.

```
+-----------------------------------------------------------
| frame<byte> |  metadata<SBE(Metadata)> | data<SBE(Data)> |
+-----------------------------------------------------------
```

### Header
A `frame` is of size 1 byte. The `frame` can take following values:  
0 : End of File  (EOF)
1 : Version 1

If the value of the frame indicates EOF, then the following bytes do not contain a valid journal record.
Otherwise a valid record must exist.
Any non-zero value indicates the version of the record.

A journal record consists of two parts. The first one is `metadata` and the second one is `data`.
Note that the record itself is not an SBE message.

### Metadata

Metadata consists of metadata of the data. `checksum` represents the CRC32C checksum of the data. The `length` represents the number of bytes in data. A journal reader can use the checksum to verify if data is corrupted, before it is read.

```
+ --------------------------------
| checksum<int64> | length<int32> |
+ --------------------------------
```

### Data

The records in the journal are ordered by the `index`. The `index` of a record must be the index of the previous record + 1.
The `asqn` is an application provided sequence number. The `asqn` can be -1 if the record does not have an `asqn`. If `asqn` is not -1, then it must be greater than the previous record's asqn.
The `data` contains the record provided by the raft module.

```
+ ------------------------------------------------------
| index<uint64> | asqn<int64> | data<SBE(RaftLogEntry)> |
+ ------------------------------------------------------
```

### RaftLogEntry


A raft log entry consists of a term that indicates the term of the leader that created this record.
The `type` denotes one of the three types:  
0 = ApplicationEntry  
1 = ConfigurationEntry   
2 = InitialEntry.   
The `entry` is the SBE serialized bytes of either an ApplicationEntry or ConfigurationEntry. InitialEntry has no data. So if the entry type denotes InitialEntry, then `entry` will be empty.

```
+ --------------------------------------------------------------------------------
| term<uint64> | type<enum> | entry<SBE(ApplicationEntry or ConfigurationEntry)> |
+ --------------------------------------------------------------------------------
```

### ApplicationEntry

This is the entry provided by the application (Zeebe process engine.) The ApplicationEntry can have one or more application record which is batched in `applicationData`.
Each application record contains a position (plus other fields which is defined by the process engine. We don't describe those fields here as it is irrelevant for this document.)
The positions of these records are expected to be always increasing by one.
The `lowestAsqn` is the position of the first record in `applicationData`.
The `highestAsqn` is the position of the last record in `applicationData`.

```
+ -----------------------------------------------------------------------
| lowestAsqn<uint64> |  highestAsqn<uint64> | applicationData<uint8[]> |
+ -----------------------------------------------------------------------
```

### ConfigurationEntry

```
+ ---------------------------------
| timestamp | members<group(member)|
+ ---------------------------------
```

A member has the following fields:
type: 0 (INACTIVE) 1 (PASSIVE) 2 (PROMOTABLE) 3 (ACTIVE)
updated : uint64
memberId : string


In summary, a record on the journal looks as follows:

A record with an ApplicationEntry

```
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Header |  sbe_headers of metadata | checksum | length | sbe_headers of Data | index | asqn | length | sbe_headers of RaftRecord | term | entrytype | sbe_headers of ApplicationEntry | lowestAsqn | highestAsqn | length | applicationData |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

A record with a ConfigurationEntry
```
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Header |  sbe_headers of metadata | checksum | length | sbe_headers of Data | index | asqn | length | sbe_headers of RaftRecord | term | entrytype | sbe_headers of ConfigurationEntry | timestamp | members |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

A record with an InitialEntry
```
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Header |  sbe_headers of metadata | checksum | length | sbe_headers of Data | index | asqn | length | sbe_headers of RaftRecord | term | entrytype |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Replication schema

One approach is to replicate the serialized journal record as it is. However the raft module which sends and receives the replicated record needs to have access to raft term.
Hence, the record needs to be (partially) deserialized at the raft layer.
The serialization logic should be then shared by the raft and the journal.
Currently the transport layer uses Kryo for serialization.
So even if we replicate the journal record as it is, we are not getting much benefits from it.
We would still need to copy the record before it is send.
However, if we can make use of operating systems' feature to transfer bytes directly from the file to the network it would make sense to transfer the entire journal record as it is.
But currently we don't see any benefit in doing so.
Hence we propose to have a separate schema for replication.

A replicated record contains following fields:
1. term (long)
2. index (long)
3. asqn (asqn)
4. checksum (long)
5. serializedRaftRecord (byte[])

Here `serializedRaftRecord` is the sbe serialized bytes of RaftRecord with the following schema.
```
+ --------------------------------------------------
| term<uint64> | entry type | entry<SBE(RaftEntry)> |
+ --------------------------------------------------
```

The raft thread that receives this record uses the term and index to verify the preconditions before writing it to the journal. It should then construct a journal record using `index`, `asqn`, `checksum` and `serializedRaftRecord` which can be appended to the journal. Journal is expected to verify if the index and checksum is correct.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Alternatives considered

1. RaftRecord does not have the field entry type, instead we can determine the type of the entry from the sbe schema.
With this approach, there will be an empty InitialEntry with only sbe headers and no fields. This approach is also fine.
However, having an entry type make its explicit. It might be also useful when adding new types or changing the existing types.
2. Instead of frame, we use the checksum field to mark a record is invalid. If the checksum is 0, we assumes that it is EOF. While this works for detecting corruption, we decided to go with the frame because of its additional benefits.


### Why do we need a Frame?

SBE can handle versions. We can add or remove fields (with some limitations).
However adding a frame before the record has the following benefits:
1. Marking valid records and marking end of file.
2. Encoding the version number to the frame allow us to change the serialization format from SBE to another without breaking compatability.
3. By encoding version number to the frame, we are more flexible in changing the journal record format.

### Why did we add index and asqn to the record?
The index and asqn are concepts defined in the journal. They are part of a valid journal record.

Previously, the index was not written together with the record. Instead it was calculated.
But this is error prone.
There is no way to detect if the log is corrupted by having an entry at the wrong index.
It is safer to persist the index with the record.

It is the responsibility of the journal to keep the mappings between the index and the asqn.
Hence, it should be possible for the journal to determine the asqn without the need for reading the raft entry.

### How should versioning work in future?
SBE supports versioning.
As long as the adding or removing fields are done according to what SBE supports, it is possible to handle multiple versions.
Multiple versions should be handled in the journal writer or reader.

For any change in the layout that is not supported by SBE, we can make use of the frame by encoding the version number.
For example, it should be possible to migrate the record layout from SBE to some other encoding.

However, the frame is only added for the journal record.
It is currently not discussed how to change the encoding of raft entries.
It is possible to add or remove fields in raft entries as long as it follows SBE.

### Versioning at different layers

Serialization is independent at each layer. Journal does not care if raft entries are serialized using SBE or not.

# Out of scope
[out-of-scope]: #out-of-scope

This document does not describe the format of engine records.

# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities
