# SmartPad update

## Context

* In the first version the update-agent only manages bundle and kuka linux updates. This functionality is to be
  expanded to include updates for the smartpad.
* The SmartPadImage has dependencies to the kuka linux version running on the controller as well as the UI container
  which comes with the base bundle. Different versions of modeling the dependencies can be seen in the
  [diagram](../concepts/resources/concept-smartpad-update.drawio.png).
* Currently, the SmartPad is updated using a dedicated .swu file and the SWU forwarder.

## Decision

* The SmartPadImage will be managed as a separate update-artifact (in form of a .swu) and not included in the kuka linux
  .swu.
  * Allows the SmartPadImage to be updated independently and potentially express different dependencies in the future
    (e.g. to the base bundle)
  * Currently, the SmartPad is already updated using .swu files.
* The dependencies to and from the SmartPadImage will not be handled by the update-agent.
  * This decreases implementation effort for the first iteration.
  * Different dependencies might be added in the future.

## Consequences

* Limited changes to existing grpc methods, however new grpc methods require to trigger install of SmartPadImage.
* As the SmartPadImage-Update is done separately from the other update types it is not a part of the transactional
  install call, i.e. a failed update of the Smartpad does not cause a rollback on the controller.
* As there is no way of expressing dependencies it is assumed that every version of the SmartPadImage works well enough
  with every software version encountered on the controller (bundles and kuka linux).
