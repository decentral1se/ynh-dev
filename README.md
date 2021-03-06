# ynh-dev - Yunohost dev environment manager

Please report issues on the following repository:

> https://github.com/yunohost/issues

# Table Of Contents

- [Introduction](#introduction)
  * [Local Development Path](#local-development-path)
  * [Remote Development Path](#remote-development-path)

- [Local Development Environment](#local-development-environment)
  * [1. Setup `ynh-dev` and the development environment](#1-setup-ynh-dev-and-the-development-environment)
  * [2. Manage YunoHost's dev LXCs](#2-manage-yunohosts-dev-lxcs)
  * [3. Development and container testing](#3-development-and-container-testing)
  * [4. Testing the web interface](#4-testing-the-web-interface)
  * [Advanced: using snapshots](#advanced-using-snapshots)
  * [Alternative: Only Virtualbox](#alternative-using-only-virtualbox)

- [Remote Development Environment](#remote-development-environment)
  * [1. Setup your VPS and install YunoHost](#1-setup-your-vps-and-install-yunohost)
  * [2. Setup `ynh-dev` and the development environment](#2-setup-ynh-dev-and-the-development-environment)
  * [3. Develop and test](#3-develop-and-test)

- [Further Resources](#further-resources)

---

# Introduction

`ynh-dev` is a CLI tool to manage your local development environment for YunoHost.

This allow you to develop on the various repositories of the YunoHost project.

In particular, it allows you to:

 * Create a directory with a clone of each repository of the YunoHost project
 * Replace Yunohost debian packages with symlinks to those git clones

Because there are many diverse constraints on the development of the Yunohost
project, there is no "official" one-size-fits-all development environment.
However, we do provide documentation for what developers are using now in
various circumstances.

Please keep this in mind when reviewing the following options with regard to
your capacities and resources when aiming to setup a development environment.

`yhn-dev` can be used for the following scenarios:

## Local Development Path

Yunohost can be developed on using a combination of the following technologies:

  * Git (any version is sufficient)
  * Vagrant (>= 2.x)
  * LXC (>= 3.x)
  * Vagrant-LXC (>= 1.4.x)

LXC is typically lightweight but you may find the initial setup complex
(in particular network configuration). Alternatively, you may be able
to setup a local environnement using Virtualbox which is kinda more
resource-hungry :

  * Virtualbox (>= 6.x)
  * Vagrant-virtualbox (>= ?.?.?)

Please keep in mind that these versions may not be available on your OS
distribution and you may be required to install them as binary or from source.
There are no guarantees of stability on newer major versions.

This local development path allows to work without an internet connection,
but be aware that it will *not* allow you to easily test your email stack
or deal with deploying SSL certificates, for example, as your machine is
likely to not be exposed on the internet. A remote machine should be used
for these cases.

Depending on your needs, this setup can be very convenient.

If choosing this path, please keep reading at the [local development
environment](#local-development-environment) section.

Please note, there is also a setup guide for a local development path which
does not require LXC. Please see the [Alternative: Only Virtualbox](#alternative-using-only-virtualbox)
section for more.

## Remote Development Path

Yunohost can be deployed as a typical install on a remote VPS. You can then use
`ynh-dev` to configure a development environment on the server.

This method can potentially be faster than the local development environment
assuming you have familiarity with working on VPS machines, if you always have
internet connectivity when working, and if you're okay with paying a fee. It
is also a good option if the required system dependencies (Vagrant, Virtualbox,
etc.) are not easily available to you on your distribution.

Please be aware that this method should **not** be used for a end-user facing
production environment.

If choosing this path, please keep reading at the [remote development
environment](#remote-development-environment) section.

# Local Development Environment

Here is the development flow:

1. Setup `ynh-dev` and the development environment
2. Manage YunoHost's development LXCs
3. Develop on your local host and testing in the container

## 1. Setup `ynh-dev` and the development environment

First you need to install the system dependencies.

`ynh-dev` essentially requires Git, Vagrant, and and the LXC ecosystem. Please
see the [local development path](#local-development-path) section for some idea
of the versions required.

Please consider using the [latest Vagrant version from their website](https://www.vagrantup.com/downloads.html), distribution versions can include weird bugs that have been fixed upstream. If you still prefer to do that, here are the instructions:

The following commands should work on **Linux Mint 19** (and possibly on any Debian Stretch?):

```bash
$ sudo apt update
$ sudo apt install git vagrant lxc-templates lxctl lxc cgroup-lite redir bridge-utils libc6 debootstrap libvirt-dev
$ vagrant plugin install vagrant-lxc
$ echo "cgroup        /sys/fs/cgroup        cgroup        defaults    0    0" | sudo tee -a /etc/fstab
$ sudo mount /sys/fs/cgroup
$ lxc-checkconfig
$ echo "veth" | sudo tee -a /etc/modules
```
If you have install libvirtd, you need to stop it and kill dnsmasq libvirtd process, to avoid conflict with dhcp. If you don't ynh-dev start will fail because the lxc container won't be able to get an ip.

On **Debian Buster**, I had to re-patch the driver.rb of vagrant-lxc plugin with [this version](https://raw.githubusercontent.com/fgrehm/vagrant-lxc/2a5510b34cc59cd3cb8f2dcedc3073852d841101/lib/vagrant-lxc/driver.rb) (especially the `roofs_path` function). I also had to install `apparmor` then `systemctl restart apparmor` for `lxc-start` to work.

Also check instruction on https://feeding.cloud.geek.nz/posts/lxc-setup-on-debian-stretch/.

If you run **Archlinux**, this page should be quite useful to setup LXC: https://github.com/fgrehm/vagrant-lxc/wiki/Usage-on-Arch-Linux-hosts

On **both Debian and Archlinux**, typically `/etc/default/lxc-net` and `/etc/lxc/default.conf` should look like this :

```
$ cat /etc/default/lxc-net
USE_LXC_BRIDGE="true"
LXC_BRIDGE="lxcbr0"
LXC_ADDR="10.0.3.1"
LXC_NETMASK="255.255.255.0"
LXC_NETWORK="10.0.3.0/24"
LXC_DHCP_RANGE="10.0.3.2,10.0.3.254"
LXC_DHCP_MAX="253"

$ cat /etc/lxc/default.conf
lxc.net.0.type = veth
lxc.net.0.link = lxcbr0
lxc.net.0.flags = up
lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
```

On **Debian Buster**, for backup stuff to work correctly with apparmor, I also had to add:

```
mount options=(ro, remount, bind, rbind)
mount options=(ro, remount, bind, relatime)
```

to `/etc/apparmor.d/lxc/lxc-default-cgns` and restart the apparmor service.

Then, go into your favorite development folder and deploy `ynh-dev` with:

```bash
curl https://raw.githubusercontent.com/yunohost/ynh-dev/master/deploy.sh | bash
```

This will create a new `ynh-dev` folder with everything you need inside.

In particular, you shall notice that there are clones or the various git
repositories. In the next step, we shall start a LXC and 'link' those folders
between the host and the LXC.

## 2. Manage YunoHost's dev LXCs

When ran on the host, the `./ynh-dev` command allows you to manage YunoHost's dev LXCs.

First, you might want to start a new LXC with:

```bash
$ cd ynh-dev  # if not already done
$ ./ynh-dev start
```

This should download an already built LXC from `build.yunohost.org`. If this does not work (or the LXC is outdated), you might want to (re)build a fresh LXC locally with `./ynh-dev rebuild`.

After starting the LXC, you should be automatically SSH'ed inside. If you later disconnect from the LXC, you can go back in with `./ynh-dev ssh`.

Later, you might want to destroy the LXC. You can do so with `./ynh-dev destroy`.

## 3. Development and container testing

After SSH-ing inside the container, you should notice that the *directory* `/ynh-dev` is a shared folder with your host. In particular, it contains the various git clones `yunohost`, `yunohost-admin` and so on - as well as the `./ynh-dev` script itself.

Inside the container, `./ynh-dev` can be used to link the git clones living in the host to the code being ran inside the container.

For instance, after running:

```bash
$ ./ynh-dev use-git yunohost
```

The code of the git clone `'yunohost'` will be directly available inside the container. Which mean that running any `yunohost` command inside the container will use the code from the host... This allows to develop with any tool you want on your host, then test the changes in the container.

The `use-git` action can be used for any package among `yunohost`, `yunohost-admin`, `moulinette` and `ssowat` with similar consequences. You might want to run use-git several times depending on what you want to develop precisely.

***Note***: The `use-git` operation can't be reverted now. Do **not** do this in production.

## 4. Testing the web interface

You should be able to access the web interface via the IP address of the container. The IP can be known from inside the container using either from `ip a` or with `./ynh-dev ip`.

If you want to access to the interface using the domain name, you shall tweak your `/etc/hosts` and add a line such as:

```bash
111.222.333.444 yolo.test
```

Note that `./ynh-dev use-git yunohost-admin` has a particular behavior: it starts a `gulp` watcher that shall re-compile automatically any changes in the javascript code. Hence this particular `use-git` will keep running until you kill it after your work is done.

## Advanced: using snapshots

Vagrant is not well integrated with LXC snapshots.

However, you may still use `lxc-snapshot` directly to manage snapshots.

## Alternative: Using Only Virtualbox

A Vagrant and Virtualbox (without LXC) guide is provided on another branch of
this repository. This is a known working setup used by some developers. Please
see the ["virtualbox" branch](https://github.com/YunoHost/ynh-dev/tree/virtualbox#develop-on-your-local-machine)
for more.

# Remote Development Environment

Here is the development flow:

1. Setup your VPS and install YunoHost
2. Setup `ynh-dev` and the development environment
3. Develop and test

## 1. Setup your VPS and install YunoHost

Setup a VPS somewhere (e.g. Scaleway, Digital Ocean, etc.) and install YunoHost following the [usual instructions](https://yunohost.org/#/install_manually).

Depending on what you want to achieve, you might want to run the postinstall right away - and/or setup a domain with an actually working DNS.

## 2. Setup `ynh-dev` and the development environment

Deploy a `ynh-dev` folder at the root of the filesystem with:

```
$ cd /
$ curl https://raw.githubusercontent.com/yunohost/ynh-dev/master/deploy.sh | bash
$ cd /ynh-dev
```

## 3. Develop and test

Inside the VPS, `./ynh-dev` can be used to link the git clones to actual the code being ran.

For instance, after running:

```bash
$ ./ynh-dev use-git yunohost
```

Any `yunohost` command will run from the code of the git clone.

The `use-git` action can be used for any package among `yunohost`, `yunohost-admin`, `moulinette` and `ssowat` with similar consequences.

# Further Resources

  * [yunohost.org/dev](https://yunohost.org/dev)
