# yazl

yet another zip library for node. For unzipping, see
[yauzl](https://github.com/thejoshwolfe/yauzl).

Design principles:

 * Don't block the JavaScript thread.
   Use and provide async APIs.
 * Keep memory usage under control.
   Don't attempt to buffer entire files in RAM at once.
 * Prefer to open input files one at a time than all at once.
   This is slightly suboptimal for time performance,
   but avoids OS-imposed limits on the number of simultaneously open file handles.

## Usage

```js
var yazl = require("yazl");

var zipfile = new yazl.ZipFile();
zipfile.addFile("file1.txt", "file1.txt");
// (add only files, not directories)
zipfile.addFile("path/to/file.txt", "path/in/zipfile.txt");
// pipe() can be called any time after the constructor
zipfile.outputStream.pipe(fs.createWriteStream("output.zip")).on("close", function() {
  console.log("done");
});
// alternate apis for adding files:
zipfile.addReadStream(process.stdin, "stdin.txt", {
  mtime: new Date(),
  mode: 0100664, // -rw-rw-r--
});
zipfile.addBuffer(new Buffer("hello"), "hello.txt", {
  mtime: new Date(),
  mode: 0100664, // -rw-rw-r--
});
// call end() after all the files have been added
zipfile.end();
```

## API

### Class: ZipFile

#### new ZipFile()

No parameters.
Nothing can go wrong.

#### addFile(realPath, metadataPath, [options])

Adds a file from the file system at `realPath` into the zipfile as `metadataPath`.
Typically `metadataPath` would be calculated as `path.relative(root, realPath)`.
Unzip programs would extract the file from the zipfile as `metadataPath`.
`realPath` is not stored in the zipfile.

A valid `metadataPath` must not be blank, must not start with `"/"` or `/[A-Za-z]:\//`,
and must not contain `"\\"` or `".."` path segments.
File paths must not end with `"/"`.

`options` may be omitted or null and has the following structure and default values:

```js
{
  mtime: stats.mtime,
  mode: stats.mode,
  compress: true,
}
```

Use `options.mtime` and/or `options.mode` to override the values
that would normally be obtained by the `fs.Stats` for the `realPath`.
The mode is the unix permission bits and file type.
The mtime and mode are stored in the zip file in the fields "last mod file time",
"last mod file date", and "external file attributes".
yazl does not store group and user ids in the zip file.

Internally, `fs.stat()` is called immediately in the `addFile` function,
and `fs.createReadStream()` is used later when the file data is actually required.
Throughout adding and encoding `n` files with `addFile()`,
the number of simultaneous open files is `O(1)`, probably just 1 at a time.

#### addReadStream(readStream, metadataPath, [options])

Adds a file to the zip file whose content is read from `readStream`.
See `addFile()` for info about the `metadataPath` parameter.
`options` may be omitted or null and has the following structure and default values:

```js
{
  mtime: new Date(),
  mode: 0100664,
  compress: true,
  size: 12345, // example value
}
```

See `addFile()` for the meaning of `mtime` and `mode`.
If `size` is given, it will be checked against the actual number of bytes in the `readStream`,
and an error will be emitted if there is a mismatch.

Note that yazl will `.pipe()` data from `readStream`, so be careful using `.on('data')`.
In certain versions of node, `.on('data')` makes `.pipe()` behave incorrectly.

#### addBuffer(buffer, metadataPath, [options])

Adds a file to the zip file whose content is `buffer`.
See `addFile()` for info about the `metadataPath` parameter.
`options` may be omitted or null and has the following structure and default values:

```js
{
  mtime: new Date(),
  mode: 0100664,
  compress: true,
}
```

See `addFile()` for the meaning of `mtime` and `mode`.

#### addEmptyDirectory(metadataPath, [options])

Adds an entry to the zip file that indicates a directory should be created,
even if no other items in the zip file are contained in the directory.
This method is only required if the zip file is intended to contain an empty directory.

See `addFile()` for info about the `metadataPath` parameter.
If `metadataPath` does not end with a `"/"`, a `"/"` will be appended.

`options` may be omitted or null and has the following structure and default values:

```js
{
  mtime: new Date(),
  mode: 040775,
}
```

See `addFile()` for the meaning of `mtime` and `mode`.

#### end([finalSizeCallback])

Indicates that no more files will be added via `addFile()`, `addReadStream()`, or `addBuffer()`.
Some time after calling this function, `outputStream` will be ended.

If specified and non-null, `finalSizeCallback` is given the parameters `(finalSize)`
sometime during or after the call to `end()`.
`finalSize` is of type `Number` and can either be `-1`
or the guaranteed eventual size in bytes of the output data that can be read from `outputStream`.

If `finalSize` is `-1`, it means means the final size is too hard to guess before processing the input file data.
This will happen if and only if the `compress` option is `true` on any call to `addFile()`, `addReadStream()`, or `addBuffer()`,
or if `addReadStream()` is called and the optional `size` option is not given.
In other words, clients should know whether they're going to get a `-1` or a real value
by looking at how they are calling this function.

The call to `finalSizeCallback` might be delayed if yazl is still waiting for `fs.Stats` for an `addFile()` entry.
If `addFile()` was never called, `finalSizeCallback` will be called during the call to `end()`.
It is not required to start piping data from `outputStream` before `finalSizeCallback` is called.
`finalSizeCallback` will be called only once, and only if this is the first call to `end()`.

#### outputStream

A readable stream that will produce the contents of the zip file.
It is typical to pipe this stream to a writable stream created from `fs.createWriteStream()`.

Internally, large amounts of file data are piped to `outputStream` using `pipe()`,
which means throttling happens appropriately when this stream is piped to a slow destination.

Data becomes available in this stream soon after calling one of `addFile()`, `addReadStream()`, or `addBuffer()`.
Clients can call `pipe()` on this stream at any time,
such as immediately after getting a new `ZipFile` instance, or long after calling `end()`.

As a reminder, be careful using both `.on('data')` and `.pipe()` with this stream.
In certain versions of node, you cannot use both `.on('data')` and `.pipe()` successfully.

### dateToDosDateTime(jsDate)

`jsDate` is a `Date` instance.
Returns `{date: date, time: time}`, where `date` and `time` are unsigned 16-bit integers.

## Output Structure

The Zip File Spec leaves a lot of flexibility up to the zip file creator.
This section explains and justifies yazl's interpretation and decisions regarding this flexibility.

This section is probably not useful to yazl clients,
but may be interesting to unzip implementors and zip file enthusiasts.

### Disk Numbers

All values related to disk numbers are `0`,
because yazl has no multi-disk archive support.

### Version Made By

Always `0x031e`.
This is the value reported by a Linux build of Info-Zip.
Instead of experimenting with different values of this field
to see how different unzip clients would behave,
yazl mimics Info-Zip, which should work everywhere.

Note that the top byte means "UNIX"
and has implications in the External File Attributes.

### Version Needed to Extract

Always `0x0014`.
Without this value, Info-Zip, and possibly other unzip implementations,
refuse to acknowledge General Purpose Bit `8`, which enables utf8 filename encoding.

### General Purpose Bit Flag

Bit `8` is always set.
Filenames are always encoded in utf8, even if the result is indistinguishable from ascii.

Bit `3` is set in the Local File Header.
To support both a streaming input and streaming output api,
it is impossible to know the crc32 before processing the file data.
File Descriptors are given after each file data with this information, as per the spec.
But remember a complete metadata listing is still always available in the central directory record,
so if unzip implementations are relying on that, like they should,
none of this paragraph will matter anyway.
Even so, Mac's Archive Utility requires File Descriptors to include the optional signature,
so yazl includes the optional file descriptor signature.

All other bits are unset.

### Internal File Attributes

Always `0`.
The "apparently an ASCII or text file" bit is always unset meaning "apparently binary".
This kind of determination is outside the scope of yazl,
and is probably not significant in any modern unzip implementation.

### External File Attributes

Always `stats.mode << 16`.
This is apparently the convention for "version made by" = `0x03xx` (UNIX).

Note that for directory entries (see `addEmptyDirectory()`),
it is conventional to use the lower 8 bits for the MS-DOS directory attribute byte.
However, the spec says this is only required if the Version Made By is DOS,
so this library does not do that.

### Directory Entries

When adding a `metadataPath` such as `"parent/file.txt"`, yazl does not add a directory entry for `"parent/"`,
because file entries imply the need for their parent directories.
Unzip clients seem to respect this style of pathing,
and the zip file spec does not specify what is standard in this regard.

In order to create empty directories, use `addEmptyDirectory()`.

## Change History

 * 2.1.1
   * Fixed stack overflow when using addBuffer() in certain ways. [issue #9](https://github.com/thejoshwolfe/yazl/issues/9)
 * 2.1.0
   * Added `addEmptyDirectory()`.
   * `options` is now optional for `addReadStream()` and `addBuffer()`.
 * 2.0.0
   * Initial release.
