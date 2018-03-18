---
layout: post
title: Turbocharging ZFS Data Recovery
published: true
---

## Intro

Anyone who has worked with ZFS for some time knows how resilient it is to data loss. Top-down checksums, self-repairing, device redundancy, metadata redundancy, snapshots and replication are all features that allow users to keep their data safe. However, despite all those great features, sometimes things go wrong. Because of human errors, a bad policy from a customer-managed storage appliance, defective hardware or software bugs, a day comes where a pool fails to import and those dreaded lines are seen:

    cannot import 'datapool': I/O error
        Destroy and re-create the pool from
        a backup source.

If this is a customer pool and a replication of the pool doesn’t exist, what would follow is a painful support session where a ZFS-internals savvy developer will try different tricks in attempt to import the pool and salvage as much user data as possible. A failure to import the pool here means total loss of data.

```
code test
test test
  another test
done
```

Having been been in a similar situation myself, I realised the tools to help recover a broken pool are quite limited. Generally, the recovery process is somewhere along the lines of:

* Use zdb to find latest txg of the pool.

* Try to examine the pool with zdb at an earlier txg. If it works, try to import the pool in read-only mode at the same txg. If it doesn’t work, or crashes the system, go to next step.

* Examine the code or use dtrace to try to figure out which part of the 750+ line *spa_load_impl()* function fails.

* Change a global variable using mdb to bypass a health check in ZFS, or deploy a patched version of ZFS that skips those checks.

* Try a combination of the steps above until the pool imports and some datasets can be mounted.

There are several issues with the process above:

1. There is very little debug information available on what exactly caused a pool to fail to import. To exacerbate this, the kernel part of the import code is monolithic and hard to read.

2. Some pool corruptions can cause the system to panic when importing the pool.

3. zdb is limited when it comes to debugging import issues (e.g. cannot modify global variables).

4. It is impossible to rewind the pool to a point before the device replacement, which may have been initiated because of the very errors we’re trying to fix! This is especially frustrating as those errors are often what prevents us from importing the pool in the first place.

## Cleanup and debugging

After joining Delphix, I had the opportunity to work on improving this workflow, and the fixes have made their way into OpenZFS over the last few months:

