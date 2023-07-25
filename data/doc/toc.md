# Additional Documentation

Following documentation covers different aspects of the system.

To better understand how the system is built, the top level sections **Architecture**, **Formats Descriptions**
and **System Description** are a good beginning.

The section **Set-Up** groups various topics that will helps you at the beginning of the project to set-up your accounts
as well as your developemnt environment.

Once you understand how the solution is build and as soon as you have a development environment up and running, you
can move on to the **How-To**s to get your hands dirty. Those sections will help you build and tests your first bundle
on a real KUKA controller.

## Architecture

- [architecture.md](./architecture.md)

### Architecture Decision Records (ADR)

Here are grouped the various **Architecture Decisions** that have been taken along the way.

- [backend-file-serving.md](./decisions/backend-file-serving.md)
- [image_verification.md](./decisions/image_verification.md)
- [logging_framework.md](./decisions/logging_framework.md)
- [podman_usage.md](./decisions/podman_usage.md)
- [programming_language.md](./decisions/programming_language.md)
- [acl.md](./decisions/acl.md)

## Formats Description

The `update-agent` is heavily based on configuration files. We have files to control **how the core component is
configured**, and on the other side files that **describe the artifacts** (software packages) that are deployed on the
KUKA controller.

- [config.md](./config.md): describes how the `update-agent` is configured
- [bundle_description.md](./bundle_description.md): describes how the bundles (the artifacts deployed on the controller
  such as the OS, base bundles, add-on bundles) as well as dependencies between them are configured.

## System Description

This section groups articles describing the various aspects the solution is build up from such as the role/right
concept [userdata.md](./userdata.md). The update-agent uses exclusively the file system to persist data and states,
and thus stores different information in different location in on the OS.
This is describes in [persistence.md](./persistence.md). Another aspect of the persistence is
the USB (see: [usb.md](./usb.md)). This document shows how this mount is used on one side a source of the bundles
and on the other side as the location where the base image for restoring the controller is stored.

- [systemd.md](./systemd.md): systemd is a system and service manager used to orchestrate the different components the
  solution is built up from
- [i18n.md](./i18n.md): describes how the internationalization has been implemented
- [logging.md](./logging.md)
- [persistence.md](./persistence.md): describes how the file system is divided and used to store data and states
- [rollback.md](./rollback.md): discribes how the system is restored when an update fails
- [usb.md](./usb.md): shows how the USB volumes are used

Notice that [logging.md](./logging.md) and [i18n.md](./i18n.md) cover the same top-level topic of **generating logs**.
[logging.md](./logging.md) is about **technical logs** whereas [i18n.md](./i18n.md) describes how **user facing**
logs are generated.

## Set-Up

Here are the main document to start with if you want to set-up your development environment.
First stop is getting the accesses you need ([accounts.md](/devdoc/accounts.md)) than you can move on with set-up your IDE
and the Vagrant Box we use to simulate the KUKA controller.

- [accounts.md](/devdoc/accounts.md): describes how you get access to the KUKA infrastructure (VPN, Artifactory).
  This is mandatory to be able to set-up a running development environment.
- [README.md](../README.md): the main documentation. This is a good start in the project.

## How-To

This session is the place for advanced user. It describes

- [how_to_bundle.md](./how_to_bundle.md): describes the steps to follow to configure and build your first bundle
- [controller_tests.md](./controller_tests.md): explains how integration tests can be execute against the controller
- [role-and-rights-configuration.md](./role-and-rights-configuration.md): give step by step instruction on how access
  to the bundle volumes can be configured form both perspective, as base-bundle author or add-on-bundle author.

## Concepts

- [userdata.md](./userdata.md): describes the role and right concept (based on Linux groups) used in the system
- [group-access-3rd-party-bundles.md](./concepts/group-access-3rd-party-bundles.md): presents a concept to control
  access (in the first run, access to the file system) from future 3rd-party add-on bundles.
