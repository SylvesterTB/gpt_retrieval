# Concept: UserData

## Data management based on group access rights

### NEW: Defining groups for bundles
* Only base-bundles define groups, which are used by volumes to manage access rights:
```yaml
...
type: base
groups:
  - name: addon-grp # this group is used to give access to all addon bundles
    gid: 2000
...
```
* The GID is always specified explicitly and internal operations only use the GID.
* When the base-bundle is started, its groupnames and ids are added to `/etc/group`
  * This only affects group names, the access rights functionality would also be working without named GIDs.

### CHANGE: Using groups for public volumes
* Previously volumes referred to host-paths on the controller and could be accessed by all bundles.
  * We change this behaviour: Volumes must have the `shared` flag set explicitly to `true` to be accessible by addons.
* The definition is therefore extended:
```yaml
type: base
volumes: # see section Volumes; only available in base bundles
  - name: Name of the volume
    hostPath: Source path on the host
    shared: true | false # allow containers and initContainers from dependent bundles to access this volume
    fileProviderAccess: true | false # allow fileProviders from dependent bundles to access this volume
    readOnlyGroups: # groups that are allowed readOnly access to the volume
      - group1
      - group2
    readWriteGroups: # groups that are allowed readWrite access to the volume (takes precedence over readOnly)
      - group1
      - group2
```
* When `shared` is `true`, containers and initContainers of dependent bundles may mount this volume.
* When `fileProviderAccess` is `true`, fileProviders of dependent bundles may mount this volume.
* *ATTENTION:* Due to the usage of ACLs for fileProviders and shared volumes, these options are only allowed for the btrfs filesystem mounted under `/opt` 
* When the base-bundle is installed and the host-path does not exists, all volume hostPaths are created with permissions set to permissions `700` (root-only access)
  * *ATTENTION:* As these created paths are not readable by containers by default groups have to be set even inside the same bundle.
* `readOnlyGroups` and `readWriteGroups` may list group-names. 
  * When a group-name is listed the group will be added to the Access Control List of the directory with either readonly (`rx`) or readwrite (`rwx`) rights.
  * If the group-name is defined in `groups`, the groupId will be used, otherwise the group will be added as is.
* The `readOnly` flag when defining a `container.spec.volume` stays in effect, but changes only the podman mounting, not the actual filesystem permissions.

### NEW: Additional groups for containers and initContainers
* To access volumes that have a specific group set, containers
```yaml
containers: # list of containers in the pod
  - spec:
      name: test-container
      image: k8s.gcr.io/pause:3.3.0
      imageId: <sha256>
      additionalGroups: # NEW: groups to add to the container 
        - group1
        - group2
```
* Effect: the containers are started with the additional groups set add via the `--group-add`
* These groups are necessary to get permissions to access a volume shared from the base-bundle

## Restricted AddOn Bundles
For Addon bundles only a subset of the BaseBundle features should be available for authors to restrict 3rd-Party authors from abusing full host acccess and enforce architectural constraints.

### CHANGE: Restrict bundle defintion
* For AddOn bundles the following aspects of the bundle-description may not be used:
  * `volumes`: May not define arbitrary mounts on host.
  * `groups`: An addon bundle may not specify groups.
* For containers in addon-bundles the following container parameters are not allowed:
  * `gidmap`
  * `uidmap`
  * `realtime`

### NEW: User restrictions for addon-bundles
* As a best practice in container security we require all containers in addon-bundles to use a specified `USER` in the range 2001-2999
  * If the image does not specify a UID or has the user-id set to "root"  we set it to 2001 by default (with `--user` option of podman)
* Addon bundles may only access volumes defined by the base-bundle.

## File Provider
### Requirement
Because of various extension mechanisms used by containers (e.g. UI, proto definitions), we want a concise mechanism for bundles to provide files that are copied to well defined local directories.
These “extension files” have to be removed, when a bundle is uninstalled.

### Solution
* As we do not handle arbitrary user-files outside of images, files may only be provided via images. These images may be created `FROM scratch` as containers are never started, only mounted.
* File-Providers run after intiContainers before starting regular containers.
* File-Providers may me defined top-level in bundle-descriptions as follows:
```yaml
fileProviders: # NEW
  - image: image reference # see containers.spec
    imageId: image id # see containers.spec
    copyOperations:
      - targetVolume: target volume name # volume to write data to
        targetPath: relative path in target volume
        sourcePaths: # full paths of files or directories in image. directories are copied recursively
          - fileA
          - fileB
          - dirA
```

* If paths are non-existing
  * In container (`sourcePaths`): Fail.
  * In Volume (`targetPath`): Create missing paths, with permissions from volume parent
* When a bundle with file providers is uninstalled (even without `purge`), all copied data is removed.
  * This is necessary to avoid inconsistencies e.g. in world-graph where a descriptor may be present, but the containers for the features are not running because of uninstallation.
  * This also implies, that FileProviders may only be used for data that is not changed by applications.
* There is no directory merge (copying contents of a directory without the directory itself), so either complete directories that are copied including their name or individual filenames have to be provided. Especially, there will be no support for a regex or glob syntax.
  * This is necessary to allow for a consistent cleanup. Otherwise we would need to journal the copied files to remove them on uninstall. Especially with nested cases these file-operations are hard to implement (no stdlib function in go and calling `cp` would yield hard to inspect results in error cases).
  * We strongly recommend that all containers that are using such a file-extension mechanism should support subdirectories because then a bundle may provide a complete directory which is completely removed when uninstalled.
* The image for a fileProvider may never be started, but only mounted to avoid security issues. Copying is done between mounted root-fs and host directory.
* On conflicts (file/directory is already existing) we clean-up already copied data and cancel the installation with an error
  * During updates we uninstall the previous bundle first, removing all files provided by the file-provider. So we have consistent updates. 
* Copied files 
  * Keep original timestamps
  * Permissions are inherited from root-directory of the volume

### Constraints
* podman does currently (3.1.2, but no indication that 4.0 fixes anything) not allow the following:
  * `image mount` or `container mount` as user (`unshare` needed first, but not provided via API)
  * `image mount` for readonly containers (which we currently use for root exclusively)
  * Best solution would be root `container create` (no options) with `container mount` to access the files in the image.
* As data may be provided for containers in other bundles, we have the strict requirement of stopping/starting all bundles during install, so the dependent containers may initialize correctly. In the future containers may be implement a file-watching mechanism which is currently not available.
* FileProviders may not be used to store data that should be persistent during uninstall without `purge`, as those fiels are always cleaned up when uninstalling.
* For the start use only one group for each volume in the bundle description and one group for a directory on the filesystem

## Additional Input
For access rights concept see https://dev.azure.com/kuka/RoX%20OS/_git/rox_architecture?path=/container/linux_container_guidelines.md&_a=preview&anchor=rootless-contaners

