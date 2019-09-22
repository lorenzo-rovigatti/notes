# My notes

## Export `reveal.js` slides to `pdf`

* `cd /tmp`
* `npm install decktape`
* `/tmp/node_modules/decktape/decktape.js reveal --size='2048x1536' file:///home/lorenzo/PATH/TO/FILE/presentazione.pdf /tmp/slides.pdf`

## Setup `zsh`

```
sudo apt install zsh
sudo apt-get install powerline fonts-powerline
git clone https://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```

Install custom plugins

```
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
```

Edit the .zshrc file to change stuff. Decomment the line

```
DISABLE_MAGIC_FUNCTIONS=true
```

Add the following lines:

```
ZSH_THEME="agnoster"

plugins=(
        git
        zsh-autosuggestions
        zsh-syntax-highlighting
        colored-man-pages
        dirhistory
        )

ZSH_AUTOSUGGEST_MANUAL_REBIND=1

alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
alias rm='rm -i'
alias ssh='ssh -Y'
```

Edit your terminal's appearance so that the default text is not black. For instance, using solarized's light them works well.

Make `zsh` the default shell:

```
chsh -s /bin/zsh

```

## Use `sudo` passwordlessly

Run `sudo visudo` and add the following line:

```
my_user ALL=(ALL:ALL) PASSWD:ALL
```

where `my_user` is the user's username. The same effect can be obtained by running the following command:

```bash
echo 'user ALL=(ALL:ALL) PASSWD:ALL' | sudo EDITOR='tee -a' visudo
```

---

## parallel-ssh cheat sheet

* Use `-o DIR` to write the `stdout` of each node to a different file in the folder `DIR`
* `-e DIR` does the same for the `stderr`
* Use `-t 0` to disable timeouts
* Use `-x '-t -t'` to run pseudo-TTY that allow using `sudo` commands

---

## Removing sensitive data from a git repository

