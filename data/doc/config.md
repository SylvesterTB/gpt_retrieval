# Config File Documentation

## Location

### Application configuration

Currently the update-agent checks for a file called `config.yaml` in `/etc/kuka/updateagent/`.

A `config.yaml` file may look like this:

	AppDir                 string   `yaml:"app-dir"`
	BackupPort             int      `yaml:"backup-port"`
	BundleCertificatePaths []string `yaml:"bundle-certificates"`
	BundleHomeDir          string   `yaml:"bundle-home-dir"`
	BundlesDir             string   `yaml:"bundles-dir"`
	DevModeStatusPath      string   `yaml:"dev-mode-status-path"`
	ExconPort              int      `yaml:"excon-port"`
	KukaLinuxDir           string   `yaml:"kuka-linux-dir"`
	IsLogLevelDebug        bool     `yaml:"loglevel-debug"`
	ServerPort             int      `yaml:"port"`
	RecoveryEnabled        bool     `yaml:"recovery-enabled"`
	RollbackScriptPath     string   `yaml:"rollback-script-path"`
	SmartpadDir            string   `yaml:"smartpad-firmware-dir"`
	UnitDir                string   `yaml:"unit-dir"`
	UpdateAgentUserID      int      `yaml:"updateagent-user-id"`
	UpdateServiceUrl       string   `yaml:"update-service-url"`
	UsbDir                 string   `yaml:"usb-dir"`

```yaml
app-dir: '/opt/operation/updateagent'
backup-port: 49883
bundle-certificates:
  - '/etc/ssl/certs/update-agent-bundle-signing.pem'
bundles-dir: '/tmp/bundles'
excon-port: 49600
kuka-linux-dir: '/tmp/kukaLinuxImages'
loglevel-debug: true
port: 49202
rollback-script-path: '/usr/sbin/switch_active_installation'
unit-dir: '/opt/operation/systemd-units'
update-service-url: 'https://api.prod.updateservice.kuka.com'
updateagent-user-id: 211
usb-dir: '/mnt/usb_update'
```

### User configuration

A second file `/opt/kuka/updateagent/config.yaml` is checked for user configurations.

	AutomaticUpdateCheck bool `yaml:"automatic-update-check"`
	UsbEnabled           bool `yaml:"usb-enabled"`

A `config.yaml` file for user configurations may look like this:

```yaml
automatic-update-check: false
usb-enabled: true
```

## Parameters

These are the parameters with a short description and their default value:

```yaml
# Port to the update-agent server
port                         int    [49202]

# Path to the app directory of updateagent.
app-dir                      string [/opt/operation/updateagent]

# Path to the directory where bundle volumes should be generated (<bundle-home-dir>/<bundle-id>)
bundle-home-dir              string [/opt/kuka]

# Path to the storage location for systemd-units
unit-dir                     string [/opt/operation/systemd-units]

# Enables the USB update. When set to false, it is equivalent to the remote case.
usb-enabled                  bool   [true]

# Mount path to the usb drive. This path is mandatory.
usb-dir                      string [/mnt/usb_update]

# Port of the KUKA execution controller
excon-port                   int    [51052]

# Checks for available bundles from backend automatically 
# Is configurable for future use, but not used for now. 
automatic-update-check       bool   [false]

# Path to the certificate for bundle signature verification
bundle-certificate           string [/etc/ssl/certs/update-agent-bundle-signing.pem]

# Download path of the KUKA linux swu files 
kuka-linux-dir               string [/tmp/kukaLinuxImages]

# Download path of the bundle files
bundles-dir                  string [/tmp/bundles]

# If this is true, only signed bundle files are excepted
signed-bundles-only          bool   [true]

# User ID of the updateagent user 
updateagent-user-id          int    [211]

# Timeout for waiting for the containers to shut down
timeout-container-stop       int    [15]

# Enable debugging output
loglevel-debug               bool   [false]

# Port for the local backup service. 
backup-port                  int    [49883]

# URL to the update service
update-service-url           string []

# This is the script, which is executed in the case of successful kuka linux upgrade 
# and failing bundle install. It makes sure the system boots to the old partition. 
# NOTE: Whenever this path is changed in debian package .install, it needs to be changed here as well
rollback-script-path         string [/vagrant/internal/adapter/system/test_scripts/switch_active_installation]

# Flag for development to disable recovery mechanisms like restore and rollback by setting it to false
recovery-enabled         bool   [true]

# The port of the persistence-data-service running on the SmartPad 
smartpad-persistent-data-service-port int [10002]

# The path to the certificate to use to authenticate gRPC calls against the SmartPad
smartpad-cert-path string [/etc/ssl/certs/smartpadpro-iiqka-grpc-client.cert.pem]

# The path to the key to use to authenticate gRPC calls against the SmartPad
smartpad-key-path string [/etc/ssl/private/smartpadpro-iiqka-grpc-client.key]

# The host name of the SmartPad
smartpad-host string [smartpadpro]

# The port to access the SmartPad SWU interface
smartpad-port int [8080]

# The initial delay for the backoff algorithm used to request the version of the SmartPad after an installation
smartpad-backoff-initial-delay int [10]

# The number of maximal retries for the backoff algorithm used to request the version of the SmartPad after an installation
smartpad-backoff-max-retries int [3]

# The time factor to use for the backoff algorithm used to request the version of the SmartPad after an installation
smartpad-backoff-factor float [1.5]

# This is a feature flag used to turn on [true] the JSON Schema validation of add-on bundle scopes and off [false]
schema-validation-enabled bool [false]
```
