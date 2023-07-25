# KUKA - Update Agent - Accounts and Setup

## Requirements

The following sessions describe necessary steps to allow access to the KUKA infrastructure.

Following will be necessary:

- a KUKA account of the form `Extern.FIRST_NAME.LAST_NAME@kuka.com` which gives access to:
  - the VPN which give access to
    - the KUKA go dependencies
    - the Artifactory (JFrog)
  - the Azure DevOps plaftorm

### Accounts

The account `Extern.FIRST_NAME.LAST_NAME@kuka.com` has to be applied by KUKA itself. After account creation, you will
get an E-Mail with your username and one part of the password. The other part will be send to you by KUKA through the
project management (on the time of editing, Dominik Grether).

This E-Mail will contain a link to:

- reset your initial password
- to access your KUKA Outlook (web)
- to setup you 2FA (use Microsoft Authenticator for that)
- and a link to downlod the VPN cient `Endpoint Security VPN Client` for your machine
- for Mac go directly to
  this [site](https://supportcenter.checkpoint.com/supportcenter/portal/user/anon/page/default.psml/media-type/html?action=portlets.DCFileAction&eventSubmit_doGetdcdetails=&fileid=123672)

### Setup VPN

The server URL ist: `kra.kuka.com`

Login with your new credentials and use your authenticator app.

If you got the message from the VPN client that you connection fails during the phase`Enforce Firewall Policy failed`,
allow it in your Mac Settings under `Security & Privacy > Firewall`

Related to this topic is also [Install and use VPN in the KUKA Vagrant Box](/devdoc/vpn_in_vagrant.md)

### Setup the Vagrant based development environment

For this step follow the steps described in the main project [README.md](../README.md)
