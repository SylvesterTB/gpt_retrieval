# How to upgrade the KRC5 in the MaibornWolff TH13 office

This guide explains how to upgrade the KRC5 and install a valid base bundle which is necessary to properly
operate the SmartPad.

## Requirements

- scp is installed on your local machine
- access to the KUKA JFrog repository via the KUKA VPN

Due to a complex chain of dependencies between the KUKA Linux OS and the base bundle, it was not possible to install
directly:

- the **latest version 1.1.12** of the **KUKA Linux OS** and
- the corresponding **base bundle** version **1.1.2**

The installation has to be done in many steps:

KUKA Linux OS 1.0.18 -> Office Base bundle 1.0.4 -> KUKA Linux OS 1.1.12 -> Office Base bundle

For more details about compatible versions, have a look at [version_compatibility.md](./../doc/version_compatibility.md)

1. First, install the `krc5_iiqka_branntweinbach_1.0.18_internal.swu` that can be downloaded from the
   [JFrog Artifactory](https://pkg.rd.kuka.com/ui/native/global-staging-generic-virtual/kuka/deploy/base/iiqka/)
   (under `branntweinbach`)
2. Download the `krc5_iiqka_bruckerbach_1.1.12_internal.swu` from JFrog
3. Create an **office** base bundle version **1.0.4** `robotics-base-bundle_1.0.4.bundle` plus the corresponding images
   `robotics-base-bundle_1.0.4_images.tar.gz` using the guide [how_to_create_base_bundle.md](../doc/how_to_create_base_bundle.md)
4. Create an **office** base bundle version **1.1.2** `robotics-base-bundle_1.1.2.bundle` plus the corresponding images
   `robotics-base-bundle_1.0.4_images.tar.gz` using the guide [how_to_create_base_bundle.md](../doc/how_to_create_base_bundle.md)
5. Upload the base bundles 1.0.4 & 1.1.2 (+ images) to the KRC5 folder `/mnt/usb_update`,
   using e.g. `$ scp robotics-base-bundle_1.0.4_images.tar.gz kuka@172.27.5.126:/mnt/usb_update`
6. Make a factory reset of the KRC5 using `krc factory-reset 172.27.5.126` as described under [controller_tests.md](./controller_tests.md)
7. Install the `krc5_iiqka_branntweinbach_1.0.18_internal.swu` via WebUI [KUKA SOFTWARE UPGRADE](http://172.27.5.126:8080/)
8. Execute the `$ sudo prepare-controller.sh` (located on the KRC5 under `/mnt/usb_update`).
   This targets the old config paths before the latest config migration.
9. Install the base bundle `robotics-base-bundle_1.0.4.bundle` using following command on the KRC5
   `$ update-agent install -b robotics-base-bundle,1.0.4`
10. Upgrade the KRC5 KUKA Linux OS to `krc5_iiqka_bruckerbach_1.1.12_internal.swu` via WebUI
    [KUKA SOFTWARE UPGRADE](http://172.27.5.126:8080/)
11. Execute the `$ prepare-controller-new-paths.sh` (located on the KRC5 under `/mnt/usb_update`).
    This targets the new path changes after the latest config migration.
12. Upgrade the base bundle to 1.1.2 `robotics-base-bundle_1.1.2.bundle` using
    `$ update-agent install -b robotics-base-bundle,1.1.2`
13. At the end, make sure that you can access the SmartPad UI
    - on the **SmartPad** or
    - under [https://172.27.5.126](https://172.27.5.126)
