# zfs-snap-manager

This role installs and configures [zfs-snap-manager](https://github.com/khenderick/zfs-snap-manager), a tool to automatically create, replicate and destroy ZFS snapshots.

## Requirements

An `apt`- or `pacman`-based distribution (target and controller as well). On the controlling machine, `git` will be installed.
Obviously, any ZFS filesystems that should be snapshotted must exist.

## Role Variables

| Name                         | Mandatory / Default      | Description                                                                             |
| ---------------------------- | ------------------------ | ------------------------------------------------------------------------------- |
| `global_cache_dir`           | $ansible_env.HOME/.cache | cache directory to clone git repo to |
| `zfs_snap_manager_version`   | `0.1.13`                 | [zfs-snap-manager](https://github.com/khenderick/zfs-snap-manager) version to use                                                                                         |
| `zfs_parent_fs`              |                          | Parent pool or filesystem (optionally prepended to all datasets)                     |
| `zfs_snapshot_defaults`      | `{}`                     | Default snapshot configuration used when there is no explicit configuration for a filesystem, see [snapshot configuration](#snapshot configuration) for content.          |
| `zfs_filesystems`            | `[]`                     | List of filesystems to be snapshotted. Snapshots can be configured using the dict item `snapshots`. For its content see [snapshot configuration](#snapshot configuration) |
| `zvols`                      | `[]`                     | List of zvols to be snapshotted. Snapshots can be configured using the dict item `snapshots`. For its content see [snapshot configuration](#snapshot configuration)                |

### Snapshot configuration

| Name                   | Default     | Description                                                                                                                                                                                                                                                                                            |
| ---------------------- | -           | -------------                                                                                                                                                                                                                                                                                          |
| `mountpoint`           | None        | Points to the location to which the dataset is mounted. Only needed when `time` = `trigger`                                                                                                                                                                                                            |
| `time`                 | `04:00`     | Can be either a timestamp in 24h notation after which a snapshot needs to be taken. It can also be `trigger` indicating that it will take a snapshot as soon as a file with name `.trigger` is found in the dataset's mountpoint. This can be used in case data is for example rsynced to the dataset. |
| `snapshot`             | `True`      | (boolean) Indicates whether a snapshot should be taken or not. It might be possible that only cleaning needs to be executed if this dataset is actually a replication target for another machine.                                                                                                      |
| `replicate_endpoint`   | (omitted)   | Can be left empty if replicating on localhost (e.g. copying snapshots to other pool). Should be omitted if no replication is required.                                                                                                                                                                 |
| `replicate_target`     | (omitted)   | The target to which the snapshots should be send. Should be omitted if no replication is required or a replication_source is specified.                                                                                                                                                                |
| `replicate_source`     | (omitted)   | The source from which to pull the snapshots to receive onto the local dataset. Should be omitted if no replication is required or a replication_target is specified.                                                                                                                                   |
| `compression`          | (omitted)   | Indicates the compression program to pipe remote replicated snapshots through (for use in low-bandwidth setups.) The compression utility should accept standard compression flags (-c for standard output, -d for decompress.)                                                                         |
| `schema`               | `7d3w11m4y` | In case the snapshots should be cleaned, this is the schema the manager will use to clean.                                                                                                                                                                                                             |
| `preexec`              | (omitted)   | A command that will be executed, before snapshot/replication. Should be omitted if nothing should be executed                                                                                                                                                                                          |
| `postexec`             | (omitted)   | A command that will be executed, after snapshot/replication, but before the cleanup. Should be omitted if nothing should be executed                                                                                                                                                                   |

Note that recursive snapshotting is not supported by zfs-snap-manager. 
For more information on configuration and constraints thereof, consider reading the documentation at [zfs-snap-manager](https://github.com/khenderick/zfs-snap-manager).

## Example Playbook

```yml
zfs_parent_fs: rpool
zfs_filesystems:
  - name: test
  - name: zroot
    snapshots:
      time: "14:00"
      schema: 10d5w1m10y
      replicate_endpoint: "ssh -p 2345 my.remote.server.org"
      replicate_target: tank/backups/rootfs
      compression: gzip
  - name: data
    snapshots:
      mountpoint: /mnt/data
      time: trigger
      snapshot: True
      schema: 5d0w0m0y
      preexec: "echo \"starting\" | mail somebody@example.com"
      postexec: "echo \"finished\" | mail somebody@exampe.com"
  - name: zvol
    snapshots:
      time: "22:00"
  - name: backups/data
    snapshots:
      mountpoint: "/mnt/backups/data"
      snapshot: False
      replicate_endpoint: "ssh other.remote.server.org"
      replicate_source: zpool/data
```

### Results 

`rpool/test` will be snapshotted locally daily at `04:00` and snapshots will be cleaned up with the default schema `7d3w11m4y`.

`rpool/zroot` will be snapshotted at `14:00` with schema `10d5w1m10y` and replicated to an example host (to destination dataset `tank/backups/rootfs` using gzip as compression).

`rpool/data` will be snapshotted by placing a `.trigger` file in `/mnt/data`. Only snapshots created in the last 5 days are kept. Before and after snapshotting, an email will be sent.

`rpool/zvol` is being snapshotted with the default schema `7d3w11m4y` every day at `22:00`.

`rpool/backups/data` will not be snapshotted but pulls snapshots from `zpool/data` residing at remote host `other.remote.server.org` to the local dataset of this name. 

## License

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).

## Author Information

 * [Michel Weitbrecht (SlothOfAnarchy)](https://github.com/SlothOfAnarchy) _michel.weitbrecht@stuvus.uni-stuttgart.de_
