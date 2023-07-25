# Podman usage

## Context

The goal was to use podman exclusively rootless. But we found that the necessary linux capabilities for real-time 
containers can not be made available for rootless containers.

## Decision

We use the rootless podman instance (with the update-agent user) for managing images and the rootful podman instance for 
creating and running containers.

## Consequences

* All containers run as root, but not necessary with all capabilities (configurable in bundle-description)
* Realtime containers can be configured with the necessary capabilities
* Because containers are pulled by a non-root user, we are less prone to system-wide attacks when an attacker manages to
  download images form an untrusted source.

