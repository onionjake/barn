BARN - Blob ARchive to Network
==============================

(or perhaps Backups Are Really Neat)

Barn is a file archival tool.  It is meant for large collections of big binary
files, where the files themselves rarely change.  It behaves like a simplified
version control system.  It was built to handle a family's collection of about
a terabyte of digital photographs and scanned slides.

A barn archive is a content-addressable store of files, indexed by SHA256
checksum.  All history is preserved by default. However, files can be removed
permanently from history with a special erase commit.

An editable copy of the archive data can be made, hereafter called a checkout.
If desired, only a subdirectory can be checked out.  It is also possible to
only check out some of the files in a directory, and placeholder symlinks will
be left in the place of files not present.

An archive can be spread across several destination disks or multiple servers.
That means it scales up to file collections far larger than any individual
computer's hard drive can store.  Each destination is called a silo.

Due to the nature of a content-addressable store, whole files are deduplicated
automatically.  Multiple trees can share the same silos, offering more
opportunity for deduplication.

Cryptographic hashes are used to verify the integrity of every file and the
change history.  Data should be checked regularly to ensure that it hasn't been
corrupted on disk.

The software doesn't need to be installed on the remote hosts.  Only ssh access
is necessary, and other backends are possible.  The repository can be GPG
encrypted.


Status
------

This project is still in its design stages.  This document contains the design.
After the design is satisfactory, a 


Sample Usage
------------

Alice likes to make videos of nature.  She has nearly 5 TB of clips spread
across her computer and disks at home.  She wants to consolidate them into one
single place, even though she doesn't have enough space free on any one
computer for the whole set.  Unfortunately, she just spent all her disposable
income on a 4K camera and can't buy a new RAID for storage.

She has two 3TB drives that she wants to leave at work for offsite storage.
They are connected to her desktop and accessible over SSH.  She has a bunch of
old external drives that she has accumulated through the years that she'd like
to use at home, with capacities averaging 500GB.


### Creating Initial Checkout

First Alice will get all the files into the archive.

```
alice@desktop:~$ cd Video/
alice@desktop:~/Video$ ls
Birds  Flowers  Rivers  Animals  Misc  favorite.mov
alice@desktop:~/Video$ barn init
alice@desktop:~/Video$ barn status
new: Birds/
new: Flowers/
new: Rivers/
new: Animals/
new: Misc/
new: favorite.mov
Total of 1,523 new files (300GB)
ERROR: No silos attached, try "silo add [server]:[path]"
alice@desktop:~/Video$ barn silo add ~/vid-barn
Directory ~/vid-barn does not exist, create with --create
alice@desktop:/Video$ barn silo add ~/vid-barn --create
Created new silo
alice@desktop:~/Pictures$ barn commit
Committing 1,523 files (300 GB) to ~/vid-barn
Error: Not enough free space available, need 157 more GB.
alice@desktop:~/Pictures$ barn commit --move
Moving 1,523 files (300 GB) to ~/vid-barn
Progress 100%, 0 minutes remaining
300 GB moved in 5 minutes
```


### Accessing files from another computer

```
alice@laptop:~$ barn checkout desktop:vid-barn Video
Barn contains 1,523 files, 300 GB.  Download with "silo get"
alice@laptop:~$ cd Video
alice@laptop:~/Video$ ls
Birds  Flowers  Rivers  Animals  Misc  favorite.mp4
# Right now all files are just broken symlinks
alice@laptop:~/Video$ ls -l favorite.mp4
lrwxrwxrwx 1 alice alice 204 Feb 10 10:56 favorite.mp4 -> /:barn-absent-file:/baa60f1e0d508288a23a00adfecc05cf321799108ec955497bc8bad5c6b2ef3e
alice@laptop:~/Video$ barn get favorite.mp4
Copying 1 file from desktop:vid-barn, 215MB
Progress 100%, 0 minutes remaining
alice@laptop:~/Video$ ls -l favorite.mp4
-rw-rw-r-- 1 alice alice 225292900 Sep  8  2013 favorite.mp4
```


### Adding more files to the archive

```
alice@laptop:~/Video$ mv ~/MoreVideos/FallingLeaves .
alice@laptop:~/Video$ barn status
new: FallingLeaves/
Total of 350 new files (70GB)
alice@laptop:~/Video$ barn commit
Committing 350 new files (70GB) to desktop:vid-barn
Progress 100%, 0 minutes remaining
70GB committed in 20 minutes
alice@laptop:~/Video$ barn drop FallingLeaves
About to drop 350 files (70GB)
Verifying 350 files on desktop:vid-barn
Progress 100%, 0 minutes remaining
Verified 70 GB in 11 minutes (x MB/s)
```

### Adding another silo for redundancy

