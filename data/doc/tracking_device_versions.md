# Tracking device versions

Occasionally a device attached to the controller is replaced.
This new device might have a different version, than the originally attached one.

The installed containers are always linked to a specific device version and podman does not track changes on the underlying linux system.
Therefore we need to manually keep track of the device versions and recreate the containers if required.

The process if split into the following steps:

1. [Tracking](#tracking): Track device version with container labels.
2. [Checking](#checking): Check whether the device version has changed.
3. [Recreating](#recreating): Recreate outdated containers.

## Tracking

To track the device versions, each container is created with a `label` for every required device, which holds the version `device_path:major.minor` (e.g. `/dev/kuka-ls-char-1=1.3`).

This version can also be found on the system via `ls -l /dev/kuka-ls-char-1`, or `stat /dev/kuka-ls-char-1`.

## Checking

The installed containers are checked for outdated devices on every system-boot.
There are two possibilities, that a container is considered outdated:

### Option 1: The container does not have a label for its devices

This is usually the case only for existing containers, which have been installed before this feature was added,
and is required to enable the feature also for existing containers.

### Option 2: The container has an outdated device

Having an outdated device version means, that the version in the containers `label` does not match the version retrieved from the system.

This is the main use case, if some hardware has been exchanged, and the device version has changed.

## Recreating

The recreation of outdated containers (see
[Option 1](#option-1-the-container-does-not-have-a-label-for-its-devices) and
[Option 2](#option-2-the-container-has-an-outdated-device))
is handled on system-boot, before the `installed-bundles.target` and the `update-agent server` is being started.

It needs to be done before the installed-bundles.target is started,
so that we do not loose time, on starting all installed bundles and
in case of an outdated container to be required to shut them down again
before replacing the container and restarting it afterwards for a second time.

There is no recovery implemented in this case, as every step would further slow down the start.
Moreover, in case of an invalid device version after a hardware change, the old system isn't operational anyway,
which also couldn't be solved by rolling back the system.

The update-agent server is also started afterwards, to prevent concurrent interactions or modifications on installed bundles,
before the containers are recreated.

The recreation is done using the create command within the container instead of the reevaluating the bundles files of installed containers,
as the evaluation inside the update-agent might have changed between the original installation of the bundle,
and the recreation of the container,
which could cause to undesired changes.
