# pbsav

`Note:` pbs av is WIP and merely a proof of concept.

Scan your virtual machine Backups on PBS using clamav.

`pbsav` consists of multiple components:

 * It uses nbdkit and the [PBS nbkit plugin](https://github.com/abbbi/cpbsnbd)
   for mapping VM backup disks from an arbitrary PBS datastore via NBD.
 * It uses Docker to run the latest clamav version.
 * It uses `guestmount` to mount the filesystems within the mapped NBD devices.
 * Task spooling and reporting is done via `task-spooler`.
 * Scan jobs can be started using the `pbsav` python script.

it does:

 * Not require root rights on the system running the scans.
 * Does not need any proxmox utilities installed for doing so.

# Setup

 * You need to compile the [PBS nbkit plugin](https://github.com/abbbi/cpbsnbd)
   as outlined in the description.
 * Install required tools (Debian recommended):

 ```
  apt-get install nbdkit task-spooler docker-ce docker-ce-cli python3-venv fuse3 git`
  echo user_allow_other | sudo tee -a /etc/fuse.conf
  git clone https://github.com/abbbi/pbsav
  python3 -m venv pbsavenv
  source pbsavenv/bin/activate
  pip3 install -r requirements.txt
  python3 pbsav -h

  # copy the nbdkit plugin to the directory
 ```

# Selecting backups for scans

As there is no functionality to set tags, the current way for selecting backups
you want to scan is to include an "pbsav" string within the comment of the
backup description.

# Starting the scan jobs

To start the scan jobs you need to pass a few arguments

 * The PBS hostname
 * The datastore to query
 * The password (or set PBS_PASSWORD environment variable)
 * The SSL Fingerprint of the PBS server
 * How many simultaneous scan Jobs you want to run
 * The mail address to send reports to

Example:

```
 export PBS_PASSWORD=password
 python3 pbsav -H 192.168.161.241 \
                -d test \
                --fingerprint "EE:FF:XX"
                -j 5 \
                -m mail@user.domain
```

The command will then query the PBS server for snapshots you want to scan.  All
disks included in the snapshot will be scanned.

```
[2025-03-01 18:18:09] INFO lib pbsav - main:  connecting PBS
[2025-03-01 18:18:09] INFO lib pbsav - main:  Setting up task spooler with limit: [1]
[2025-03-01 18:18:09] INFO lib pbsav - main:  Considering following backups for scan:
[2025-03-01 18:18:09] INFO lib pbsav - get_snapshots:  Ignoring snapshot: [123/2025-02-25T10:35:51Z]: no pbsav comment set
[2025-03-01 18:18:09] INFO lib pbsav - get_snapshots:  Ignoring snapshot: [103/2025-02-25T15:48:38Z]: no pbsav comment set
[2025-03-01 18:18:09] INFO lib pbsav - get_snapshots:  Ignoring snapshot: [103/2025-02-25T11:06:00Z]: no pbsav comment set
[2025-03-01 18:18:09] INFO lib pbsav - get_snapshots:  Ignoring snapshot: [120/2025-02-25T15:38:53Z]: no pbsav comment set
[2025-03-01 18:18:09] INFO lib pbsav - main:
+-------------+----------------------+-----------------+-----------------+
|   Backup ID | Backup Time (UTC)    | Comment         | Disk images     |
+=============+======================+=================+=================+
|         103 | 2025-02-25T10:32:54Z | ci-vm-foo pbsav | drive-scsi0.img |
+-------------+----------------------+-----------------+-----------------+
[2025-03-01 18:18:09] INFO lib pbsav - main:  Job for snapshot [2025-02-25T10:32:54Z] disk [drive-scsi0.img] submitted.
```

# Monitoring the scans

The scanjobs are enqueued using the tasks-pooler system. You can get a list of
running scans and their state using the `tsp` command, like so:

```
$ tsp
ID   State      Output               E-Level  Times(r/u/s)   Command [run=1/1]
20   running    /tmp/ts-out.2wjx0w                           nbdkit ./nbdkit-pbs-plugin.so image=drive-scsi0.img vmid=103 timestamp=2025-02-25T10:32:54Z repo=root@pam@192.168.161.241:test fingerprint=EE:FF:XX -U - --run /home/abi/pbsav/scan $uri
21   queued     (file)                                       nbdkit ./nbdkit-pbs-plugin.so image=drive-scsi0.img vmid=103 timestamp=2025-02-25T10:32:54Z repo=root@pam@192.168.161.241:test fingerprint=EE:FF:XX -U - --run /home/abi/pbsav/scan $uri
16   finished   /tmp/ts-out.ZG3RI3   0        572.28/25.67/9.63 nbdkit ./nbdkit-pbs-plugin.so image=drive-scsi0.img vmid=103 timestamp=2025-02-25T15:48:38Z repo=root@pam@192.168.161.241:test fingerprint=EE:FF:XX -U - --run /home/abi/pbsav/scan $uri
```

If the system running the scan jobs has configured proper sendmail connectivity
(via /usr/sbin/sendmail) reports will be sent via e-mail once an job finished.

The results are copied to the local results folder after finishing, containing
the clamav scan output or any other errors that resulted in failure.

```
find results/
results/
results/failed
results/failed/ts-out.JsAMyD
results/success
results/success/ts-out.8xVjLl
results/success/ts-out.ZG3RI3
results/success/ts-out.WzkiLi
```