```
alice@laptop:~/Video$ barn numcopies 2
Setting numcopies to 2 for 1873 files (370GB)
WARNING: 1873 files (370GB) do not have enough copies
Since there is only one silo, more will be needed to meet copy quota
alice@laptop:~/Video$ barn silo add officeA office:/media/external-A/barn --create
Created new silo
alice@laptop:~/Video$ barn copy origin officeA
Copying 1873 files (370GB) from desktop:vid-barn to office:/media/external-A/barn
Progress 100%, 0 minutes remaining
Copied 370GB in 24 hours (x MB/s)
```


### Checking for Corruption

```
alice@laptop:~/Video$ barn check
Verifying contents of 1 file (215 MB)
Progress 100%, 0 minutes remaining
No corruption found
alice@laptop:~/Video$ barn check officeA --timeout=1h
Verifying contents of 1873 files (370GB)
Progress 15%, 11 hours remaining
Check aborted due to timeout, progress saved

Making changes
alice@laptop:~/Video$ rm favorite.mp4
alice@laptop:~/Video$ barn status
deleted: favorite.mp4
alice@laptop:~/Video$ barn commit
Committed 1 deleted file
```


Silo File Layout
----------------
The archive format is designed to be as simple as possible.  Compared to other
systems, less effort has been spent on minimizing disk space used, since the
overhead is going to be insignificant compared with the large files that are
being stored.  A silo has two subdirectories, "files" and "metadata".

"files" contains the blobs.  Each file is stored with its filename being the
sha256 of the file contents.  In order to speed up remote directory listings
(which are only needed for garbage collection), these are divided into
subdirectories by the first character of the hash, and then further into more
subdirectories by the next character.  A file named

`e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`

would be stored in

`e/e3/e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`.

The files are stored without the write bit set, as they only ever need to be
written once.

"metadata" contains commits to the system.  Imagine that there is a key-value
store with the path to the file as the key and the value being the tuple of
(sha256, mtime, unix file mode).  Each commit lists the changes to the KV
store, which are either "SET key, value" or "DELETE key".  The commit is
serialized to JSON, and the sha256 of that JSON string is computed.  The
results are then compressed with gzip and written to "metadata/$sha256.gz".

FIXME: Is there a format that is simple like JSON but better for grepping?
YAML fits the bill but is relatively complicated.  Maybe pretty-printed JSON
would be good enough.  A newline-separated file where complicated strings are
JSON encoded would be even better.

FIXME: We also need to support RENAME for directory trees, since they could
represent a substantial amount of SET and DELETE operations otherwise.

FIXME: Finally, we need to decide how the DESTROY operation would be stored.

Each commit references zero or more parent commits.  The commit with no parents
is the root commit.  Most commits will have only one parent.  Typical use
expects divergence to be rare.  When a divergence is detected, a merge commit
with more than one parent commit is created automatically.

FIXME: Is it even necessary to have multiple parents?  Since we're going to
merge automatically, we can always rebase before committing, giving a
completely linear history.


Since the "metadata" directory will need to be listed frequently, after it
contains 512 entries, entries can be packed together.  Packing consists of
taking a set of entries, sorting them by their sha256 hash, uncompressing them,
concatenating them, and taking the sha256 of that concatenation, the
concatenation is then discarded.  A directory is made called
"metadata/$sha256.pack", and the entire set is moved into that directory.

FIXME: In order to let multiple separate trees share the same CAS, we'd want to
subdivide the "metadata" directory.


