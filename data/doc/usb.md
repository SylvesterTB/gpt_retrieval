# Install from USB

## Archives and bundles

* In order to install a bundle from USB, both the bundle file as well as an image archive for the bundle need to be
  present on the USB stick.

* Image archives need to be in docker format, tar'ed and gzip'ed

* Bundle files and image archives need to be on the top-level folder of the USB stick. Sub-directories are not scanned
  for bundle files or image archives.

* The name of the image archive needs to be the same as the bundle base filename, appended with `_images.tar.gz`

  * Example: `my-bundle_1.0.0.bundle` needs to be accompanied by `my-bundle_1.0.0_images.tar.gz`

* Each bundle requires its own archive containing all images required for the bundle install. Additional images within
  an archive will not be loaded.

  * This is necessary in order for the update agent to clean up the image store correctly.

## USB media access

The USB media (partition) is mounted by access base below `/mnt/usb_update`. After access is finished the media will be
unmounted within 1 second. This is done by a `systemd-mount-service`, created by an `udev-rule`. The `udev-rules` file
is installed by this package.

Normally the first partition (`/dev/sdb1`) will be used. When the first partition is labeled `efi`, the `udev-rule` will
NOT create an mount-service. This will be the case for KUKA USB media (rescue-stick).  

To use the `KUKA USB media` for testing purposes, an additional partition labeled `updates` can be created:

* when using an KRC, prepare the machine to be able to install apt packages
* `$ sudo apt install parted`
* attach USB-stick
* `$ sudo parted /dev/sdb print`, choose "Fix" when error occur
* `$ sudo parted /dev/sdb --script -a optimal mkpart updates ext4 2074MiB 3074MiB`
* `$ sudo mkfs.ext4 -F /dev/sdb3 -L update_label`

Currently there is no additional check for a mounted/mountable device. This could be done "manually" by:

* check for the presence of an USB media, e.g. by checking `/dev/sdb*`
* creation of an known (hidden) file `.update_agent` on USB media and check for it OR
* just check for presence of `/mnt/usb_update/lost+found/` in case of an ext partition OR
* remove any USB media and create an (hidden) file `/mnt/usb_update/.usb_not_mounted` --> when mount is working this
  file will be covered and is invisible

## Duplicate bundles

If a bundle with the same id and version exists remotely as well as in the usb location, the bundle from usb will always
be used by the update-agent. This can only be circumvented by disconnecting the USB media device or removing the bundle
from it.
