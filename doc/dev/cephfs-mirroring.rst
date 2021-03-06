================
CephFS Mirroring
================

.. note:: CephFS mirroring feature is currently under development. Interfaces detailed in
          this document might change during the development cycle.

CephFS will support asynchronous replication of snapshots to a remote CephFS file system via
`cephfs-mirror` tool. Snapshots are synchronized by mirroring snapshot data followed by
creating a snapshot with the same name (for a given directory on the remote file system) as
the snapshot being synchronized.

Requirements
------------

The primary (local) and secondary (remote) Ceph clusters version should be Pacific or above.

Key Idea
--------

For a given snapshot pair in a directory, `cephfs-mirror` daemon uses readdir diff to
identify changes in a directory tree. The diffs are applied to directory in the remote
file system thereby only synchronizing files that have changed between two snapshots.

This feature is tracked here: https://tracker.ceph.com/issues/47034

.. note:: Synchronizing hardlinks is not supported -- hardlinked files get synchronized
          as separate files.

Creating Users
--------------

Start by creating a user (on the primary/local cluster) for the mirror daemon. This user
has restrictive capabilities on the MDS and the OSD::

  $ ceph auth get-or-create client.mirror mon 'allow r' mds 'allow r' osd 'allow rw tag cephfs metadata=*, allow r tag cephfs data=*' mgr 'allow r'

Create a user for each file system peer (on the secondary/remote cluster). This user needs
to have full capabilities on the MDS (to take snapshots) and the OSDs::

  $ ceph fs authorize <fs_name> client.mirror_remote / rwps

This user should be used (as part of peer specification) when adding a peer.

Starting Mirror Daemon
----------------------

Mirror daemon should be spawned using `systemctl(1)` unit files::

  $ systemctl enable cephfs-mirror@mirror
  $ systemctl start cephfs-mirror@mirror

`cephfs-mirror` daemon can be run in foreground using::

  $ cephfs-mirror --id mirror --cluster site-a -f

.. note:: User used here is `mirror` as created in the `Creating Users` section.

Mirroring Design
----------------

CephFS will support asynchronous replication of snapshots to a remote CephFS file system
via `cephfs-mirror` tool. For a given directory, snapshots are synchronized by transferring
snapshot data to the remote file system and creating a snapshot with the same name as the
snapshot being synchronized.

Change Detection
----------------

The first snapshot to get synchronized involves bulk transfer of data to the remotee
file system. Subsequent snapshot synchronization transfers changed/modified files.

Identifying changed files between two snapshots (one of which is already synchronized),
involves (efficiently) walking two (snapshot) trees. Since one of the snapshots is
already synchronized, scanning the two snapshots locally would suffice. For the most
part, identifying changed files between two snapshot does not involve querying the
remote file system. However, there are cases where it is essential to query the remote
file system:

Failed Synchronization
----------------------

Failure to transfer snapshot data (which could be due to a variety of reasons) leaves
the remote file system with incomplete data. Although retrying synchronization most
likely overcomes transient issues, non-recoverable errors would need to be handled.
When a snapshot synchronization hits a non-recoverable error, the snapshot is noted
for failed synchronization and gets skipped. This is done either manually by the
operator or when the number of consecutive retries (post-failure) hits a configured
limit.

Skipping a "failed" snapshot leaves incomplete data on the remote file system. When the
next snapshot is chosen for synchronization, local scanning of (two) snapshots is not
feasible, since certain files (which were part of the "failed" snapshot) may have been
transferred to the remote file system but do not exist in the "next" chosen snapshot.
In such a case, scanning the two snapshots "locally" for changes would ignore the
incomplete data on the remote file system left behind. In such a case identifying
changes would need to compare files between the snapshot and the remote file system.
Alternatively, the remote file system data could be purged followed by bulk transfer of
data from the next chosen snapshot. Subsequent snapshots can then rely on scanning local
snapshots for identifying changes to be transferred.

