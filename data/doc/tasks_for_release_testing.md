# Tasks for release testing

A general description how to test on the controller can be found in [controller_tests.md](./controller_tests.md).
In this document we will focus on the different scenarios for manual tests, that can be executed in order to ensure the
expected behavior.
In some cases we need to prepare the system in order to be able to test the desired behavior. In this case the section
[How to manually prepare the controller for tests](#howto-manually-prepare-the-controller-for-tests)
can become handy.
In test cases where we manually create a snapshot, we could also manually destroy some parts of the system, deleting
container, disabling targets before we run our test-case.

❗ Remark: In case we want to do a real update of the os without rollback we need to keep in mind, that this is possible
only once, as we can only upgrade the version, but not install the same version again.
If we want to test it more often we can do a factory reset, install an older version of the system, and finally proceed
with our test case, which upgrades the os.

## Test-Cases

* [Scenario 1: successful installation of an os image](#scenario-1-successful-installation-of-an-os-image)
* [Scenario 2: successful un-/installation of bundle(s)](#scenario-2-successful-un-installation-of-bundles)
* [Scenario 3: successful combined installation of os & bundles](#scenario-3-successful-combined-installation-of-os--bundles)
* [Scenario 4: failed installation of an os image - Recovery: EnableBundleTargets & DeleteInstallationData](#scenario-4-failed-installation-of-an-os-image---recovery-enablebundletargets--deleteinstallationdata)
  * ❔ Here we have an unexpected behavior when starting without a bundle
* [Scenario 5: failed installation of bundles - Recovery: RestoreAndStart (bundle failed to start)](#scenario-5-failed-installation-of-bundles---recovery-restoreandstart-bundle-failed-to-start)
* [Scenario 6: failed installation of os (& bundles) - Recovery: RestoreAndStart (booted to old partition)](#scenario-6-failed-installation-of-os--bundles---recovery-restoreandstart-booted-to-old-partition)
* [Scenario 7: failed installation of os (& bundles) - Recovery: DeleteInstallationData (booted to old partition)](#scenario-7-failed-installation-of-os--bundles---recovery-deleteinstallationdata-booted-to-old-partition)
  * ❔ Test by pulling the plug during os installation instead of manually preparing the system
* [Scenario 8: failed combined installation of os & bundles - Recovery: RestoreAndRollback (bundle install started)](#scenario-8-failed-combined-installation-of-os--bundles---recovery-restoreandrollback-bundle-install-started)
  * ❗ Remark: Failing rollback script on controller in munich, which has impact on all 'Rollback' tests.
* [Scenario 9: failed combined installation of os & bundles - Recovery: RestoreAndRollback (bundle not found)](#scenario-9-failed-combined-installation-of-os--bundles---recovery-restoreandrollback-bundle-not-found)
* [Scenario 10: failed combined installation of os & bundles - Recovery: RestoreAndRollback (bundle not starting)](#scenario-10-failed-combined-installation-of-os--bundles---recovery-restoreandrollback-bundle-not-starting)
  * ❌ Currently I get a `bundle not found` after manually preparing the system
  * ❌ On a second test I got an error `file exists`, when the update-agent tries to enable the installed bundles (
    after manually preparing the system)
  * ❔ Test by installing os & failing bundle without manually preparing the system
* [Scenario 11: successful combined installation of os and firmware](#scenario-11-successful-combined-installation-of-os-and-firmware)
* [Scenario 12: successful recreation of container with outdated device version](#scenario-12-successful-recreation-of-container-with-outdated-device-version)

---

## Scenario 1: successful installation of an os image

Test:

* Call updateagent with a valid os image

Expected:

* Will successfully install the new os
* System is rebooted

---

## Scenario 2: successful un-/installation of bundle(s)

This scenario consists of two parts, first we uninstall a bundle, and afterwards we install it again in a second
iteration.

These steps can also be exchanged, by first installing a new bundle and afterwards uninstalling the very same bundle.

### Part A (uninstall)

Test:

* Call updateagent with valid (installed) bundle(s)

Expected:

* Will successfully uninstall the bundles
* System is not rebooted

### Part B (install)

Test:

* Call updateagent with valid (not installed) bundle(s)

Expected:

* Will successfully install the bundles
* System is not rebooted

---

## Scenario 3: successful combined installation of os & bundles

This scenario can already cover the cases of installing os & bundles separately.

Test:

* Call updateagent with a valid os image & bundles

Expected:

* Will successfully install the new os and bundles
* System is rebooted

---

## Scenario 4: failed installation of an os image - Recovery: EnableBundleTargets & DeleteInstallationData

Preparation:

* Create an empty os image (if not yet existing), e.g.:

    ```sh
    touch /mnt/usb_update/krc5_iiqka_failing_image_42.0.0_internal.swu
    ```

Test:

* Install the failing os

  ❓ Remark: This will cause the expected error, but the bundle should not be required here. See 'FIXME' below.

    ```sh
    sudo -u updateagent update-agent install -k 42.0.0 -b test-addon-bundle-will-not-start,1.0.1
    ```

    ```log
    kuka@sysa /mnt/usb_update $ sudo -u updateagent update-agent install -k 42.0.0 -b test-addon-bundle-will-not-start,1.0.1
    INFO[0000] Using app config file: /etc/kuka/updateagent/config.yaml
    INFO[0000] Using user config file: /opt/kuka/updateagent/config.yaml
    Error: rpc error: code = InvalidArgument desc = [business/install.(*installAction).Execute] install: [business/install.(*installAction).installKukaLinuxImage] InstallImage: exit status 1, SWUpdate failed
    ```

* FIXME ❓❗ - This will fail on enabling targets, as they haven't been disabled before when no bundle is provided:

    ```sh
    sudo -u updateagent update-agent install -k 42.0.0
    ```

    ```log
    kuka@sysa /mnt/usb_update $  sudo -u updateagent update-agent install -k 42.0.0
    INFO[0000] Using app config file: /etc/kuka/updateagent/config.yaml
    INFO[0000] Using user config file: /opt/kuka/updateagent/config.yaml
    Error: rpc error: code = InvalidArgument desc = [business/install.(*installAction).Execute] install: [business/service.EnableBundleTargets] units.EnableBundleTarget: [adapter/systemd.(*units).EnableBundleTarget] fs.Symlink: symlink /opt/operation/systemd-units/robotics-base-bundle.target /opt/operation/systemd-units/installed-bundles.target.wants/robotics-base-bundle.target: file exists: error while processing bundle: robotics-base-bundle; enable bundle targets triggered by: [business/install.(*installAction).installKukaLinuxImage] InstallImage: exit status 1, SWUpdate failed
    ```

Expeced:

* Error: `InstallImage: exit status 1, SWUpdate failed`
* Will call `EnableBundleTargets` & `DeleteInstallationData` as recovery
* systemd-targets should be enabled again
* No reboot expected

---

## Scenario 5: failed installation of bundles - Recovery: RestoreAndStart (bundle failed to start)

Test:

* Install bundle that won't start properly

    ```sh
    sudo -u updateagent update-agent install -b test-addon-bundle-will-not-start,1.0.1
    ```

Expected:

* Error: `finalizeInstallationAndStart: bundle failed to start`
* Will call `RestoreAndStart` as recovery
* system should be up and running again
* No reboot expected

---

## Scenario 6: failed installation of os (& bundles) - Recovery: RestoreAndStart (booted to old partition)

Preparation:

* Create a snapshot
* Manually create a markerfile with a valid configuration
  * PreviousPartition == currentPartition
  * BackupId != ""

    ```sh
    sudo -u updateagent nano /opt/operation/updateagent/.bundlesToInstall
    ```

    ```yaml
    to-install: # Can also be empty. (From the logical point of view it shouldn't be empty.)
      - id: gripper-toolbox-bundle
        version: 1.1.0
    to-uninstall:
    backup-id: /opt/.snapshots/backupname_bundledata_bruckerbach_1.1.3_coact-toolbox-bundle_1.1.0_robotics-base-bundle_1.0.4_2023-03-06T110315Z # Backup, which was created manually beforehand.
    previous-partition: /dev/sda3 # The currently active partition.
    bundle-install-started: false
    ```

Test:

* Restart updateagent

    ```sh
    sudo -u updateagent systemctl restart update-agent
    ```

Expected:

* Error: `booted to old partition. kuka linux update failed`
* Will call `RestoreAndStart` as recovery
* system should be up and running again
* No reboot expected

---

## Scenario 7: failed installation of os (& bundles) - Recovery: DeleteInstallationData (booted to old partition)

Alternative [not yet tested]:

* It should also be possible without any manual preparation by just pulling the plug, after an os installation started.

Preparation:

* Manually create a markerfile with a valid configuration:
  * PreviousPartition == currentPartition
  * BackupId == ""

```yaml
to-install:
to-uninstall:
backup-id:
previous-partition: /dev/sda3 # The currently active partition.
bundle-install-started: false
```

Test:

* Restart updateagent

Expected:

* Error: `booted to old partition. kuka linux update failed`
* Will call `DeleteInstallationData` as recovery
* Bundle install marker is deleted <br> (check
  with: `sudo -u updateagent ls /opt/operation/updateagent/.bundlesToInstall`)
* No reboot expected

---

## Scenario 8: failed combined installation of os & bundles - Recovery: RestoreAndRollback (bundle install started)

Preparation:

* Create a snapshot
* Manually create a markerfile with a valid configuration
  * PreviousPartition != currentPartition
  * BackupId != ""
  * BundleInstallStarted == true

```yaml
to-install:
to-uninstall: # Can also be empty. (From the logical point of view it shouldn't be empty.)
  - id: gripper-toolbox-bundle
    version: 1.1.0
backup-id: /opt/.snapshots/backupname_bundledata_bruckerbach_1.1.3_coact-toolbox-bundle_1.1.0_gripper-toolbox-bundle_1.1.0_robotics-base-bundle_1.0.4_2023-03-06T110315Z # Backup, which was created manually beforehand.
previous-partition: /dev/sdaX # Some other partition, than the currently active one.
bundle-install-started: true
```

Test:

* Restart updateagent

Expected:

* Error: `booted to old partition. shutdown during bundle install`
* Will call `RestoreAndRollback` as recovery
* Reboot to other partition expected
* system should be up and running again

❗ Remark:

* Currently the controller in the office in munich does not reboot to the other partition, as the
  switch_active_installation script fails during execution.
  In this case the update-agent itself did still successfully do its job and will just reboot to the same partition as a
  fallback. Also in this case the controller should be active and operational afterwards.

    ```log
    Mar 06 12:02:41 sysa /usr/bin/update-agent[15840]: level=info msg="Bundle Install failed after reboot. Switching partition."
    Mar 06 12:02:41 sysa sudo[16083]: updateagent : PWD=/ ; USER=root ; COMMAND=/usr/sbin/switch_active_installation
    Mar 06 12:02:41 sysa sudo[16083]: pam_unix(sudo:session): session opened for user root(uid=0) by (uid=211)
    Mar 06 12:02:41 sysa update-agent[15840]: mount: /dev/sda4 mounted on /tmp/tmp.yh7LXv8TXD.
    Mar 06 12:02:41 sysa update-agent[15840]: mount: /tmp/tmp.yh7LXv8TXD/boot/efi: mount point does not exist.
    Mar 06 12:02:41 sysa sudo[16083]: pam_unix(sudo:session): session closed for user root
    Mar 06 12:02:41 sysa /usr/bin/update-agent[15840]: level=error msg="ROLLBACK SCRIPT EXECUTION FAILED: exit status 32, rollback failed"
    ```

---

## Scenario 9: failed combined installation of os & bundles - Recovery: RestoreAndRollback (bundle not found)

This test will make `getBundlesToInstallAndUninstall` fail when continuing an installation.

Preparation:

* Create a snapshot
* Manually create a markerfile with a valid configuration
  * PreviousPartition != currentPartition
  * BackupId != ""
  * BundleInstallStarted == false
  * ToInstall != [] || ToUninstall != []

```yaml
to-install:
  - id: bundle-does-not-exist # Any bundle, that does not exist
    version: 4.2.0
to-uninstall:
backup-id: /opt/.snapshots/backupname_bundledata_bruckerbach_1.1.3_coact-toolbox-bundle_1.1.0_gripper-toolbox-bundle_1.1.0_robotics-base-bundle_1.0.4_2023-03-06T110315Z # Backup, which was created manually beforehand.
previous-partition: /dev/sdaX # Some other partition, than the currently active one.
bundle-install-started: false
```

Test:

* Restart updateagent

Expected:

* Error: `bundle not found`
* Will call `RestoreAndRollback` as recovery
* Reboot to other partition expected
* system should be up and running again

---

## Scenario 10: failed combined installation of os & bundles - Recovery: RestoreAndRollback (bundle not starting)

This test will fail on the last stage on `performBundleOperations` when continuing an installation after a reboot.

Alternatives [not yet tested]:

* performBundleOperations could also fail when enabling targets.
  This might be achieved, by manually enabling them before, or creating some file/symlink with the same name in the
  directory
* This test case could as well be achieved without manual preparation by using a valid os image & a failing bundle.

Preparation:

* Create a snapshot
* Manually create a markerfile with a valid configuration
  * PreviousPartition != currentPartition
  * BackupId != ""
  * BundleInstallStarted == false
  * ToInstall != [] || ToUninstall != []
* In case of bundle to install it needs to be placed inside `/tmp/bundles` (or in a respective directory, if
  the `bundles-dir` is set within `/etc/kuka/updateagent/config.yaml`) -> currently this still leads to an
  error `bundle not found`❓
* Currently I get the
  error: `performBundleOperations - enabling installed bundle targets [...] file exists: error while processing bundle: robotics-base-bundle`
  ❓

```yaml
to-install:
  - id: test-addon-bundle-will-not-start # A test bundle, that won't start properly.
    version: 1.0.1
to-uninstall:
backup-id: /opt/.snapshots/backupname_bundledata_bruckerbach_1.1.3_coact-toolbox-bundle_1.1.0_gripper-toolbox-bundle_1.1.0_robotics-base-bundle_1.0.4_2023-03-06T110315Z # Backup, which was created manually beforehand.
previous-partition: /dev/sdaX # Some other partition, than the currently active one.
bundle-install-started: false
```

Test:

* Restart updateagent

Expected:

* Will call `RestoreAndRollback` as recovery
* Reboot to other partition expected
* system should be up and running again

---

## Scenario 11: successful combined installation of os and firmware

This test could also of course be executed without manual preparation and a valid os image.
This is just a shortcut, if no matching os image of a higher version is available.

Preparation:

* Disable recovery:

  ```sh
  sudo sed -i "s/recovery-enabled:[[:space:]]*true/recovery-enabled: false/" /etc/kuka/updateagent/config.yaml
  ```

* Restart updateagent (to load new config):

  ```sh
  sudo systemctl restart update-agent
  ```

* Start installation with desired bundle, but failing os (to create all necessary files):

  ```sh
  update-agent install -k 42.0.0 -f 1.3.2
  ```

* Wait until the update fails with: `SWUpdate failed`

* Edit markerfile by manually changing the previous partition to something other than the one, that was written to it:

  ```yaml
  ...
  previous-partition: /dev/sdaX # Some other partition, than the currently active one.
  ...
  ```

* Enable recovery:

  ```sh
  sudo sed -i "s/recovery-enabled:[[:space:]]*false/recovery-enabled: true/" /etc/kuka/updateagent/config.yaml
  ```

Test:

* Restart updateagent

Expected:

* Smartpad firmware will be installed successfully

---

## Scenario 12: successful recreation of container with outdated device version

This test verifies the correct recreation of installed container, which include devices, with outdated versions.

Assure, that you have an installed container with outdated devices.
Containers with devices (included in the robotics-base-bundle) are e.g. `ethercat` and `motion-control`.

A container is considered outdated, if the container does not yet have a label with the device version, or the version in the label does not match the version of the device.

Preparation:

If you already have an outdated container, no preparation is required, otherwise
you could use one of the following options, to recreate one of the `installed` containers manually.

The currently valid device version can be retrieved like described in [Device version](#device-version). Note, the the label will contain the version in decimal, with a `.` as separator.

The create command can be retrieved from the container (e.g. `testContainer=ethercat`) via:

```sh
$ sudo podman inspect $testContainer --format {{.Config.CreateCommand}}

[podman create --name ethercat --device=/dev/null --label /dev/null=1.3 k8s.gcr.io/pause:3.3]
```

* Option 1: skip the `label` for the device completely

  ```sh
  # by using `--replace`, we do not need to delete the container beforehand
  sudo podman create --replace --name ethercat --device=/dev/null k8s.gcr.io/pause:3.3
  ```

* Option 2: change the `label` for the device, to have a different version

  ```sh
  sudo podman create --replace --name ethercat --device=/dev/null --label /dev/null=1.1 k8s.gcr.io/pause:3.3
  ```

Test:

* Reboot the system

Expected:

* The container was recreated. This can be verified via:

  ```sh
  $ sudo podman ps -a | grep -e CREATED -e $testContainer

  CONTAINER ID  IMAGE                 COMMAND  CREATED        STATUS   PORTS   NAMES
  3f93956f426e  k8s.gcr.io/pause:3.3           6 minutes ago  Created          ethercat
  ```

  The creation time in the column `CREATED`, and the `CONTAINER_ID` should have changed.

* The container contains the correct labels. This can be verified via:

  ```sh
  $ sudo podman inspect $testContainer --format {{.Config.Labels}}

  map[/dev/null:1.3]
  ```

* To be sure, that it was recreated correctly you could also compare the complete inspect information `sudo podman inspect $testContainer` before/after the test execution. But be aware, that the ordering of the elements might have changed, which is ok. Here a sorting like described in [sort container inspect info](#sort-container-inspect-info) could become handy.

---

# Howto manually prepare the controller for tests

Table of contents:

* [Active partition](#active-partition)
* [Backup](#backup)
* [Markerfile](#markerfile)
* [OS-Version](#os-version)
* [Device version](#device-version)
* [Failing bundle](#failing-bundle)
* [Sort container inspect info](#sort-container-inspect-info)

## Active partition

Get currently active partition (this is the way how our program is getting the value):

```sh
cat /proc/mounts | awk '$2 == "/" {print $1}'
```

Alternatively we could ask swupdate:

```sh
sudo swupdate --get-root
```

## Backup

Create a new backup on the controller:

```sh
sudo backup-cli create -type bundledata
```

Creating a backup should be done `before` the marker file is created, otherwise it will be part of the backup, and
therefore it won't be deleted properly when restoring the system after a failed update.

## Markerfile

Example structure & values, that can be written to the marker file `.bundlesToInstall`:

```sh
sudo -u updateagent nano /opt/operation/updateagent/.bundlesToInstall
```

```yaml
to-install:
  - id: bundle-a
    version: 1.1.2
  - id: bundle-c
    version: 1.0.1
to-uninstall:
  - id: bundle-a
    version: 1.1.0
  - id: bundle-b
    version: 1.0.0
backup-id: /opt/.snapshots/backupname_bundledata_bruckerbach_1.1.3_bundle-a_1.1.0_bundle-b_1.0.0_robotics-base-bundle_1.0.4_2023-03-06T110315Z
previous-partition: /dev/sda3
bundle-install-started: false
```

## OS-Version

Version of the currently installed os:

```sh
cat /etc/os-release | grep "VERSION="
```

## Device version

The version of a device (e.g. `/dev/null`) can be retrieved via:

```sh
# This will return the device version in hex.
$ stat --format="%t.%T" /dev/null
1.3

# This will return the device version in decimal.
$ ls -l /dev/null | awk '{ print $5$6 }'
1,3
```

## Failing bundle

A bundle that won't start can be found on the controller in munich
on `/mnt/usb_update/test-addon-bundle-will-not-start_1.0.1.bundle`.

It was created according to [controller_tests.md](./controller_tests.md) from the
following `test-addon-bundle-will-not-start.yaml`:

``` yaml
apiVersion: v1
id: test-addon-bundle-will-not-start
name: test-addon-bundle-will-not-start
version: 1.0.1
type: addon
kukaLinuxVersion: '>=0.1.0'
description: Simple single test bundle that will not start properly
bundleStartTimeout: 10s
containers:
  - spec:
      name: test-container
      image: k8s.gcr.io/pause:3.1
      imageId: da86e6ba6ca197bf6bc5e9d900febd906b133eaa4750e6bed647b0fbe50ed43e
    startTimeoutInSeconds: 10
    startLimitBurst: 1
```

It can be installed via: `sudo -u updateagent update-agent install -b test-addon-bundle-will-not-start,1.0.1`, which
will lead to a rollback, as it does not become active.

## Sort container inspect info

The inspect info of a container might rearrange, which does not have any effects on the container,
but makes it cumbersome to compare the info e.g. before and after recreation.

To make it more easy, the inspect info could be sorted before comparison, by utilizing the script `tools/sort-container-inspect.py`.

Example:

```sh
podman inspect ethercat > inspect-ethercat.json
python3 tools/sort-container-inspect.py inspect-ethercat.json
```

Of course this file could be piped to a file as well:

```sh
python3 tools/sort-container-inspect.py inspect-ethercat.json > inspect-ethercat-sorted.json
```
