Simple Object Storage, in golang
--------------------------------

The Simple Object Storage (SOS) is a HTTP-based object-storage system
which allows files to be uploaded, and later retrieved by ID.

Files can be replicated across a number of hosts to ensure redundancy,
and despite the naive implementation it does scale to millions of files.

The code is written in [golang](http://golang.com/), which should ease deployment.

Building the code should pretty idiomatic for a golang user:

     #
     # Download the code to $GOPATH/src
     # If already present is is updated.
     #
     go get -u github.com/skx/sos/...

If you prefer to build manually:

     $ git clone https://github.com/skx/sos.git
     $ cd sos
     $ make



Simple Design
-------------

The implementation of the object-store is built upon the primitive of a "blob server".  A blob server is a dumb service which provides three simple operations:

* Store a particular chunk of binary data with a specific name.
* Given a name retrieve the chunk of binary data associated with it.
* Return a list of all known names.

These primitives are sufficient to provide a robust replicating storage system, because it is possible to easily mirror their contents, providing we assume that the IDs only ever hold a particular set of data (i.e. data is immutable).

To replicate the contents of `blob-server-a` to `blob-server-b` the algorithm is obvious:

* Get the list of known-names of the blobs stored on `blob-server-a`.
* For each name, fetch the data associated with that name.
    * Now store that data, with the same name, on `blob-server-b`.

In real world situations the replication might become more complex over time, as different blob-servers might be constrained by differing amounts of disk-space, etc.  But the core-operation is both obvious and simple to implement.

(In the future you could imagine switching to from the HTTP-based blob-server to using something else: [redis](http://redis.io/), [memcached](https://memcached.org/), or [postgresql](http://postgresql.org/) would be obvious candidates!)

Ultimately the blob-servers provide the storage for the object-store, and the upload/download service just needs to mediate between them.  There isn't fancy logic or state to maintain, beyond that local to each node, so it is possible to run multiple blob-servers and multiple API-servers if required.

The important thing is to ensure that a replication-job is launched regularly, to ensure that blob-servers __are__ replicated:

    ./bin/replicate -v


Quick Start
-----------

In an ideal deployment at least two servers would be used:

* One server would run the API-server, which allows uploads to be made, and later retrieved.
* Each of the two servers would run a blob-service, allowing a single uploaded object to be replicated upon both hosts.

We can replicate this upon a single host though, for the purposes of testing.  You'll just need to make sure you have four terminals open to run the appropriate daemons.

First of all you'll want to launch a pair of blob-servers:

    $ blob-server -store data1 -port 4001
    $ blob-server -store data2 -port 4002

> **NOTE**: The storage-paths (`./data1` and `./data2` in the example above) is where the uploaded-content will be stored.  These directories will be created if missing.

In production usage you'd generally record the names of the blob-servers in a configuration file, either `/etc/sos.conf`, or `~/.sos.conf`, however they may also be specified upon the command line.

We start the `sos-server` ensuring that it knows about the blob-servers to store content in:

    $ sos-server -blob-server http://localhost:4001,http://localhost:4002
    Launching API-server
    ..


Now you, or your code, can connect to the server and start uploading/downloading objects.  By default the following ports will be used by the `sos-server`:

|service          | port |
|---------------- | ---- |
| upload service   | 9991 |
| download service | 9992 |

Providing you've started all three daemons you can now perform a test upload with `curl`:

    $ curl -X POST --data-binary @/etc/passwd  http://localhost:9991/upload
    {"id":"cd5bd649c4dc46b0bbdf8c94ee53c1198780e430","size":2306,"status":"OK"}

If all goes well you'll receive a JSON-response as shown, and you can use the ID which is returned to retrieve your download:

    $ curl http://localhost:9992/fetch/cd5bd649c4dc46b0bbdf8c94ee53c1198780e430
    ..
    $

> **NOTE**: The download service runs on a different port.  This is so that you can make policy decisions about uploads/downloads via your local firewall.

At the point you run the upload the contents will only be present on one of the blob-servers, chosen at random, to ensure that it is mirrored you'll want to replicate the contents:

    $ ./bin/replicate --verbose

The default is to replicate all files into two servers, if you were running three blob-servers you could ensure that each one has all the files:

    $ ./bin/replicate --verbose --min-copies=3




Production Usage
----------------

* The API service must be visible to clients, to allow downloads to be made.

* It is assumed you might wish to restrict uploads to particular clients, rather than allow the world to make uploads.  The simplest way of doing this is to use a local firewall.

* The blob-servers should be reachable by the hosts running the API-service, but they do not need to be publicly visible, these should be firewalled.

* None of the servers need to be launched as root, because they don't bind to privileged ports, or require special access.
    * **NOTE**: [issue #6](https://github.com/skx/sos/issues/6) improved the security of the `blob-server` by invoking `chroot()`.  However `chroot()` will fail if the server is not launched as root, which is harmless.



Future Changes?
---------------

There are two specific changes which would be useful to see in the future:

* Marking particular blob-servers as preferred, or read-only.
     * If you have 10 servers, 8 of which are full, then it becomes useful to know that explicitly, rather than learning at runtime when many operations have to be retried, repeated, or canceled.
* Caching the association between object-IDs and the blob-server(s) upon which it is stored.
     * This would become more useful as the number of the blob-servers rises.

It would be possible to switch to using _chunked_ storage, for example breaking up each file that is uploaded into 128Mb sections and treating them as distinct.  The reason that is not done at the moment is because it relies upon state:

* The public server needs to be able to know that the file with ID "NNNNABCDEF1234" is comprised of chunks "11111", "222222", "AAAAAA", "BBBBBB", & etc.
* That data must be always kept up to date and accessible.

At the moment the API-server is stateless.  You could even run 20 of them, behind a load-balancer, with no concerns about locking or sharing!  Adding state spoils that, and the complexity has not yet been judged worthwhile.


Questions?
----------

Questions/Changes are most welcome; just report an issue.

Steve
--
