# Controller Tests

To start a connection has to be established with the KRC5 via port forwarding (see Run Tests). 
For `controller_template_test.go` the template files are in the folder `controller_yml_templates` 
and the bundles are created from a .yaml file and the images are downloaded. 
These are saved locally and then copied to the KRC5 into the USB directory.
At the end of the test, the files are cleaned up and removed from the KRC5.

The `controller_office_test.go` installs a kuka robotics-base-bundle. If several versions are available it will install
the latest version. You can also install addon-bundles by changing the bundle identifier in the code. 
It requires a bundle and the images archived (as .tar.gz file) on the usb stick `/mnt/usb_update/`

## Setup-Tests 

Before the tests run for the first time, the credential template must be renamed `controller_access.yaml` 
and the credentials for KRC5 entered.

## Run Tests

Establish a connection with the KRC5 `ssh -L 49202:127.0.0.1:49202 -N kuka@172.27.5.126` in your vagrant terminal

To run a test: `go test -tags controllertemplate -v ./controllertest -run TestInstallUninstall_SyntheticBundles`
or all tests in the package with `go test -tags controllertemplate -v ./controllertest` 


# How to generate kuka office robotic base bundle

1) Install Ansible on your Machine
2) Clone the [Rox_Deployment Repo](https://dev.azure.com/kuka/RoX%20OS/_git/rox_deployment)
3) Run the playbook `pb_create_bundles` 

`ansible-playbook -vv "/rox_deployment/ansible/pb_create_bundles.yml"`

4) You find the `robotics-base-bundle_{version}.yaml` in `/rox_deployment/bundles/office-base-bundle/`


5) Copy the `.yaml` into `operation_management_update_agent` repo and create bundle with
`go run updateagent.go bundle create -k internal/model/bundle/testdata/test-private-key.pem robotics-base-bundle_1.0.0.yaml`

    or use shell script from Rox Deployment
    `ansible_new/roles/dev_install_bundle_development/shell_script/create_bundle.sh` 

    with command:

    `./create_bundle.sh pathToYaml/robotics-base-bundle_1.0.0.yaml`

    The shell script will also download and pack the required images to a .tar.gz file. 

# How to setup controller with new KUKA Linux

1) Install [Script from Controller Tools iiQKA.OS](https://wiki.rd.kuka.com/devwiki/Controller_Tools_iiQKA.OS)
2) Do a factory reset with `krc factory-reset 172.27.5.126`. Instead of using the wipe-script it is recommended 
   to do factory-reset to have a clean state of controller. 
3) Install the .swu via WebUI [KUKA SOFTWARE UPGRADE](http://172.27.5.126:8080/)
4) After this you can login again via ssh
