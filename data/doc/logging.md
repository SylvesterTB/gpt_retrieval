# Logging

There are two categories of logs:

1. **UserLogs** that are shown to the end user, and thus have to be **internationalized**. This topic is treated
   in [i18n.md](./i18n.md).
2. **Technical Logs** generated using the [Logrus](https://github.com/sirupsen/logrus) Golang library. This is the topic
   of this document.

The update agent consistently logs debug output. In order to navigate the output more easily the output client also logs
certain key progress points as log level info.

Logging is implemented with [logrus](https://github.com/sirupsen/logrus) and configured to log to `syslog` when available.
To avoid double logging via systemd, the update-agent does not log to `stdout` when syslog is available.

## Key progress points

Each functionality of the update agent has its own **key points** and log outputs defined in `internal/util/log.go`.

The help the developer to better identify the stage of the flow that has been reached during a debugging session. 

### Download

* "DOWNLOAD: REQUEST"
* "DOWNLOAD: START"
* "DOWNLOAD: KUKA LINUX""
* "DOWNLOAD: BUNDLES"

### Install

* "INSTALL: REQUEST"
* "INSTALL: START"
* "INSTALL: KUKA LINUX"
* "INSTALL: BUNDLES"
* "INSTALL: CONTINUE"

### ListAvailableBundles

* "LIST UPDATES: REQUEST"
* "LIST UPDATES: KUKA LINUX"
* "LIST UPDATES: BUNDLES"
