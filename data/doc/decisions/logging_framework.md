# Logging Framework

## Context

We need to have expressive logging to trace installation procedures and report details on errors. For the RoX Controller
[syslog](https://geek-university.com/linux/syslog-protocol-explained/) should be used for logging.

## Decision

We use [logrus](https://github.com/sirupsen/logrus) for logging.

While there are newer and faster options for logging (e.g. the widely used [zap](https://github.com/uber-go/zap)) and 
logrus is in maintenance mode, it is still the most used logging framework for go.

## Consequence

* Due to broad adoption developer resources and documentation are very good.
* Supports syslog [out-of-the-box](https://pkg.go.dev/github.com/sirupsen/logrus/hooks/syslog)
* Is slower than alternatives, but as a edge-device service logging performance is of secondary concern (we wont produce
  1000s of logs per second)
* Supports structured logging, for adding e.g. a installation context to log entries.