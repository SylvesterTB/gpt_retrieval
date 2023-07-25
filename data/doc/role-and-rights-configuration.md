# User Guide: How to configure file access rights for bundles

The software necessary on the controller consists of bundles (base bundle and a collection of add-on bundles) that
runs on a KUKA distribution of the Linux OS.

These bundles uses the underlying file system to share data.

This guide describes how access rights on the different file system paths can be configured.

## Which components access and share data?

A bundle consist of a collection of containers managed by Podman. Each container provides a specific software
component that shares data (through the local file system of the controller) with other components.

From the perspective of a container, a `volume` is a folder (i.e. a specific path on the controller).

Sharing data between bundles (i.e. between containers) is implement on
the OS (Linux) level based on the known concept of users and groups having access rights to specific folders.

On one hand, specifying and configuring these access rights is done by the base bundle by defining specific groups with
dedicated rights (read-only, read-write). On the other hand using these rights to access the various volumes
(folders on the file system) is configured in both the base bundle and the different add-on bundles.

The base bundle is the central and only component which is allowed to define user groups with specific access right on
a list of paths (volumes).

This is shown in the figure below. The arrow **(1)** denotes how the base bundle defines and configures access to
various paths on the file system (on the group level), and arrow **(2)** shows that add-on bundles are using their
assigned right (through the groups they belong to) to access the different paths (volumes) on the controller.

**Note:**

In the following chapters, the term `file path`, `volume`, `mount` are used to denote the same thing: a specific folder
on the controller (on a Linux OS) that can be shared or not between different bundles/containers.

```
┌─────────────────┬──────────────────────────────────────────────────────┐
│ KUKA Controller │                                                      │
├─────────────────┘                                                      │
│ ┌─────────────┬─────┐     ┌ ─ ─ ─ ─ ─ ┬ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│ │ File System │     │        Podman                                  │ │
│ ├─────────────┘     │     ├ ─ ─ ─ ─ ─ ┘                                │
│ │                   │                                                │ │
│ │ │                 │     │                                            │
│ │ ├── opt           │                                                │ │
│ │ │    │            │     │                                            │
│ │ │    ├── kuka     │         ┌ ─ ─ ─ ─ ─ ─ ─ ┐   ┌ ─ ─ ─ ─ ─ ─ ─    │ │
│ │ │    │        ◀──(2)───┐│        Add-on              Add-on    │     │
│ │ │    └── var  ◀──(1)─┐ │    │    Bundle     │   │    Bundle        │ │
│ │ │                 │  │ ││    ─ ─ ─ ─ ─ ─ ─ ─     ─ ─ ─ ─ ─ ─ ─ ┤     │
│ │ │                 │  │ │    │ ┌────────────┐│   │┌────────────┐    │ │
│ │ └── mnt           │  │ └┼───  │Container#1 │ ... │Container#1 ││     │
│ │                   │  │      │ └────────────┘│   │└────────────┘    │ │
│ │                   │  │  │     ┌────────────┐     ┌────────────┐│     │
│ │                   │  │      │ │Container#n ││   ││Container#n │    │ │
│ │                   │  │  │     └────────────┘     └────────────┘│     │
│ │                   │  │      └ ─ ─ ─ ─ ─ ─ ─ ┘   └ ─ ─ ─ ─ ─ ─ ─    │ │
│ │                   │  │  │                                            │
│ │                   │  │      ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │ │
│ │                   │  │  │               Base Bundle            │     │
│ │                   │  │      ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │ │
│ │                   │  │  │                                      │     │
│ │                   │  └──────┤┌────────────┐     ┌────────────┐     │ │
│ │                   │     │    │ Container#1│ ... │ Container#n│ │     │
│ │                   │         │└────────────┘     └────────────┘     │ │
│ │                   │     │    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘     │
│ │                   │                                                │ │
│ └───────────────────┘     └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
└────────────────────────────────────────────────────────────────────────┘
```
**Figure 1 - Show how bundle (base and add-ons) are organized and how access on the various volumes is done**

## How files are accessed and/or modified?

On the controller there are three dedicated ways of accessing and sharing files:

