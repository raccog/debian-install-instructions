# Debian Server Installation Notes

While bringing up a Debian 11.3.0 server on one of my PCs, I want to record the process so that it can be easily reproduced. My goal is to record every single action that I make while setting up the server.

## Table of Contents

1. [Debian Installer](#debian-installer)
	1. [Partitioning Method](#partitioning-method)
	2. [Software to Install](#software-to-install)
2. [Users](#users)
	1. [List of Users](#list-of-users)
	2. [User Setup](#user-setup)
	3. [root](#root)
	4. [ryan](#Ryan)
	5. [backup-manager](#backup-manager)
3. [Storage](#storage)
	1. [Archives](#archives)
	2. [fstab](#fstab)
4. [Installing Packages](#installing-packages)
	1. [List of Packages](#list-of-packages)
	2. [neovim](#neovim)
	3. [sudo](#sudo)
5. [Running Services](#running-services)
	1. [List of Services](#list-of-services)
	2. [sshd](#sshd)
	3. [rsync](#rsync)

## Debian Installer

1. Language: `English`
2. Location: `United States`
3. Keymap: `American English`
4. Hostname: *Anything will do*
5. Domain Name: *Empty*
6. Root Password: *Long password*
7. Full Name for New User: `Ryan`
8. Username for Account: `Ryan`
9. Password for New User: *Common password*
10. Time Zone: `Eastern`
11. Partitioning Method: *See [Partitioning Method](#partitioning-method)*
12. Debian Archive Mirror Country: `United States`
13. Debian Archive Mirror: `deb.debian.org`
14. HTTP Proxy Info: *Empty*
15. Participate in Package Usage Survey: `No`
16. Choose Software to Install: *See [Software to Install](#software-to-install)*
17. Is the System Clock Set to UTC?: `Yes`

Installation is now complete. Remove the USB installer drive and reboot the system.

### Partitioning Method

If using an entire disk for root of filesystem with encryption:

1. Select `Guided - use entire disk and set up encrypted LVM`
2. Choose disk for root
3. Select `Separate /home, /var, and /tmp partitions`
4. Wait for disk to be written with random bits (takes a while)
5. Set encryption password
6. Set volume group size to fill the disk
7. Finish partitioning and write to disk

Else if using an entire disk for root of filesystem without encryption:

1. Select `Guided - use entire disk`
2. Choose disk for root
3. Select `Separate /home, /var, and /tmp partitions`
4. Finish partitioning and write to disk

Else if using manual partitioning:

*TODO: Write down steps for manual partitioning*

1. Select `Manual`
2. Create partitions manually (ideally encrypting the root filesystem).

### Software to Install

Always choose `standard system utilities`.

If using a desktop environment, choose the following:

* `Debian desktop environment`
* `Xfce`

If running a web server, choose `web server`.

If running a ssh server, chose `SSH server`

## Users

These instructions describe the general user configuration used throughout multiple systems as well as the steps for setting up users.

My main user is named `ryan`, but this name can be substituted for any other name.

### List of Users

* [root](#root)
* [ryan](#ryan)
* [backup-manager](#backup-manager)

### User Setup

Users `root` and `ryan` are properly setup by default with `uid=0` and `uid=1000` respectively. User `backup-manager` does not exist by default.

The following commands setup each user. User `ryan` will be added to the `users` group. User `backup-manager` will be created with a new home directory.

```
usermod -aG users ryan
useradd -m backup-manager
```

### root

Access to the `root` user will be blocked off as much as possible to prevent accidental or malicious changes to the system.

#### Access

* Cannot login through ssh
* Has a very strong password

#### Sudo Access

Users in the `sudo` group have permission to run commands as root using `sudo <COMMAND>`. Note that these users need to enter their password to run `sudo <COMMAND>`. Only user `ryan` is in the `sudo` group for now. (NOTE: Maybe give the user `ryan` sudo permissions without the group if no other user will need the permissions)

User `backup-manager` can run the command `sudo rsync` without needing a password. This is used for storage backup purposes. User `backup-manager` cannot run any other command under sudo except for `rsync`.

### ryan

User `ryan` is essentially user `root` when the `sudo` package is installed because `ryan` has permission run commands as `root`. However, this functionality is locked behind `ryan`'s password.

#### Access

* Can login through ssh
* Has a decently strong password

#### Permissions

* Can only access files inside its own home directory (unless using `sudo`)
* Can run `sudo <COMMAND>` to run a command as `root`.

### backup-manager

The `backup-manager` user is used to provide client access to storage backups; both reads and writes.

#### Access

* Can login through ssh
* Can have any password

#### Permissions

* Can only access files inside its own home directory
* Can write files to the `/archive` directory when a client sends files to the rsync server

*TODO: Move the following line to rsync service section*

The backup files written through rsync has identical permissions, ownership, and timestamps to the original files.

## Storage

As each system has different priorities and devices, storage setup will be very different for each system. General actions and settings will be listed here.

### Archives

Backup archives should be stored in `/archive`.

I have no requirements or preferences for how `/archive` should be structured. It could be a single directory of system backups, or a directory tree of sorted backups. It could also contain multiple partitions. Generally each partition should be the same filesystem, but this shouldn't matter too much.

My current server has a single 2.7 TB partition with an ext4 filesystem mounted to `/archive`. However, I plan to use multiple physical partitions in `/archive` for my desktop system, so I will see how I want to set that up eventually.

### fstab

Partitions listed in `/etc/fstab` are auto-mounted on system boot.

To match device partitions to UUIDS:

`lsblk -o NAME,SIZE,UUID`

For adding an ext4 partition to fstab:

```
echo "UUID={UUID} /path/to/mount ext4 defaults 0 0" >> /etc/fstab
```

## Installing Packages

These instructions show the installation and configuration for every package. Only the packages that need explicit configuration changes have their own sub-sections.

Note that most configuration files require a working ssh key connected to my Github account to clone the config repository.

### List of Packages

The following packages will be installed using `apt install <PACKAGE>`:

* [neovim](#neovim)
* [sudo](#sudo)
* rsync

If running an ssh server:

* openssh-server

### neovim

My default text editor.

#### Note

This package is used as a fallback, as neovim will also be installed from source. This is because debian currently supports neovim version `0.4` while version `0.6` is needed for a default runtime with `toml` syntax highlighting. I need the newer version because I edit Rust toml files often.

Thus, the obsolete version is only used if my installation from source is not available for some reason.

#### Configuration

Config directory: `~/.config/nvim/`

My config files are hosted on Github.

Make sure to install in all `/home/<USER>` directories for users that use neovim (possibly including `root` at `/root`).

#### Environment Variable

*TODO: Move the following line to their respective shell's section*

Set `nvim` as `EDITOR` environment variable:

* bash: 

```
echo 'export EDITOR=nvim' >> ~/.profile
```

* fish: 

```
set -Ux EDITOR nvim
```

### sudo

Creates extra permissions for running processes as user `root`.

#### Configuration

Add `ryan` to the `sudo` group:

```
usermod -aG sudo ryan
```

## Running Services

These instructions are for setting up individual services and their purposes.

Note that most configuration files require a working ssh key connected to my Github account to clone the config repository.

### List of Services

Services that will be enabled using `systemctl start <SERVICE>` and `systemctl enable <SERVICE>`:

* [sshd](#sshd)
* [rsync](#rsync)

### sshd

Allow clients to access this machine through the ssh protocol.

#### Configuration

Config files: 

* `/etc/ssh/sshd_config`
* `~/.ssh/authorized_keys`

The configuration process needs to be done in the following steps:

1. Start running the `sshd` service with the default config file.
2. Create ssh keys for each potential client and send them to the ssh server using `scp`.
3. Add these keys to `~/.ssh/authorized_keys` file for each user that can login through ssh.
4. Install custom config file from Github to allow ssh keys and disable passwords.

#### Client Configuration

On the client machines, do not forget to add the created keys to the `~/.ssh/config` file.

### rsync

Allow clients to read and write backups to the server's local storage.

#### Configuration

Config file: `/etc/rsyncd.conf`

Example config file that allows reads/writes to/from the `/archive` directory:

```
[archive]
        path = /archive
        comment = Backup Archive
        read only = no
        uid = 0
        gid = 0
```

#### Sudo Permissions

Run `visudo` and add the following line:

```
backup-manager ALL= NOPASSWD:/usr/bin/rsync
```

This allows user `backup-manager` to run only `sudo rsync` or `sudo /usr/bin/rsync`. Trying to run any other command with sudo will fail for this user. To backup a directory from a client machine, an ssh key for `backup-manager` is needed.

#### Backing Up Storage From a Client

From a client machine, the following command can be run to backup a directory on the client machine to the `/archive` directory on the server:

```
sudo rsync -avP --rsync-path="sudo rsync" -e "ssh" /path/to/backup rsync://backup-manager@<SERVER_ADDR>/archive
```

The command `sudo` is used to ensure file permissions and ownership stay unaltered.