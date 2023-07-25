# How to locally test the gRPC interfaces

This documentation describes how the gRPC interfaces of the update-agent can be tested within the Vagrant box.

### Install `grpccurl` in Vagrant

To send gRPC calls, a CLI tool is necessary. `grpccurl` can e.g. be used.

To install it in the Vagrant box do the following (after `vagrant up`, `cd /vagrant`):

```shell
curl -sSL "https://github.com/fullstorydev/grpcurl/releases/download/v1.8.7/grpcurl_1.8.7_linux_x86_64.tar.gz" | sudo tar -xz -C /usr/local/bin
```

### Set the reflection flag in `server.go`

In the `internal/api/server/server.go` file, set the reflection flag `reflection.Register(grpcServer)`.

See the snippet below:

```go
func Start(opts *ServerOpts, applicationCtx *appcontext.ApplicationContext) error {
...
// set reflection flag here
reflection.Register(grpcServer)

log.Info("[Server Startup] Starting to listen on ", address)
...
return nil
}
```

### Start the update-agent

In the Vagrant box, execute the following:

```shell
go run updateagent.go server
```

### Option 1:  Make your gRPC calls in Vagrant

gRPC calls can now be executed. See some examples below:

```shell
grpcurl -plaintext 127.0.0.1:49202 list

grpcurl -plaintext 127.0.0.1:49202 kuka.operationmanagement.updateagent.v1.UpdateAgentService.GetInstalledOsImage
```

### Option 2:  Make your gRPC calls outside Vagrant

Add your SSH-Key in Vagrant:

1) Set in `/etc/ssh/sshd_config` the parameter `PubkeyAuthentication yes` <br>
   and `AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2` by uncomment
2) Add your public (id_rsa.pub) key to `/home/vagrant/.ssh/authorized_keys`
3) Restart ssh daemon `sudo systemctl restart sshd`

Do port forwarding to your vagrant box:

`ssh -p 2222 -L 49202:localhost:49202 -N vagrant@localhost`

grPC calls can now be executed

###

This documentation is related to [README_testing.md](../internal/adapter/backend/README_testing.md)