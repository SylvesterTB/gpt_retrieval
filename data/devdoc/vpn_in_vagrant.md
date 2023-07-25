# Install and use VPN in the KUKA Vagrant Box

The advantage with this approach is that your host (your working machine) can use a different VPN (MW) then the one
(KUKA VPN) started in the vagrant box to access KUKA ressources.

## Installation

1. `sudo dpkg --add-architecture i386` (it's ok if this does not run properly)
2. `sudo apt update`
3. `sudo apt install -y libx11-6:i386 libpam0g:i386 libstdc++5:i386 bzip2`
4. Download snx_install_linux30.sh from the link below
5. `sudo sh snx_install_linux30.sh`

The shell script `snx_install_linux30.sh` can be downloaded from:

[snx_install_linux30.sh](https://supportcenter.checkpoint.com/supportcenter/portal/user/anon/page/default.psml/media-type/html?action=portlets.DCFileAction&eventSubmit_doGetdcdetails=&fileid=22824)

It can also be found in this project under `./tools/snx_install_linux30.sh`

## Start VPN

```shell
snx -d ; sleep 1; snx -s kra.kuka.com -u extern.first_name.last_name@kuka.com
```

Enter your password and then should come a push notification from your Microsoft Authenticator on your mobile.

## Stop VPN

```shell
snx -d
```

Test using:

```shell
ping wiki.rd.kuka.com
```

## Notes

- disconnect if there was a connection before