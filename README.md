# lnn (LiNk + Networks)

**lnn** extends the Unix `ln` command with network capabilities — create persistent links to files on remote machines as if they were local.

The name doubles as the protocol: **LiNk + Networks** — Unix `ln` (link) plus an extra `n` for *network*. The daemon speaks **LNNP**, the *LiNk + Networks Protocol*.

> Work in progress — link operations are in development.

```bash
lnn 192.168.1.34://var/log/app.log ./app.log   # symlink to a remote file
lnn --add-peer 192.168.1.34                   # trust a peer
lnn --peers                                   # list known peers
```

## How it works

`lnn` is the command-line client (C). It is a drop-in replacement for `ln` with network extensions.

For remote operations the CLI does not talk to other machines itself: it sends a request over a local Unix socket to the **[lnnd](https://github.com/d3xxdev/lnnd)** daemon, which handles peer connections, authentication (Ed25519 mutual handshake), the **LNNP** wire protocol and request persistence. The daemon must be running for remote operations to work.

See [docs/architecture.md](docs/architecture.md) for how the CLI talks to the daemon.

## Build

**Requirements:** GCC, Make

```bash
git clone https://github.com/d3xxdev/lnn.git
cd lnn
make
```

## Usage

### Remote links

Remote links have two orthogonal dimensions: **resolution** (symlink = path-based, hard link = inode-based) and **locality** (virtual = data stays on remote peer, local = data copied to this machine).

| | Virtual (no local copy) | Local (copy) |
|---|---|---|
| **Symlink (`-s`)** | `lnn -s peer://path ./link` | `lnn -s --local peer://path ./link` |
| **Hard link** | `lnn --virtual peer://path ./link` | `lnn peer://path ./link` |

Default for `-s` is virtual (no copy); default for hard link is local. Both defaults follow Unix convention. See [docs/link-semantics.md](docs/link-semantics.md).

```bash
lnn -s 192.168.1.34://path/to/file.txt ./link          # symlink, virtual — lnnd serves on access
lnn -s --local 192.168.1.34://path/to/file.txt ./link  # symlink, local copy
lnn 192.168.1.34://path/to/file.txt ./link             # hard link, local copy
lnn --virtual 192.168.1.34://path/to/file.txt ./link   # hard link, virtual — lnnd tracks inode
lnn 192.168.1.34 ./local_link                          # open request — remote chooses the file
```

### Peer management

```bash
lnn --add-peer 192.168.1.34                      # add peer using default trusted template
lnn --add-peer 192.168.1.34 --max                # add peer with max trust (any path)
lnn --add-peer 192.168.1.34 --template mytempl   # add peer using a custom template
lnn --peer 192.168.1.34 --allow /path/to/dir     # grant extra path to an existing peer
lnn --peer 192.168.1.34 --anti-flooding 4hours;3requests  # set rate limit for a peer
lnn --block 192.168.1.34                         # block a peer
lnn --name raspi 192.168.1.34                    # give a peer a friendly name
lnn --peers                                      # list known peers
lnn --peers 192.168.1.34                         # show peer info, paths, and rate limit
```

```bash
lnn --remove-peer 192.168.1.34                   # strip all paths (revert to unknown template)
lnn --remove-peer 192.168.1.34 --template tmpl   # remove only the paths from that template
lnn --remove-peer 192.168.1.34 /path/to/dir      # remove a single path
```

Trusted peers can create links directly (no request needed) inside the paths their template grants them. The template is a snapshot — modifying the template later does not change already-added peers. The permission model — templates, anti-flood rules, path matching — is specified in the [lnnd trust model](https://github.com/mixx86/docs/trust-model.md).

### Alerts

```bash
lnn --alert        # show unread alerts
lnn --alert all    # show full alert history
```

Alerts are generated when a peer exceeds their request rate limit or attempts a link to an unauthorized path.

### Incoming requests

Requests from peers without direct permissions are persisted by the daemon as soon as they arrive, so they are not lost if it restarts. They stay pending until you resolve them.

```bash
lnn --requests                       # list pending requests
lnn --requests all                   # list all requests
lnn --accept 192.168.1.34:id /path   # accept and resolve a request
```

### Local (`ln` compatible)

```bash
lnn file.txt link.txt       # hard link
lnn -s file.txt link.txt    # symbolic link
```

## Documentation

| Document | Contents |
|----------|----------|
| [docs/architecture.md](docs/architecture.md)   | CLI role and how it talks to lnnd |
| [docs/link-semantics.md](docs/link-semantics.md) | Symlink vs hard link, virtual vs local |

Daemon-side documentation (protocol, IPC contract, trust model) lives in the [lnnd repository](https://github.com/mixx86/lnnd).

## Status

| Feature                                      | Status      |
|----------------------------------------------|-------------|
| Remote symlinks                              | In progress |
| Remote hard links                            | In progress |
| Incoming requests (`--requests`, `--accept`) | In progress |
| Peer management (`--add-peer`, ...)          | Planned     |
| Alerts (`--alert`)                           | Planned     |
| Sync mode (`--sync`)                         | Planned     |
| Smart links                                  | Planned     |

Daemon-side status is tracked in the [lnnd README](https://github.com/mixx86/lnnd#status).

## License

Copyright (C) 2026 Gabriel Quintáns Souto

Released under the [GPLv3](LICENSE).
