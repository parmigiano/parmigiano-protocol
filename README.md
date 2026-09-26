# Parmigiano protocol

[Русская версия](README.ru.md)

Protobuf protocol, realtime chat packets, presence, and client updates.

This repository is the wire contract for Parmigiano. It is not an application. Clients and servers import the schema, generate code, and exchange length-prefixed protobuf packets. The published module is [buf.build/parmigiano/protocol](https://buf.build/parmigiano/protocol).

Field-by-field meaning of every message is in [proto/parmigiano/README.md](proto/parmigiano/README.md).

## How a packet works

The connection carries one packet at a time:

```text
[ uint32 length ][ protobuf bytes ]
```

`length` is a big-endian unsigned 32-bit integer. It counts only the protobuf bytes. The 4-byte prefix itself is not included.

The client serializes `ClientMessage`. The server serializes `ServerMessage`. Each message has exactly one `payload` branch set. `ClientMessage.user_id` is the sender and is not repeated inside the payload.

Package name: `parmigiano.v1`.

| File | Who sends it | What it contains |
| --- | --- | --- |
| `proto/parmigiano/v1/common.proto` | both | Shared types: content kind, files, directories |
| `proto/parmigiano/v1/client.proto` | client | Connect, chat actions, presence, update check |
| `proto/parmigiano/v1/server.proto` | server | Chat events, public keys, disconnect, update manifest |

## Repository layout

```text
proto/parmigiano/v1/          schema
proto/parmigiano/README.md    message reference, English and Russian
buf.yaml                      Buf module buf.build/parmigiano/protocol
buf.gen.examples/             example buf generate templates
.github/workflows/check.yml   lint and breaking checks on a pull request
.github/workflows/publish.yml push to the Buf Schema Registry
```

Generated code is written to `gen/` and is not committed.

## What to install

Install the [Buf CLI](https://buf.build/docs/installation/). Check it:

```bash
buf --version
```

You need Buf to lint the schema and to generate code. Application runtimes are separate and depend on the language you generate for. The examples below name the runtime each plugin expects.

A Git checkout of this repository is for people who change the schema or generate from local files. Application developers can depend on the module in the Buf Schema Registry after it has been published and skip cloning.

## Change the schema

1. Edit the `.proto` files under `proto/parmigiano/v1/`.
2. Keep the indentation at 4 spaces. `buf format` rewrites files to 2 spaces, so do not run it on this repository.
3. Update [proto/parmigiano/README.md](proto/parmigiano/README.md) in both languages when a field or message changes meaning.
4. From the repository root, check the schema:

```bash
buf lint
buf build
```

Rules that keep old and new code compatible:

- Field numbers are the wire contract. Do not change the number of an existing field, and do not reuse a number.
- To remove a field or an enum value, delete it and add `reserved` for its number and name.
- Add a new field with the next free number.
- A new client or server action is a new message plus a new branch in the `oneof payload`.
- The first enum value is `0` and its name ends with `_UNSPECIFIED`.
- Other enum value names start with the enum name in `UPPER_SNAKE_CASE`. `DeleteReason` uses `DELETE_REASON_`. Two enums in one package cannot share a value name.
- `oneof` carries one payload. Do not put a second body on the same message.

Compatible additions are new fields, new `oneof` branches, and new enum values. Incompatible changes are renamed types, changed field types, and deleted numbers. Put an incompatible schema in a new package such as `parmigiano.v2` instead of rewriting `v1`.

## Pull requests

Open a pull request into `main`. The Check workflow runs `buf lint` and `buf breaking` against `main`. It does not publish.

`buf breaking` uses the `FILE` rules in `buf.yaml`. It rejects deletes and type changes that would break the schema already on `main`.

If the break is intentional, add the label `buf skip breaking` to the pull request. Lint still runs. Use that label only when you mean to break compatibility, and prefer a new package version for a real wire break.

Pushing straight to `main` does not run Check. Review goes through a pull request.

## Releases and tags

Merging into `main` runs Publish. That workflow runs `buf push` and updates the module on the Buf Schema Registry. It does not lint or run breaking checks again. Those already ran on the pull request.

Publish also runs when you push a tag that matches `v*.*.*`:

```bash
git tag v1.2.0
git push origin v1.2.0
```

Use [semantic versions](https://semver.org/):

| Tag | When |
| --- | --- |
| `v1.2.3` | Documentation or other change with no wire effect |
| `v1.3.0` | Compatible addition: new fields, messages, or enum values |
| `v2.0.0` | Incompatible change, together with a new protobuf package |

The tag records the release. The BSR module label `main` moves on every merge. A version tag is the label consumers should pin.

Publishing uses the GitHub Actions secret `BUF_TOKEN`. It is a Buf API token with `module.push` on `parmigiano/protocol`. Create it in the Buf account settings and store it under that exact name in the repository secrets. `GITHUB_TOKEN` is provided by GitHub Actions and is not a secret you create.

## Use the schema

Do not generate code in the application repository. After Publish, the Buf Schema Registry builds a package for every language it supports. Install that package with the language's package manager.

The module is `buf.build/parmigiano/protocol`. Copy-paste snippets for the current commit are on the [SDKs tab](https://buf.build/parmigiano/protocol/sdks). The commands below install the latest commit on the default label. Pin a version from that page for a release build.

Overview of generated SDKs: [buf.build/docs/bsr/generated-sdks/overview](https://buf.build/docs/bsr/generated-sdks/overview/).

| Language | Install | Guide |
| --- | --- | --- |
| Go | `go get buf.build/gen/go/parmigiano/protocol/protocolbuffers/go@latest` | [Go](https://buf.build/docs/bsr/generated-sdks/go/) |
| JavaScript and TypeScript | `npm install @buf/parmigiano_protocol.bufbuild_es` | [npm](https://buf.build/docs/bsr/generated-sdks/npm/) |
| Java | `build.buf.gen:parmigiano_protocol_protocolbuffers_java` | [Maven](https://buf.build/docs/bsr/generated-sdks/maven/) |
| Kotlin | `build.buf.gen:parmigiano_protocol_protocolbuffers_kotlin` | [Maven](https://buf.build/docs/bsr/generated-sdks/maven/) |
| Swift | package `buf.parmigiano_protocol_apple_swift` | [Swift](https://buf.build/docs/bsr/generated-sdks/swift/) |
| Python | `pip install parmigiano-protocol-protocolbuffers-python` | [Python](https://buf.build/docs/bsr/generated-sdks/python/) |
| Rust | `cargo add --registry buf parmigiano_protocol_community_neoeinstein-prost` | [Cargo](https://buf.build/docs/bsr/generated-sdks/cargo/) |
| C# | `dotnet add package BSR.Parmigiano.Protocol.Protocolbuffers.Csharp` | [NuGet](https://buf.build/docs/bsr/generated-sdks/nuget/) |
| C++ | CMake `FetchContent` target `parmigiano_protocol_protocolbuffers_cpp` | [CMake](https://buf.build/docs/bsr/generated-sdks/cmake/) |

C is not published as a generated SDK. Generate it locally with `buf.gen.examples/c` if you need protobuf-c.

### Go

[Go SDKs](https://buf.build/docs/bsr/generated-sdks/go/). A fresh push can take up to about 30 minutes to show up through `proxy.golang.org`. Set `GOPRIVATE=buf.build/gen/go` to fetch from the BSR directly.

```bash
go get buf.build/gen/go/parmigiano/protocol/protocolbuffers/go@latest
```

Import path: `buf.build/gen/go/parmigiano/protocol/protocolbuffers/go/parmigiano/v1`.

### JavaScript and TypeScript

[npm SDKs](https://buf.build/docs/bsr/generated-sdks/npm/). One package covers both languages. Point the `@buf` scope at the BSR once:

```bash
npm config set @buf:registry https://buf.build/gen/npm/v1
npm install @buf/parmigiano_protocol.bufbuild_es
```

### Java and Kotlin

[Maven and Gradle SDKs](https://buf.build/docs/bsr/generated-sdks/maven/). Add the repository `https://buf.build/gen/maven`, then depend on:

```text
build.buf.gen:parmigiano_protocol_protocolbuffers_java
build.buf.gen:parmigiano_protocol_protocolbuffers_kotlin
```

Take the version string from the [SDKs tab](https://buf.build/parmigiano/protocol/sdks). Kotlin code uses the Java types, so an application that calls Kotlin APIs still needs the Java artifact.

### Swift

[Swift SDKs](https://buf.build/docs/bsr/generated-sdks/swift/).

```bash
swift package-registry set https://buf.build/gen/swift --scope=buf
```

Package id: `buf.parmigiano_protocol_apple_swift`. Product name: `Parmigiano_Protocol_Apple_Swift`. For an Xcode app target, paste the Git URL from the SDKs tab instead of using the registry.

### Python

[Python SDKs](https://buf.build/docs/bsr/generated-sdks/python/).

```bash
pip install parmigiano-protocol-protocolbuffers-python --extra-index-url https://buf.build/gen/python
```

### Rust

[Cargo SDKs](https://buf.build/docs/bsr/generated-sdks/cargo/). Rust crates are generated when the module page is opened or when Cargo asks for them, then on later pushes to the default label.

`.cargo/config.toml`:

```toml
[registries.buf]
index = "sparse+https://buf.build/gen/cargo/"
credential-provider = "cargo:token"
```

```bash
cargo login --registry buf "Bearer TOKEN"
cargo add --registry buf parmigiano_protocol_community_neoeinstein-prost
```

### C#

[NuGet SDKs](https://buf.build/docs/bsr/generated-sdks/nuget/). Add the feed `https://buf.build/gen/nuget/index.json` and map package ids `BSR.*` to it, as that page shows. Then:

```bash
dotnet add package BSR.Parmigiano.Protocol.Protocolbuffers.Csharp
```

### C++

[CMake SDKs](https://buf.build/docs/bsr/generated-sdks/cmake/). CMake 3.14 or later. Run `buf registry login` once. Copy `FetchContent_Declare` from the [SDKs tab](https://buf.build/parmigiano/protocol/sdks) for plugin `protocolbuffers/cpp`. The library name is `parmigiano_protocol_protocolbuffers_cpp`. There is no gRPC service in this schema, so do not add `grpc/cpp`.

The SDK gives message classes and `SerializeToArray` / `ParseFromArray`. The 4-byte length prefix stays in application code.

### Generate locally

`buf.gen.examples/` is only for the case where a package manager cannot be used. C has no BSR SDK and always uses this path. From the repository root:

```bash
buf generate --template buf.gen.examples/go/buf.gen.yml
buf generate --template buf.gen.examples/java/buf.gen.yml
buf generate --template buf.gen.examples/kotlin/buf.gen.yml
buf generate --template buf.gen.examples/swift/buf.gen.yml
buf generate --template buf.gen.examples/python/buf.gen.yml
buf generate --template buf.gen.examples/rust/buf.gen.yml
buf generate --template buf.gen.examples/csharp/buf.gen.yml
buf generate --template buf.gen.examples/cpp/buf.gen.yml
buf generate --template buf.gen.examples/c/buf.gen.yml
buf generate --template buf.gen.examples/typescript/buf.gen.yml
buf generate --template buf.gen.examples/javascript/buf.gen.yml
```

| Template | Output |
| --- | --- |
| `go` | `gen/go` |
| `java` | `gen/java` |
| `kotlin` | `gen/kotlin` |
| `swift` | `gen/swift` |
| `python` | `gen/python` |
| `rust` | `gen/rust` |
| `csharp` | `gen/csharp` |
| `cpp` | `gen/cpp` |
| `c` | `gen/c` |
| `typescript` | `gen/ts` |
| `javascript` | `gen/js` |

`gen/` is gitignored.

## License

See [LICENSE](LICENSE).
