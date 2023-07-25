# Programming language

## Context

We need a programming language for the RoX update-agent

## Decision

We use [Go](golang.org)

## Consequences

* We can use the [podman go bindings](https://github.com/containers/podman/blob/master/pkg/bindings/README.md)
* The expected programming team that takes over at KUKA is fluent in go
* Great [GRPC support](https://grpc.io/docs/languages/go/basics/)

