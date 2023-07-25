# Concept: Default Volumes

## Requirement

AddOn-Bundles need the possibility of persistent storage for bundle specific data, that is allowed to be deleted 
when an uninstall with `prune` takes place.
* As of the already implemented [Concept User Data](../userdata.md) this is not possible with current volumes 
  as they are only defined by the base-bundle.
* The OverlayFS inside the container may also not be used, as there are new containers created with updates, 
  which would lose the persistent data.

## Option A: Generated Bundle-specific Groups (discarded)

### Rationale

* Currently, groups are used to provide access rights to volumes. To apply that concept for bundle-specific storage on 
  the host via volumes, we need to introduce bundle-specific groups.
* GroupIDs for bundles need to be unique to avoid other bundles from accessing the default-volume.
* GroupIDs for bundles need to stay the same when updating (uninstall without prune and installing), 
  to avoid loosing access rights.
* The bundle-specific groups need to be named (in the groups-file) and 
used alongside the groups defined in the base-bundle.
* To avoid clashes with user-defined Groups in the base-bundle, 
  we need a dedicated number range for bundle-specific groups.
* Addon-Bundles may still not define groups in the bundle-description.
* Currently, the base-bundle creates a temporary group-file which contains all user-defined groups on install, 
  that is used to add groups on the installed-bundles.target startup. 
  We extend that mechanism to add bundle-specific groups.
* On debian these are the restrictions for naming groups (from `man useradd`):
```
       It is usually recommended to only use usernames that begin with a lower case letter or an underscore,
       followed by lower case letters, digits, underscores, or dashes. They can end with a dollar sign. In
       regular expression terms: [a-z_][a-z0-9_-]*[$]?

       On Debian, the only constraints are that usernames must neither start with a dash ('-') nor plus ('+')
       nor tilde ('~') nor contain a colon (':'), a comma (','), or a whitespace (space: ' ', end of line:
       '\n', tabulation: '\t', etc.). Note that using a slash ('/') may break the default algorithm for the
       definition of the user's home directory.

       Usernames may only be up to 32 characters long.
```
* This means we cannot safely use the bundle-id (which may be longer) to safely name the groups. 
  We propose not to name bundle-specific groups in the groups-file.

### Changes

* Bundle specific groups use the GroupID range `2500-3000`. The range for defined groups need 
  to be restricted accordingly (currently `2000-3000` enforced by `provide_groups`-script)
* We maintain a mapping of BundleIDs to GroupsIDs in a file under `<appDir>/bundle_groups`
* When a bundle is installed, the mapping finds the next free number in the range and assigns it to the bundleID
* When a bundle is uninstalled with prune, the mapping for that bundle is removed. The GroupID is then free to reuse.
* The bundle-specific GroupID is then used to SetACLs for the default-volume (see below)

## Option B: Configurable Bundle-Groups

### Rationale

* For each Addon-Bundle a distinct Group-ID and Group-Name is added during the release pipeline of the bundle.
* The release process is responsible for making sure GroupIDs are globally unique
  * There should be an index that assigns each bundle-id a globally unique GroupId and GroupName
  * ATTENTION: 3rd-party-specific GroupIDs may not be reused, even when 3rd-party does not provide bundles anymore 
  (due to access rights)
* Currently, it is planned that only KUKA releases bundles and therefore can make sure that distinct GroupIDs are assigned to each Bundle-ID
* As the bundle-description-yaml is the only file we currently sign, the groups Name/ID is added to the bundle description to avoid major retooling.
* A default volume is only created, when the group ID/Name is given in the description, to avoid changing semantics for existing bundles.

### Changes

* The bundle description is extended by the following top-level properties:
```yaml
bundleGroup:
  name: bundle-group # group name restrictions of Option A apply
  id: 2020 # shares the same ID range as volume groups
```
* Default-Volumes (see below) are only set up, when bundleGroup is given
* Bundle-Validation is extended so that:
  * GroupNames conform to `useradd` restrictions
  * When `default` Volume is referenced it is checked, that a bundle-group is given
  * GroupID is checked to be in the reserved range `2000-3000`

### Usage example

Use a default volume inside of container of an Addon-Bundle:
```yaml
apiVersion: v1
id: test
name: default-bundle-usage-example
version: 1.0.0
type: addon
bundleGroup:
  name: bundle-group
  id: 2020
containers:
  - spec:
      name: test-container
      image: k8s.gcr.io/pause:3.3
      imageId: <sha256>
      volumes:
        - name: default
          dest: /workdir # path inside container 
      additionalGroups: 
        - bundle-group # needed to have access rights on the volume
```

## Provide Default-Volumes for Addon-Bundles

### Rationale

* We introduce an automatically generated  `volume` for addon-bundles called default-volume.
* “Generated” in this case means that the volume is present without needing it to specify in the bundle-description, 
  but it follows the capabilities of regular volumes.
* Access to this volume is only allowed by the bundle-specific GroupID/GroupName, defined by the `bundleGroup` property

### Changes

* As bundles now have no way of using persistent storage, 
  we introduce an auto-defined volume for the bundle which follows the following convention:
```yaml
volumes:
  - name: default
    hostPath: /opt/kuka/<bundle-id>
    shared: false 
    fileProviderAccess: false 
    readWriteAccess:
      - <bundle-specific-group-name/-id> # for Option A: group-name, Option B: group-id
```
* As we currently only allow named groups in the `*Access` definitions it is up to the implementation if we use that mechanism or custom logic for addon-bundles.
* The auto-defined volume is usable by containers by its name (`default`) as it would be if defined explicitly.
* When a bundle is uninstalled with `purge` the default-volumes hostPath is deleted from the host and 
  for Optiona A the group is also removed from the mapping-file.

## Open Question

* Should the base-bundle also get a bundle-volume? Not included in base-bundle.
* Is `bundleGroup` a good name for the new property? Should there be a way to give the default-volume a custom name (conflicts with validation of bundleGroup necessity)? Dominik provides feedback.
* Are Group-Names needed for default-volumes? Not necessarily, but Option B provides a good way of dealing with that.