Deleted Snapshots
-----------------

Snapshots on the primary file system can be deleted even during synchronization.
This needs to be handled by the mirror daemon and poses the same issues as mention
above. Moreover, mirroring can be disabled in midst of snapshots being synchronized
and enabled after deleting snapshots (including the one under synchronization) and
poses the same set of issues.

Snapshot Synchronization Order
------------------------------

Although the order in which snapshots get chosen for synchronization does not matter,
it may be beneficial to pick snapshots based on creation order (using snap-id).

Snapshot Incarnation
--------------------

A snapshot may be deleted and recreated (with the same name) with different contents.
An "old" snapshot could have been synchronized (earlier) and the recreation of the
snapshot could have been done when mirroring was disabled. Using snapshot names to
infer the point-of-continuation would result in the "new" snapshot (incarnation)
never getting picked up for synchronization.

Snapshots on the secondary file system stores the snap-id of the snapshot it was
synchronized from. This metadata is stored in `SnapInfo` structure on the MDS.

Interfaces
----------

`Mirroring` module (manager plugin) provides interfaces for managing directory snapshot
mirroring. Manager interfaces are (mostly) wrappers around monitor commands for managing
file system mirroring and is the recommended control interface. See `Internal Interfaces`
(monitor commands) are detailed below.

.. note:: `Mirroring` module is work in progress and the interface is detailed in the
          `Mirroring Module and Interface` section.

Internal Interfaces
-------------------

Mirroring needs be enabled for a Ceph file system for `cephfs-mirror` daemons to start
mirroring directory snapshots::

  $ ceph fs mirror enable <fsname>

Once mirroring is enabled, mirror peers can be added. `cephfs-mirror` daemon mirrors
directory snapshots to mirror peers. A mirror peer is represented as a client name
and cluster name tuple. To add a mirror peer::

  $ ceph fs mirror peer_add <fsname> <client@cluster> <remote_fsname>

Each peer is assigned a unique identifier which can be fetched via `ceph fs dump` or
`ceph fs get` commands as below::

  $ ceph fs get <fsname>
  ...
  ...
  [peers={uuid=e3739ebf-dbce-460a-bf9c-c66b57697c9a, remote_cluster={client_name=client.site-a, cluster_name=site-a, fs_name=backup}}]

To remove a mirror peer use the following::

  $ ceph fs mirror peer_remove <uuid>

Mirroring can be disabled for a Ceph file system with::

  $ ceph fs mirror disable <fsname>

Mirror status (enabled/disabled) and file system mirror peers are persisted in `FSMap`.
This enables any entity in a Ceph cluster to subscribe to `FSMap` updates and get
notified about changes in mirror status and/or peers. `cephfs-mirror` daemon subscribes
to `FSMap` and gets notified on mirror status and/or peer updates. Peer changes are
handled by starting or stopping mirroring to when a new peer is added or an existing peer
is removed.

Mirroring Module and Interface
------------------------------