FIXME: How do we have a consistent spreading of files to a distributed archive?
Should those rules be defined in a config file inside of the archive itself?
That'd save the need for the "peers" file for a evenly distributed database.
(It wouldn't allow for an ad-hoc distribution though, the "pile of USB hard
drives" approach that some people would use.)


Conflicts
---------

If any conflicting changes to a file are detected (which is expected to be very
rare in typical usage scenarios), the user will be notified and multiple copies
of the file will be created, with the filename munged to represent the state of
the conflict.  The user can resolve the conflict by deleting the copy of the
file not desired and committing again.


Read-only server views
----------------------

On a computer with a silo, a read-only view can be created of the contents of
the silo.  A separate directory tree is created where each file is a symlink
(or hardlink) to the corresponding entry in the archive by SHA256.  Care must
be taken to ensure that these files won't be written to, such as by using a
read-only bind mount point.  (Note that some operations are safe, such as
deleting files, adding new files, and replacing files wholesale.  It is only
partially rewrites that are damaging.


Checkouts
---------

Checkouts have a .barn directory at their root which track upstream sources for
the checkout.  Each checkout should have more than one main source, since for
archival it is common to want two or more copies.

The .barn directory keeps a copy of the metadata and configuration.

Any cache files will live in ~/.cache by default instead of in .barn, because
barn checkouts should be usable on weaker filesystems, such as on sshfs, NFS,
samba, etc.

If a file is dropped locally, a symlink pointing to `/:barn:/$sha256` will take
its place.  These placeholders mean that commands like rm and cp can still be
used on these files, even if they aren't present.


Out of band checkout management
-------------------------------

Instead of having a .barn directory inside of the checkout tree, it's also
possible to have it stored in ~/.config/barn (FIXME: Whatever the XDG compliant
directory is).  This allows the user to manage files on a shared drive or cloud
service where the .barn directory's presence wouldn't be desireable.

When the barn software is invoked, it would manage this by first checking in
~/.config/barn for the current directory, and then search the ancestor
directories for a .barn directory (until it crosses a filesystem boundary).


Encryption
----------

Archives can be encrypted using gpg.  The layout is the same except each file
is encrypted with gpg before uploading.  The sha256 filenames would be XOR'd
with a fixed key in order to obfuscate file contents.  The filename obfuscation
key would need to be stored on the remote archive, also gpg encrypted.  We'd
still leak other data like file sizes, deemed an acceptable amount of leakage.


Safety
------

Files are checked for consistency whenever they're moved around.  This means
that we'll be CPU bound for many operations, which means multithreading would
be important.


Command Line Usage
------------------


`barn checkout server:/path` - gets a copy of the history, doesn't download any
files

`barn checkout server:/path/sub/directory` - makes a partial checkout, gets
history

`barn init`

`barn get [path]` - fetches all files in current directory or $path.  This can
optionally pull from another checkout, since we can verify that the file isn't
corrupted.

`barn drop [path]` - removes local copies of files (note that a checkout's
files don't contribute to numcopies because they aren't guaranteed to be safe)

`barn status [path]` - show status of current directory or $path, using mtime
and size to detect changes.  While git shows the status of the entire
repository, for performance reasons the user might not want to wait for the
system to check the entire tree.

`barn commit [path]` - commit changes and write them to upstream servers, with
an optional message

`barn update [path]` - fetch remote commits and merge them

`barn update --with-contents` - does a barn update and a barn get on any new
files (saves a transversal of the directory tree for large repositories)

`barn destroy` - delete a file and erase it from history; can also take a sha256

`barn log` - show history

`barn check` - verify files haven't changed on disk even though mtime and size
haven't; this automatically keeps track of its last position so that it will
eventually visit all files even if only ran for an hour a day

`barn incoming` - dry run of barn update

`barn find` - say which silos contain a file


Additional commands that can be run against a silo (either locally or remotely):

`barn update --with-history`

`barn get --with-history`


FIXME: we can commit without pushing the file data BUT that means that we'll
need to make temporary local copies of the files in question until push time so
that they can't be changed and those changes lost.


Backends
--------

Originally we'd ship with only two backends, file and ssh.  Since a major speed
limitation is hashing files, multiple concurrent instances of the backend will
be spawned.  In addition, barn will communicate with multiple copies of the
archive at once.  Finally, the other major speed limitation is the round-trip
time on the network, so remote backends are expected to be able to pipeline
requests.


Running on Android
------------------

A GUI with a button to commit and another button to drop files older than 30
days would be enough to meet common needs on android.  Maybe a browser for all
files that would let you mark some to "get" for a movie collection.

An HTTP backend could be made to eliminate the ssh dependency so that stock
jruby, with no gems, would be sufficient, or net-ssh could be used.


Further Work
------------

In the future a simple FUSE filesystem could mount a local or remote archive,
with local caching of files.  This could even be written to, with local changes
waiting until the user decides to commit.  As soon as a file is closed it could
be uploaded to the CAS, with the commits happening only occasionally.  In
between commits, all changes could be logged to a 'partial commits' file for
that client, which could be used to reconstruct a commit if discovered.


Similar Projects
----------------

"git-annex" was the inspiration for this project.  git-annex is complex to use,
as it builds on top of git, so a non-programmer would have to learn both git
and git-annex.  It depends on using symlinks for each file, which is ugly and
slow to traverse.  It has "direct" mode, which doesn't use symlinks, but then
partial checkouts aren't possible.

"git-annex" has a gui called git-annex assistant which simplifies usage at the
cost of changes being synced automatically.

"bup", "zbackup", "obnam", and "attic" are backup systems that detect duplicate
data instead of only duplicate files, which makes it better for databases and
log files.  However, none of these allow a repository to be copied safely to
another location.  (rsync is very efficient at making the copy but it doesn't
have any way of verifying that the files haven't corrupted on disk.)  Also,
they don't allow for spreading across multiple disks.  (Note that RAID0 or LVM
allow many disks to act as a single logical disk, but in that scenario losing a
single disk means that the entire logical disks contents are lost, whereas
having a silo on each disk individually means that only the files stored on
that silo are lost.

FIXME: Is there opportunity to use bup or one of the others as a backing store
for barn?  It would certainly be possible, but the main advantage
(deduplicating parts of the file) makes the most sense for file types that
wouldn't typically be directly edited by a user.
