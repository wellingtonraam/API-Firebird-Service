# binary-object

Encode json objects into a binary format.
Inspired by msgpack. With reduces memory usage.

[![npm Package Version](https://img.shields.io/npm/v/binary-object.svg?maxAge=3600)](https://www.npmjs.com/package/binary-object)

## Features
- Encode any value to binary format (and decode back)
- Memory-efficient
- Auto detect object schema to make the binary format more compact (optional)
- This library is highly composable, it can be extended/reused to support different encoding schema and input/output channel

- Support varies encode/decode pipes:
  - [x] Binary Object
  - [x] Binary JSON
  - [x] Object Schema (store object keys only once, similar to csv, but nested)
  - [x] [compress-json](https://www.npmjs.com/package/compress-json)
  - [x] [msgpack](https://www.npmjs.com/package/msgpack)

- Support varies input/output channel:
  - [ ] Buffer
  - [x] File
  - [ ] Stream (e.g. fs / net stream)
  - [x] Callback (for producer / consumer pattern)
  - [x] Array (for in-memory mock test)

Wide range of javascript data type are supported:
- string
- number
- bigint
- boolean
- Buffer
- Array
- Map
- Set
- Date
- object
- symbol
- function
- undefined
- null

## Why not [MsgPack](https://github.com/msgpack/msgpack-node)?
MsgPack cannot reclaim the memory usage.

When I use MsgPack to encode ~211k objects, and write them to a file (via fs.writeSync, fs.appendFileSync or fs.createWriteStream), the node runs out of memory.

In the test, each object comes from LMDB, a sync-mode embedded database.
The key and value of each object packed and written to file separately.
It doesn't cause out of memory error if I load all the objects into memory, then pack them as a single array (then write to file) but this batching approach doesn't work for long stream of data.

I tried to use setTimeout to slow down the writing, and even explicitly call `process.gc()` but the problem persist.

## Why not [BON](https://github.com/bon-org/bon-js)?
BON does not support parsing from file / stream.

The current implementation of BON requires the binary data to be passed into the `decode()` or `decode_all()` function in complete.

Also, the data schema of BON does not specify the length of list and string, which is compact in turns of the resulting binary data.
However, lacking this information means the decoding process cannot be preciously fetch the right amount of bytes from the file / readable stream.

As a result, BON cannot support continuous decoding from a large file / long stream.

## How this library works?
This library re-use the buffer when encoding across multiple calls.
Effective like the object pooling but for buffer.

Also, the length of each data chunk is deterministic from the header (data type and length).
Therefore, the decoder knows exactly how many bytes should be read to process.

In addition, some source type support `*iterator()` generator method which help to decode the binary data iteratively.

This library implement the I/O and decode/encode as pipes, namely Sink and Source.

Multiple sinks or sources can be stacked together in flexible manner. e.g.
```typescript
const sink = new SchemaSink(new BinaryJsonSink(FileSink.fromFile('db.log')))

const source = new SchemaSource(new BinaryJsonSource(FileSource.fromFile('db.log')))
```

Or you can use some pre-defined factory functions to construct the pipes
```typescript
const sink = BinaryObjectFileSink.fromFile('db.log')

const source = BinaryJsonFileSource.fromFile('db.log')
```

## Does this work?
The correctness is tested and passed.

The benchmarking is not done.

## Combination & Performance

### Sample Data
266430 sample json data crawled from online forum.

Total size: 843M

The objects have consistent shape.

Some data are duplicated, e.g. user name, and some common comments.

### Benchmarks

The sample data are read line by line, instead of loading into memory at once to avoid "Out of Memory Error".

**high-write-speed**:
data > json > raw-line-file
| 843M
| 38.07s write
| 38.74s read

**high-read-speed**:
data > binary-json > file
| 843M
| 42.95s write
| 30.02s read

**-high-read-speed-disk-space-efficient**:
data > schema > msgpack > file
| 640M
| 51.80s write
| 18.34s read

**more-disk-space-efficient**:
data > unique-value > line-file
| 506M
| 151.64s write
| 59.26s read
(estimated from 50% sample to avoid Out of Memory Error)

## LICENSE
[BSD-2-Clause](./LICENSE) (Free Open Sourced Software)
