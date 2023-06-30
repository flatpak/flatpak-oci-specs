tardiff
=======

The purpose of a tardiff is to represent the difference from one uncompressed tar file (the source), to another (the target).

The algorithm is designed to reconstruct the exact bytes of the target tarball so that they can be verified by checksum to match the source tarball, and to be independent of the details of tar file formats.

Delta compression is based on [bsdiff](http://www.daemonology.net/bsdiff/).

Algorithm
---------
 - unpack the first tar file to create a directory tree, which will be referenced by the tardiff
 - apply the sequence of operations in the tardiff file, producing a stream of bytes for the new tar file. 

File Format
-----------

A delta file consists of a header, with the fixed bytes:

```
{ 't', 'a', 'r', 'd', 'f', '1', '\n', 0}
```

Followed by a [zstd](https://facebook.github.io/zstd/) compressed stream, with a sequence of operations, each operation is encoded as follows:

```
op: 1 byte
size: uint64 encoded as a varint 
data: <size> bytes. data is absent for DeltaOpCopy and DeltaOpSeek
```

For varint encoding, see: https://developers.google.com/protocol-buffers/docs/encoding#varints

```
DeltaOpData = 0
DeltaOpOpen = 1
DeltaOpCopy = 2
DeltaOpAddData = 3
DeltaOpSeek = 4
```

***DeltaOpData***
Emit the bytes from `<data>` into the output stream.

***DeltaOpOpen***
`<data>` is a the path to a file within the original tarball. Set the source for subsequent DeltaOpCopy and DeltaAddData operations to this file, and reset the source position to 0.

***DeltaOpCopy***
Emit `<size>` bytes from the source file to the output stream, and advance the source position by `<size>`

***DeltaOpAddData***
Read `<size>` bytes from the source file, and advance the source position by `<size>`. Add these bytes bytewise to the bytes in `<data>` with unsigned wrapping, and emit the result to the output stream.

***DeltaOpSeek***
Set the source position to `<size>`
