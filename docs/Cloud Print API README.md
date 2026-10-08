# Cloud Print API

Make your first authenticated call to the Zebra Cloud Print API. This covers gRPC and gRPC-web, from
submitting a job through checking status and cancelling. It is not a full endpoint reference.

---

## Overview

Submit print jobs and track them programmatically over **gRPC**, or **gRPC-web** from a browser.
Authentication is an **API Token** sent as a `Bearer` token.

There is one service: `nucleus.client.printer.print.v1.PrintService`.

**You'll need**
- A tenant with access to **Developer → API Tokens** in the Nucleus web UI
- The **serial numbers** of your target devices (Step 4)
- [`grpcurl`](https://github.com/fullstorydev/grpcurl) for the examples, or generated stubs. See the
  proto bundle README for codegen.

---

## Step 1: Create an API Token

In the Nucleus web UI, go to **Developer → API Tokens → Create Token**. Name it and optionally set
an expiry. Token lifecycle is UI-only; there is no API to mint tokens.

**Copy the value immediately.** It is shown once and cannot be retrieved afterward. If you lose it,
revoke it and create a new one.

```
zbr_nuc_<64 hex chars>
```

Store the token like a password, since anyone holding it can call the API as you. Revoked and
expired tokens stop working immediately.

---

## Step 2: Endpoint

```
api.nucleus.zebra.com:443 (TLS)
```

---

## Step 3: Authenticate

Send the token as request metadata on every call:

```
authorization: Bearer zbr_nuc_<your-token>
```

The gateway resolves your tenant and permissions from the token, so you never send tenant IDs or
user headers yourself.

---

## Step 4: Get a target serial number

Jobs address devices by **serial number**, not friendly name, and there is no device-listing RPC.
Read the serials from the Nucleus web UI, which also shows whether each device is connected.

---

## Step 5: Submit a print job

`PrintService.Print` is **server-streaming**, so you get one `PrintResponse` per device as jobs are
accepted.

> [!note] No gRPC reflection on these endpoints
> `grpcurl` needs the proto bundle to resolve the service. Set this once and every example below
> reuses it. `grpcurl` has the `google.protobuf.*` well-known types built in, so only the bundle's
> own `protos/` needs to be on the import path.
>
> macOS/Linux:
> ```bash
> BUNDLE=/path/to/nucleus-apis-v1
> GRPC="-import-path $BUNDLE/protos -proto nucleus/client/printer/print/v1/print_service.proto"
> ```
>
> Windows (PowerShell):
> ```powershell
> $BUNDLE = "C:\path\to\nucleus-apis-v1"
> $GRPC = "-import-path", "$BUNDLE\protos", "-proto", "nucleus/client/printer/print/v1/print_service.proto"
> ```
>
> Not needed with generated stubs.

`PrintRequest` takes two fields:
- `serial_numbers`: one or more targets, at least one
- `content`: a `oneof`, so set exactly one of
  - `data`, raw label bytes such as ZPL, base64-encoded in JSON, max 10 MB
  - `remote_file`, an object of `{ uri, crc32, trust_chain }`

macOS/Linux:
```bash
grpcurl $GRPC \
  -H "authorization: Bearer zbr_nuc_<your-token>" \
  -d '{
        "serial_numbers": ["<serial-number>"],
        "content": {"data": "<base64-encoded-label-bytes>"}
      }' \
  api.nucleus.zebra.com:443 \
  nucleus.client.printer.print.v1.PrintService/Print
```

Windows (PowerShell):
```powershell
grpcurl $GRPC `
  -H "authorization: Bearer zbr_nuc_<your-token>" `
  -d '{"serial_numbers": ["<serial-number>"], "content": {"data": "<base64-encoded-label-bytes>"}}' `
  api.nucleus.zebra.com:443 `
  nucleus.client.printer.print.v1.PrintService/Print
```

Remote file instead of inline bytes:

```bash
  -d '{
        "serial_numbers": ["<serial-number>"],
        "content": {"remote_file": {"uri": "https://example.com/label.zpl"}}
      }'
```

Each `PrintResponse` carries the `serial_number`, either a `job_id` or an `error`, and any
`current_alerts` on that device.

**Track a job.** Pass the `job_id` from `Print` as `command_id`:

macOS/Linux:
```bash
grpcurl $GRPC \
  -H "authorization: Bearer zbr_nuc_<your-token>" \
  -d '{"command_id": "<job-id>"}' \
  api.nucleus.zebra.com:443 \
  nucleus.client.printer.print.v1.PrintService/GetPrintJobStatus
```

Windows (PowerShell):
```powershell
grpcurl $GRPC `
  -H "authorization: Bearer zbr_nuc_<your-token>" `
  -d '{"command_id": "<job-id>"}' `
  api.nucleus.zebra.com:443 `
  nucleus.client.printer.print.v1.PrintService/GetPrintJobStatus
```

That returns a `PrintJob` with `status` (`PENDING`, `IN_PROGRESS`, `COMPLETED`, `FAILED` or
`CANCELLED`), `created_at`, `completed_at`, and `error_details` on failure.

**Cancel a job.** `CancelPrintJob` takes the same `{"command_id": "<job-id>"}` request and reports
`CANCELLED` or `NOT_CANCELLABLE`, for instance when the job has already printed.

macOS/Linux:
```bash
grpcurl $GRPC \
  -H "authorization: Bearer zbr_nuc_<your-token>" \
  -d '{"command_id": "<job-id>"}' \
  api.nucleus.zebra.com:443 \
  nucleus.client.printer.print.v1.PrintService/CancelPrintJob
```

Windows (PowerShell):
```powershell
grpcurl $GRPC `
  -H "authorization: Bearer zbr_nuc_<your-token>" `
  -d '{"command_id": "<job-id>"}' `
  api.nucleus.zebra.com:443 `
  nucleus.client.printer.print.v1.PrintService/CancelPrintJob
```

---

## Browser clients (gRPC-web)

Generate gRPC-web stubs from the `nucleus.client.printer.print.*` protos and point them at
`https://api.nucleus.zebra.com`. The gateway translates to gRPC, and auth is the same `authorization`
metadata.

---

## Errors

| Status | Meaning | What to do |
|---|---|---|
| `UNAUTHENTICATED` | Missing, invalid or expired token | Check the `Bearer` header. The token may have been revoked, so mint a new one |
| `PERMISSION_DENIED` | Token lacks the required permission | Check the token's permissions in the Nucleus web UI |
| `INVALID_ARGUMENT` | Malformed request | Check required fields against the proto |
| `NOT_FOUND` | Serial number or job doesn't exist in your tenant | Verify the id |
| `RESOURCE_EXHAUSTED` | Free-call budget exhausted | Back off, retry after daily reset |
| `UNAVAILABLE` | Transient | Retry with backoff |
