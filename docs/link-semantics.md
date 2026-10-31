# Link semantics

This page explains what the link flags of `lnn` mean. The behavior itself is implemented by [lnnd](https://github.com/mixx86/lnnd).

Remote links have two orthogonal dimensions.

**Resolution** — how lnnd refers to the file on the remote peer:
- **Symlink (`-s`)**: path-based. If the remote file is renamed or moved, the link breaks.
- **Hard link**: inode-based. lnnd tracks the inode on the remote peer; the link follows the file even if renamed or moved.

**Locality** — where the data lives:
- **Virtual** (default for `-s`): no local copy. Data lives on the remote peer; lnnd serves it on access. Always reflects the current state of the remote file.
- **Local** (default for hard link): data is copied to this machine. Works offline; reflects the state at link-creation time unless `--sync` is active.

The four combinations:

| | Virtual (no local copy) | Local (copy) |
|---|---|---|
| **Symlink (`-s`)** | `lnn -s peer://path ./link` | `lnn -s --local peer://path ./link` |
| **Hard link** | `lnn --virtual peer://path ./link` | `lnn peer://path ./link` |

Default behaviors follow Unix convention: symlinks do not duplicate data; hard links produce a local inode.

The resolution dimension (sym vs hard) is meaningful in all four cases: a symlink always refers to the remote path, so a rename on the remote peer breaks it regardless of locality; a hard link always refers to the remote inode, so lnnd follows renames transparently regardless of locality.