Mirroring module provides interface for managing directory snapshot mirroring. The module
is implemented as a Ceph Manager plugin. Mirroring module does not manage spawning (and
terminating) the mirror daemons. Right now the preferred way would be to start/stop
mirror daemons via `systemctl(1)`. Going forward, deploying mirror daemons would be
managed by `cephadm` (Tracker: http://tracker.ceph.com/issues/47261).

The manager module is responsible for assigning directories to mirror daemons for
synchronization. Multiple mirror daemons can be spawned to achieve concurrency in
directory snapshot synchronization. When mirror daemons are spawned (or terminated)
, the mirroring module discovers the modified set of mirror daemons and rebalances
the directory assignment amongst the new set thus providing high-availability.

.. note:: Multiple mirror daemons is currently untested. Only a single mirror daemon
          is recommended.

Mirroring module is disabled by default. To enable mirroring use::

  $ ceph mgr module enable mirroring

Mirroring module provides a family of commands to control mirroring of directory
snapshots. To add or remove directories, mirroring needs to be enabled for a given
file system. To enable mirroring use::

  $ ceph fs snapshot mirror enable <fs>

.. note:: Mirroring module commands use `fs snapshot mirror` prefix as compared to
          the monitor commands which `fs mirror` prefix. Make sure to use module
          commands.

To disable mirroring, use::

  $ ceph fs snapshot mirror disable <fs>

Once mirroring is enabled, add a peer to which directory snapshots are to be mirrored.
Peers follow `<client>@<cluster>` specification and get assigned a unique-id (UUID)
when added. See `Creating Users` section on how to create Ceph users for mirroring.

To add a peer use::

  $ ceph fs snapshot mirror peer_add <fs> <remote_cluster_spec> [<remote_fs_name>]

`<remote_fs_name>` is optional, and default to `<fs>` (on the remote cluster).

.. note:: Only a single peer is supported right now.

To remove a peer use::

  $ ceph fs snapshot mirror peer_remove <fs> <peer_uuid>

.. note:: See `Mirror Daemon Status` section on how to figure out Peer UUID.

To configure a directory for mirroring, use::

  $ ceph fs snapshot mirror add <fs> <path>

To stop a mirroring directory snapshots use::

  $ ceph fs snapshot mirror remove <fs> <path>

Only absolute directory paths are allowed. Also, paths are normalized by the mirroring
module, therfore, `/a/b/../b` is equivalent to `/a/b`.

  $ mkdir -p /d0/d1/d2
  $ ceph fs snapshot mirror add cephfs /d0/d1/d2
  {}
  $ ceph fs snapshot mirror add cephfs /d0/d1/../d1/d2
  Error EEXIST: directory /d0/d1/d2 is already tracked

Once a directory is added for mirroring, its subdirectory or ancestor directories are
disallowed to be added for mirorring::

  $ ceph fs snapshot mirror add cephfs /d0/d1
  Error EINVAL: /d0/d1 is a ancestor of tracked path /d0/d1/d2
  $ ceph fs snapshot mirror add cephfs /d0/d1/d2/d3
  Error EINVAL: /d0/d1/d2/d3 is a subtree of tracked path /d0/d1/d2

Commands to check directory mapping (to mirror daemons) and directory distribution are
detailed in `Mirror Daemon Status` section.

Mirror Daemon Status
--------------------

Mirror daemons get asynchronously notified about changes in file system mirroring status
and/or peer updates. CephFS mirror daemons provide admin socket commands for querying
mirror status. To check available commands for mirror status use::

  $ ceph --admin-daemon /path/to/mirror/daemon/admin/socket help
  {
      ....
      ....
      "fs mirror status cephfs@360": "get filesystem mirror status",
      ....
      ....
  }

Commands with `fs mirror status` prefix provide mirror status for mirror enabled
file systems. Note that `cephfs@360` is of format `filesystem-name@filesystem-id`.
This format is required since mirror daemons get asynchronously notified regarding
file system mirror status (A file system can be deleted and recreated with the same
name).

Right now, the command provides minimal information regarding mirror status::

  $ ceph --admin-daemon /var/run/ceph/cephfs-mirror.asok fs mirror status cephfs@360
  {
    "rados_inst": "192.168.0.5:0/1476644347",
    "peers": {
        "a2dc7784-e7a1-4723-b103-03ee8d8768f8": {
            "remote": {
                "client_name": "client.mirror_remote",
                "cluster_name": "site-a",
                "fs_name": "backup_fs"
            }
        }
    },
    "snap_dirs": {
        "dir_count": 1
    }
  }

`Peers` section in the command output above shows the peer information such as unique
peer-id (UUID) and specification. The peer-id is required to remove an existing peer
as mentioned in the `Mirror Module and Interface` section.

Command with `fs mirror peer status` prefix provide peer synchronization status. This
command is of format `filesystem-name@filesystem-id peer-uuid`::

  $ ceph --admin-daemon /var/run/ceph/cephfs-mirror.asok fs mirror peer status cephfs@360 a2dc7784-e7a1-4723-b103-03ee8d8768f8
  {
    "/d0": {
        "state": "idle",
        "last_synced_snap": {
            "id": 120,
            "name": "snap1",
            "sync_duration": 0.079997898999999997,
            "sync_time_stamp": "274900.558797s"
        },
        "snaps_synced": 2,
        "snaps_deleted": 0,
        "snaps_renamed": 0
    }
  }

Synchronization stats such as `snaps_synced`, `snaps_deleted` and `snaps_renamed` are reset
on daemon restart and/or when a directory is reassigned to another mirror daemon (when
multiple mirror daemons are deployed).

A directory can be in one of the following states::

  - `idle`: The directory is currently not being synchronized
  - `syncing`: The directory is currently being synchronized
  - `failed`: The directory has hit upper limit of consecutive failures

When a directory hits a configured number of consecutive synchronization failures, the
mirror daemon marks it as `failed`. Synchronization for these directories are retried.
By default, the number of consecutive failures before a directory is marked as failed
is controlled by `cephfs_mirror_max_consecutive_failures_per_directory` configuration
option (default: 10) and the retry interval for failed directories is controlled via
`cephfs_mirror_retry_failed_directories_interval` configuration option (default: 60s).

E.g., adding a regular file for synchronization would result in failed status::

  $ ceph fs snapshot mirror add cephfs /f0
  $ ceph --admin-daemon /var/run/ceph/cephfs-mirror.asok fs mirror peer status cephfs@360 a2dc7784-e7a1-4723-b103-03ee8d8768f8
  {
    "/d0": {
        "state": "idle",
        "last_synced_snap": {
            "id": 120,
            "name": "snap1",
            "sync_duration": 0.079997898999999997,
            "sync_time_stamp": "274900.558797s"
        },
        "snaps_synced": 2,
        "snaps_deleted": 0,
        "snaps_renamed": 0
    },
    "/f0": {
        "state": "failed",
        "snaps_synced": 0,
        "snaps_deleted": 0,
        "snaps_renamed": 0
    }
  }

This allows a user to add a non-existent directory for synchronization. The mirror daemon
would mark the directory as failed and retry (less frequently). When the directory comes
to existence, the mirror daemons would unmark the failed state upon successfull snapshot
synchronization.

When mirroring is disabled, the respective `fs mirror status` command for the file system
will not show up in command help.

Mirroring module provides a couple of commands to display directory mapping and distribution
information. To check which mirror daemon a directory has been mapped to use::

  $ ceph fs snapshot mirror dirmap cephfs /d0/d1/d2
  {
    "instance_id": "404148",
    "last_shuffled": 1601284516.10986,
    "state": "mapped"
  }

.. note:: `instance_id` is the RAODS instance-id associated with a mirror daemon. This
          will include the RADOS instance address to help map to a mirror daemon.

Other information such as `state` and `last_shuffled` are interesting when running
multiple mirror daemons.

When no mirror daemons are running the above command shows::

  $ ceph fs snapshot mirror dirmap cephfs /d0/d1/d2
  {
    "reason": "no mirror daemons running",
    "state": "stalled"
  }

Signifying that no mirror daemons are running and mirroring is stalled.

Re-adding Peers
---------------

When re-adding (reassigning) a peer to a file system in another cluster, ensure that
all mirror daemons have stopped synchronization to the peer. This can be checked
via `fs mirror status` admin socket command (the `Peer UUID` should not show up
in the command output). Also, it is recommended to purge synchronized directories
from the peer  before re-adding it to another file system (especially those directories
which might exist in the new primary file system). This is not required if re-adding
a peer to the same primary file system it was earlier synchronized from.

Feature Status
--------------

`cephfs-mirror` daemon is built by default (follows `WITH_CEPHFS` CMake rule). However, the
feature is in development phase.
