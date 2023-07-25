# Storage Locations

All remote update specific data that is installation specific has to be stored in the `/opt/kuka` mount.

This is in respect to the use cases

* A/B OS-Update: where all other storage locations are overriden.
* Backup/Restore: where only `/opt/kuka` is snapshoted and restored

| Content                | Host Path              | Description            |
| ---------------------- | ---------------------- | ---------------------- |
| podman user space | `/opt/operation/containers` | Data that is managed by podman and its commands (including images) |
| update agent files | `/opt/operation/updateagent` | Data that is managed by the updateagent |
| container volumes | `/opt/kuka/{container-name}` | Data that is managed and used by a single container |
| shared volumes | `/opt/kuka/common/{shared-folder-name}` | HostPath for shared volumes |
| app configuration | `/etc/kuka/updateagent/config.yaml` | Configuration data used by update agent. |
| user configuration | `/opt/kuka/updateagent/config.yaml` | Configuration data used by update agent managed by user. |

For container and shared volumes refer to [Folder Mappings for Linux Containers](https://dev.azure.com/kuka/RoX%20OS/_git/rox_architecture?path=%2Fstandards_guidelines%2Fnaming_guidelines%2Frox_linux_container_folders.md&_a=preview).

> **Prerequisite:** The update-agent-user has to have read/write-access to `/opt/kuka`.

## Podman User Space

Podmans `graphroot`-Setting specifies were it stores its native data. This has to be set in the podman `storage.conf`
either by changing the default in `/etc/containers` or setting it for the update-agent-user
(as all podman is done root-less) in `~/.config/containers`.

The relevant setting in teh `storage.conf` has to set the `graphroot`, e.g.

```
[storage]
driver="overlay"
graphroot="/opt/operation/root-containers"
.
.
.
```

> **Prerequisite:** The correct `storage.conf` has to be provided by the OS-Image.

## Update-Agent Files

### Bundle Descriptions

All bundles are stored in their zipped format in specific folders according to their life-cycle in the installation
process.

> **Prerequiste**: All bundle-descriptions have to have their ID and SemVer in the filename to avoid collisions.

All folders are relative to `/opt/operation/updateagent`:

* `/bundles/available/`: bundles that have been downloaded from the update server. When the Update-Server is queried for
  updates, all bundles in this folder have to be updated and potentially deleted if now longer available.
* `/bundles/downloaded/`: bundles for which the corresponding images have already been downloaded, but are not installed.
* `/bundles/installed/`: bundles that are installed and containers are running.
* `/bundles/uninstalled/`: bundles that are not running, but which user-data (i.e. their unshared volumes) is not pruned.
   The bundles in this directory are only deleted when the user *prunes* the corresponding user-data.

Bundles are moved by the update-agent according to their installation status.

For available updates the update-agent has to take USB-Mounts into account which may be available under `/mnt/usb_update`
(see [Documentation for USB](usb.md)).

### SystemD Units

The update-agent will generate SystemD-Units which are used to start and stop bundles via common bundle-targets.

See also [systemd.md](systemd.md) for more details.

These files have to be located in a path evaluated by systemd (see [systemd.unit](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)).

Use the command `$ systemd-path systemd-search-system-unit` to see a list for the current system.

In order to backup the unit files, following is done:

* During package setup: Create the folder structure `/usr/local/lib/systemd`, which is recognized by systemd, but not
  used yet
* During package setup: Create the folder structure `/opt/operation/systemd-units`
* During package setup: Create a directory-symlink `/usr/local/lib/systemd/system -> /opt/operation/systemd-units`
* Setup/Update: Create/update unit files in `/opt/operation/systemd-units`
* Backup/Restore: The folder `/opt/operation/systemd-units` must be part of backup/restore, than SystemD-Units will be
  automatically restored.
* On OSUpdate/OSRollback: the package setup has already created the directory-symlink, because located in "rfs"

> "Package setup" is part of OS-Creation process. This means also an package-update will be done by OSUpdate.

Many options have been evaluated, to see which one, have a look at the ADR
[persisting-systemd-unit-files.md](./decisions/persisting-systemd-unit-files.md)

