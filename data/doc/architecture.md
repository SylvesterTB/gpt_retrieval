# Architectural Documentation

- [Introduction and Goals](#introduction-and-goals)
  - [Requirements Overview](#requirements-overview)
  - [Quality Goals](#quality-goals)
  - [Stakeholders](#stakeholders)
- [Architecture Constraints](#architecture-constraints)
- [Scope and Context](#scope-and-context)
  - [Business Context](#business-context)
  - [Technical Context](#technical-context)
- [Solution Strategy](#solution-strategy)
- [Building Block View](#building-block-view)
- [Runtime View](#runtime-view)
- [Deployment View](#deployment-view)
- [Cross-cutting Concepts](#cross-cutting-concepts)
- [Architectural Decisions](#architectural-decisions)
- [Quality Requirements](#quality-requirements)
- [Risks and Technical Debts](#risks-and-technical-debts)

## Introduction and Goals

The `update-agent` is an edge device service written in [Go](https://golang.org/). It enables retrieval and installation
of remote software updates for controllers of the KUKA RoX platform.
The installable update-artifacts are versioned according to [SemVer 2.0.0](https://semver.org/). Only installation of
newer update-artifacts (i.e. higher version) is supported. A downgrade to older versions is not supported.

### Requirements Overview

The `update-agent` is mainly responsible for:

- Downloading and interpreting the custom [description](./bundle_description.md) for KUKA-Update-Packages from an
  Update-Backend
- Instrumenting [Podman](https://podman.io/) and [systemd](https://www.freedesktop.org/wiki/Software/systemd/)
  for creating and starting the OCI-Containers of a package.
- Installing OS image updates by instrumenting [SWUpdate](https://sbabic.github.io/swupdate/swupdate.html)
- Providing a GRPC interface for triggering and monitoring the update process

### Quality Goals

| Quality Goal               | Scenario                                                                                                                                                                                                                                 |
|----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Recoverability             | An installation changes the persistent state of the controller. All changes made by an install transaction must be recoverable to the previous state                                                                                     |
| Authenticity               | All installed artifacts have to be verified to be from a trusted source.                                                                                                                                                                 |
| Fault Tolerance            | Errors during installation of an update-artifact should always lead to a rollback to the previous state to avoid inconsistent controller states                                                                                          |
| Functional Appropriateness | The update-agent is *NOT* responsible for the runtime behavior of installed software components. Starting and stopping of installed software is done by `systemd`, the update-agent should be idle when no transaction is triggered by the API |
| Testability                | To enable testing on non-controller OS, there have to be small, mockable interfaces for all interactions with system software (e.g. `systemd`, `podman`)                                                                                 |
| Consistency                | The canonical way to interact with the update-agent is the provided GRPC-API provided by the update-agent system service. For command-line access the update-agent uses this API to provide a consistent behavior                       |

### Stakeholders

| Role/Name                 | Contact                  | Expectations                                                             |
|---------------------------|--------------------------|--------------------------------------------------------------------------|
| KUKA R&D Developers       | <ML-RD_Team_OS@kuka.com> | Detailed information for the creation of update-artifacts and especially the bundle-description |
| KUKA UI Developers        | - | The GRPC-Interface is documented and works                                                      |
| KUKA security accountable | - | All update-artifacts are signed and installed only from trusted sources                         |

## Architecture Constraints

|Constraint|Explanation|
|----------|-----------|
| Existing software artifacts | The update-agent has to install existing artifacts in the form of swupdate's `.swu` files and OCI-containers for application software |
| Separation of OS and application software | The OS and application software must be installable independent from each other |
| Enabling real-time containers | Application software with real-time requirements has to be installed and runnable without constraints |

## Scope and Context

![Component-View](./diagrams/component-view.drawio.png)

### Business Context

| Neighbor        | Description                                                                                    |
|-----------------|------------------------------------------------------------------------------------------------|
| backup-service  | Create and restore snapshots of the data partition                                             |
| swupdate-client | Install OS updates. Additional scripts are used for triggering rollback and reboot on failures |
| systemd         | Start and stop units. Additionally, file creation/deletion of units is used                    |
| podman          | Creating and removing of containers and running of init-containers                             |
| AWS Cognito     | User authentication via ID-Token provided from GRPC-Clients                                    |
| Update-Service  | Download of current update-definitions (bundles) and swu-files                                 |
| Image-Registry  | Pulling of images defined in update-bundles                                                    |

### Technical Context

The following technical interfaces are needed for the operation of the update-agent.

| Type | Technology | Location       | Usage                                                                                                                                       |
|------|------------|----------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| used | CLI        | on-controller  | calls `swupdate-client`, `rollback` and `reboot`            |
| used | dbus       | on-controller  | instrumenting systemd                                                       |
| used | HTTP       | on-controller  | interaction with podman by using the [go-bindings](https://github.com/containers/podman/blob/main/pkg/bindings/README.md) provided by podman |
| used | HTTPS      | off-controller | communication with backend, download and image pull |
| provided | CLI | on-controller | expose select functionality for development users |
| provided | GRPC | off-controller | expose API for UI |

## Solution Strategy

1. The update process is implemented on-controller as a standalone utility that enables complex dependency resolution
   and implementation of a GRPC interface for controlling the update.
2. The update-agent is implemented in [go](https://go.dev/) for its efficiency and compatibility with core components
   (`podman` is also written in go, good support for GRPC)
3. The update-agent is provided as a debian package, which is preinstalled on the KUKA-Linux platform
4. The installation logic for OS updates is implemented inside the swu-Files and not in the scope of the update-agent
5. The installation logic for containers is fully implemented in the update-agent based on the static configuration in
   the bundle-description.
6. The update-agent only supports the **upgrade** of installed artifacts to higher version numbers.

## Building Block View

### Folders

- `internal` go-packages for packages that are not usable outside of the project (contains the server implementation)
- `config` groups configurations files
- `itest` go implemented integration testing (only on environments with all dependencies installed)
- `pipelines` azure build pipelines
- `tools` tools for integration test and Vagrant dev-environment
- `debian` all files for building the debian package of the update-agent
- `doc` documentation in markdown
- `devdoc` documentation only relevant for the non-KUKA development team
- `proto-api` separate module for the GRPC-Interface
- `i18n` internationalization files
- `controllertest` contains files relevant for integration tests that run on a controller
- `testdata` this directory contains some data, which can be used for controller tests

### Go packages

| Package                                | Responsibility                                                                        |
|---------------------------------------|---------------------------------------------------------------------------------------|
| main                                  | entrypoint. does not contain logic                                                    |
| internal/api/cli                      | provides CLI, including command for starting the server                               |
| internal/api/server                   | implementation of the GRPC API and mapping of internal to interface transport objects |
| internal/config                       | handling of the configuration of the update-agent                                     |
| internal/model/bundle                 | bundle description parsing and validation                                             |
| internal/business/install             | main installation logic                                                               |
| internal/business/recreate            | recreation logic to recreate installed containers with outdated device versions       |
| internal/adapter/podman               | podman interface                                                                      |
| internal/adapter/systemd              | systemd interface and unit creation logic                                             |
| internal/persistence/filesystem       | helper methods for filesystem access                                                  |
| internal/adapter/backend              | interface with update-service on AWS                                                  |
| internal/adapter/backup               | interface with backup-service                                                         |
| internal/adapter/execution-controller | interface with execution-controller for checking safety                               |
| internal/model/kukalinux              | interface with system-information                                                     |
| internal/adapter/system               | interface with scripts for rollback                                                   |
| internal/business/dependency          | provides functionalities to manage the order, grouping and dependencies of bundles    |

The aforementioned table is not comprehensive and only includes packages that require further explanation.

### Unit Structure

The generated systemd units are documented in [systemd.md](./systemd.md)

### Bundle definition

The bundle definitions are documented in [bundle_description.md](./bundle_description.md)

### Persistence

Persistence on-controller and used storage locations are documented in [persistence.md](./persistence.md)

### GRPC-API

The GRPC API is documented in the corresponding [proto-definition](../proto-api/operation-management-update-agent-api/proto/kuka/operationmanagement/updateagent/v1/update_agent_service.proto)

### Configuration

Configuration parameters are detailed in [config.md](./config.md)

## Runtime View

### Installation

The Installation logic separates three main update-flows:

1. Only Bundles (application software)
2. Only KUKA-Linux
3. Combined update of KUKA-Linux and then Bundles

![Update-Flow](./diagrams/combined-update-flow-with-states.drawio.png)

### Rollback

Rollback handling is documented in [rollback.md](./rollback.md)

### Request cancellation

Via the `Cancel` GRPC method a running `Install` call may be cancelled.

Cancellation is implemented via the `context` go-package. An install transaction has a corresponding context that is
cancellable via the API call.  The effect of cancellation is that installation via `podman`, `systemd` or `swu` is no
longer possible and the install transaction is rolled back.

### Source verification

For all update-artifacts the source has to be verified:

- **OS**: SWUpdate allows only for signed `swu` files to be installed. As we only instrument swupdate this is not in
  scope for update-agent.
- **Bundles**: Bundle files include a signature for the bundle-description yaml-file which is verified against the
  configured certificate. Unsigned bundles or bundles with wrong signatures cannot be installed.
- **Images**: The bundle-description must contain an image-id for each used image. Containers are only created when the
  image id in the description matches the id on-controller.

## Deployment View

See [component-view](#scope-and-context)

## Cross-cutting Concepts

### Logging

See [logging.md](logging.md)

### Testing

There are some test which are executable on generic build-environments. These are marked with the build tag `!noIntegration`.
Most tests however required the on-controller dependencies (see [component-view](#scope-and-context)) to be installed
and are executed inside a virtual machine set up by [Vagrant](https://www.vagrantup.com/) with the project
[Vagrantfile](../Vagrantfile).

## Architectural Decisions

For easy of use architectural decisions are documented in separate files in the [decisions folder](./decisions).

## Quality Requirements

See [Quality Goals](#quality-goals)

## Risks and Technical Debts

### Risks

| Risk                                 | Description                                                                                                                                                                                                                                                                   |
|--------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Inconsistent controller state        | We rely heavily on the restore feature of the backup-service. All failed installs result in a restore of the current on disk state. In case of failure during the restore we have an unknown system state and both the update-agent and the controller may not work correctly |
| Malicious use of limited root access | The update-agent has access to the root podman instance (via socket access rights) and to manage root systemd-units via polkit which may be used maliciously.                                                                                                                 |

### Technical Debts

| Technical Debt              | Description                                                                                                                                                 |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Complex install logic       | The combined installation logic is very complex mainly due to error-handling requirements and state-handling. The code could be simplified when refactored. |
| Inconsistent error handling | Error sentinels and custom errors are used for different purposes. This could be simplified by a refactoring                                                |
| Logging                     | Logrus is used and should be changed to the special KUKA logging framework                                                                                  |
