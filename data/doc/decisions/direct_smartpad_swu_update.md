# Use the SWU REST/WebSocket API to update the SmartPad

## Context

The goal is to update the controller and firmware of the SmartPad. Using SWU’s forward mechanism leads to a controller restart, even when only the SmartPad has been updated, as configured.

## Decision

Work around the restart behavior of the controller by directly accessing the SWU instance running on the SmartPad itself. This can be achieved by using the REST/WebSocket endpoints SWU provides.

## Consequences

- The **forward** version of the SmartPad firmware is no longer needed.
- The update-agent requires a more complex implementation to manage the firmware update.

