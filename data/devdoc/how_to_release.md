# How to release the update-agent

### 1. Changelog

Edit the Changelog file `debian/changelog`

Copy the last entry from the changelog

Example:

```
operation-management-update-agent (2.0.2) unstable; urgency=medium

* CHANGE: new rollback mechanism from KUKA-OS is used(183842)
* ADD: gRPC API with function to remove downloaded bundles (165462)
* ADD: get podman createCommand via cli (173736)

-- Team Linux OS <ML-RD_Team_OS@kuka.com>  Tue, 16 Feb 2023 11:30:07 +0100
```

#### Version:

We change our versions according to [Semantic Versioning](https://semver.org/) Guidelines

`operation-management-update-agent (2.0.3) unstable; urgency=medium`

#### Changes:

According to our closed Stories in Azure Devops since the last release
we can add the following text in description

```
* ADD: New features
* CHANGE: For changes we did
* FIX: for fixes we did
```

Use a meaningful description and add the ticket number

#### Date

Change the date to the actual release date and use a rounded up time.

The correct short form must be used for the indication of the date

`-- Team Linux OS <ML-RD_Team_OS@kuka.com>  Tue, 16 Feb 2023 11:30:07 +0100`

### 2. Release with Pipeline

Go to our [Update-Agent Pipeline](https://dev.azure.com/kuka/RoX%20OS/_build?definitionId=1652)

Click the button `Run pipeline` in the top right corner

#### Options:

The Branch is master

Select from the dropdown menu `Repository name where package dependencies are resolved from` <br>
the target KUKA-OS release

Following boxed should be unchecked/checked:

- [ ] Queue debugging active
- [ ] Skip publishing of built packages to PublishRepository.
- [X] BlackDuck scan active
- [ ] Use production repo and BlackDuck scan for release