[7638 Refactor spa_load_impl into several functions](https://github.com/illumos/illumos-gate/commit/1fd3785ff6601d3e391378c2dcbf4c5f27e1fe32)

[8961 SPA load/import should tell us why it failed](https://github.com/illumos/illumos-gate/commit/3ee8c80c747c4aa3f83351a6920f30c411236e1b)

[7277 zdb should be able to print zfs_dbgmsg's](https://github.com/illumos/illumos-gate/commit/29bdd2f916366ece37c4748bca6b3d61f57a223b)

[7280 Allow changing global libzpool variables in zdb and ztest through command line](https://github.com/illumos/illumos-gate/commit/0e60744c982adecd0a1f146f5637475d07ab1069)

[9075 Improve ZFS pool import/load process and corrupted pool recovery](https://github.com/openzfs/openzfs/commit/619c0123a513678d5824d6b1c4d7e8fad3c63e76)

The [first improvement](https://github.com/illumos/illumos-gate/commit/1fd3785ff6601d3e391378c2dcbf4c5f27e1fe32) was to break down the *spa_load_impl()* code into several functions to make it more readable. You’ll notice many new functions starting with *spa_ld_*.

[Debug messages](https://github.com/illumos/illumos-gate/commit/3ee8c80c747c4aa3f83351a6920f30c411236e1b) were then added to key areas and failure points of the import process. We decided to log those messages into the *zfs_dbgmsg* log as it allows dumping a larger amount of data. In order to help preserve backward compatibility with third party applications, ZFS user commands return the same error messages as before.

The *zfs_dbgmsg* buffer can be displayed using a few different methods:

1. Live updates can be tracked with dtrace:
$ sudo dtrace -qn 'zfs-dbgmsg{printf("%s\n", stringof(arg0))}'

2. Can be dumped in mdb:
$ sudo mdb -ke "::zfs_dbgmsg ! cat"
Note: the pipe to cat is a trick to prevent mdb from wrapping lines at 80 characters

3. Use new -G [feature](https://github.com/illumos/illumos-gate/commit/29bdd2f916366ece37c4748bca6b3d61f57a223b) in zdb:
$ sudo zdb <flags> -G <pool>

This brings us to the improvements in zdb. Besides being able to display the new debug information, zdb has another new feature that brings its capabilities on par with the kernel: the ability to [set global libzpool variables](https://github.com/illumos/illumos-gate/commit/0e60744c982adecd0a1f146f5637475d07ab1069). 

For example, we want to rewind a pool 20 txgs back. This is considered an extreme rewind and zfs doesn’t guarantee that some blocks from 20 txgs back haven’t been overwritten, therefore it does a complete sanity scan of the pool before importing it by default. This can take a significant amount of time if the pool is large. 

Note that in the example below we have previously determined that the latest good txg is 50.

$ zdb -de -p /files -X -t 50 -G datapool

Dataset mos [META], ID 0, …

<We truncate the output of -d as we are only interested in the debug buffer here>

ZFS_DBGMSG(zdb):

spa_import: importing datapool, max_txg=50 (RECOVERY MODE)

spa_load(datapool, config untrusted): LOADING

file vdev '/files/dsk1': best uberblock found for spa datapool. txg 48

file vdev '/files/dsk1': label discarded as txg is too large (69 > 48)

file vdev '/files/dsk1': failed to read label config. Trying again without txg restrictions.

spa_load(datapool, config untrusted): using uberblock with txg=48

spa_load(datapool, config trusted): performing a complete scan of the pool since extreme rewind is on. This may take a very long time.

  (spa_load_verify_data=1, spa_load_verify_metadata=1)

spa_load(datapool, config trusted): LOADED

spa=datapool async request task=32

Here we can see the new debug messages generated by the *spa_load()*. By default zdb won’t display them, unless invoked with the -G flag.

If we want to turn off the complete scan of the pool on rewind, all we’d have to do in the kernel would be to set *spa_load_verify_metadata* to 0 by calling:

$ sudo mdb -kwe "spa_load_verify_metadata/W 0"

To achieve the same results in zdb, we can do the following:

$ zdb -d -ep /files -t 50 -G **-o spa_load_verify_metadata=0** datapool

ZFS_DBGMSG(zdb):

spa_import: importing datapool, max_txg=50 (RECOVERY MODE)

spa_load(datapool, config untrusted): LOADING

file vdev '/files/dsk1': best uberblock found for spa datapool. txg 48

file vdev '/files/dsk1': label discarded as txg is too large (69 > 48)

file vdev '/files/dsk1': failed to read label config. Trying again without txg restrictions.

spa_load(datapool, config untrusted): using uberblock with txg=48

spa_load(datapool, config trusted): LOADED

spa=datapool async request task=32

As you can see, the scan of the pool has been skipped this time.

From zdb’s man page:

       -o var=value ...

           Set the given global libzpool variable to the provided value. The

           value must be an unsigned 32-bit integer. Currently only little-

           endian systems are supported to avoid accidentally setting the high

           32 bits of 64-bit variables.

## Allow import with different vdev configurations

So far we’ve only covered changes that improve debuggability and code readability. We also made some [functional changes](https://github.com/openzfs/openzfs/commit/619c0123a513678d5824d6b1c4d7e8fad3c63e76), the most important of which is that now we do not trust the pool configuration provided from userland at all (note that we also do not trust the configuration that is automatically assembled from vdev labels). Instead, we just use it to retrieve the configuration that was stored on the pool (in the MOS), and then reopen the pool using that configuration.

What we were doing before was to compare the provided configuration with the one stored in the MOS and fail to import the pool if there were differences in the topology of the two vdev configurations. Then we would copy some data from the MOS configuration into the userland configuration.

Whenever we import or open a pool, we’d always do it using a configuration provided from userland. Since the configuration of a pool can change over time, the configuration provided might be out of date (if taken from a cachefile, by default */etc/zfs/zpool.cache*), or might be "in the future" (in the case where we are doing a rewind). Historically, this would either cause kernel panics as we are referencing vdevs that aren’t yet known, or we’d fail to import the pool as we detect a difference in topology between the configuration provided and the configuration stored on the pool. Hence we have introduced a new variable in the spa, *spa_trust_config*, which determines whether or not we trust the current configuration. 

Whenever we open the pool, the initial configuration is not trusted. This means that we do a few checks differently. First, if block pointers have DVAs that point to vdevs that aren’t known at that point in time, we ignore those DVAs (which was the main cause of kernel panics). Secondly, the pool is read-only in order to avoid accidentally repairing blocks that are actually good. Finally, we allow opening the pool even if some top-level vdevs appear to be missing. The sole objective of the pool in this mode is to retrieve the configuration for the txg that we are opening. Once we retrieved it, we change our mode to trusted and reopen the pool using the retrieved configuration.

The advantage of the new method is that the provided configuration only needs to be accurate enough to be able to retrieve metadata from the MOS, which has a triple level of redundancy. Moreover, since we do not need to compare the old config to the new config, it allows us to rewind past a configuration change. A good case for this is when an error on the pool triggered auto-replacement of a device and we want to rewind past the txg where this error was introduced.

Here’s an example:

I created *tank*, a raidz pool with a spare.

	NAME                            STATE     READ WRITE CKSUM

	tank                            ONLINE       0     0     0

	  raidz1-0                      ONLINE       0     0     0

	    /home/pzakharov/dsks/disk1  ONLINE       0     0     0

	    /home/pzakharov/dsks/disk2  ONLINE       0     0     0

	    /home/pzakharov/dsks/disk3  ONLINE       0     0     0

	spares

	  /home/pzakharov/dsks/disk4    AVAIL

I’ve created 3 datasets on it, synced, then backed up disk3 for future use. I used zdb to find out our current txg is 25. disk3 will be our victim.

I created a 4th dataset with data, synced to make sure the data is on the pool, then copied back the backed up disk3 in place of the current disk3 to simulate data loss on disk (note that those "disks" are actually files, although the ZFS logic is the same for both).

When I re-import the pool and do a scrub, a lot of errors are detected on the pool and device replacement is automatically triggered:

  pool: tank

 state: DEGRADED

status: One or more devices has experienced an unrecoverable error.  An

	attempt was made to correct the error.  Applications are unaffected.

action: Determine if the device needs to be replaced, and clear the errors

	using 'zpool clear' or replace the device with 'zpool replace'.

   see: http://illumos.org/msg/ZFS-8000-9P

  scan: resilvered 50.2M in 0h0m with 0 errors on Fri Feb  3 19:54:43 2017

config:

	NAME                              STATE     READ WRITE CKSUM

	tank                              DEGRADED     0     0     0

	  raidz1-0                        DEGRADED     0     0     0

	    /home/pzakharov/dsks/disk1    ONLINE       0     0     0

	    /home/pzakharov/dsks/disk2    ONLINE       0     0     0

	    spare-2                       DEGRADED     0     0     0

	      /home/pzakharov/dsks/disk3  DEGRADED     0     0   269  too many errors

	      /home/pzakharov/dsks/disk4  ONLINE       0     0     0

	spares

	  /home/pzakharov/dsks/disk4      INUSE     currently in use

Now that device replacement has occurred, the configuration of the pool has changed. Without the *spa_load* improvements, if we try to rewind past the point where the error was introduced it will fail due to a mismatch between the pool configuration stored in the MOS and the configuration in the device labels:

$ sudo zpool import -o readonly -d /home/pzakharov/dsks -T 25 tank

cannot import 'tank': one or more devices is currently unavailable

A similar error message with zdb:

$ sudo zdb -ep /home/pzakharov/dsks/ -d -t 25 tank

zdb: can't open 'tank': No such device or address

With the new improvements however, this process happens automatically:

$ sudo zdb -ep /home/pzakharov/dsks/ -G -t 25 tank

…

ZFS_DBGMSG(zdb):

spa_import: importing tank

spa_load(tank, config untrusted): LOADING

file vdev '/home/pzakharov/dsks/disk1': best uberblock found for spa tank. txg 25

file vdev '/home/pzakharov/dsks/disk1': label discarded as txg is too large (63 > 25)

file vdev '/home/pzakharov/dsks/disk1': failed to read label config. Trying again without txg restrictions.

spa_load(tank, config untrusted): using uberblock with txg=25

spare-2 vdev (guid 1678298609174670425): vdev_copy_path: vdev type mismatch: spare != file

spa_load(tank, config untrusted): provided vdev tree:

  vdev 0: root, guid: 9071892073341600451, path: N/A, healthy

    vdev 0: raidz, guid: 12371771463436077262, path: N/A, healthy

      vdev 0: file, guid: 15878061772973117011, path: /home/pzakharov/dsks/disk1, healthy

      vdev 1: file, guid: 4361664636657269715, path: /home/pzakharov/dsks/disk2, healthy

      vdev 2: spare, guid: 1678298609174670425, path: N/A, healthy

        vdev 0: file, guid: 1738272117167295233, path: /home/pzakharov/dsks/disk3, healthy

        vdev 1: file, guid: 6743904684371266774, path: /home/pzakharov/dsks/disk4, healthy

spa_load(tank, config untrusted): MOS vdev tree:

  vdev 0: root, guid: 9071892073341600451, path: N/A, closed

    vdev 0: raidz, guid: 12371771463436077262, path: N/A, closed

      vdev 0: file, guid: 15878061772973117011, path: /home/pzakharov/dsks/disk1, closed

      vdev 1: file, guid: 4361664636657269715, path: /home/pzakharov/dsks/disk2, closed

      vdev 2: file, guid: 1738272117167295233, path: /home/pzakharov/dsks/disk3, closed

spa_load(tank, config untrusted): vdev_copy_path_strict failed, falling back to vdev_copy_path_relaxed

spa_load(tank, config trusted): final vdev tree:

  vdev 0: root, guid: 9071892073341600451, path: N/A, healthy

    vdev 0: raidz, guid: 12371771463436077262, path: N/A, healthy

      vdev 0: file, guid: 15878061772973117011, path: /home/pzakharov/dsks/disk1, healthy

      vdev 1: file, guid: 4361664636657269715, path: /home/pzakharov/dsks/disk2, healthy

      vdev 2: file, guid: 1738272117167295233, path: /home/pzakharov/dsks/disk3, healthy

spa_load(tank, config trusted): LOADED

In the example above we discard the untrusted config almost entirely. The only thing we copy over is the disk paths, as in the case of regular *cXtYdZ* disks (*sdX* on Linux) hardware modifications can cause the paths to change, and thus the paths stored in the pool’s MOS may become obsolete. This step is especially important when importing the pool on a different system.

Following our success with zdb we can try to import the pool:

$ sudo zpool import -d /home/pzakharov/dsks/ -T 25 tank

$ sudo zpool scrub tank

$ sudo zpool status tank

  pool: tank

 state: ONLINE

  scan: scrub repaired 0 in 0h0m with 0 errors on Fri Feb  3 19:58:37 2017

config:

        NAME                            STATE     READ WRITE CKSUM

        tank                            ONLINE       0     0     0

          raidz1-0                      ONLINE       0     0     0

            /home/pzakharov/dsks/disk1  ONLINE       0     0     0

            /home/pzakharov/dsks/disk2  ONLINE       0     0     0

            /home/pzakharov/dsks/disk3  ONLINE       0     0     0

        spares

          /home/pzakharov/dsks/disk4    AVAIL

We got lucky here as none of the old blocks have been overwritten. Note that in a real recovery scenario it would be more cautious to first import the pool in read-only mode. Also we’d probably want to set *spa_load_verify_metadata*=0 in mdb to skip a full pool scan as it will take a very long time on a large pool. Although it is currently not supported, in the future we might enable scrub of a read-only pool to identify data errors.

## Import with missing top level vdevs

The [changes to the pool configuration logic](https://github.com/openzfs/openzfs/commit/619c0123a513678d5824d6b1c4d7e8fad3c63e76) have enabled another great improvement: the ability to import a pool with missing or faulted top-level vdevs. Since some data will almost certainly be missing, a pool with missing top-level vdevs can only be imported read-only, and the failmode is set to "continue" (*failmode=continue* means that when encountering errors the pool will continue running, as opposed to being suspended or panicking).

To enable this feature, we’ve added a new global variable: zfs_max_missing_tvds, which defines how many missing top level vdevs we can tolerate before marking a pool as unopenable. It is set to 0 by default, and should be changed to other values only temporarily, while performing an extreme pool recovery. 

Here as an example we create a pool with two vdevs and write some data to a first dataset; we then add a third vdev and write some data to a second dataset. Finally we physically remove the new vdev (simulating, for instance, a device failure) and try to import the pool using the new feature.

        NAME           STATE     READ WRITE CKSUM

        datapool       ONLINE       0     0     0

          /files/dsk1  ONLINE       0     0     0

          /files/dsk2  ONLINE       0     0     0

          /files/dsk3  ONLINE       0     0     0

After removing *dsk3*:

$ sudo zpool import -d /files -o readonly=on datapool

cannot import 'datapool': no such pool or dataset

        Destroy and re-create the pool from

        a backup source.

$ zdb -dep /files -G datapool

zdb: can't open 'datapool': No such file or directory

ZFS_DBGMSG(zdb):

spa_import: importing datapool

spa_load(datapool, config untrusted): LOADING

file vdev '/files/dsk1': best uberblock found for spa datapool. txg 33

spa_load(datapool, config untrusted): using uberblock with txg=33

spa_load(datapool, config trusted): vdev tree has 1 missing top-level vdevs.

spa_load(datapool, config trusted): current settings allow for maximum 0 missing top-level vdevs at this stage.

spa_load(datapool, config trusted): FAILED: unable to open vdev tree [error=2]

  vdev 0: root, guid: 18429886357526972942, path: N/A, can't open

    vdev 0: file, guid: 16757086639684086562, path: /files/dsk1, healthy

    vdev 1: file, guid: 11488705042820759313, path: /files/dsk2, healthy

    vdev 2: file, guid: 14944961330847529528, path: /files/dsk3, can't open

spa_load(datapool, config trusted): UNLOADING

Trying zdb with the new feature:

$ zdb -dep /files -G -o zfs_max_missing_tvds=1 datapool

Dataset mos [META], ID 0, …

…

ZFS_DBGMSG(zdb):

spa_import: importing datapool

spa_load(datapool, config untrusted): LOADING

file vdev '/files/dsk1': best uberblock found for spa datapool. txg 33

spa_load(datapool, config untrusted): using uberblock with txg=33

spa_load(datapool, config trusted): vdev tree has 1 missing top-level vdevs.

spa_load(datapool, config trusted): current settings allow for maximum 1 missing top-level vdevs at this stage.

  vdev 0: root, guid: 18429886357526972942, path: N/A, degraded

    vdev 0: file, guid: 16757086639684086562, path: /files/dsk1, healthy

    vdev 1: file, guid: 11488705042820759313, path: /files/dsk2, healthy

    vdev 2: file, guid: 14944961330847529528, path: /files/dsk3, can't open

spa_load(datapool, config trusted): forcing failmode to 'continue' as some top level vdevs are missing

spa_load(datapool, config trusted): LOADED

This looks promising, so let’s attempt to import the pool and look at its contents.

$ sudo mdb -kwe "zfs_max_missing_tvds/W 1"

zfs_max_missing_tvds:           0               =       0x1

$ sudo zpool import -d /files -o readonly=on datapool

$ zpool status datapool

  pool: datapool

 state: DEGRADED

status: One or more devices could not be opened.  Sufficient replicas exist for

        the pool to continue functioning in a degraded state.

action: Attach the missing device and online it using 'zpool online'.

   see: http://illumos.org/msg/ZFS-8000-2Q

  scan: none requested

config:

        NAME                    STATE     READ WRITE CKSUM

        datapool                DEGRADED     0     0     0

          /files/dsk1           ONLINE       0     0     0

          /files/dsk2           ONLINE       0     0     0

          14944961330847529528  UNAVAIL      0     0     0  was /files/dsk3

As we can see, the pool was imported and no errors were detected as of yet.

We list the contents of the pool:

$ ls -lh -R /datapool

/datapool:

total 6

drwxrwxrwx   2 root     root          11 Nov  4 17:44 ds1

drwxrwxrwx   2 root     root          11 Nov  4 17:45 ds2

/datapool/ds1:

total 36927

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:44 file1

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:44 file2

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:44 file3

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:44 file4

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:44 file5

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:44 file6

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:44 file7

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:44 file8

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:44 file9

/datapool/ds2:

total 36927

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:45 file1

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:45 file2

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:45 file3

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:45 file4

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:45 file5

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:45 file6

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:45 file7

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:45 file8

-rw-r--r--   1 pzakharov staff       2.0M Nov  4 17:45 file9

This works without issues as file metadata always has 2 copies that are written to different vdevs. Now if we were to look at the contents of those files, we would expect the files in *ds1* to be intact as they were created before we added *dsk3*. Since we created the files in *ds2* after adding *dsk3* some of their blocks could have been written to *dsk3* and will be missing now.

$ find /datapool/ -type f | xargs md5sum

md5sum: /datapool/ds2/file6: No such device or address

md5sum: /datapool/ds2/file1: No such device or address

md5sum: /datapool/ds2/file5: No such device or address

md5sum: /datapool/ds2/file2: No such device or address

md5sum: /datapool/ds2/file8: No such device or address

md5sum: /datapool/ds2/file7: No such device or address

md5sum: /datapool/ds2/file9: No such device or address

md5sum: /datapool/ds2/file3: No such device or address

md5sum: /datapool/ds2/file4: No such device or address

78ea9bd4f294e78c51fa3df1698c4cc7  /datapool/ds1/file6

7170ac4b37b71d33d535db8151217508  /datapool/ds1/file1

0890d868fee6c2c8ed9a96c578ff28d6  /datapool/ds1/file5

c820d5b9133cc01a8630ac26c4a65281  /datapool/ds1/file8

ea5649083969a09a3e31d8a55b04d154  /datapool/ds1/file2

b7adcd615032636662656465b52ba5d2  /datapool/ds1/file7

2235d876f928b0a7dcb3fc3b0ed2aa2a  /datapool/ds1/file3

087e4844ca7cfd884ac63c28394c39d2  /datapool/ds1/file9

a9a7618f3e69a114dc9487ade867d176  /datapool/ds1/file4

As expected, all the files in ds1 are intact, and trying to read the contents of the files in ds2 returns ENXIO (No such device or address). The reason why we cannot recover a single file from ds2 is because ZFS spreads file blocks across every top-level vdev in a round-robin fashion, thus while certain parts of the files are readable, none of those files are fully intact. Also note that we get ENXIO here instead of EIO because of the way ZIO propagates errors; it is somewhat misleading here but would require additional changes in ZFS for little benefit in order to correct it.

Zpool status now properly reports files with permanent errors:

$ sudo zpool status -v datapool

  pool: datapool

 state: DEGRADED

status: One or more devices has experienced an error resulting in data

        corruption.  Applications may be affected.

action: Restore the file in question if possible.  Otherwise restore the

        entire pool from backup.

   see: http://illumos.org/msg/ZFS-8000-8A

  scan: none requested

config:

        NAME                    STATE     READ WRITE CKSUM

        datapool                DEGRADED    19     0     0

          /files/dsk1           ONLINE       0     0     0

          /files/dsk2           ONLINE       0     0     0

          14944961330847529528  UNAVAIL      0     0     0  was /files/dsk3

errors: Permanent errors have been detected in the following files:

        /datapool/ds2/file1

        /datapool/ds2/file2

        /datapool/ds2/file3

        /datapool/ds2/file4

        /datapool/ds2/file5

        /datapool/ds2/file6

        /datapool/ds2/file7

        /datapool/ds2/file8

        /datapool/ds2/file9

At this point, a user would probably want to do a backup of the data to another pool. Replication will only work for totally intact datasets, and standard copying or archiving tools will have to be used for the other datasets (If replication really is needed you can set *zfs_send_corrupt_data* to 1 in mdb, so that it will ignore all missing or corrupted data). Note that had we made some snapshots prior to the addition of dsk3, we would be able to fully recover older versions of files even if they were overwritten (and thus corrupted) later.

This pool recovery feature was designed to allow you to set *zfs_max_missing_tvds* to any positive value, although in practice if the pool misses more than one top-level vdev then it will have problems mounting datasets or even importing. Also, as noted above, this kind of recovery is the most effective when the missing vdev was added recently and when snapshots were made before the vdev was added.

## Recover a device that was accidentally added to another pool

Let’s say we have a pool with two vdevs:

	NAME           STATE     READ WRITE CKSUM

	datapool       ONLINE       0     0     0

	  /files/dsk1  ONLINE       0     0     0

	  /files/dsk2  ONLINE       0     0     0

Now, we want to create another pool, but accidentally used *dsk2* from the first pool. Since we are really unlucky, we also used *-f*. We realized our typo right away, but the label in *dsk2* has already been overwritten, so now we get this message when trying to import datapool:

$ sudo zpool import -d /files datapool

cannot import 'datapool': one or more devices is currently unavailable

When this happens, the first thing to do is to export the new pool to prevent further damage to the data on *dsk2*. We can then use zdb to figure out what exactly is causing the error:

$ zdb -dep /files -G datapool

zdb: can't open 'datapool': No such device or address

ZFS_DBGMSG(zdb):

spa_import: importing datapool

spa_load(datapool, config untrusted): LOADING

file vdev '/files/dsk1': best uberblock found for spa datapool. txg 68

spa_load(datapool, config untrusted): using uberblock with txg=68

file vdev '/files/dsk2': vdev_validate: vdev label pool_guid doesn't match config (8114759366981915206 != 888796509862721908)

spa_load(datapool, config trusted): FAILED: cannot open vdev tree after invalidating some vdevs

  vdev 0: root, guid: 888796509862721908, path: N/A, can't open

    vdev 0: file, guid: 5914869504010138739, path: /files/dsk1, healthy

    vdev 1: file, guid: 14132494099893290853, path: /files/dsk2, can't open

spa_load(datapool, config trusted): UNLOADING

Let’s look at *dsk2*’s first label:

$ sudo zdb -l /files/dsk2 | head -n 27

--------------------------------------------

LABEL 0

--------------------------------------------

    version: 5000

    name: 'newpool'

    state: 1

    txg: 11

    pool_guid: 8114759366981915206

    hostid: 1830391373

    hostname: 'pavel2.dcenter'

    top_guid: 18118652663715475783

    guid: 18118652663715475783

    vdev_children: 2

    vdev_tree:

        type: 'file'

        id: 1

        guid: 18118652663715475783

        path: '/files/dsk2'

        metaslab_array: 43

        metaslab_shift: 24

        ashift: 9

        asize: 414711808

        is_log: 0

        create_txg: 5

    features_for_read:

        com.delphix:hole_birth

        com.delphix:embedded_data

We can see that *vdev_validate()* is doing the right thing in preventing to open /files/dsk2 as it now officially belongs to another pool called *newpool*. If we are lucky enough, we didn’t have time to overwrite much of its original contents besides the labels. A good strategy would be to import datapool while bypassing the *vdev_validate()* checks and let it overwrite the label. We will first check if this strategy is viable by using zdb. We’ve added a new variable, *vdev_validate_skip*, which can be used to bypass vdev validation.

$ zdb -dep /files -G **-o vdev_validate_skip=1** datapool

Dataset mos [META], ID 0, …

ZFS_DBGMSG(zdb):

spa_import: importing datapool

spa_load(datapool, config untrusted): LOADING

file vdev '/files/dsk1': best uberblock found for spa datapool. txg 68

spa_load(datapool, config untrusted): using uberblock with txg=68

spa_load(datapool, config trusted): LOADED

The strategy seems to work so we will proceed with it:

$ sudo mdb -kwe "vdev_validate_skip/W 1"

vdev_validate_skip:             0               =       0x1

$ sudo zpool import -d /files datapool

$ zpool status datapool

…

        NAME           STATE     READ WRITE CKSUM

        datapool       ONLINE       0     0     0

          /files/dsk1  ONLINE       0     0     0

          /files/dsk2  ONLINE       0     0     3

errors: No known data errors

As we imported the pool, it forced an update of the vdev labels. We export the pool, turn vdev validation back on, re-import the pool and do a scrub to assess the extent of the corruption:

$ sudo zpool export datapool

$ sudo mdb -kwe "vdev_validate_skip/W 0"

vdev_validate_skip:             0x1             =       0x0

$ sudo zpool import -d /files datapool

$ sudo zpool scrub datapool

$ sudo zpool status datapool

  pool: datapool

 state: ONLINE

status: One or more devices has experienced an unrecoverable error.  An

        attempt was made to correct the error.  Applications are unaffected.

action: Determine if the device needs to be replaced, and clear the errors

        using 'zpool clear' or replace the device with 'zpool replace'.

   see: http://illumos.org/msg/ZFS-8000-9P

  scan: scrub repaired 1.50K in 0h0m with 0 errors on Thu Nov  3 16:29:22 2016

config:

        NAME           STATE     READ WRITE CKSUM

        datapool       ONLINE       0     0     0

          /files/dsk1  ONLINE       0     0     0

          /files/dsk2  ONLINE       0     0     4

errors: No known data errors

Luckily, it looks like only metadata was affected, and could be repaired using ditto blocks from the good device. This is the ideal case as we can completely recover the pool.

Finally, we do a zpool clear:

$ sudo zpool clear datapool

$ zpool status datapool

  pool: datapool

 state: ONLINE

  scan: scrub repaired 1.50K in 0h0m with 0 errors on Thu Nov  3 16:29:22 2016

config:

        NAME           STATE     READ WRITE CKSUM

        datapool       ONLINE       0     0     0

          /files/dsk1  ONLINE       0     0     0

          /files/dsk2  ONLINE       0     0     0

errors: No known data errors

## Summary of new tunables

Here’s a summary of new tunables that were introduced:

* *zfs_max_missing_tvds* [integer, default = 0]: Allow importing a pool with *zfs_max_missing_tvds* top-level vdevs missing.

* *vdev_validate_skip* [boolean, default = 0]: Do not validate labels stored on the vdevs against the information in the pool configuration. Allows importing pool with vdevs with corrupted labels or vdevs that were accidentally added to another pool.

* *spa_load_print_vdev_tree* [boolean, default = 0]: Always print the whole vdev tree in the *zfs_dbgmsg* log when loading a pool.

Other useful tunables:

* *spa_load_verify_metadata* [boolean, default = 1]: Verify pool consistency during import / load. It is usually recommended to set this to 0 when performing a pool rewind as otherwise the import command can take hours on a large pool.

## When to expect new feature to be available on my platform?

As of March 2018, it has landed on OpenZFS and Illumos but not yet on FreeBSD and Linux, where I’d expect it to be upstreamed in the next few months. The first OS that will get this feature will probably be [OmniOS Community Edition](https://www.omniosce.org/), although I do not have an exact timeline.
