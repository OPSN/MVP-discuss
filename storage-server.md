BUGS: this file is not yet in markdown syntax. Specific errors: @oresmus is not a link.

==

This will contain my (@oresmus) current thoughts about one simple way to define and implement an OPSN server,
or more specifically a "pool storage server" or just "OPSN storage server",
which does nothing but store stuff for one user, with read permissions to be granted as a unit (all or nothing).

Basic idea: basically just a filesystem, which the owner can write (with an efficient append operation) and readers can only read.

(The owner can effectively control everything about it, roughly like for a filesystem they own.)

==

OWNER permissions:

Needed:
- PUT (write new contents for a file, atomically)
- DELETE (remove a file)
- GET (retrieve file contents)
- efficient APPEND (not a standard part of HTTP, so in an HTTP API we'd use a POST operation)
  - this should be atomic, which is difficult to make efficient;
    it might suffice instead to allow specifying the expected prior file length, and failing if it's wrong
- get a directory listing (needed to recover from some write errors, if nothing else)

Optional:
- RENAME (ie move) [I'll put this in if it's simple enough, since it lets read convention be much simpler]
- make symbolic links [not worth the trouble]
- directory listing includes file lengths [good]
- directory listing can include file hashes [not worth the trouble for now]
- keep logs of all operations [we'll do this if it's easy, for debugging if nothing else]

==

READER permissions:
- read any file given its name
  (standard file extensions control its mimetype, as in a typical web server)
- but nothing else (in particular, no directory listings)

Optional reader permissions:
- directory listing for a finite list of specified full pathnames [purely an optimization, so not worth the trouble for now]

==

Nature of access:
- OWNER needs an HTTP API
- READER could either use an HTTP API (ideal for some purposes), or just GET from a web address (ideal for other purposes), or both
  - note that GET from web address doesn't imply the presence of an ordinarily-usefully-browseable website,
    though writers could produce one there if desired (the simple server would not produce it, though)

Current thoughts about implementation: one 'grain' in Sandstorm.io, using one HTTP API for both interfaces (OWNER and READER)

==

Everything else about OPSN is done by conventions followed by writers and readers.
For example:
- known filenames contain basic info, including OPSN directory format.
- unguessable filenames are possible.
  - use them for writing to specific readers (or just to owner), unseen by other readers.
- blob filenames include their hash, perhaps their length, and perhaps a file extension (not hashed, just hint about type).
- Various filenames can be converted to related filenames with appendable or mutable metainfo or log info, etc.
- Appendable files will be readable as a sequence of records.
  We'll use them for:
  - sets of small hashed blobs, as the writer classifies them.
  - misc mutable-sets of exprs, as the writer desires (eg current state of various weights, perhaps never hashed).

==

Possible specific convention:

README.txt - for people
index.html - for anyone browsing there by accident, tells them what you want the public to know about the right way to read it
README-MACHINE.txt - for programs -- gives machine-readable format description (at least format version) of entire directory
MANIFEST? INDEX? - for reader programs - list of pathnames readers need to know but would not guess (more or less)

blob/<hash> - a long blob (maybe include length in name? plus optional file extension, which reader needs to know in advance)
- we might need a way of splitting into subdirs if the main dir gets too long;
  - it'd be nice if the underlying filesystem could do this (remain efficient for huge directories)
  - if it doesn't, maybe the storage server itself should?

news/<date or uuid or nickname> - an appendable "log" of changes to an OPSN graph, with conventions for which ones to read

private/<unguessable>/ - subdirectory for blob/ and news/ in private channels

topic/<name>/ - subdirectory -- not needed except to optimize for readers who don't care about a large subtopic

And more, not yet designed here (unless part of news?), for very small very mutable data, like weights and other metainfo

==

Advantages of that scheme:
- the server is simpler
  - thus easier to write, debug, and port to other languages or hosts
  - it needs no database, just the filesystem
- and reusable for many purposes (unrelated to OPSN),
- and less often modified during development (perhaps never, once it works).

- the server could be replaced (with small changes to interface, and possible need for reader proxy to enforce limits)
  by any fully controlled filesystem under a web server,
  or any other shared filesystem,
  so porting OPSN clients to totally different kinds of servers is also easier.

Disadvantages:
- there are useful things the server can't do, like
  - validate hashes
  - recover gracefully from interrupted write-connections, or even notice errors caused by those
  - OPSN-specific logging


