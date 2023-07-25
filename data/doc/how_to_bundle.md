# How to create a bundle file

A bundle file is nothing more than a zipped tar archive, that contains a bundle description yaml, a signature file,
maybe an EULA file and whatever else you want to have in there. The update agent provides functionality to create such a
file from a given YAML.

## Initial thoughts

Initially there are three things you have to think about.

1. Containers
2. Init Containers
3. Volumes

### 1. Containers

These are [OCI containers](https://opencontainers.org/) which will be running permanently on the controller. They will
be enabled as systemd services and as such will start with every boot. See below for more info.

### 2. Init Containers

Init containers **run exactly once**. They run during installation, before the other containers are even touched. They
also run during an update. They run sequentially, which means each of them is started and when it is done, the next init
container is started. Each init container has a timeout. This timeout is important, because if the container is not done
when the timeout is over, the installation or update fails and everything is reverted. The requires field is ignored
because no other container of that bundle is running parallel to an init container.

### 3. Volumes

Volumes in the sense of the update agent are **podman bind mounts**, which means they have a host directory 
(source path) and a container directory (destination path). For convenience and to prevent spelling errors they also 
have a name. All volumes to be used in one of the containers must be declared at root level in the YAML. They are 
declared by the host path and a name.

See also [role-and-rights-configuration.md](role-and-rights-configuration.md)).

```yaml
volumes:
  - name: models
    hostPath: /opt/kuka/common/models
```

If the directories declared here don't exist on the host machine, they will be created by the update-agent recursively,
so in this example `/opt/kuka/common` would also be created. Also the update-agent makes sure, that the mounted
directories do have permissions 0775, but recursively created parent directories are effected by umask.

Then any container can bind to that volume by mentioning its name and the destination path.

```yaml
containers:
  - spec:
      name: test-container
      image: k8s.gcr.io/pause
      volumes:
        - name: models
          dest: /models
          readOnly: true
```

Please keep in mind the guidelines for storage locations: [persistence.md](persistence.md)

## Create bundle description YAML

First of all a description of all the parameters of a bundle description yaml file can be found in
the [bundle_description.md](bundle_description.md) file. This tutorial is about what all the parameters actually do.

This is a simple, working bundle description file:

```yaml
apiVersion: v1
id: test-bundle-id
name: test-bundle
version: 1.0.0
type: base
kukaLinuxVersion: '>=0.1.0'
description: Simple single container test description
containers:
  - spec:
      name: my-container-name
      image: my-image:latest
initContainers:
  - spec:
      name: my-init-container-name
      image: my-init-image:latest
```

The bundle needs a unique id and a version. Those two are the actual identifier. A bundle can be a **base** or an 
**addon** bundle (**there is only one base bundle**). Also an OS version compatibility range is mandatory. Then there is
a description and finally the containers.

### Container

Because the containers are the heart of any bundle, here's a little more detail on them. If containers are described
like in the simple example above they are internally created like this:

```
podman create --name my-container-name my-image:latest
```

Then they are started by systemd with the command:

```
podman start my-container-name
```

The container dependencies from the requires field end up in the required field of the systemd service. All other
optional parameters you find in the [bundle_description.md](bundle_description.md) are setting the corresponding podman
flags during creation of the container.

