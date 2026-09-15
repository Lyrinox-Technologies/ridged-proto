# ridged-proto (`rdgproto`)

`ridged-proto` is a Go binary framing and payload framework. Import it as
`github.com/Lyrinox-Technologies/ridged-proto/rdgproto`. It provides message
headers, message IDs, length-delimited payloads, payload factories, optional
signing hooks, streaming, and transport-agnostic client/server helpers. It
does not define Lyre authentication, organizations, contracts, or routing.

## Installation

Go `1.18+` is required.

```sh
go get github.com/Lyrinox-Technologies/ridged-proto/rdgproto
go test ./...
```

Message types `250-252` are reserved for internal streaming. Applications
should use values below `250`. The default streaming threshold is 1 MiB and
the default chunk size is 64 KiB. Length-prefixed strings are limited to 1 MiB
on read; byte slices are limited to 1 GiB. Applications must still enforce
their own protocol-specific limits.

## Payloads

Implement `Marshal() ([]byte, error)` and/or `Unmarshal([]byte) error`, then
register a factory for decoding:

```go
type Ping struct { Text string }

func (p *Ping) Marshal() ([]byte, error) {
    b := rdgproto.GetBuffer(); defer rdgproto.PutBuffer(b)
    if err := rdgproto.WriteString(b, p.Text); err != nil { return nil, err }
    out := append([]byte(nil), b.Bytes()...)
    return out, nil
}

func (p *Ping) Unmarshal(data []byte) error {
    p.Text, _ = rdgproto.ReadString(bytes.NewReader(data))
    return nil
}

func init() {
    rdgproto.RegisterPayloadType(1, func() rdgproto.PayloadUnmarshaler {
        return &Ping{}
    })
}
```

Use `NewPayloadRegistry` for an isolated registry; the global registration
helpers are convenient for one application protocol. Unknown message types are
delivered as raw bytes.

## Client and Server

Any `io.Reader`, `io.Writer`, and `io.Closer` combination implements
`rdgproto.Connection`:

```go
client := rdgproto.NewClient(conn, nil)
client.SetHandler(func(msg *rdgproto.Message, payload interface{}) error { return nil })
if err := client.Start(); err != nil { return err }
_, err := client.Send(1, &Ping{Text: "hello"})
return err
```

`Client` also provides `SendWithID`, `SendRaw`, `Wait`, `Errors`, `Done`,
`Close`, and `Protocol`. `NewServer` accepts a `net.Listener` or custom
`Listener`; install a connection handler and call `Start` or `StartAsync`.
The server provides `Broadcast`, `ClientCount`, and `Stop`.

## Serialization Helpers

The package exposes varint and fixed-width `uint32`/`uint64` helpers,
`WriteString`/`ReadString`, `WriteBytes`/`ReadBytes`, and boolean helpers.
`GetBuffer`/`PutBuffer` provide a buffer pool; copy returned bytes before
returning a buffer to the pool.

`Signer` and `Verifier` allow applications to provide signing and verification.
Authentication, authorization, encryption, replay policy, and transport
selection remain application responsibilities.

## Development

```sh
go test ./...
go test -race ./...
go test ./benchmark/...
```

The `benchmark/` directory contains local benchmarks. Keep custom payloads
bounded and reject malformed lengths before allocating large buffers.
