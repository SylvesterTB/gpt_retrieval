# SmartPad Firmware Update CLI

The SmartPad Firmware Update CLI (`tools/swu-smartpad-update-cli.go`) is a command-line utility designed to update the firmware of a SmartPad device.
The tool utilizes an SSH connection to communicate with the SmartPad device and performs an update using the provided
SWU (Software Update) image file.

## Features

- Connects to the SmartPad device via SSH.
- Retrieves the current firmware version before updating.
- Updates the SmartPad device using the specified SWU image file.
- Retrieves the firmware version after the update.
- Implements an exponential backoff retry strategy for checking the device version.

## Usage

```shell
go run swu-smartpad-update-cli.go --swu_file <SWU_IMAGE_FILE> --host_name <HOST_NAME> [--port <PORT>] [--timeout <TIMEOUT>] [--log_level <LOG_LEVEL>] [--color <COLOR>]
```

### Arguments

- `--swu_file <SWU_IMAGE_FILE>`: (Required) Path to the SWU image file.
- `--host_name <HOST_NAME>`: (Required) Hostname or IP address of the SmartPad device.
- `--port <PORT>`: (Optional) Port number for the SSH connection (default: 8080).
- `--timeout <TIMEOUT>`: (Optional) Timeout in seconds for the whole SWUpdate process (default: 300).
- `--log_level <LOG_LEVEL>`: (Optional) Change the log level (error, info, warning, debug) (default: "debug").
- `--color <COLOR>`: (Optional) Colorize messages (auto, always, never) (default: "auto").

### Example

```shell
go run swu-smartpad-update-cli.go --swu_file /Users/dimitri.missoh/dev/workspace/kuka/swus/smartpadpro_iiqka_1.3.0-dev.1181420_image.swu --host_name localhost
```

#### Output

The CLI tool will output the firmware version before and after the update process. Additionally, it will display whether the update was successful or not.

Example output:

```shell
INFO[0002] SmartPad version before the update: 1.3.0-dev.1176800
INFO[0002] Start uploading the image /Users/dimitri.missoh/dev/workspace/kuka/swus/smartpadpro_iiqka_1.3.0-dev.1181420_image.swu...
INFO[0002] Waiting for messages on websocket connection
INFO[0002] [network_thread] : Incoming network request: processing...  fields.level=7
INFO[0002] Software Update started !                     fields.level=6
INFO[0002] [network_initializer] : Software update started  fields.level=7
.
.
.
INFO[0151] [run_system_cmd] : /tmp/scripts/image_postinstall_finalize  sys_a command returned 0  fields.level=7
INFO[0151] SWUPDATE successful !                         fields.level=6
INFO[0151] Restarting the SmartPad...
INFO[0152] Restart SmartPad request response status: 201 OK
INFO[0152] Try (multiple times if necessary with exponential backoff). Initial delay: 1s, backoff factor: 1.5, and max retry: 5
INFO[0152] Make the first call after an initial delay of 1s ...
INFO[0153] Trying to retrieve the version...
INFO[0163] Successfully retrieved the current version of SmartPad: 1.3.0-dev.1181420
INFO[0163] The SmartPad has been successfully update! SmartPad version before update: 1.3.0-dev.1176800, SmartPad version after upadate: 1.3.0-dev.1181420
```

## Using this tool on your local machine

To use this tool from your local machine following is necessary:

- VPN connection to the MaibornWolff network
- Port-Forwarding (log-in with the known controller password): `ssh -N -L 8080:smartpadpro:8080 kuka@172.27.5.126` (this will block and is cancelable using CTR+C)
- Then call the CLI tool as explained above (but use `localhost` since the port-forwarding is doing to mapping to the `smaprtpadpro` host on the controller).

## Prerequisites

- Go programming language installed.
- Access to the KUKA Controller device via SSH.
- SWU image file for the update.

## Some example snippets

```sh
ssh -N -L 8080:smartpadpro:8080 kuka@172.27.5.126

go run swu-smartpad-update-cli.go --swu_file /Users/dimitri.missoh/dev/workspace/kuka/swus/smartpadpro_iiqka_1.3.0-dev.1181420_image.swu --host_name localhost --log_level debug

go run swu-smartpad-update-cli.go --swu_file /Users/dimitri.missoh/dev/workspace/kuka/swus/smartpadpro_iiqka_1.3.0-dev.1176800_image.swu --host_name localhost --log_level debug

curl -X POST -m 1 http://localhost:8080/restart
```
