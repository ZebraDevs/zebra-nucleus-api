# Nucleus API: Proto Bundle

Protobuf definitions for the Nucleus APIs. This is a point-in-time snapshot of the contract, so
pin a copy and re-export when you need a newer version.

## Contents

```
protos/nucleus/client/printer/print/v1/print_service.proto      # PrintService: Print, GetPrintJobStatus, CancelPrintJob
protos/nucleus/client/printer/print/v1/print_messages.proto     # request/response messages
protos/nucleus/client/printer/print/v1/print_resources.proto    # PrintJob, PrintContent, enums
protos/nucleus/client/printer/v1/printer_resources.proto        # Alert, AlertState and AlertType only
protos/buf/validate/validate.proto                              # proto validate constraints (dependency)
google/protobuf/timestamp.proto                                 # vendored well-known type
google/protobuf/duration.proto                                  # vendored well-known type
google/protobuf/field_mask.proto                                # vendored well-known type
google/protobuf/descriptor.proto                                # vendored well-known type (buf/validate dependency)
```

`PrintService` is `nucleus.client.printer.print.v1.PrintService`, and `Print` is server-streaming.
Device management is a separate service, despite the `printer` path segment.

The `google.protobuf.*` well-known types needed by this bundle are vendored under `google/`, so the
bundle compiles standalone without relying on `protoc` or `buf` shipping them.

## Generate a client

Run these from the bundle root, `nucleus-apis-v1/`. Pass both `protos/` and `google/` as import
paths — application protos live under the former, vendored well-known types under the latter.
Output goes to `gen/`, so keep it out of the bundle you re-export.

Verify everything compiles:
```bash
protoc -I protos -I google --descriptor_set_out=/dev/null $(find protos -name '*.proto')
```

### Java

`--java_out` is built into protoc but only emits message classes. The gRPC service stub needs
`protoc-gen-grpc-java`, which ships separately as a per-platform binary on Maven Central under
`io.grpc:protoc-gen-grpc-java`. Grab the `.exe` artifact matching your platform classifier, then
invoke it as the `protoc` plugin.

#### macOS / Linux

Classifier: `osx-x86_64` (Intel Mac), `osx-aarch_64` (Apple Silicon), `linux-x86_64`, or
`linux-aarch_64` (ARM Linux).

```bash
mkdir -p gen/java
chmod +x protoc-gen-grpc-java
protoc -I protos -I google \
  --plugin=protoc-gen-grpc-java=./protoc-gen-grpc-java \
  --java_out=gen/java --grpc-java_out=gen/java \
  $(find protos -name '*.proto')
```

#### Windows

Classifier: `windows-x86_64`. The plugin path needs the `.exe` extension, and `find` isn't
available in `cmd`/PowerShell, so list the proto files explicitly or use a shell that has it (Git
Bash, WSL).

```powershell
mkdir gen\java -Force
protoc -I protos -I google `
  --plugin=protoc-gen-grpc-java=.\protoc-gen-grpc-java.exe `
  --java_out=gen\java --grpc-java_out=gen\java `
  protos\nucleus\client\printer\print\v1\print_messages.proto `
  protos\nucleus\client\printer\print\v1\print_resources.proto `
  protos\nucleus\client\printer\print\v1\print_service.proto `
  protos\nucleus\client\printer\v1\printer_resources.proto `
  protos\buf\validate\validate.proto
```

That produces the message classes plus `PrintServiceGrpc`. The protos already set
`option java_package`, so the output lands under `com.zebra.proto.nucleus.*`.

On a Gradle or Maven build, use the `com.google.protobuf` plugin instead. It resolves the same
grpc-java artifact for you, so there is no binary to download by hand.

### Python

`grpcio-tools` covers both messages and the service stub in one install.

```bash
pip install grpcio-tools
mkdir -p gen/python
python -m grpc_tools.protoc -I protos -I google \
  --python_out=gen/python --grpc_python_out=gen/python --pyi_out=gen/python \
  $(find protos -name '*.proto')
```

You get `*_pb2.py` for messages and `*_pb2_grpc.py` for the `PrintServiceStub`.

### Other languages

gRPC supports other languages, and each has a quickstart that covers generating stubs from
`.proto` files: <https://grpc.io/docs/languages/>

Note that protobuf and gRPC codegen are separate steps in most toolchains, as the Java section
above shows. For languages other than Java or Python, see the protobuf
generated-code reference: <https://protobuf.dev/reference/>
