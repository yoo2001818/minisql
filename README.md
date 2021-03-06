# minisql
An experimental mini SQL database for Node.js

minisql is an experimental SQL database written in TypeScript for Node.js.
It is so experimental and only for learning, it'll only support basic CRUD,
without ACID, or index scanning support. (Meaning that it'll only always
perform full scan! .... for now.)

## Components
minisql is separated to a number of components.

- Storage engine
- Query executor
- Query planner
- Frontend / Gateway
- Client

Unlike other databases, it doesn't have any task queueing / controlling
facility since it doesn't support multithreading, and ACID support is not
in consideration - it's too complicated to implement.

### Storage Engine
Storage engine is responsible for storing data onto disk, and fetching them.

It'll be constructed using this hierarchy of components:

- Raw disk I/O driver
- Low-level block management system (A filesystem)
- Table storage / B+Tree storage
- Metadata storage
- Storage optimizer (defrag, vacuum, etc)

### Query Executor
Query executor actually performs the query by connecting to the storage engine.
It'd perform low-level operations instead of SQL, something like Pig Latin.

### Query Planner
Query planner plans the query from SQL and converts to intermediate execution
code.

### Frontend / Gateway
Frontend is built using a simple stdin/stdout, and it'd be parsed and sent to
the query planner. Or it can use WebSocket or TCP server for remote connection.

### Client
Actual user interacts from here. Other processes should be able to interact with
DB, so we need a client driver for each platform.

## Storage Engine
Storage engine resembles a simple file system, then a table / index layout
is built upon it.

### Filesystem
Since all the database should be stored on a single file, it is required to
build in-house filesystem - but it can be really simple.

It splits each block to 4KiB, then puts file inode and file data in any
order. Luckily, since this is limited to internal use for DB, we don't need to
store any metadata like filename, date, permission, etc. We can just use
block ID for the filename.

Since the inode will only have file size and pointers for data, inode
shouldn't be too large. So, each 'index' block can be splited to multiple
inodes.

Each inode can be small as 128 bytes - so each block can store 32 inodes.
Since the blocks are not organized in any way, there must be an pointer for the
next inode block. Also, for future use for GC or something, storing the
block information as an bitset is required too.

File and 'inode' both are synonyms - Each file can reference inode
to create a B-Tree.

1th file stores a bitset for block information - preferably a byte for each
block to mark the left amount of free nodes for each inode block, too.

0th inode in the inode block stores next inode block, to point other
inode information.

For each inode, we should employ same technique that ext4 is using -
a single / double / triple pointer. That'll allow for approximately
(12 + 12 + 12*12 + 12*12*12)*4096 bytes.

Then, we have everything we need for storing raw bytes in a organized chaos!
We can start building a database upon it.

### Table / B+Tree storage
After building a filesystem, we need to build a table and B+Tree storage
upon it.

#### Table storage
Table storage is an incrementing file with offset pointers and big data chunk.

Basically, all data accumulates on the data chunk file, and offset pointers are
written in the offset file.

Offset file has a deterministic positioning - offset file can be located using
(12 * row ID).

This way, the data will only accumulate - it'll never shrink! so we need
vacuuming - trimming deleted data out. Of course, we need to mark the deleted
data - so we need to manage deleted fragments list too.

We also need a metadata file - containing auto increment key, row ID, length,
to-be-deleted fragment list.

#### B+Tree storage
B+Tree storage is an incrementing file with fixed B+Tree node blocks.

Since B+Tree node's size is completely fixed, that means string's max size must
be specified as well - otherwise it can't be filled up. We're setting node's
hard limit to 512 bytes.

Each node's size is user-customizable, but the default would be around 64kB.

B+Tree node can appear in random position, and root node can move, so we need to
manage the position / metadata / block bitmap too.

We can put that kind of data in the first sector of the index, or another
file. Putting them in another file will be much easier.