1. through direct file access of a volume from a container
2. through the use of `FileProviders` (see the [Glossary](#fileprovider) below)
3. through file manipulation (copy, move, overwrite) that a shell script do from an `InitContainer`
   (see the Glossary below)

## Where to configure access rights?

Access rights (through volume and group configuration) are defined in the respective **bundle description** files.

Examples for both Base Bundles and Add-on Bundles can be seen here:

- [Example of a Base Bundle description file - robotics-base-bundle_1.0.0.yaml](https://dev.azure.com/kuka/RoX%20OS/_git/rox_deployment?path=/bundles/robotics-base-bundle/robotics-base-bundle_1.0.0.yaml)
- [Example of a Add-on Bundle description file - gripper-toolbox-bundle_1.1.0.yaml](https://dev.azure.com/kuka/RoX%20OS/_git/rox_deployment?path=/bundles/gripper-toolbox-bundle/gripper-toolbox-bundle_1.1.0.yaml)

## How to define and configure access rights

The sections below describes how to configure access rights based on two perspectives:

- from the perspective of a **base bundle author** who is defining and configuring access rights to specific volumes
- from the perspective of an **add-on bundle author** who wants to access specific volumes

The starting point (an reference) is the base bundle, since this is the only place where groups and volumes are defined.

## What's the workflow?

To sum up, the whole workflow looks as follow:

1. the **Base Bundle Author** defines in the base bundle description file (YAML) **groups** and **volumes**
2. the **Add-on Bundle Author** uses the **groups** and **volumes** (defined by the **Base Bundle Author**) to configure
   access to shared volumes in the add-on bundle description file (YAML).

## How to define access rights on a volume - As a Base Bundle author

As a base bundle author we set which groups and volumes are available. The configuration is done in a corresponding
base bundle description file. In this example we will take the real case example of
`rox_deployment/bundles/robotics-base-bundle/robotics-base-bundle_1.0.0.yaml`.

### Step 1 - Define groups

In the `groups` block (see `(1)` in the snippet below), one or many groups are defined with:

- a `name` `(2)` and a
- group id `gid` `(3)` 


### Step 2 - Define Volume and assign a group

#### Format definition

Volumes are defined following the format below:

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

#### Real world example

In `rox_deployment/bundles/robotics-base-bundle/robotics-base-bundle_1.0.0.yaml` the definition of the volumes is done
in the `volume` block `(4)`, by entering:

- `name` `(4)`: Name of the volume
- `hostPath` `(5)`: Source path on the host
- `shared` `(6)`: allow containers and initContainers from dependent bundles to access this volume
- `fileProviderAccess` `(7)`: allow fileProviders from dependent bundles to access this volume
- `readWriteGroups` and/or `readOnlyGroups` `(8)`: groups that are allowed readWrite access to the volume
  (takes precedence over readOnly)

**Important**

- Notice that setting `shared` to `true` is important here if the corresponding path/volume
  should be accessible from other bundles.
- Setting `fileProviderAccess` to `true` is important here if you want `FileProviders` to access this volume.


**Example file:** `rox_deployment/bundles/robotics-base-bundle/robotics-base-bundle_1.0.0.yaml`
```yaml
apiVersion: v1
id: robotics-base-bundle
name: iiQKA OS robotics software
version: 1.0.0
type: base
kukaLinuxVersion: '>=1.0.11'
description: Provides the essential software for iiQKA.OS.
groups: # (1)
  - name: addoninit # (2)
    gid: 2010 # (3)
  - name: robotics-base
    gid: 2001
  .
  .
  .
  - name: addon-protodescriptors # (4)
    gid: 2004

volumes: # (4)
  - name: ipc_sockets_mapping # (4)
    hostPath: /opt/run # (5)
    shared: false # (6)
    fileProviderAccess: false # (7)
    readWriteGroups: # (8)
      - robotics-base # (9)
  - name: addon_protodescriptors # (10)
    hostPath: /opt/kuka/common/proto/descriptor # (11)
    shared: true # (12)
    fileProviderAccess: true # (13)
    readOnlyGroups: # (14)
      - addon-protodescriptors # (15)
.
.
.
```

The defined groups and volumes can now be either used:

- within the base bundle itself
- or in add-on bundles

Let's now take the role of an **add-on bundle author**.

## How to configure an addon bundle to access specific volumes defined by the base bundle

As an **add-on bundle author** I know I can only access **volumes** defined by the base bundle based on the groups
(also defined in the base bundle).

### Step 1 - Read the base bundle description file

The **first step** here is to **read** the base bundle description file to see which **groups** and **volumes** are
available for use.

In this example we are referring to `rox_deployment/bundles/robotics-base-bundle/robotics-base-bundle_1.0.0.yaml`.

### Step 2 - Adapt your add-on bundle description file

We are going to illustrate the next steps base on two real world examples:

- `rox_deployment/bundles/robot-data/robot-data-lbr3r760-iisy_200V_1.0.0.yaml` and
- `rox_deployment/bundles/gripper-toolbox-bundle/gripper-toolbox-bundle_1.1.0.yaml`

#### Case 1 - Configuring direct access to a volume

Let's say, the add-on bundle author want to configure direct access to a **volume** using **groups**
(both defined in the base bundle).

In this case, the syntax below should be used, where the block `additionalGroups` under either the block `containers` or
`initContainers` references a given `group`.

##### Format definition

The snippet below shows the syntax to use when defining access for the case 1

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

##### Real world example

The real world example we are using here is `rox_deployment/bundles/robot-data/robot-data-lbr3r760-iisy_200V_1.0.0.yaml`.

###### Step 1 - Pick the group

To define a direct access to a volume. Pick the volume you are interested in from the base bundle description file and write down an associated `group` name (a `group`gives you access to a list of volumes defined in the base bundle ).

In our example, we want to access some of the paths/volumes that are available for the `group` `robotics-base`. Have a
look at the base bundle bundle `rox_deployment/bundles/robotics-base-bundle/robotics-base-bundle_1.0.0.yaml` where the
`robotics-base` group give access e.g. to:

- `/opt/kuka/common/rt_system_configuration`
- `/opt/kuka/common/symlink`

which are all **shared** since `shared: true`, and thus accessible to add-on bundles.

In contrast the volume
`/opt/kuka/common/config/threading` defined as `readWriteGroups` in the base bundle is **NOT** accessible to add-on
bundles since `shared: false`. By the way  `/opt/kuka/common/config/threading` is also not accessible to File Providers
since `fileProviderAccess: false`.

###### Step 2 - Assign the group

Add a block `additionalGroups` (`16`) and list the groups you are interested in. In this case `robotics-base` (`17`).

**Example file:** `rox_deployment/bundles/robot-data/robot-data-lbr3r760-iisy_200V_1.0.0.yaml`
```yaml
apiVersion: v1
id: robot-data-lbr3r760-iisy-200V
name: iiQKA OS robot data for lbr3r760-iisy 200V
version: 1.0.0
type: addon
kukaLinuxVersion: '>=1.0.10'
description: Provides robot machine data for lbr3r760-iisy 200V
requires:
  - id: robotics-base-bundle
    version: '>=1.0.0'
initContainers:
  - spec:
      name: motion-init
      image: kuka/runtime/rt-container-lbr3r760_iisy-init:1.0.3
      imageId: c53b4a3126dd390b8eca4ff9ab18484309dfed47e807083a69a7769fbfc5ca77
      additionalGroups: # (16)
        - robotics-base # (17)
.
.
.
```

#### Case 2 - Configuring access to a volume through a `FileProvider`

Here is the case where we want to use a FileProvider to populate a volume with some files defined within a container.

##### Format definition

The snippet below shows the syntax to use when defining access for the case 2 (definition of the access for a File
Provider).

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

##### Real world example

For this case, let's take `rox_deployment/bundles/gripper-toolbox-bundle/gripper-toolbox-bundle_1.1.0.yaml` where a 
`fileProviders` is defined.

###### Step 1 - Pick the right volume

The volume (defined in the base bundle) we are interested in is `addon_protodescriptors`. See: `rox_deployment/bundles/robotics-base-bundle/robotics-base-bundle_1.0.0.yaml`.

It has been defined as follow:

```yaml
.
.
.
  - name: addon_protodescriptors
    hostPath: /opt/kuka/common/proto/descriptor
    shared: true
    fileProviderAccess: true
    readOnlyGroups:
      - addon-protodescriptors
.
.
.
```

and thus is:

- **shared** (`shared: true`)
- is accessible for **File Providers** (`fileProviderAccess: true`)
- but is read-only `readOnlyGroups`

###### Step 2 - Configure the volume for our File Provider

So the configuration in this case needs to set `targetVolume` to the volume we are interested in
`addon_protodescriptors` (`18`).

**Example file:** `rox_deployment/bundles/gripper-toolbox-bundle/gripper-toolbox-bundle_1.1.0.yaml`
```yaml
apiVersion: v1
id: gripper-toolbox-bundle
name: KUKA iiQKA.Gripper Toolbox
version: 1.1.0
type: addon
kukaLinuxVersion: '>=1.0.5'
description: 'The Gripper Toolbox is an extension of the basic system functions...'
author:
license:
requires:
  - id: robotics-base-bundle
    version: '>=1.0.0'
.
.
.
fileProviders:
  - image: kuka/toolbox/gripper/gripper-service:1.1.0
    imageId: e1896f0013fdb86a4413625f8e61b15f7755abb8cd436e3ec631ebea88bd194d
    copyOperations:
      - targetVolume: addon_protodescriptors # (18)
        targetPath: gripper-toolbox-bundle
        sourcePaths:
          - /data/proto/kuka.toolbox.gripper.custom-gripper-config.desc
          - /data/proto/kuka.toolbox.gripper.gripper-service.desc
      - targetVolume: addon_type_libraries
        targetPath: gripper-toolbox-bundle
        sourcePaths:
          - /data/type-library/kuka.toolbox.custom.gripper.womtypes
.
.
.
```

# Technical Details

Here are grouped important technical details regarding the use and configuration of `FileProviders` and `gid` (Group IDs)  

## FileProviders

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

## Definition and use of group ids (gid)

> * The `gid` must be specified explicitly and internal operations only use the `gid`.
> * When the base-bundle is started, its group names and ids are added to `/etc/group`
    >   * This only affects group names, the access rights functionality would also be working without named `gid`s.

**Some notes regarding group definition**

> * *ATTENTION:* Due to the usage of ACLs for fileProviders and shared volumes, these options are only allowed for the
    btrfs filesystem mounted under `/opt`
> * When the base-bundle is installed and the host-path does not exists, all volume hostPaths are created with
    permissions set to permissions `700` (root-only access)
    >  * *ATTENTION:* As these created paths are not readable by containers by default groups have to be set even inside the same bundle.
> * `readOnlyGroups` and `readWriteGroups` may list group-names.
    >   * When a group-name is listed the group will be added to the Access Control List of the directory with either
          readonly (`rx`) or read-write (`rwx`) rights.
>  * If the group-name is defined in `groups`, the groupId will be used, otherwise the group will be added as is.
> * The `readOnly` flag when defining a `container.spec.volume` stays in effect, but changes only the podman mounting,
    not the actual filesystem permissions.

# Additional Resources

- For more details and especially differences between the former implementation and the current one, have a look at the
[concept](./userdata.md).
- For access rights concept see [linux_container_guidelines.md](https://dev.azure.com/kuka/RoX%20OS/_git/rox_architecture?path=/container/linux_container_guidelines.md&_a=preview&anchor=rootless-contaners)

# Glossary

## FileProvider

FileProviders are used for providing data for the base and addon-bundles. The most important use case is to populate
**Volumes** with data that is shared between containers.

Data from FileProviders are copied in the order specified before starting the regular **Containers**. Attention: they
are not started, but only used to copy data from.

See also:

- [File Provider under bundle_description.md](./bundle_description.md)
- [File Provider in the concept document userdata.md](./userdata.md)

## InitContainer

InitContainers are run in the order specified before starting the regular **Containers**. They have to terminate without
an error in order for the bundle to be installed.

InitContainers are used to implement initialization logic. The most important use case is to populate **Volumes** with
data that is shared between containers.

See also: [InitContainers under bundle_description.md](./bundle_description.md)

## Volumes == HostPaths

In the context of the `update-agent`, `volumes` are simple `HostPaths` which we use as `bind-mounts`.

## Mount (bind-mounts)

When you use a bind mount, a file or directory on the host machine is mounted into a container.
The file or directory is referenced by its absolute path on the host machine. 
See also [how_to_bundle.md](how_to_bundle.md).
