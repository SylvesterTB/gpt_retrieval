# Cheat Sheet

## Container
Podman run command for an created container: `sudo podman inspect containername --format {{.Config.CreateCommand}}`