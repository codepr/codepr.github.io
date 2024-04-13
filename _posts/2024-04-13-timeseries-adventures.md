---
layout: post
title: "Time-series adventures"
description: "Small summary of my journey implementing a little time-series library"
categories: c unix system-programming database
---

Databases, how they interact with the filesystem at low-level, have always
represented a fascinating topic for me. I implemented countless in-memory
key-value stores at various level of abstraction and various stages of
development; with multiple languages, Scala, Python, Elixir, Go. Gradually
going more and more in details, it was finally time to try and write something
a bit more challenging.

A relational database was too big of a step to begin with and being the
persistence the very first big problem I could think of facing almost
immediately, I wanted something that could be implemented based on a simple but
extremely versatile and powerful structure, the log.
I wanted something that could be implemented on top of a this simple concept,
in my mind the system should've basically been a variation of a Kafka commit
log, but instead of forwarding binary chunks to connected consumers (sendfile
and how Kafka works internally is another quite interesting topic I explored
and I'd like to dig into a bit more in the future); a time-series seemed
interesting and fitting my initial thoughts.

The programming language to write it in was the next decision to make, I considered a couple of choices:

- **Elixir/Erlang** - a marvel, fantastic back-end language, the BEAM is a piece of art, I use Elixir daily at work though so I craved for something of a change my scenery
- **Go** - great design, simplicity and productivity like no others, also a bit boring too
- **Rust** - different beast from anything else, requires discipline and perseverance to retain productivity, I like it, but for it's very nature and focus on safety, I always feel kinda constrained and fighting the compiler more than the problem I'm solving (gotta put some more effort into it to make it click completely). Will certainly pick it up again for something else in the future.
- **C++** - Nope, this was a joke, didn't consider this at all

Eventually I always find myself coming back to a place of comfort with C. I
can't really say the reason, it's a powerful tool with many dangers and when I
used it for my toy projects in the past, I almost never felt enough confidence
to say "this could to be used in a prod environment". I fought with myself on
it for many years, learning Rust, flirting with Go, but I eventually gave up
and embraced my comfort place. They're great languages, I used Go for work for
a while and Rust seems a good C++ replacement most of time, provided you use it
with consistency (borrow checker and lifetimes can be hellish to deal and
re-learn if out of shape).

With C, I reckon it all boils down to it's compactness, conceptually simple and
leaving very little to imagination of what's happening under the hood. I'm
perfectly conscious how easy it is to introduce memory-related bugs, and
implementing the same dynamic array for each project can get old pretty fast;
but what matters to me, ultimately, is having fun, and C provides me with that.
Timeseries have always been a fascinating topic for me

## Design

Conceptually it's not really an efficient or particularly smart architecture,
I approached the implementation on a best-effort way, the idea being to delay
as much as possible the need of non-basic data-structures such as arrays.

The two main segments are just fixed size arrays where each position stores a
pointer to the first position of a dynamic array. This to be able to store all
the data points included in the time range that the segment covers

![Segments]({{site.url}}{{site.baseurl}}/assets/images/Roach_segment.png#content-image-1)

Each time a new record is to be inserted, first it is appended as a new entry
in the Write-Ahead-Log (WAL) and only then it is store in the in-memory segment
where it belongs. The WAL acts as a disaster recovery policy, it is solely
responsible for storing incoming records in order to be able to read them and
re-populate the segment on restarts.

![Disk_1]({{site.url}}{{site.baseurl}}/assets/images/roach_disk_1.png#content-image-1)

In short:

- two segments of 15 minutes each
    - each position represents a second, so 900 is the length of the segments
    - each second points to a dynamic array storing positions based on the microsecond portion of the timestamp of the point
- each time a new point is inserted, it is first stored on a WAL on disk in order to be able to recover in case of crash
- once the main segment is full, or a timestamp too far in the future is received (i.e. more than 15 minutes past the 1st point store)
    - the tail segment gets persisted on disk and becomes immutable, WAL is cleared
    - the head segment gets persisted on disk and becomes immutable, WAL is cleared
    - the head segment becomes the new tail segment
    - a new head in-memory segment is generated
- immutable segments on disk are paired with an index file to read records from the past

## The current state

At the current stage of development, it's still a very crude core set of
features but it seems to be working as expected, with definitely many edge
cases and assertions to solve; the heart of the DB is there, and can be built
into a dynamic library to be used on a server. The repository can be found at
[https://github.com/codepr/roach](https://github.com/codepr/roach).

### Main features

- **Fixed size records:** to keep things simple each record is represented by just a timestamp with nanoseconds precision and a double
- **In memory segments:** Data is stored in time series format, allowing efficient querying and retrieval based on timestamp, with the last slice of data in memory, composed by two segments (currently covering 15 minutes of data each)
    - The last 15 minutes of data
    - The previous 15 minutes for records out of order, totaling 30 minutes
- **Commit Log:** Persistence is achieved using a commit log at the base, ensuring durability of data on disk.
- **Write-Ahead Log (WAL):** In-memory segments are managed using a write-ahead log, providing durability and recovery in case of crashes or failures.

### What's in the road map

- Duplicate points policy
- CRC32 of records for data integrity
- Adopt an arena for memory allocations
- Memory mapped indexes, above a threshold enable binary search
- Schema definitions

### Timeseries library APIs

- `tsdb_init(1)` creates a new database
- `tsdb_close(1)` closes the database
- `ts_create(3)` creates a new Timeseries in a given database
- `ts_get(2)` retrieve an existing Timeseries from a database
- `ts_insert(3)` inserts a new point into the Timeseries
- `ts_find(3)` finds a point inside the Timeseries
- `ts_range(4)` finds a range of points in the Timeseries, returning a vector with the results
- `ts_close(1)` closes a Timeseries

#### Writing a Makefile

A simple `Makefile` to build the library as a `.so` file that can be linked to any project as an external lightweight dependency or used alone.
<hr>
<hr>

{% highlight makefile %}
CC=gcc
CFLAGS=-Wall -Wextra -Werror -Wunused -std=c11 -pedantic -ggdb -D_DEFAULT_SOURCE=200809L -Iinclude -Isrc
LDFLAGS=-L. -ltimeseries

LIB_SOURCES=src/timeseries.c src/partition.c src/wal.c src/disk_io.c src/binary.c src/logging.c src/persistent_index.c src/commit_log.c
LIB_OBJECTS=$(LIB_SOURCES:.c=.o)

libtimeseries.so: $(LIB_OBJECTS)
	$(CC) -shared -o $@ $(LIB_OBJECTS)
%.o: %.c
	$(CC) $(CFLAGS) -fPIC -c $< -o $@
clean:
	@rm -f $(LIB_OBJECTS) libtimeseries.so

{% endhighlight %}

Building the library is a simple thing now, just a single command `make`, to link it to a main, just a one liner
```bash
gcc -o my_project main.c -I/path/to/timeseries/include -L/path/to/timeseries -ltimeseries
```

`LD_LIBRARY_PATH=/path/to/timeseries.so ./my_project` to run the main, a basic example of interaction with the library
<hr>
<hr>

{% highlight c %}

#include "timeseries.h"

int main() {
    // Initialize the database
    Timeseries_DB *db = tsdb_init("testdb");
    if (!db)
        abort();
    // Create a timeseries, retention is not implemented yet
    Timeseries *ts = ts_create(db, "temperatures", 0, DP_IGNORE);
    if (!ts)
        abort();
    // Insert records into the timeseries
    ts_insert(&ts, 1710033421702081792, 25.5);
    ts_insert(&ts, 1710033422047657984, 26.0);
    // Find a record by timestamp
    Record r;
    int result = ts_find(&ts, 1710033422047657984, &r);
    if (result == 0)
        printf("Record found: timestamp=%lu, value=%.2lf\n", r.timestamp, r.value);
    else
        printf("Record not found.\n");
    // Release the timeseries
    ts_close(&ts);
    // Close the database
    tsdb_close(db);

    return 0;
}
{% endhighlight %}


## A server draft

Event based server (rely on [ev](https://github.com/codepr/ev.git) at least
initially), TCP as the main transport protocol, text-based custom protocol
inspired by RESP but simpler:

- `$` string type
- `!` error type
- `#` array type
- `:` integer type
- `;` float type
- `\r\n` delimiter

With the following encoding:

`<type><length>\r\n<payload>\r\n`

For example a simple hello string would be

`$5\r\nHello\r\n`

### Simple query language

Definition of a simple, text-based format for clients to interact with the server, allowing them to send commands and receive responses.

#### Basic outline

- **Text-Based Format:** Use a text-based format where each command and response is represented as a single line of text.
- **Commands:** Define a set of commands that clients can send to the server to perform various operations such as inserting data, querying data, and managing the database.
- **Responses:** Define the format of responses that the server sends back to clients after processing commands. Responses should provide relevant information or acknowledge the completion of the requested operation.

#### Core commands

Define the basic operations in a SQL-like query language

- **CREATE** creates a database or a timeseries

    `CREATE <database name>`

    `CREATE <timeseries name> INTO <database name> [<retention period>] [<duplication policy>]`

- **INSERT** insertion of point(s) in a timeseries

    `INSERT <timeseries name> INTO <database name> <timestamp | *> <value>, ...`

- **SELECT** query a timeseries, selection of point(s) and aggregations

    `SELECT <timeseries name> FROM <database name> AT/RANGE <start_timestamp> TO <end_timestamp> WHERE value [>|<|=|<=|>=|!=] <literal> AGGREGATE [AVG|MIN|MAX] BY <literal>`

- **DELETE** delete a timeseries or a database

    `DELETE <database name> DELETE <timeseries name> FROM <database name>`

### Flow:

1. **Client Sends Command:** Clients send commands to the server in the specified text format.

2. **Server Parses Command:** The server parses the received command and executes the corresponding operation on the timeseries database.

3. **Server Sends Response:** After processing the command, the server sends a response back to the client indicating the result of the operation or providing requested data.

4. **Client Processes Response:** Clients receive and process the response from the server, handling success or error conditions accordingly.
