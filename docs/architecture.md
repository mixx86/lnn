# lnn architecture

`lnn` is a thin command-line client. It parses the user's command and, for remote operations, delegates the work to the local **[lnnd](https://github.com/d3xxdev/lnnd)** daemon. The CLI never opens connections to other machines.

```
 User
  │
  ▼
┌────────┐   Unix socket (JSON)   ┌────────┐   TCP / LNNP   ┌──────────────┐
│  lnn   │ ─────────────────────► │  lnnd  │ ─────────────► │ Remote peers │
│ (CLI)  │ ◄───────────────────── │        │                └──────────────┘
└────────┘      JSON response     └────────┘
```

Everything on the right of the socket — peer sessions, authentication, the wire protocol, request persistence, scheduling — is documented in the lnnd repository:

- [Architecture](https://github.com/d3xxdev/lnnd/blob/main/docs/architecture.md)
- [LNNP protocol](https://github.com/d3xxdev/lnnd/blob/main/docs/protocol.md)
- [Trust model](https://github.com/d3xxdev/lnnd/blob/main/docs/trust-model.md)

## Talking to the daemon

The daemon's IPC contract is specified in [lnnd/docs/ipc.md](https://github.com/d3xxdev/lnnd/blob/main/docs/ipc.md). From the client's side, every command that needs the daemon is one short exchange:

1. Connect to the daemon's Unix socket (the path configured as `LNND_IPC_SOCKET` in lnnd).
2. Write **one** JSON request.
3. **Half-close the write side** of the connection (`shutdown(fd, SHUT_WR)`). The daemon reads until EOF, so it will not process the request until it sees this.
4. Read the JSON response until EOF.

Request:

```json
{ "op": "link", "type": "symreq", "src": "192.168.1.34://path/to/file", "dst": "./local_link" }
```

| Field  | Meaning |
|--------|---------|
| `op`   | Operation. Currently `link`. |
| `type` | `symreq` — ask a peer to create a symlink. `hard` is not implemented by the daemon yet. |
| `src`  | `<ip>://<path>`. The IP must be valid; the path is cleaned and made absolute by the daemon. |
| `dst`  | Destination path (optional). |

Response:

```json
{ "ok": true }
{ "ok": false, "err": "unsupported transport: quic" }
```

On `ok: false` the CLI should report `err` to the user and exit with a non-zero status. `ok: true` means the daemon handed the request to the peer, not that the peer has accepted it — acceptance happens later, on the peer's side.

## Source format

Remote paths are written `<ip>://<path>`, for example `192.168.1.34://var/log/app.log`. The IP part is validated by the daemon; a source without `://`, or with an invalid IP, is rejected.

## Link flags

The meaning of `-s`, `--local` and `--virtual` is described in [link-semantics.md](link-semantics.md).
