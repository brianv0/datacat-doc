# Datacat

## A brief History

* Currently in use for Fermi and other experiments through common SRS
  * Been in use since 2007
  * Manages tens of millions of datasets for Fermi (23 million)
  * Around 4.7 PB of files ~(200MB/file)
  * SRS sees very similar file sizes, although only manages 1.6 million files over several different projects

### We've been refining the details of it over the last 18 months.
* Why? Because more people want to use it, and we want more people to be able to use it
* Web-First, REST api, generic, etc...
* Server implementation reimagined as a caching VFS with a database-backed datastore.
   * Database-enforced ACLs are hard, so VFS handles that
* Datastore interface doesn't limit to SQL DBMS only, but that's the only implementation so far

**"Datacat" is an ongoing parallel improvement of the Fermi Data Catalog, intended for better modularity and reuse across experiments.**

**The biggest limitation for using the rewrite in the next 3 months is probably the direct user interaction through Web User Interface, which is currently Oracle-only, but prototype implementations using the VFS exist.**

### And how we might describe "Datacat" going forward...
Datacat is a semantic meta-file system with metadata indexing built for the web. It is designed for the tracking and organization of files and their replicas across heterogeneous storage systems. We've designed it to be web-first with a REST api to facilitate ease of use among globally diverse systems and users.

Overview
========

## Primitives
Datacat defines two main file system-like primitives for interaction
*   **Dataset**, the abstract representation of a data file
*   **Containers (groups and folders)**, which are analagous to POSIX directories.

### Datasets
A Dataset consists of one or more versions of a file, each with one or more possible physical locations.
*   Metadata can be applied to a specific version
*   By applying a view, you can disambiguate to a specific version at specific location(s)
    *   (`version=1234;site=ALL`, `version=1;site=SLAC`)
*   If no version is specified, the view is interpreted to use the latest version
    *   (i.e. `site=NCSA` == `version=latest;site=NCSA`)

### Container
*   **Folders** are the simplest containers. They are completely analagous to a POSIX directory
*   **Groups** are a special container which may only contain datasets. They are meant to logically group datasets which might be treated as a macro dataset.

# Components
Conceptually, Datacat can be viewed as these main components:
*   A Datastore interface and implementation ** 
*   VFS to interact with the Datastore interface
*   RESTful APIs to interact with the VFS
*   A Web Application/WUI which uses the RESTful APIs
*   Client libraries which use the RESTful APIs
*   A CLI implemented with the Client libraries
*   A Crawler implemented with the Client libraries

**Note**: Currently we use Oracle, with unit tests using HSQLDB. SQLite, MySQL, and PostgreSQL should work fine.

## Crawler
Listens/polls for resource updates (i.e. new datasets created/replicas added to existing datasets). The Crawler is meant to automatically extract metadata out of the file which may be useful for indexing and searching on. Typically, this includes the file size, checksum, event count, user metadata.

## REST api
We believe an open, well-defined REST API is the best way to support a globally diverse project without limiting users to a specific language.
*   All supported clients are implemented using this REST API.

## Clients
These are clients we directly support.
*   Support Web Application Client/WUI (Web User Interface)
*   Support native python client and a java client
*   Support a CLI implemented with python client
*   Through WUI, will probably Javascript client for rich web interaction/node.js support (if desired)

## Searching
Datacat indexes it's metadata, allowing for fast search.

Typically, search is conducted by defining container targets to search, and a query string. Optional arguments include sort fields.

For more information on this, see [Searching](./Searching.md).

## Core (Internals and concepts)
### VFS
The core of Datacat Server is implemented as a VFS
*   The VFS layer enforces permissions
*   The VFS layer implements caching
*   The VFS layer abstracts interactions with the Datastore layer

### ACLs
Datacat implements a subset of AFS ACLs. User authentication and group checking is pluggable
*   Authentication handled by the REST web application
*   Authorization handled in the VFS by checking user's groups to container's ACL
    *  i.e `True in [(e.group in userGroups and e.permission == requestedPermission) for e in acl]`
*   It's recommended to use a web-based SSO system which has group support&nbsp;
    *  Ideally, it should support groups
*   We have a &quot;GroupManager&quot; service which can retrieve the list of groups a user is in and manage group membership

### Metadata
There are several types of metadata for any given Datacat file:

1.  Datacat metadata - dataset/container name, creation time, etc...
2.  File metadata - size in bytes, checksum, etc...
3.  User metadata - any key:value pair (where value is a string, number, or timestamp)
4.  Foreign metadata - Relational metadata not maintained by the datacatalog (i.e. not file or user)

The first three types are indexed in our current SQL implementation. Still, there's room for improvement depending on the underlying DBMS (i.e partitioning).

##### Foreign Metadata
Foreign metadata is implemented by plugins which define a pseudo-table. Typically, foreign metadata is another table in the same database as the core datacat tables.
*   That pseudo table MUST be relatable to some other form of data for a given dataset or container.
    *   But it doesn't actually have to be a table! It should just act like one. (For example, SIMBAD database?)
*   Similar to PostgreSQL Foreign Data Wrappers
*   More work still needed.

### Datastore
Datacat defines an interface to a datastore.
*   Currently one implementation exists which uses a SQL database for the store
    *   SQL database must implement temporary tables for Search
*   Other datastores implementations are possible, such as BerkeleyDB, Cassandra, or even potentially a file system (btrfs?)
*   Application-level replication could be implemented for high availability/high performance deployments
    *   Through use of Paxos, Raft, zab, etc...
*   Datastore-level replication could be used as well
   *   i.e. use a distributed database with many workers (and add interfaces for VFS cache coherency)




# Python client example (Search)

*Dump a Datacat path, resource path, vetoEvents metadata value for datasets at SLAC*

```python
#!/usr/bin/python
from datacat import Client
from datacat.config import autoconfig

client = Client(**autoconfig())                  # autoconfig discovers URL, user info

container_pattern = "/LSST/Data/Raw/*"           # Search inside child containers of Raw
query = 'run > 6200 and runQuality =~ "GO*"'     # Filter query (in SQL: run > 6200 AND runQuality LIKE 'GO%')
sort = ["run-", "events"]                        # Sort run desc, events asc (asc default). These are retrieved
show = ["vetoEvents"]                            # Retrieve vetoEvents metadata as well

datasets = client.search(container_pattern, site="SLAC", query=query, sort=sort, show=show)

print("Path\tResource\tvetoEvents")

for dataset in datasets:
  print("%s\t%s\t%s", %(dataset.path, dataset.resource, dataset.metadata['vetoEvents']))
  with f as open(dataset.resource, "r+b"):
      dat = f.read()
      # do some work with the file

```