Here is a link to the podman create
docs: [https://docs.podman.io/en/latest/markdown/podman-create.1.html](https://docs.podman.io/en/latest/markdown/podman-create.1.html)

## Validate the bundle description yaml

First you should validate the bundle. It will tell you what is going on.

```shell
$ update-agent bundle validate bundle-description.yaml
INFO[0000] Using app config file: /etc/kuka/updateagent/config.yaml
INFO[0000] Using user config file: /opt/kuka/updateagent/config.yaml
Bundle file bundle-description.yaml successfully validated.
```

In case something is set in the wrong way, the update agent will tell you so.

```shell
$ update-agent bundle validate basic_noContainerImage.yaml
INFO[0000] Using app config file: /etc/kuka/updateagent/config.yaml
INFO[0000] Using user config file: /opt/kuka/updateagent/config.yaml
Error: error validating bundle: required field [container.spec.image] is not set
[...]
```

## Create .bundle

Once your YAML is validated, you can create a bundle from it. You have to give it a path to a private key file. It will
be used to sign the YAML. If you give more than one file, those files will also be added to the bundle. The update-agent
validates the YAML again, signs it, and puts it in the .bundle archive together with the other files you give it.

```shell
$ update-agent bundle create -k key.pem bundle-description.yaml EULA.txt
INFO[0000] Using app config file: /etc/kuka/updateagent/config.yaml
INFO[0000] Using user config file: /opt/kuka/updateagent/config.yaml
Signing description file bundle-description.yaml
Bundle file ./test-bundle-id_1.0.0.bundle successfully created.
```

In case you don't have the update-agent, this is how you could also do it but keep in mind that in this case the YAML
won't be checked.

### Create a base bundle with additional JSON Schema

In order to create a base bundle with e.g. the additional JSON Schemas `required-user-and-realtime.json` and `not_allowed_cap_add.json` that will be used for scope validation, you can execute the following:

```shell
update-agent bundle create -k test-private-key.pem robotics-base-bundle_1.1.0.yaml required-user-and-realtime.json not_allowed_cap_add.json EULA.txt
```

In this example `required-user-and-realtime.json` and `not_allowed_cap_add.json` defines two grammar as described in
[./scope_validation.md](./scope_validation.md).

Please be aware that the signing process involves not just the `robotics-base-bundle_1.1.0.yaml` but also any JSON files that are included as additional parameters. This implies that in the earlier example, both `required-user-and-realtime.json.sign` and `not_allowed_cap_add.json.sign` are created and incorporated into the .*bundle file.

```shell
openssl dgst -sha256 -sign key.pem -out basic_baseBundle.yaml.sign basic_baseBundle.yaml
tar -cvzf test-bundle_0.0.1.bundle basic_baseBundle.yaml basic_baseBundle.yaml.sign EULA.txt
```

The .bundle file is only valid and usable by the update-agent if the filename follows the naming convention. The name
must consist of two parts separated by `_` and followed by `.bundle`. The first part is a name and can only consist of
the letters a-Z, digits and `-`. The second part is the version, and must follow the semver format. The entire regex is
shown below.

```
^[a-zA-Z0-9-]+_(?P<major>0|[1-9]\d*)\.(?P<minor>0|[1-9]\d*)\.(?P<patch>0|[1-9]\d*)(?:-(?P<prerelease>(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\+(?P<buildmetadata>[0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?\.bundle$
```

## A note on installation from USB

If the installation happens from a flash drive, the controller is likely offline, which means it is not able to download
the images for the bundle. In this case the update-agent is looking for a file with the same name as the .bundle file,
but with an "_images.tar.gz" at the end, i.e. `$bundleId_$bundleVersion_images.tar.gz`. This file should contain all
the required images. You can create it like this:

```shell
podman save -m docker.io/library/alpine docker.io/library/busybox | gzip > test-bundle-id_1.0.0_images.tar.gz
```

## A note on creation of SWU files

During the installation of SWU updates, the update-agent looks for files with predefined pattern names. It means that
the update-agent verifies whether the SWU files have a certain predefined pattern name. Therefore, the names of SWU 
files should contain the following information:

* Controller
* Maturity
* Version
* Codename
* Arch

The structure of the file name should look like this:

```shell
$(Image.Controller)_$(Image.Maturity)_$(Image.Version)_$(Image.Codename)_$(Image.Arch).swu
```

Here are the examples:

```yaml
krc5_stable_0.0.4_marlin_amd64.swu       15-Apr-2021 15:30  369.00 MB
krc5_stable_0.0.5_marlin_amd64.swu       01-Jun-2021 08:19  368.98 MB
krc5_stable_0.0.6_marlin_amd64.swu       02-Jun-2021 12:52  368.33 MB
krc5_stable_0.0.7_marlin_amd64.swu       07-Jun-2021 15:32  369.17 MB
```

# Naming Convention for files

The following describes the naming conventions used for:

- the **bundle file** itself (*.bundle) and the files it contains:
    - bundle description file (*.yaml)
    - bundle description file signature (*.sign)
    - End-user license agreement (aka. EULA)
- SWU file

## Bundle File (*.bundle)

A bundle file is a compressed tar archive.

The .bundle file is only valid and usable by the update-agent if the filename follows the naming convention. The name
must consist of two parts separated by `_` and followed by `.bundle`. The first part is a name and can only consist of
the letters a-Z, digits and `-`. The second part is the version, and must follow the semver format. The entire regex is
shown below.

```regex
^[a-zA-Z0-9-]+_(?P<major>0|[1-9]\d*)\.(?P<minor>0|[1-9]\d*)\.(?P<patch>0|[1-9]\d*)(?:-(?P<prerelease>(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\+(?P<buildmetadata>[0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?\.bundle$
```

Examples:

- `robotics-base-bundle_1.0.0.bundle`
- `gripper-toolbox-bundle_1.1.0.bundle`
- `robot-data-lbr3r760-iisy-230V_1.0.0.bundle`

### Convention for the local case

For the **local case** i.e. while installing from an USB, the naming convention is:

```
$bundleId_$bundleVersion_images.tar.gz
```

Be aware of the ending `_images.tar.gz`.

See also **A note on installation from USB** above for more details.

### Content of the bundle file

The `.bundle` file itself contains:

- A manifest / description file with:
    - The unique bundle id (unique across all existing bundles)
    - The unique bundle version (unique within one bundle id context)
    - Lists with container images within multiple named sections which contain
        - The container image name
        - The container image version
- A `licenses` folder which contains at least an `kuka-EULA.txt` document, but could contain more license files.
- A signature file for the manifest / description.

```
├── robotics-base-bundle_1.0.0
│   ├── licenses
│   │   └── kuka-EULA.txt
│   ├── robotics-base-bundle_1.0.0.yaml
│   └── robotics-base-bundle_1.0.0.yaml.sign
```

### The Bundle Description File (.yaml)

The Bundle Description File (*.yaml) follows the conventions of the **Bundle file** itself (only the extension changes):

```
^[a-zA-Z0-9-]+_(?P<major>0|[1-9]\d*)\.(?P<minor>0|[1-9]\d*)\.(?P<patch>0|[1-9]\d*)(?:-(?P<prerelease>(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\+(?P<buildmetadata>[0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?\.yaml
```

The signature counterpart of the Bundle Description File (*.yaml), just prefixes the file name with `.sign`.

Examples:

- inside the `gripper-toolbox-bundle_1.1.0.bundle` bundle file you will find
    - the Bundle Description File `gripper-toolbox-bundle_1.1.0.yaml` and it's corresponding signed counterpart
    - `gripper-toolbox-bundle_1.1.0.yaml.sign`

### The EULA files

EULA files do not follow a specific naming convention, but it is expected to have a folder named `licences` that
contains at least one file with the extension `*.txt`.

Examples:

```
├── robotics-base-bundle_1.0.0
│   ├── licenses
│   │   └── kuka-EULA.txt
│   ├── robotics-base-bundle_1.0.0.yaml
│   └── robotics-base-bundle_1.0.0.yaml.sign
```

or

```
├── gripper-toolbox-bundle_1.1.0 copy
│   ├── gripper-toolbox-bundle_1.1.0.yaml
│   ├── gripper-toolbox-bundle_1.1.0.yaml.sign
│   └── licenses
│       ├── kuka-EULA.txt
│       ├── kuka-license.txt
│       └── kuka-license_de.txt
```

### SWU (*.swu)

#### KUKA Linux OS

The KUKA Linux OS SWU files follow this naming convention:

`$(Image.Controller)_$(Image.Maturity)_$(Image.Version)_$(Image.Codename)_$(Image.Arch).swu`

Example: `krc5_stable_0.0.7_marlin_amd64.swu`

To match KUKA Linux OS SWU files, the following RegEx pattern is used:

`^(krc5_iiqka_[a-zA-Z-]+)_([0-9]+\.[0-9]+\.[0-9]+)[a-zA-Z0-9_]*\.swu$`

#### SmartPad Firmware

To match SmartPad firmware SWU files, the following RegEx pattern is used:

`^((.*_)?smartpad-pro(_[a-zA-Z-]+)*)_(\d+\.\d+\.\d+)(_.*)?\.swu$`

Example of a SWU file for the SmartPad: `smartpad-pro_iiqka_1.3.0_image.swu`
