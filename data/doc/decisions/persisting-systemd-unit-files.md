# Persisting Systemd unit files to allow backup

## Context

Systemd configuration file (units files) are added to the file system during set-up. We want those files not only be
accessible to systemd, but also being able to backup them easily.

## Proposal: Possible Solution

#### Possible Solution 1: User-Directory not on OS-Partition (preferred)

* The whole user-directory of the update-agent is preserved during OS-Update
* The user-directory is part of the backup/restore set.
* On Restore: SystemD-Units are automatically restored, but we still have to update the WantedBy for e.g.
  multi-user.target by enabling the service. A nice workaround could be that we store a generic target
  (e.g. containers.target) that is registered to multi-user and we set our bundles to be wantedBy containers.target.
* On OSUpdate: SystemD-Units are untouched, the workaround above could take care of the rest.
* Bonus: No need to tinker with podman user storage locations as described above, because we can use the default user
  locations

#### Possible Solution 2: Copy files from /opt to SystemD-Location

* Create unit files somewhere in `/opt/operation/updateagent`
* Move them to e.g. `~/.config/systemd/user/` to be recognized by systemd.
* On Restore: find out with e.g. a marker-file that we have restored the user partition and delete the old systemd files
  and copy the ones form /opt over.
* On OSUpdate: Find out that we have a new OS-Image (again marker-file created after first run) and copy over all
  SystemD-Units from /opt/operation/updateagent.

#### Possible Solution 3: Link Units from /opt/kuka

* Create unit files somewhere in `/opt/operation/updateagent`
* Use `systemctl link --user` to create symlinks to the originals
* On Restore/OSUpdate: same as solution 2 only with symlinks.

#### Possible Solution 4: Configure SystemD Unit Path

* Add an `/opt` directory to the SystemD UnitPath via `SYSTEMD_UNIT_PATH` environment variable.
* Seems as if it has to be set
  [before systemd is started](https://unix.stackexchange.com/questions/646991/sytemd-how-to-set-the-systemd-unit-path-variable),
  which seems next to impossible for PID1
* On Restore/OSUpdate: Similar to solution 1.

## Decision

As a solution, we opted for a mix of "Solution 3" and "Solution 4", see above and especially the session
**SystemD Units** under [persistence.md](../persistence.md).

## Consequences

Systemd unit files can easily be backup.



