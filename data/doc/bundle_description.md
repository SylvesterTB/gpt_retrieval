# Bundle-Description

An update bundle is defined by the bundle-description. The bundle description is a [YAML](https://yaml.org/) file
containing all meta-data about the bundle. It references the container images that are provided by a container
repository as well as the arguments used to create and run the containers.

For SemVer versions and ranges we use the capabilities defined by
the [Go-Implementation](https://github.com/Masterminds/semver#checking-version-constraints) that is also used by
Helm-Charts.

Only the parameters referenced in this document may be used when specifying bundles.

## Format

```yaml
# mandatory fields
apiVersion: v1 # (string) version of the bundle-description format
id: robotics-base-bundle # (string) identifier for the bundle
name: iiQKA OS robotics software # (string) name of the bundle
version: 0.1.0 # (string) version of the bundle, in strict SemVer2 format, see Reference 4
type: base # (string) type of the Bundle, possible values: base | addon

# mandatory fields for base bundle / optional for addon bundle
kukaLinuxVersion: '>=0.1.10' # (string) range of compatible Kuka linux versions, interpreted as a SemVer2 range

# optional fields
description: Provides the essential software for iiQKA.OS. # (string) single-sentence description of the bundle
author: Kuka # (string) author of the bundle
license: testLicense # (string) license under which the bundle is being distributed
bundleStartTimeout: 10m # (string) timeout for bundle start, interpreted as duration string consisting of number and unit suffixes ms | s | m | h, default: 10m  
requires: # (list) required bundles
  - id: robotics-base-bundle # (string) id of a required bundle
    version: '>=0.1.1' # (string) range of compatible versions, in strict SemVer2 format, see Reference 4
groups: # (list) users groups, only for base bundle
  - name: robotics-base # (string) group name
    gid: 2001 # (string) group id 
volumes: # (list) defined volumes, only available in base bundles, see section Volumes
  - name: addon_type_libraries # (string) name of the volume
    hostPath: /opt/kuka/common/type_libraries # (string) source path on the host
    shared: true # (bool) allow containers and initContainers from dependent bundles to access this volume
    fileProviderAccess: true # allow fileProviders from dependent (type: addon) bundles to access this volume. Ignored for base bundles.
    readOnlyGroups: # (list) groups that are allowed readOnly access to the volume
      - robotics-base # (string) name of the group as defined in groups
    readWriteGroups: # (list) groups that are allowed readWrite access to the volume (takes precedence over readOnly)
      - robotics-base # (string) name of the group as defined in groups
containers: # (list) containers included in bundle, see section Containers
  # mandatory fields
  - spec: # container specification
      name: state-service # (string) name of the container
      image: kuka/os/platform/system/state-service:0.3.5 # (string) image reference with a semver version separated by ":"
      imageId: df5391c409838f8a660fe1402b0e5d769ab6560d9524d38148dde53a98e7c3de # (string) image ID representing a SHA256 hash

      # optional fields
      user: 2002 # (int) the id of the user to use. Set to 0 for root or a value in the range [2001, 3000] otherwise. This value overwrites the one set in the container image. This is only taken into account when a scope is defined, otherwise it is ignored!
      sdnotify: container # (string) sets the behaviour of sd-notify, possible values: container | conmon | ignore, default: container
      addHost: # (list) additional host mappings (given to podman create --add-host)
        - 'containers.spec.Name:127.0.0.1' # (string) host and ip seperated by ":", default: containers.spec.Name:127.0.0.1
      uidmap: # (list) uid maps for the user name space [only in base bundle]
        - containerId: # (int) container uid
          hostId: # (int) host uid
          size: # (int) amount of consecutive uids to be mapped
      gidmap: # (list) gid maps for the user name space, given to podman create --gidmap [only in base bundle]
        - containerId: # (int) container gid
          hostId: # (int) host gid
          size: # (int) amount of consecutive gids to be mapped
      devices: # (list) Host devices added to container, see References 5 
        - '/dev/kuka-ls-char-0:/dev/kuka-ls-char-0:rw' # (string) host device
      additionalGroups: # (list) groups to add to the container 
        - robotics-base # (string) reference to group by name
      volumes: # (list) volumes used by the container, see section volumes 
        - name: dev_log # (string) name of the volume
          dest: /dev/log # (string) destination path inside the container
          readOnly: false # (bool) specifies if the volume is read-only
          bindPropagation: rprivate # (string) sets bind propagation for podman create --mount, possible values: shared | slave | private | unbindable | rshared | rslave | runbindable | rprivate, default : rprivate
      env: # (map) environment variables to set inside the container.
        KRC_ENVIRONMENT: CABINET # (string):(string) key value pair to be set 
      capAdd: # (list) capabilities to be added to the container
        - CAP_NET_RAW # (string) linux capability
      capDrop: # (list) capabilities to be dropped from the container
        - CAP_NET_RAW # (string) linux capability
      cgroupParent: # (string) definition of cgroup-parent
      cmd: # (list) command to be run inside of container
        - '-c' # (string) part of the command
      entrypoint: # (list) entry points of the container
        - '/bin/bash' # (string) entry point 
      dns: 127.0.0.53 # (string) IPs of DNS servers to be used by the container, default: 127.0.0.53
      network: host # (string) network mode for the container, default: host
      workingDir: /usr/share/kuka/ # (string) working directory inside the container
      realtime: true # (bool) realtime container, see section Realtime [In base-bundles anyway. In add-on bundles when allowed over scope]
      maxMemory: 250m # (string) maximum memory for the container, see section MaxMemory 
      tty: true # (bool) allocate a pseudo-tty and attach to STDIN of container
      restart: on-failure # (string) systemd service option Restart, possible values: no | on-success | on-failure | on-abnormal | on-watchdog | on-abort | always, default: on-failure
      attach: false # (bool) podman attach option, default: false
      unit: # specification of additional systemd dependencies, directly mapped to systemd properties
        requires: # (list) systemd services related with requires
          - 'irq-affinities.service' # (string) name of the service
        wants: # (list) systemd services related with wants
          - 'irq-affinities.service' # (string) name of the service
        bindsTo: # (list) systemd services related with bindsTo
          - 'irq-affinities.service' # (string) name of the service
        after: # (list) systemd services related with after
          - 'irq-affinities.service' # (string) name of the service
        requisite: # (list) systemd services related with requisite
          - 'irq-affinities.service' # (string) name of the service
    requires: # (list) required containers
      - ethercat # (string) name of the container 
    startTimeoutInSeconds: 120 # (int) max amount of time until which the container has to start and send `systemd-notify --ready` signal
    stopTimeoutInSeconds: 120 # (int) max amount of time until which the container has to stop before it will be killed forcefully
    restartSec: 100ms # (string) time to wait before restarting, see References 1
    startLimitBurst: 5 # (int) limit for start tries, see References 2
    startLimitIntervalSec: 10s # (string) limit for start duration, see References 3
initContainers: # (list) containers for initial setup, see section InitContainers
  - spec: # same as for containers except that "requires" is not allowed
    maxWaitTimeInSeconds: 20 # (int) maximum wait time for init container to stop before it will be force deleted 
fileProviders: # (list) containers providing data for copying, see section FileProviders
  - image: kuka/license/robotics-base-bundle:0.1.1 # (string) image reference with a semver version separated by ":"
    imageId: cfc4dd0169642f6e65365d4cf6e4fea4d59577c2b7f2c1fe322d8c3f5fb165a2 # (string) image ID representing a SHA256 hash
    copyOperations: # (list) copy operations to be performed
      - targetVolume: addon_licenses # (string) volume to write data to
        targetPath: "/" # (string) relative path inside target volume, default: "/"
        sourcePaths: # (list) files or directories in image that will be copied, directories are copied recursively
          - /licenses/robotics-base-bundle/robotics-base-bundle.txt # (string) full path to file or directory
scopeDefinitions: # (list) of available scopes as grammar (can only be defined in a base bundle)
  - 'scope-x'
  - 'scope-y'
scope: 'scope-x' # references one of the scopes defined in the base bundle under `scopeDefinitions` (can only be defined in an add-on bundle)
```

**Note:** The update-description has to be signed by the Update-Service when uploaded and the signature has to be
checked before installation by the Update-Agent.

**Note:** Dependent on usage of volumes, user restrictions may apply.
See [blog: volumes-and-rootless-podman](https://blog.christophersmart.com/2021/01/31/volumes-and-rootless-podman/) for details.

**References**:

1. [systemd.service: RestartSec](https://www.freedesktop.org/software/systemd/man/systemd.service.html#RestartSec=)
2. [systemd.unit: StartLimitIntervalSec](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#StartLimitIntervalSec=interval)
3. [semver](https://semver.org/)
4. [podman-create: device-permissions](https://docs.podman.io/en/latest/markdown/podman-create.1.html#device-host-device-container-device-permissions)

## InitContainers

InitContainers are used to implement initialization logic. The most important use case is to populate **Volumes** with
data that is shared between containers.

InitContainers are run in the order specified before starting the regular **Containers**. They have to terminate without
an error in order for the bundle to be installed.

## FileProviders

FileProviders are used for providing data for the base and addon-bundles. The most important use case is to populate **
Volumes** with data that is shared between containers.

Data from FileProviders are copied in the order specified before starting the regular **Containers**. Attention: The
containers specified in the FileProvider section are only used to copy data from and are not started.
Additional information on FileProviders can be found in [userdata.md](userdata.md).

## Containers

Containers describe services that have to be started on the controller. They can access data from other containers
with _volumeMounts_.

## Volumes

- Volumes define Host-Directories that may be mounted by containers.
- All `volumes` have to be defined on the top-level in the `base-bundle`
- With the `shared` and `fileProviderAccess` options, those volumes may be used in bundles dependent on the `base-bundle`
- Use `container.spec.volumes` to map defined volumes to a path inside the container using
  [podman mounts](https://docs.podman.io/en/v3.1.2/markdown/podman-run.1.html#mount-type-type-type-specific-option)
  of `type=bind`. You can also set the `bindPropagation` for the mount.
- When a volumes host path is not existent on the system, it is created automatically.
- Volumes from other service containers should only be mounted read-only to avoid changing important shared files.
  An exception is made for init-containers (see use case below) and file-providers.
- For details see [userdata.md](userdata.md)

## Example Use Case: Language-Packs

Language-Packs are Update-Packages that provide localization data (i.e. translations). Most Service-Containers contain
preliminary translations which should be replaced by the contents of a Language-Pack when installed. To achieve this the
process could be as follows:

- When an Update-Package with Services is installed or updated an InitContainer mounts an "i18n"-Volume, and copies its
  preliminary translations to it, but only when they are not existent.
- When a Language-Pack is installed, an InitContainer mounts the "i18n"-Volume and replaces all translations including
  the preliminary one.
  This makes sure that a translation is always existing and that Language-Packs take precedence over Service-Containers.

## Realtime

Containers inside the base bundle can be specified as realtime containers. If set to true the following options are set
via podman :

- ipc=host
- pid=host
- ulimit msgqueue=-1
- ulimit nofile=768
- ulimit stack=524288:524288

## MaxMemory

The maximum memory usage for a container can be limited using this option. The expected format is "number[unit]", a unit
can be b (bytes), k (kilobytes), m (megabytes), or g (gigabytes)
