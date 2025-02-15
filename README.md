<div align="center">

<img width="100" height="100" src="media/assemblyscript-logo.svg" alt="AssemblyScript logo">
<img width="100" height="100" src="media/protobuf-logo.svg" alt="Protobuf logo">

<h1>as-proto</h1>
<p>Protobuf implementation in AssemblyScript</p>

[![npm](https://img.shields.io/npm/v/as-proto)](https://www.npmjs.com/package/as-proto)

</div>

## Features 
 * Encodes and decodes protobuf messages
 * Generates AssemblyScript files using `protoc` plugin
 * Produces relatively small `.wasm` files
 * Relatively fast, especially for messages that contains only primitive types

## Installation
This package requires **Node 10.4+** or modern browser with [WebAssembly][1] support.
Requires [`protoc`][2] installed for code generation. 

```sh
# with npm
npm install --save as-proto
npm install --save-dev as-proto-gen

# with yarn
yarn add as-proto
yarn add --dev as-proto-gen
```

## Code generation
To generate AssemblyScript file from `.proto` file, use following command:
```sh
protoc --plugin=protoc-gen-as=./node_modules/.bin/as-proto-gen --as_out=. ./file.proto
```
This command will create `./file.ts` file from `./file.proto` file.

<details>
<summary>Generated code example:</summary>

```protobuf
// star-repo-message.proto
syntax = "proto3";

message StarRepoMessage {
  string author = 1;
  string repo   = 2;
}
```
```typescript
// star-repo-message.ts
import { Writer, Reader } from "as-proto";

export class StarRepoMessage {
  static encode(message: StarRepoMessage, writer: Writer): void {
    writer.uint32(10);
    writer.string(message.author);

    writer.uint32(18);
    writer.string(message.repo);
  }

  static decode(reader: Reader, length: i32): StarRepoMessage {
    const end: usize = length < 0 ? reader.end : reader.ptr + length;
    const message = new StarRepoMessage();

    while (reader.ptr < end) {
      const tag = reader.uint32();
      switch (tag >>> 3) {
        case 1:
          message.author = reader.string();
          break;

        case 2:
          message.repo = reader.string();
          break;

        default:
          reader.skipType(tag & 7);
          break;
      }
    }

    return message;
  }

  author: string;
  repo: string;

  constructor(author: string = "", repo: string = "") {
    this.author = author;
    this.repo = repo;
  }
}
```

</details>

## Usage
To encode and decode protobuf messages, all you need is `Protobuf` class and 
generated message class:

```typescript
import { Protobuf } from 'as-proto';
import { StarRepoMessage } from './star-repo-message'; // generated file

const message = new StarRepoMessage('piotr-oles', 'as-proto');

// encode
const encoded = Protobuf.encode(message, StarRepoMessage.encode);
assert(encoded instanceof Uint8Array);

// decode
const decoded = Protobuf.decode(encoded, StarRepoMessage.decode);
assert(decoded instanceof StarRepoMessage);
assert(decoded.author === 'piotr-oles');
assert(decoded.repo === 'as-proto')
```

Currently the package doesn't support GRPC definitions - only basic Protobuf messages.

## Performance
I used performance benchmark from [`ts-proto`][3] library and added case for `as-proto`.
The results on Intel Core i7 2.2 Ghz (MacBook Pro 2015):

```
benchmarking encoding performance ...

as-proto x 1,295,297 ops/sec ±0.30% (92 runs sampled)
protobuf.js (reflect) x 589,073 ops/sec ±0.27% (88 runs sampled)
protobuf.js (static) x 589,866 ops/sec ±1.66% (89 runs sampled)
JSON (string) x 379,723 ops/sec ±0.30% (95 runs sampled)
JSON (buffer) x 295,340 ops/sec ±0.26% (93 runs sampled)
google-protobuf x 338,984 ops/sec ±1.25% (84 runs sampled)

               as-proto was fastest
  protobuf.js (reflect) was 54.5% ops/sec slower (factor 2.2)
   protobuf.js (static) was 55.1% ops/sec slower (factor 2.2)
          JSON (string) was 70.7% ops/sec slower (factor 3.4)
        google-protobuf was 74.1% ops/sec slower (factor 3.9)
          JSON (buffer) was 77.2% ops/sec slower (factor 4.4)

benchmarking decoding performance ...

as-proto x 889,283 ops/sec ±0.51% (94 runs sampled)
protobuf.js (reflect) x 1,308,310 ops/sec ±0.24% (95 runs sampled)
protobuf.js (static) x 1,375,425 ops/sec ±2.86% (92 runs sampled)
JSON (string) x 387,722 ops/sec ±0.56% (95 runs sampled)
JSON (buffer) x 345,785 ops/sec ±0.33% (94 runs sampled)
google-protobuf x 359,038 ops/sec ±0.32% (94 runs sampled)

   protobuf.js (static) was fastest
  protobuf.js (reflect) was 2.4% ops/sec slower (factor 1.0)
               as-proto was 33.8% ops/sec slower (factor 1.5)
          JSON (string) was 71.2% ops/sec slower (factor 3.5)
        google-protobuf was 73.2% ops/sec slower (factor 3.7)
          JSON (buffer) was 74.2% ops/sec slower (factor 3.9)

benchmarking combined performance ...

as-proto x 548,291 ops/sec ±0.41% (96 runs sampled)
protobuf.js (reflect) x 421,963 ops/sec ±1.41% (89 runs sampled)
protobuf.js (static) x 439,242 ops/sec ±0.85% (96 runs sampled)
JSON (string) x 186,513 ops/sec ±0.25% (94 runs sampled)
JSON (buffer) x 153,775 ops/sec ±0.54% (94 runs sampled)
google-protobuf x 160,281 ops/sec ±0.46% (91 runs sampled)

               as-proto was fastest
   protobuf.js (static) was 20.2% ops/sec slower (factor 1.3)
  protobuf.js (reflect) was 23.8% ops/sec slower (factor 1.3)
          JSON (string) was 65.9% ops/sec slower (factor 2.9)
        google-protobuf was 70.8% ops/sec slower (factor 3.4)
          JSON (buffer) was 72.0% ops/sec slower (factor 3.6)
```

The library is slower on decoding mostly because of GC - AssemblyScript provides very simple (and small) GC
which is not as good as V8 GC. The `as-proto` beats JavaScript on decoding when messages contain
only primitive values or other messages (no strings and arrays). In this case the generator creates
`@unmanaged` classes which are much faster for memory management.

## License
MIT

[1]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly#browser_compatibility
[2]: https://grpc.io/docs/protoc-installation/
[3]: https://github.com/stephenh/ts-proto