There are many guides and tutorials online, but here I will report a very straightforward recipe taken from [here](http://palexander.posthaven.com/remove-a-password-from-gits-commit-history-wi):

```bash
# Sync with the remote master
git pull

# Force your clone to look like HEAD
git reset --hard

# AGAIN, A WARNING: This can really break stuff!

# Run your filter branch command, replacing all instances of "old_text" with "new_text"
# The example looks for python files ("*.py"), you can change this to match your needs
git filter-branch --tree-filter 'git ls-files -z "*.py" |xargs -0 perl -p -i -e "s#(old_text)#new_text#g"' -- --all

# Overwrite your master with local changes
git push origin master --force
```

## Setting up a Cisco VPN on kubuntu

1. `sudo apt-get install network-manager-vpnc`
2. If the system connection is not managed by `network-manager`, open the `/etc/NetworkManager/NetworkManager.conf` file, set `managed=true` and use the `[ifupdown]` connection.
3. There are some issues with the gnome keyring. When using KDE, these issues can be overcome by saving the VPN passwords for all the users.

---

## Setting up NFS

[NFS](https://en.wikipedia.org/wiki/Network_File_System) (Network File System) is a protocol that makes it possible to remotely (and transparently) access a file system over the network. Here I will explain how to setup a remote machine (aka the *server*) so that a local machine (aka the *client*) can have access to one or more of its directories.

 1. Install the NFS server. On Ubuntu, this boils down to installing the `nfs-kernel-server` package.
 2. Open (with root privilegies) the `/etc/exports` file. Add a line for each folder you want to export. The syntax is
 
     ```PATH_TO_DIRECTORY  CLIENT(OPTIONS)```
     
     Note the lack of whitespace between CLIENT and the open parenthesis. The CLIENT field can be an IP address or a DNS name and can contain wildcards (*e.g.* `*` or `?`). The list of available options depend on the NFS server version and can be found online (for example [here](https://linux.die.net/man/5/exports)). Valid configuration examples are
     
     ```/home/lorenzo/RESULTS 192.168.0.10(rw,no_root_squash)```
     ```/home/lorenzo/RESULTS *(ro,no_root_squash)```
     
     The above lines export the `/home/lorenzo/RESULTS` directory in read/write mode for the client identified by the IP address `192.168.0.10` and in read-only mode for everybody else.
     
 3. Restart the NFS server. On many Linux distros this can be done with the command `sudo service nfs-kernel-server restart`
 4. On the client, install the NFS client (the `nfs-common` package on Ubuntu)
 5. The remote filesystem can be mounted either manually (`sudo mount 192.168.0.1:/home/lorenzo/RESULTS /home/lorenzo/SERVER_RESULTS`) or automatically by adding a line like this to the `/etc/fstab` file:
 
     ```192.168.0.1:/home/lorenzo/RESULTS /home/lorenzo/SERVER_RESULTS nfs rw,user,auto,exec,bg,retry=10 0 0```
     
     and then using `sudo mount -a`.  Here I assumed that the server has IP address `192.168.0.1` and that the directory `/home/lorenzo/CLIENT_RESULTS` exists on the client. The `bg` option tells `mount` to try to mount the drive in the background, while `retry=10` tells it to keep trying mounting it for 10 minutes.
     
**Nota Bene:** on default installations, the shared directories are owned by the user and group identified by the `uid` and `gid` of the server, respectively. If the user on the client has a different `uid` and/or `gid`, its access to these directories may be limited. If this is the case (and if you are sure there will be no issues arising from such a drastic change) the best course of action is to change the user and group ids on either machine in order to make them match. This can done with the `usermod` command (*e.g.* `usermod -g 1110 -u 1205 lorenzo` will change lorenzo's `gid` and `uid` to 1110 and 1205, respectively). Note that `usermod` will change the ownership of all the files and directories that are in the user's home directory and are owned by them. The ownership of any other file or directory must be fixed manually.

---

## Setting up slurm

Many people recommend to compile slurm from the source, but if you do not need the cutting-edge version just use the one packaged for your distro. Once you have installed it consider the next few points:

1. Use [this form](https://slurm.schedmd.com/configurator.html) to assist in the generation of the `slurm.conf` file.
2. On my Ubuntu box I had to run `/usr/sbin/create-munge-key` (with root permissions) to make it possible to start `munged`. Note that the generated key file, which by default is `/etc/munge/munge.key`, should be then copied over to all the nodes of the cluster.
3. Slurm requires a mail program. This can be set in the `slurm.conf` file with the `MailProg` key which, if not set, points to `/bin/mail`. The simplest way of setting up a mailserver is to install postfix and makes it relay email to another server (see for example [here](https://linode.com/docs/email/postfix/postfix-smtp-debian7/)).

Notes:

* **The clock of all nodes must be synchronized!** This can be done on a per-node basis by issuing `sudo ntpdate -s ntp.ubuntu.com`
* If slurm stops sending emails, remember that we are using the CNR mail which requires a change of password every $X$ (maybe 6?) months!

---

## Making movies

We would like to generate a movie out of a (maybe very long) list of png images. We first create a file containing the list of png files sorted according to the order with which they should appear in the movie. If the names of the files are `img-1.png`, `img-2.png`, *etc.*, this can be accomplished by typing

```bash
$ ls -1v img-*.png > list.dat
```

The `-v` option is a GNU extension that tells `ls` to order entries according to the natural sorting of the numbers contained in the entry names. If on your system this option is not supported or it does something else (this might be the case on BSD/OSX systems), then you can use the following command

```bash
$ ls -1 img-*.png | sort -n -t'-' -k 2 > list.dat
```

`-n` applies natural sorting, `-t'-'` tells sort to use dashes as the separators to split the entries it acts on into fields and `-k 2`  to sort according to the value of the second field

A movie of width WIDTH, height HEIGHT and framerate FPS can now be generated by using `mencoder` with the following options:

```bash
$ mencoder mf://@list.dat -mf w=WIDTH:h=HEIGHT:fps=FPS:type=png -ovc xvid -xvidencopts bitrate=200 -o output.avi
```

The `-ovc xvid` controls the type of output and may not be suitable on all systems (or for all the purposes). The `-xvidencopts bitrate=200` option sets a sort of *overall* quality of the output.

---

## Adding a fading, rounded border to figures with GIMP

1. Select the whole picture (`ctrl + a`)
2. Shrink the border by an appropriate amount of pixels (Select &rarr; Shrink). I would say no more than 10% of the figure size.
3. Round the selection (Select &rarr; Rounded Rectangle)
4. Invert the selection (`ctrl + i`)
5. Feather the border (Select &rarr; Feather). I usually choose a number similar to the one used in step 2.
6. Delete the selected region (press delete)
7. Turn white into alpha (Colours &rarr; Colour to Alpha)
8. Create a new layer (same width and height as the figure) and put it underneath all other layers
9. Fill the layer with the background colour you want your picture to fade to at the border (usually black)

---

## Installing a PXE server

We have a machine that is connected to a LAN throught an ethernet port. 

### Second ethernet port

Let's say our machine connects to the outside world through the ethernet port `enp5s0` on the network `192.168.0.0`with IP `192.168.0.1` and to the client machine through `enp6s0` on the network `192.168.1.0` with IP `192.168.1.1`. We first add the following line to the hosts file:

```bash
echo "192.168.1.1 pxe pxe.local" >> /etc/hosts
```

### DHCP server

There is no DHCP on the LAN. We first install a DHCP server:

```bash
apt-get install isc-dhcp-server
```

Configure it by editing the `/etc/dhcp/dhcpd.conf` file:

```
ddns-update-style none;
DHCPDARGS=enp6s0;
default-lease-time 86400;
max-lease-time 604800;
authoritative;

subnet 192.168.1.0 netmask 255.255.255.0 {
        range 192.168.1.10 192.168.1.30;
        filename "pxelinux.0";
        option subnet-mask 255.255.255.0;
}
```

Restart it:

```bash
systemctl enable isc-dhcp-server
systemctl restart isc-dhcp-server
```

### TFTP server

The Trivial File Transfer Protocol makes it possible for PXE to transfer files over the network. Install it with

```
apt-get install tftpd-hpa
```

Configure it by editing `/etc/default/tftpd-hpa`:

```
RUN_DAEMON="yes"
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/tftpboot"
OPTIONS="-l -s /tftpboot"
TFTP_ADDRESS="[::]:69"
TFTP_OPTIONS="--secure"
```

Create the boot folder, set the right permissions and start the TFTP daemon:

```
mkdir -p /tftpboot/preseed
chmod -R 777 /tftpboot
systemctl enable tftpd-hpa
systemctl restart tftpd-hpa
```

### PXE configuration

Download the Ubuntu netboot installer, untar it and copy the relevant files to the boot folder:

```bash
wget http://archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/current/images/netboot/netboot.tar.gz
mkdir netboot
cd netboot
tar xvf ../netboot.tar.gz
cp -a ubuntu-installer /tftpboot
cd ubuntu-installer
cp amd64/pxelinux.0 /tftpboot/
cp amd64/boot-screens/ldlinux.c32 /tftpboot/
mkdir /tftpboot/boot-screens
cd boot-screens
cp libcom32.c32 libutil.c32 vesamenu.c32 /tftpboot/boot-screens/
mkdir /tftpboot/pxelinux.cfg
cd /tftpboot/pxelinux.cfg
ln -s ../boot-screens/syslinux.cfg default
```

Put the following lines in the `/tftpboot/boot-screens/syslinux.cfg`:

```
path boot-screens
include boot-screens/menu.cfg
default boot-screens/vesamenu.c32
prompt 0
timeout 100
```

Setup the Boot Menu in the `/tftpboot/boot-screens/menu.cfg` file:

```
menu hshift 13
menu width 49
menu margin 8
menu tabmsg
menu title Installer boot menu
label auto-ubuntu-18.04
menu label ^Ubuntu 18.04 automated install
kernel ubuntu-installer/amd64/linux
append auto=true priority=critical vga=788 initrd=ubuntu-installer/amd64/initrd.gz preseed/
url=tftp://192.168.1.1/preseed/Ubuntu16.cfg preseed/interactive=false
menu begin ubuntu-16.04
menu title Ubuntu 16.04
label mainmenu
menu label ^Back..
menu exit
include ubuntu-installer/amd64/boot-screens/menu.cfg
menu end
```

Add the following lines to the `/tftpboot/preseed/Ubuntu18.cfg` file:

```
d-i debian-installer/locale string en_US
d-i debian-installer/language string en
d-i debian-installer/country string IT
d-i keyboard-configuration/xkb-keymap select it

d-i passwd/user-fullname string
d-i passwd/username string lorenzo
d-i passwd/root-password password sucker17
d-i passwd/root-password-again password sucker17
d-i passwd/user-password password sucker17
d-i passwd/user-password-again password sucker17
d-i user-setup/allow-password-weak boolean true

d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string unassigned-hostname
d-i netcfg/get_domain string unassigned-domain

d-i mirror/country string manual
d-i mirror/http/hostname string http://en.archive.ubuntu.com
d-i mirror/http/directory string /ubuntu
d-i mirror/http/proxy string

d-i clock-setup/utc boolean true
d-i clock-setup/ntp boolean true
d-i time/zone string Europe/Rome

d-i partman-auto/disk string /dev/sda
d-i partman-auto/method string lvm
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-auto-lvm/guided_size string max
d-i partman-auto/choose_recipe select multi
d-i partman/default_filesystem string ext4
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true

d-i grub-installer/grub2_instead_of_grub_legacy boolean true
d-i grub-installer/only_debian boolean true
d-i grub-installer/bootdev string /dev/[sv]da

d-i pkgsel/update-policy select none
d-i pkgsel/include string unity ubuntu-desktop openssh-server
d-i pkgsel/updatedb boolean true
d-i base-installer/kernel/override-image string linux-server
tasksel tasksel/first multiselect standard
d-i pkgsel/include string lsb-release openssh-server screen sysstat wget ldap-utils docker.io
popularity-contest popularity-contest/participate boolean false
d-i finish-install/reboot_in_progress note
d-i apt-setup/services-select multiselect security
d-i apt-setup/security_host string security.ubuntu.com
d-i apt-setup/security_path string /ubuntu

d-i preseed/late_command string \
in-target /bin/bash -c 'echo -e "192.168.1.1 pxe pxe.local" >> /etc/hosts'

d-i preseed/early_command string /bin/killall.sh; /bin/netcfg
d-i finish-install/reboot_in_progress note
```
