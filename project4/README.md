# Distributed File System

In this assignment, you will be developing a working distributed file
server. We provide you with only the `gunrock_web` HTTP framework; you
have to build the rest.

The goals of this project are:
- To learn the basics of on-disk structures for file systems
- To learn about file system internals
- To learn about distributed storage systems

This project consists of three main parts: [reading on-disk storage
using command line utilities](#read-only-file-system-utilities), [modifying
on-disk storage](#readwrite-file-system-utilities), and using the local file
system to impelemnt a [distributed file
system](#distributed-file-system-1). We recommend implementing your
server in this order where you ensure that you have a solid foundation
before moving on to the next part.

Fundamentally, this project is all about persistent storage. However, the
interfaces you have to work with are all different. The disk operates only
on disk blocks, the local file system organizes disk blocks into a heirarchical
file system, and the API layer uses the local file system to provide an object
abstraction. Each of these components have their own unique physical reality
to work with and illusion they provide to software running above.

## Distributed File System

**The distributed file system component is optional this quarter.** You
should focus on implementing and testing the local file system, described
below.

### Background
The main idea behind a distribute file system is that you can have
multiple clients access the same file system at the same time. One
popular distributed file system, which serves as inspiration for this
project, is Amazon's S3 storage system. S3 is used widely and provides
clear semantics with a simple REST/HTTP API for accessing data. With
these basics in place, S3 provides the storage layer that powers many
of the modern apps we all use every day.

Like local file systems, distributed file systems support a number of
high level file system operations. In this project you'll implement
`read()`, `write()`, and `delete()` operations on objects using
HTTP APIs.

### HTTP/REST API

In your distributed file system you will have two different entities:
`files` and `directories`. You will access these entities using
standard HTTP/REST network calls. All URL paths begin with `/ds3/`,
which defines the root of your file system.

To create or update a file, use HTTP `PUT` method where the URL
defines the file name and path and the body of your PUT request
contains the entire contents of the file. If the file already exists,
the PUT call overwrites the contents with the new data sent via the
body.

In the system, directories are created implicitly. If a client PUTs a
file located at `/ds3/a/b/c.txt` and directories `a` and `b` do not
already exist, you will create them as a part of handling the
request. If one of the directories on the path exists as a file
already, like `/a/b`, then it is an error.

To read a file, you use the HTTP `GET` method, specifying the file
location as the path of your URL and your server will return the
contents of the file as the body in the response. To read a directory,
you also use the HTTP `GET` method, but directories list the entries
in a directory.  To encode directory entries, you put each directory
entry on a new line and regular files are listed directly, and
directories are listed with a trailing "/". For `GET` on a directory
omit the entries for `.` and `..`. For example, `GET` on `/ds3/a/b` will
return:

`c.txt`

And `GET` on `/ds3/a/` will return:

`b/`

The listed entries should be sorted using standard string comparison
sorting functions.

To delete a file, you use the HTTP `DELETE` method, specifying the
file location as the path of your URL. To delete a directory, you also
use `DELETE` but deleting a directory that is not empty it
is an error.

You will implement your API handlers in
[DistributedFileSystemService.cpp](gunrock_web/DistributedFileSystemService.cpp).

Since Gunrock is a HTTP server, you can use command line utilities,
like cURL to help test it out. Here are a few example cURL command:

```bash
% curl -X PUT -d "file contents" http://localhost:8080/ds3/a/b/c.txt 
% curl http://localhost:8080/ds3/a/b/c.txt                          
file contents
% curl http://localhost:8080/ds3/a/b/     
c.txt
% curl http://localhost:8080/ds3/a/b 
c.txt
% curl http://localhost:8080/ds3/a  
b/
% curl -X DELETE http://localhost:8080/ds3/a/b/c.txt
% curl http://localhost:8080/ds3/a/b/               
% 
```

### Dealing with errors

To implement your distributed storage
interface, you will use a sequence of LocalFileSystem calls.  Although
each of these calls individually will ensure that they will not modify
the disk when they have an error, since your implementation uses
several LocalFileSystem calls it needs to clean up when something goes
wrong. The key principle is that if an API call has an error, there
should not be any changes to the underlying disk or local file system.

To clean up the LocalFileSystem on errors, you can use the
[Disk](gunrock_web/include/Disk.h) interface for transactions.  To use
transactions, when you know that a file system call can change the
disk's state, then start a transaction by calling
`beginTransaction`. As your implementation for an API call proceeds,
if the call is successful then you can `commit` your transaction to
ensure that all file system modifications persist. If the call ends
with an error, you can call `rollback` to reverse any writes that
happened before the error.

There are four types of errors that your distributed file system can
return. First, use `ClientError::notFound()` for any API calls that
try to access a resouce that does not exist. Second, use
`ClientError::insufficientStorage()` for operations that modify the
file system and need to allocate new blocks but the disk does not have
enough storage to satisfy them.  Third, use `ClientError::conflict()`
if an API call tries to create a directory in a location where a file
already exists. Fourth, use `ClientError::badRequest()` for all other
errors.

To return an error to the client, in your
`DistributedFileSystemService.cpp` file, throw an exception using the
[ClientError](gunrock_web/include/ClientError.h) exception class, and
the gunrock web framework will catch these errors and convert them to
the right HTTP response and status code for that error.

## Local file system

### On-Disk File System: A Basic Unix File System

The on-disk file system structures follow that of the
very simple file system discussed
[here](https://pages.cs.wisc.edu/~remzi/OSTEP/file-implementation.pdf). On-disk,
the structures are as follows:
- A single block (4KB) super block
- An inode bitmap (can be one or more 4KB blocks, depending on the number of inodes)
- A data bitmap (can be one or more 4KB blocks, depending on the number of data blocks)
- The inode table (a multiple of 4KB-sized blocks, depending on the number of inodes)
- The data region (some number of 4KB blocks, depending on the number of data blocks)

More details about on-disk structures can be found in the header
[ufs.h](ufs.h), which you should use. Specifically, this has a very
specific format for the super block, inode, and directory
entries. Bitmaps just have one bit per allocated unit as described in
the book.

As for directories, here is a little more detail.  Each directory has
an inode, and points to one or more data blocks that contain directory
entries. Each directory entry should be simple, and consist of 32
bytes: a name and an inode number pair. The name should be a
fixed-length field of size 28 bytes; the inode number is just an
integer (4 bytes). When a directory is created, it should contain two
entries: the name `.` (dot), which refers to this new directory's
inode number, and `..` (dot-dot), which refers to the parent
directory's inode number. For the root directory in a file system,
both `.` and `..` refer to the root directory.

When your server is started, it is passed the name of the file system
image file. The image is created by a tool we provide, called `mkfs`.
It is pretty self-explanatory and can be found
[here](gunrock_web/mkfs.c).

When accessing the files on an image, your server should read in the
superblock, bitmaps, and inode table from disk as needed. When writing
to the image, you should update these on-disk structures accordingly.

One important aspect of your on-disk structure is that you need to
assume that your server can crash at any time, so all disk writes need
to leave your file system in a consistent state. To maintain
consistency typically you'd need to order writes carefully. But for
this assigment, instead you just need to make sure that your disk is
in a consistent state after each `LocalFileSystem` call returns.

Importantly, you cannot change the file-system on-disk format.

For more detailed documentation on the local file system specification,
please see [LocalFileSystem.h](gunrock_web/include/LocalFileSystem.h)
and the stub [LocalFileSystem.cpp](gunrock_web/LocalFileSystem.cpp). Also,
please see [Disk.h](gunrock_web/include/Disk.h) for the interface for accessing
the disk.

### Bitmaps for block allocation
We use on-disk bitmaps to keep track of entries (inodes and data blocks) that
the file system has allocated. For our bitmaps for each byte, the least
significant bit (LSB) is considered the first bit, and the most significant
bit (MSB) is considered the last bit. So if the first bit of a two byte
bitmap is set, it will look like this in hex:

```
byte position:  0  1
hex value:     01 00

bit position   0                               15
bit value:     1 0 0 0  0 0 0 0  0 0 0 0  0 0 0 0
```

and if the last bit is set it will look like this in hex:

```
byte position:  0  1
hex value:     00 80

bit position   0                               15
bit value:     0 0 0 0  0 0 0 0  0 0 0 0  0 0 0 1
```

### LocalFileSystem `write` and `read` semantics
In our file system, we don't have a notion of appending or modifying data.
Conceptually, when we get a `write` call we overwrite the entire contents
of the file with the new contents of the file, and write calls specify
the complete contents of the file.

Calls to `write`, `create`, and `unlink` need to reuse existing data
blocks. If the new file uses fewer data blocks, you must free the
extra data blocks. If the new file needs more data blocks, you must
add these new blocks while still reusing the current blocks for the
first part of the data.

Calls to `read` always read data starting from the beginning of the file,
but if the caller specifies a size of less than the size of the object
then you should return only these bytes. If the caller specifies a size
of larger than the size of the object, then you only return the bytes
in the object.

### Allocating data blocks and inodes

When you need to allocate a new data block or inode, you must always
use the lowest numbered entry that is available.

### LocalFileSystem out of storage errors
One important class of errors that your `LocalFileSystem` needs to handle is
out of storage errors. Out of storage errors can happen when one of the
file system modification calls -- `create` and `write` -- does not
have enough availabe storage to complete the request.

For `write` calls, you should write as many bytes as you can and
return success to the caller with the number of bytes actually
written.

For `create` calls, you'll need to allocate both an inode and a disk
block for directories. If you have allocated one of these entities but
can't allocate the other, make sure you free allocated inodes or disk
blocks before returning an error from the call.

## Read-only file system utilities

To help debug your disk images, you will create seven small
command-line utilities that access a given disk image.  We have
included an example disk image and expected outputs in the
[disk_testing](gunrock_web/tests/disk_images) directory, but make sure that
your utilities can handle multiple different disk image configurations
and contents.

For the `ds3ls`, `ds3bits`, and `ds3cat` utilities, you'll only need
to implement the "read" calls from your local file system. These read
calls are `lookup`, `stat`, and `read`.  For `ds3bits` you'll want to
implement `readSuperBlock`, `readInodeBitmap`, and
`readDataBitmap`. We recommend getting these utilities and
`LocalFileSystem.cpp` functions working first.

We have included the full set of test cases that we use to test the
read-only parts of your local file system with this repo. You can run
them using the `test-readonly.sh` testing script. It uses a similar
setup as our test cases for project 1 and project 2.

Note: We will only test your utilities on correct disk images -- you
can assume that all data on disk in the test cases is consistent and
correct.

### Error handling

In general, most of the error handling can be done by your
`LocalFileSystem.cpp` implementation. These utilities call the
underlying file system methods and convey an error when an error
occurs, where the actual error checking logic happens in the file
system implementation. For all of our utilities, we have a single
string that you write to standard error for all errors that you may
encounter.

### The `ds3ls` utility

The `ds3ls` prints the contents of a directory. This utility takes two
arguments: the name of the disk image file to use and the path of the
directory or file to list within the disk image. For directories, it
prints all of the entries in the directory, sorted using the
`std::sort` function. Each entry goes on its own line. For files, it
prints just the information for that file. Each entry will include the
inode number, a tab, the name of the entry, and finishing it off with
a newline.

For all errors, we'll use a single error string: `Directory not found`
printed to standard error (e.g., cerr), and your process will exit
with a return code of 1. On success, your program's return code is 0.

### The `ds3cat` utility

The `ds3cat` utility prints the contents of a file to standard
output. It takes the name of the disk image file and an inode number
as the only arguments. It prints the contents of the file that is
specified by the inode number.

For this utility, first print the string `File blocks` with a newline
at the end and then print each of the disk block numbers for the file
to standard out, one disk block number per line. After printing the
file blocks, print an empty line with only a newline.

Next, print the string `File data` with a newline at the end and then
print the full contents of the file to standard out. You do not need
to differentiate between files and directories, for this utility
everything is considered to be data and should be printed to standard
out.

Calling `ds3cat` on a directory is an error. For all errors print the
string `Error reading file" to standard error and set your processes
return code to 1.

### The `ds3bits` utility

The `ds3bits` utility prints metadata for the file system on a disk
image. It takes a single command line argument: the name of a disk
image file.

First, it prints metadata about the file system starting with the
string `Super` on a line, then `inode_region_addr` followed by a space
and then the inode region address from the super block. Next, it
should print the string `inode_region_len` followed by a space and the
value from the super block. Finally, it should print the string
`num_inodes` followed by a space and the value from the super
block. Repeat this process for the data region.

After printing the super block contents, print an empty line.

Next it prints the inode and data bitmaps. Each bitmap starts with a
string on its own line, `Inode bitmap` and `Data bitmap`
respectively. For each bitmap, print the byte value, formatted as an
`unsigned int` followed by a space. For each byte in your bitmap your
print statement might look something like this:

```
cout << (unsigned int) bitmap[idx] << " ";
```

Where each byte is followed by a space, including the last byte and
after you're done printing all of the bytes print a single newline
character.

Print the indoe bitmap first, followed by blank line consisting of
only a single newline character, then print the data bitmap.

## Read/write file system utilities

In addition to the utilities that you'll use to read a file system
image, you will also write four simple utilities to modify disk images
using your `LocalFileSystem.cpp` implementation.

Note: On success, none of these utilities print anything to standard
out, but they will use a process return code of 0 to signify success.

We _highly_ recommend getting your read-only utilities working before
moving on to the read/write utilities. We use your read-only utilities
to generate the outputs that we use for testing your read/write
utilities, so you need to make sure that that part of your
`LocalFileSystem.cpp` works.

### The `ds3mkdir` and `ds3touch` utilities

The `ds3mkdir` and `ds3touch` utilities creates directories and files
respectively in your file system. On the command line, these utilities
takes the disk image file, the parentInode for the directory where you
will create the new entry, and the name of your new entry.

For all errors, print the string `Error creating directory`
(`ds3mkdir`) or `Error creating file` (`ds3touch`) to standard error
and exit your program with a return code of 1. As with the underlying
`LocalFileSystem.cpp` implementation, creating a directory or file
that already exists is not an error as long as the type (file or
directory) is consistent with the existing entity.

### The `ds3cp` utility

The `ds3cp` utility copies a file from your computer into the disk
image using your `LocalFileSystem.cpp` implementation. It takes three
command line arguments, the disk image file, the source file from your
computer that you want to copy in, and the inode for the destination
file within your disk image.

For all errors, print the string `Could not write to dst_file` to
standard error and exit with return code 1. We will always use files
that exist on the computer for testing, so if you encoutner an error
opening or reading from the source file you can print any error
message that makes sense.

### The `ds3rm` utility

The `ds3rm` utility removes a file or empty directory from your disk
image's file system. It takes three arguments: the disk image file
name, the inode for the parent directory, and the name of the file or
directory that you want to delete. For all errors, print the string
`Error removing entry` to standard error and exit with return code 1.

## Hints

Here are a few hints to help you get started:

- For the read/write utilities, we will use the read-only utilities in
  our testing scripts to generate the outputs that we use in our
  testing framework. Make sure that the read-only utilities are
  working well before moving on to the read/write utilities.

- It's ok to have extra read and write disk operations. Normally in a
  file system implementation, the file system creators put a lot of
  care into minimizing reads and writes. However, to keep your
  implementation simple you are free to issue extra disk reads and
  writes as needed.

- If you encounter an error while you are in the middle of an
  operation, it's ok to clean up data that you stored on disk
  previously. In a real file system you would avoid writing anything
  to disk that could leave your file system in an inconsistent state,
  but for this assignment it's ok to violate the public invariant for
  on-disk structures temporarily as long as you make sure that it's
  consistent before returning to the caller.

- A clean implementation of the modification `LocalFileSystem.cpp`
  methods should be able to reuse the same internal "write" function
  to update data blocks. Spend some time thinking through what you'd
  need for `create`, `write`, and `unlink` and make sure that you
  structure your code to reuse the same internal function. It'll make
  it much simpler to implement these functions. The key will be
  deciding which error checks go in the internal function vs the
  calling function.

- For `ds3cat`, when you read data from your file system you're
  reading bytes, which is not NULL terminated. When you print the
  data, make sure that you NULL terminate the data before printing it
  to the screen.

- To calculate the number of disk blocks given a file size, use this
  logic:

```
int blocks = fileSize / UFS_BLOCK_SIZE;
if ((fileSize % UFS_BLOCK_SIZE) != 0) {
  blocks += 1;
}
```

## Visual studio code and debugging

Using the debugger in Visual Studio Code is important for this
assigment. We configured vscode so that it includes debugging tasks
for each of your utilities. Us it, it will make your life much simpler
when you are debugging. See the repo's [README](../README.md) for more
details on how to setup and use Visual Studio Code with Docker for
development and debugging.

## Gradescope

We are using Gradescope to autograde your projects. You should submit
the following files to Gradescope: `LocalFileSystem.cpp`,
`DistributedFileSystemService.cpp`, `ds3ls.cpp`, `ds3cat.cpp`,
`ds3bits.cpp`, `ds3mkdir.cpp`, `ds3touch.cpp`, `ds3cp.cpp`, and
`ds3rm.cpp`. The autograder requires all files to run so we provided
stubs for each of these that you can use until you have your own
implementation of them.

