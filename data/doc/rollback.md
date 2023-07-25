# Rollback handling

## Methods

There are two different rollback methods in place, one for the OS and one for bundles
(i.e. containers and services).

### Partition switching for OS

The OS is always installed on a secondary partition, if the OS installation succeeds it becomes the boot partition. In
case the OS installation fails or a combined install of OS and bundle fails, the update-agent keeps the previous
partition as the boot partition.

### Snapshots for Bundles

A bundle installation consists of bundle files, container-images, containers and systemd-units. These are all
stored in their respective places in the `/opt` partition (see [persistence.md](./persistence.md)). At the beginning of
an installation/uninstallation/update of bundles or update to the os the update agent will create a snapshot. When a
bundle installation fails, all files are reset by restoring of the previously made snapshot using the
[backup-service](https://dev.azure.com/kuka/RoX%20OS/_git/operation-management_backup-service) of type BUNDLE_DATA

## Triggers

During the installation/uninstallation/update of bundles, or update to the os the update agent always creates a snapshot
and may depending on occurring errors trigger a rollback. A rollback can be caused by:

* __failed installation/uninstallation/update of bundles__: After the operation is performed the update agent cannot
  correctly start all services.
* __failed update of the OS__ : The system is rebooted successfully but not to the correct partition. This implicit
  rollback is done by `swupdate` which is used for OS install.
* __failed bundle installation after OS update__: If the installation of bundles fails after a successful OS install,
  bundles are rolled back using a snapshot created before the OS install. The partition is switched back to the previous
  OS by the update agent. This is followed by a reboot to boot to the system state before the whole installation
  operation.

All installation sequences with the associated states are
visualized [here](./diagrams/combined-update-flow-with-states.drawio.png).

## Failure during boot

The update agent handles errors during the installation of a new os as described above. Additionally, it recognizes
whether it was booted to the old partition or a new one. However the update agent has no way to handle errors that occur
during the boot/reboot itself. At this time, 02.02.2022, there is no system in place to switch back to the old partition
if a boot with the new one fails, instead the new partition will always be booted. This means that successfully
installing an os via the update agent (via swupdate) which then crashes during boot will leave the system in an unusable
state.

This is also visualized [here](./diagrams/combined-update-flow-with-states.drawio.png). The red bubbles represent logic
of the update agent that currently never runs.

## States

In order to give feedback about the status of running operations the update agent keeps track of its own state and makes
it available through a dedicated gRPC method called `SubscribeUpdateAgentStatus`. Once registered, the update agent
sends the current state immediately and then each time the state changes. The message sent contains the current and 
previous state.

There are 4 possible states: __IDLE, STARTING, ROLLING_BACK, INSTALLING__.

### STARTING_UNSPECIFIED

This is the initial state of the update agent used during startup, when it is not yet determined if an installation is
to be continued after a reboot.

### IDLE

By default, the update agent is always __IDLE__ after startup. This only changes when an 
installation/uninstallation/update is triggered. The update agent remains in state __IDLE__ during every other 
operation. After the installation the agent will always return to this state, even if the operation failed.

### INSTALLING

At the beginning of installation/uninstallation/update the state changes to __INSTALLING__. During the installation of
bundles the state can change directly back to __IDLE__ in case of no critical errors or switch to __ROLLING_BACK__ in
case of errors.

### ROLLING_BACK

In case of critical errors during the installation/uninstallation/update of bundles or updates to the OS the update
agent will trigger a rollback to a previously created snapshot. Until the snapshot is restored and all services are
running again the update agent remains in the state __ROLLING_BACK__. Afterwards the state changes back to __IDLE__.

When an installation errors before creating a snapshot or doing an OS install the __ROLLING_BACK__ state is never 
reached, but the install call returns an error.

