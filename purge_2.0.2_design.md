Tombstone Purging Design Specification
=================================================

# INTRODUCTION

## Purpose of this Document

This document describes the software design and implementation to support the purging of deletion tombstones in Couchbase server 2.0.2 and the component changes necessary, and implications of the design on correctness and performance.


## Motivation

When documents in Couchbase are deleted, there is still a recored of the document being in the deleted state. This is because XDCR, indexing, incremental backup, and any components relying on receiving asynchronous incremental changes need to know when to remove records from their state. If the deleted documents are completely removed from storage without a trace, it's very possible for the other modules to never know when to remove the same documents and be out of sync with storage forever.

The problem with storing the tombstones is they take up storage file index space and can slow down disk operations for the server. We want a system that gives of the ability to remove old deletion records that have a high probability of being seen all other interested modules, and can provide the ability for modules to know when they are potentially out of sync and take corrective measures.

## Scope of this Document

This document will describe the high level data structures, and components necessary to implement.

# SYSTEM ARCHITECTURAL DESIGN

## Overview of Modules and Components

### Couchbase Server

This is a single node of a Couchbase installation. It contains an implementation of all components.

### Couchstore

This is the storage engine component of Couchbase Server. For each partition, there is a storage file and it arranges a record of each document by update sequence ID, a  monotonically increasing value, in a btree. Each document mutation is a assigned a new sequence number, for existing documents in the store, the old sequence value is deleted. This allows servers to quickly scan recent documents and deletions persisted since a particular sequence number, or from zero if wants all changes.

It also has a btree arranging the same documents by key/id, which also contains the same meta information for a document.

A new field, High-Purge-Sequence, will be added to the storage file header that indicates the highest sequence number of any tombstone ever purged. There is already an unused Purge Sequence field we can use, making file format backwards compatibility a non-issue.

### Couchstore Compactor

This is the module and separate executable that is responsible for copy compaction of a Couchstore partition. It will also be responsible for "purging" tombstone records as compaction happens, and updating tombstones from older versions of Couchbase with a deletion time.

### ns_server

This is the module responsible for invoking the Couchstore compactor. It will also be responsible for keeping the purge "confidence interval" configuration setting, which a integer that represents the minimal time between when a tombstone is created and can be purged.

When it invokes the compactor, it will pass it a --purge-tombstones-before=%unixtime% command line parameter, that the compactor will then use to drop tombstones with a deletion time before in the input value.

### Tombstone

When a document is deleted, a deletion record, or "tombstone" is stored in the 2 Couchstore btrees to signify that the document is in the deleted state. These are the records we want to purge after enough time, a "confidence interval", has passed. The tombstone record is in the identical format of the live records, except that a deleted bit is set to 1 and the body pointer has a zero/null value.

The records in the btree have a expiration time field, expressed in 32bit unix time. This field previously was unused in deletion tombstones. For deleted documents, we will record the system time when the document was deleted, so when the confidence interval has passed the tombstones can be purged.

If the compactor runs and sees a tombstone with no deletion time (the time is zero), it will populate the tombstone deletion time with the current time.

### Incremental MapReduce View Indexer

This is the component responsible for scanning storage files and brings secondary MapReduce indexes up to date with storage files. After it scans and indexes a storage file, it records the high sequence number in the index header, so that it can pick up where it left off on the next indexing scan.

Now, before beginning an incremental index, it will compare it's high index sequence number with the storage file High-Purge-Sequence, and if the storage file has a higher sequence, it means it's possible there are deletions the indexer isn't aware of. When this occurs, the indexer will be forced to destroy indexes and start over from scratch, performing a full index of all storage files.


# Correctness and Performance Considerations

## Incremental MapReduce View Indexer

Implemented correctly, it should be impossible for the view index to unknowingly miss deletions and therefore to ever get out of sync with storage. However, when the view indexer notices it's possibly out of sync, it will have to destroy the index and restart from scratch, introducing an unbounded pause for view queries, using lots of CPU and disk IO and affecting the file system cache, possibly slowing the whole server to an unknown degree.

## XDCR

With this scheme if a XDCR task is delayed longer than the confidence interval, it's possible XDCR will miss deletions and the remote cluster will forever preserve documents that should have been deleted, and the clusters will forever be out of sync.

With a relatively high purge confidence interval (on the order days), enough bandwidth and properly sized cluster and machines, this should be an exceedingly rare occurrence. 

Both the detection and repair of this is currently out of scope for this document.

However, if it does happen, the clusters can be brought back into sync by deleting and recreating the XDCR tasks on both ends. With unidirectional XDCR, it should be made bidirectional for long enough for the remote cluster to check and sync all data with the primary cluster. However this has the effect of "resurrecting" deleted documents on all clusters, which might be undesirable.

With unidirectional XDCR, if resurrection is undesirable, it's possible to write a client that scans all changes in the secondary cluster and to check if it's available on the primary cluster. If not, delete the document on the secondary cluster. Also, creating a new secondary cluster and an XDCR task to initialize it is a reliable solution.

With bidirectional XDCR, if resurrection is undesirable, there is no known algorithm to reliably remove documents from clusters that missed the deletion record.

## Elastic Search Integration

For reasons similar to XDCR, it's possible -- but should be exceedingly rare -- for the elastic search cluster to miss purged deletions. Repairing this might be possible if detected, but the detection and repairing of this is currently out of scope for this document.

## UPR clients (2.1 or later)

In the future, UPR clients will be able to register persistent UPR state records that can be used to more aggressively purge tombstones, and prevent the purging of any tombstones not seen by a registered UPR client. The facilities and implementation have not yet been specified.
