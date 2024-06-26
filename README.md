## Docker on Android 🐋📱

All packages, except for Tini have been added to [termux-root](https://github.com/termux/termux-root-packages). To install them, simply `pkg install root-repo && pkg install docker`. This will install the whole docker suite, left only Tini to be compiled manually.

---

# Summary

1. [Intro](#1-intro)
2. [Building](#2-building)
    1. [Rooting](#21-rooting)
    2. [Kernel](#22-kernel)
        1. [General compiling instructions](#221-general-compiling-instructions)
        2. [Modifications](#222-modifications)
        3. [Patching](#223-patching)
    3. [Docker](#23-docker)
        1. [dockercli](#231-dockercli)
        2. [dockerd](#232-dockerd)
        3. [tini](#233-tini)
        4. [libnetwork](#234-libnetwork)
        5. [containerd](#235-containerd)
        6. [runc](#236-runc)
3. [Running](#3-running)
    1. [Caveats](#31-caveats)
        1. [Internet access](#311-internet-access)
        2. [Shared volumes](#312-shared-volumes)
    2. [GUI](#32-gui)
        1. [X11 Forwarding](#321-x11-forwarding)
        2. [VNC server within the container](#322-vnc-server-within-the-container)
    3. [Steam (work in progress)](#33-steam-work-in-progress)
4. [Attachments](#4-attachments)
    1. [Kernel patches](#41-kernel-patches)
    2. [docker-cli patches](#42-docker-cli-patches)
    3. [dockerd patches](#43-dockerd-patches)
    4. [containerd patches](#44-containerd-patches)
5. [Aknowledgements ](#5-aknowledgements)
6. [Final notes](#6-final-notes)

---

# 1. Intro

This tutorial presents a step by step guide on how to run docker containers directly on Android. By directly I mean there's no VM involved nor chrooting inside a GNU/Linux rootfs. This is docker purely in Android. Yes, ***it is*** possible.

Bear in mind that you'll have to root your phone, mess with and compile your phone's kernel and docker suite. So, be prepared to get your hands dirty.

# 2. Building 

## 2.1. Rooting

This step is pretty device specific, so there's no way to write a generic tutorial here. You'll need to google for instructions for your device and follow them.

Just be aware that you may lose your phone's warrant and all your data will be erased after unlocking the bootloader, so make a backup of your important stuff.

## 2.2. Kernel

### 2.2.1. General compiling instructions

Compiling the phone's kernel is also device specific, but some major tips may help you out.

First, google about instructions for your phone. Start by compiling the kernel without any modification. Flash it and hope for the best. If everything went well, then you can proceed to the modifications.

Note that flashing the kernel won't erase any data in your phone. The worst that can happen is you get stuck in a boot loop. In this case, you can flash a kernel that's known to be working or just flash a working ROM, since it contains a kernel with it. None of these operations erase any data in your phone.

### 2.2.2. Modifications

Now that you (hopefully) are able to compile the kernel, let's talk about what matters. Docker needs a lot of features that are disabled by default in Android's kernel.

To check the necessary features list, first install the [Termux app](https://github.com/termux/termux-app) in your phone. This is the terminal emulator that we're going to use throughout this guide. It has a package manager with many tools compiled for Android.

Next, open Termux and download a script to check your kernel:

```
$ pkg install wget
$ wget https://raw.githubusercontent.com/moby/moby/master/contrib/check-config.sh
$ chmod +x check-config.sh
$ sed -i '1s_.*_#!/data/data/com.termux/files/usr/bin/bash_' check-config.sh
$ sudo ./check-config.sh
```

Now, in your computer, open the kernel's configuration menu. This menu is a modified version of dialog, a ncurses window menu, in which you can enable and disable the kernel features. To look for some item in particular, you can press the `/` key and type the item name and hit Enter. This will show the description and location of the item.

For now, we want to enable the `Generally Necessary` items, the `Network Drivers` items and some `Optional Features`. For the `Storage Drivers` we'll be using the `overlay`.

### 2.2.3. Patching

Before compiling the kernel there are two files that need to be patched.

#### kernel/Makefile

The first one is the `kernel/Makefile`. Although not strictly necessary to modify this file, it will help by making it possible to check if your kernel has all the necessary features docker needs.

If you do not apply this patch, the output of the `check-config.sh` script used above won't be reliable after recompiling the kernel.

Check the [patch at the attachments section](#41-kernel-patches) and modify your Makefile accordingly.

#### net/netfilter/xt_qtaguid.c

This second file *needs to be patched* because of a bug introduced by Google. After you run any container, a seg fault will be generated due to a null pointer dereference and your phone will freeze and reboot. If you work at Google or know someone who does, warn him/her about it.

Check the [patch at the attachments section](#41-kernel-patches) and modify your xt_qtaguid.c accordingly.

---

Now that everything is setup, compile and flash the kernel. If you applied the Makefile patch, you'll see this warning everytime your phone boots:

![IMG_20210110_203818](https://user-images.githubusercontent.com/9597536/104138646-43f48000-539d-11eb-800d-b741b63c8bcd.jpg)

Don't worry though, this is a harmless warning remembering you that you're using a modified kernel.

## 2.3. Docker

See [Edit](#edit-).

Once you have a supported kernel, it's time to compile the docker suite. It's a suite because it's not just one program, but rather a set of different programs that we'll need to compile separately. So hands on.

Firts, let's install the packages we're gonna use to build docker in Termux:

```
$ pkg install go make cmake ndk-multilib tsu
```

Now we're ready to start compiling things. Create a work directory where the packages will be downloaded and built:

```
$ mkdir $TMPDIR/docker-build
$ cd $TMPDIR/docker-build
```

Download all the patches files into there and let's begin. All commands for the differents packages that'll be compiled next is meant to be executed inside this folder.

### 2.3.1. dockercli

See [Edit](#edit-).

This is the docker client, which will talk to the docker daemon. This package will compile a binary named `docker` and all docker man pages. To build and install it:

```
$ cd $TMPDIR/docker-build
$ wget https://github.com/docker/cli/archive/v20.10.2.tar.gz -O cli-20.10.2.tar.gz
$ tar xf cli-20.10.2.tar.gz
$ mkdir -p src/github.com/docker
$ mv cli-20.10.2 src/github.com/docker/cli
$ export GOPATH=$(pwd)
$ export VERSION=v20.10.2-ce
$ export DISABLE_WARN_OUTSIDE_CONTAINER=1
$ cd src/github.com/docker/cli
$ xargs sed -i 's_/var/\(run/docker\.sock\)_/data/docker/\1_g' < <(grep -R /var/run/docker\.sock | cut -d':' -f1 | sort | uniq)
$ patch vendor/github.com/containerd/containerd/platforms/database.go ../../../../database.go.patch.txt
$ patch scripts/docs/generate-man.sh ../../../../generate-man.sh.patch.txt
$ patch man/md2man-all.sh ../../../../md2man-all.sh.patch.txt
$ patch cli/config/config.go ../../../../config.go.patch.txt
$ make dynbinary
$ make manpages
$ install -Dm 0700 build/docker-android-* $PREFIX/bin/docker
$ install -Dm 600 -t $PREFIX/share/man/man1 man/man1/*
$ install -Dm 600 -t $PREFIX/share/man/man5 man/man5/*
$ install -Dm 600 -t $PREFIX/share/man/man8 man/man8/*
```

### 2.3.2. dockerd

See [Edit](#edit-).

The docker daemon is the most problematic binary that's gonna be compiled. It needs so many patches that's easier to modify the code in a batch with sed. Despite the need of modifying a lot of files, the modifications by themselfs are rather simple:

1. Substitute every occurrence of `runtime.GOOS` by the string `"linux"`;
2. Remove unneeded imports of the `runtime` lib.

By doing that, we are in essence spoofing our operating system as a Linux one: everytime the code would do the `runtime.GOOS == "linux"` comparison (which would become `"android" == "linux"`, and thus `false`) it will now do `"linux" == "linux"` and thus `true`.

To make the substitution across every file, we'll run a sed command. After that, some files will now give the extremely annoying unturnable-off go lang "feature" `imported and not used` error, because the only function these files were using from the `runtime` package was the `runtime.GOOS`. So, to fix it we'll use an horrible but simple solution: we'll keep trying to compile the code and at each failed attempt we'll fix the reported files till we get it to compile successfully.

```
$ cd $TMPDIR/docker-build
$ wget https://github.com/moby/moby/archive/v20.10.2.tar.gz -O moby-20.10.2.tar.gz
$ tar xf moby-20.10.2.tar.gz
$ cd moby-20.10.2
$ export DOCKER_GITCOMMIT=8891c58a43
$ export DOCKER_BUILDTAGS='exclude_graphdriver_btrfs exclude_graphdriver_devicemapper exclude_graphdriver_quota selinux exclude_graphdriver_aufs'
$ patch cmd/dockerd/daemon.go ../daemon.go.patch
$ xargs sed -i "s_\(/etc/docker\)_$PREFIX\1_g" < <(grep -R /etc/docker | cut -d':' -f1 | sort | uniq)
$ xargs sed -i 's_\(/run/docker/plugins\)_/data/docker\1_g' < <(grep -R '/run/docker/plugins' | cut -d':' -f1 | sort | uniq)
$ xargs sed -i 's/[a-zA-Z0-9]*\.GOOS/"linux"/g' < <(grep -R '[a-zA-Z0-9]*\.GOOS' | cut -d':' -f1 | sort | uniq)
$ (while ! IFS='' files=$(AUTO_GOPATH=1 PREFIX='' hack/make.sh dynbinary 2>&1 1>/dev/null); do if ! xargs sed -i 's/\("runtime"\)/_ \1/' < <(echo $files | grep runtime | cut -d':' -f1 | cut -c38-); then echo $files; exit 1; fi; done)
$ install -Dm 0700 bundles/dynbinary-daemon/dockerd $PREFIX/bin/dockerd-dev
```

A binary called dockerd-dev was compiled and installed, but in order to it run correctly, the cgroups need to be mounted. Since Android mounts the cgroups in a non standard location we need to fix this. To do so, a script named dockerd will be created that will mount crgoups in the correct path if needed and call dockerd-dev next.

```
$ cat << "EOF" > $PREFIX/bin/dockerd
#!/data/data/com.termux/files/usr/bin/bash

export PATH="${PATH}:/system/xbin:/system/bin"
opts='rw,nosuid,nodev,noexec,relatime'
cgroups='blkio cpu cpuacct cpuset devices freezer memory pids schedtune'

# try to mount cgroup root dir and exit in case of failure
if ! mountpoint -q /sys/fs/cgroup 2>/dev/null; then
  mkdir -p /sys/fs/cgroup
  mount -t tmpfs -o "${opts}" cgroup_root /sys/fs/cgroup || exit
fi

# try to mount cgroup2
if ! mountpoint -q /sys/fs/cgroup/cg2_bpf 2>/dev/null; then
  mkdir -p /sys/fs/cgroup/cg2_bpf
  mount -t cgroup2 -o "${opts}" cgroup2_root /sys/fs/cgroup/cg2_bpf
fi

# try to mount differents cgroups
for cg in ${cgroups}; do
  if ! mountpoint -q "/sys/fs/cgroup/${cg}" 2>/dev/null; then
    mkdir -p "/sys/fs/cgroup/${cg}"
    mount -t cgroup -o "${opts},${cg}" "${cg}" "/sys/fs/cgroup/${cg}" \
    || rmdir "/sys/fs/cgroup/${cg}"
  fi
done

# start the docker daemon
$PREFIX/bin/dockerd-dev $@
EOF
```

Make the script executable:

```
$ chmod +x $PREFIX/bin/dockerd
```

And finally configure some dockerd options:

```
$ mkdir -p $PREFIX/etc/docker
$ cat << "EOF" > $PREFIX/etc/docker/daemon.json
{
    "data-root": "/data/docker/lib/docker",
    "exec-root": "/data/docker/run/docker",
    "pidfile": "/data/docker/run/docker.pid",
    "hosts": [
        "unix:///data/docker/run/docker.sock"
    ],
    "storage-driver": "overlay2"
}
EOF
```

> **Warning:** dockerd will store all its files, like containers, images, volumes, etc inside the `/data/docker` folder, which means you'll lose everything if you format the phone (flash a ROM). This folder was chosen instead of storing things inside Termux installation folder, because dockerd fails when setting up the overlay storage driver there. It seems Android mounts the `/data/data` folder with some options that prevent overlayfs to work, or the filesystem doesn't support it.

### 2.3.3. tini

tini is an optional dependency of dockerd in case you want the `init` process to be the first process of the container being ran (for this use the `--init` flag when creating a container). Having `init` as the parent of all other proccess ensures that a proper clean up inside the container is made regarding zombie processes. For a detailed explanation on its benefits and when to use it, check here: https://github.com/krallin/tini/issues/8
 
```
$ cd $TMPDIR/docker-build
$ wget https://github.com/krallin/tini/archive/v0.19.0.tar.gz
$ tar xf v0.19.0.tar.gz
$ cd tini-0.19.0
$ mkdir build
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$PREFIX ..
$ make -j8
$ make install
$ ln -s $PREFIX/bin/tini-static $PREFIX/bin/docker-init
```

### 2.3.4. libnetwork

See [Edit](#edit-).

Another dockerd dependency needed when using the `-p` flag while creating a container:

```
$ cd $TMPDIR/docker-build
$ wget https://github.com/moby/libnetwork/archive/448016ef11309bd67541dcf4d72f1f5b7de94862.tar.gz
$ tar xf 448016ef11309bd67541dcf4d72f1f5b7de94862.tar.gz
$ mkdir -p src/github.com/docker
$ mv libnetwork-448016ef11309bd67541dcf4d72f1f5b7de94862 src/github.com/docker/libnetwork
$ export GOPATH="$(pwd)"
$ cd src/github.com/docker/libnetwork
$ go build -o docker-proxy github.com/docker/libnetwork/cmd/proxy
$ strip docker-proxy
$ install -Dm 0700 docker-proxy $PREFIX/bin/docker-proxy
```

### 2.3.5. containerd

See [Edit](#edit-).

This is a dockerd dependency. Some patches are needed to fix path locations, build the manuals correctly and compile extra binaries used by dockerd that are not build by default by the Makefile:

```
$ cd $TMPDIR/docker-build
$ wget https://github.com/containerd/containerd/archive/v1.4.3.tar.gz
$ tar xf v1.4.3.tar.gz
$ mkdir -p src/github.com/containerd
$ mv containerd-1.4.3 src/github.com/containerd/containerd
$ export GOPATH=$(pwd)
$ cd src/github.com/containerd/containerd
$ xargs sed -i "s_\(/etc/containerd\)_$PREFIX\1_g" < <(grep -R /etc/containerd | cut -d':' -f1 | sort | uniq)
$ patch runtime/v1/linux/bundle.go ../../../../bundle.go.patch.txt
$ patch runtime/v2/shim/util_unix.go ../../../../util_unix.go.patch.txt
$ patch Makefile ../../../../Makefile.patch
$ patch platforms/database.go ../../../../database.go.patch.txt
$ patch vendor/github.com/cpuguy83/go-md2man/v2/md2man.go ../../../../md2man.go.patch.txt
$ BUILDTAGS=no_btrfs make -j8
$ make -j8 man
$ DESTDIR=$PREFIX make install
$ DESTDIR=$PREFIX/share make install-man
```

Lastly, some configurations:

```
$ mkdir -p $PREFIX/etc/containerd
$ cat << "EOF" > $PREFIX/etc/containerd/config.toml
root = "/data/docker/var/lib/containerd"
state = "/data/docker/run/containerd"
imports = ["$PREFIX/etc/containerd/runtime_*.toml", "./debug.toml"]

[grpc]
  address = "/data/docker/run/containerd/containerd.sock"

[debug]
  address = "/data/docker/run/containerd/debug.sock"

[plugins]
  [plugins.opt]
    path = "/data/docker/opt"
  [plugins.cri.cni]
    bin_dir = "/data/docker/opt/cni/bin"
    conf_dir = "/data/docker/etc/cni/net.d"
EOF
```

> **Note:** unfortunately containerd files also can't be stored inside Termux installation folder, failing with an error when creating the socket it uses.

### 2.3.6. runc

See [Edit](#edit-).

runc is a dependency of containerd. Conveniently for us, it's already provided by Termux's repository. Install it by simply:

```
$ pkg install root-repo
$ pkg install runc
```

# 3. Running

Now comes the truth time. To run the containers, first we need to start the daemon manually. To do so, it's advisable to install a terminal multiplexer so you can run the daemon in one pane and the container in others panes:

```
$ pkg install tmux
```

In one pane start dockerd:

```
$ sudo dockerd --iptables=false
```

And in others panes you can run the containers:

```
$ sudo docker run hello-world
```

> **Note:** Teaching how to use tmux is out of the scope of this guide, you can find good tutorials on YouTube. If you don't wanna use a terminal multiplexer, you can run dockerd in the background instead, with `sudo dockerd &>/dev/null &`.

## 3.1. Caveats

### 3.1.1. Internet access

The two [network drivers](https://docs.docker.com/network/) tested so far are `bridge` and `host`. Here's how to get each of them working.

#### bridge

This is the default netwok driver. If you don't specify a driver, this is the type of network you are creating. [Bridge networks](https://docs.docker.com/network/bridge/) isolate the container network by editing the iptables rules and creating a network interface called `Docker0` that serves as a bridge. All containers created with the bridge driver will use this interface. This is analogous to creating a VLAN and running the containers inside it.

But, there's a catch in Android: iptables rules policy is different here than on a conventional GNU/Linux system (more info [here](https://gist.github.com/FreddieOliveira/efe850df7ff3951cb62d74bd770dce27#gistcomment-3605349)). For the bridge driver to work, you'll have to manually edit the iptable by running;

```
$ sudo ip route add default via 192.168.1.1 dev wlan0
$ sudo ip rule add from all lookup main pref 30000
```

> **Note:** change 192.168.1.1 according to your gateway IP.

Unfortunately, this means that changing networks will require you to re-configure the rules again.

#### host

Using the [host driver](https://docs.docker.com/network/host/), means to remove network isolation between the container and the Docker host, and use the host’s networking directly. This way, the container will use the same network interface as your device (e.g. wlan0) and thus will share the same IP address.

To use this driver give the `--net=host --dns=8.8.8.8` flags when running a container.

### 3.1.2. Shared volumes

An easy way to share folders and files between containers and the host is to use a shared volume. For example, using the `-v ~/Documents/docker-share:/root/docker-share` flag when running a container, will make the `~/Documents/docker-share` folder from the host to be accessible inside the container `/root/docker-share` folder.

But, when talking about Android, things seems to never be as easy and straightforward as expected. Due to Android file system encryption, if you `ls` the `/root/docker-share` folder inside the container you might see a bunch of random letters, numbers and symbols instead of the folders and files names:

```
# ls /root/docker-share
+2xKy7JIRrcGrCf+o6KSeB  T6BJkyIa5OedXNrSyRKLbB  cwoDh,Nzt1l,5BsKA4hH8D
2aHRCQEyK8yYiiK9PEI9SA  Ue39lJVm4kIxGrS1bV07zB  lEpWZhTY9dNqJxCu+GqBuA
5ZRDLfHMwyik6RMe,f0WPA  X+yGLxXSgwxbCsFGRXuczC  y4ZWVvVBBjcxSWlJ9conED
GljgSZK5gFr7D4Fk7BHNeB  X1ATNoqhp,,ZsKjFXqKFiA
I3N5j0R4zmaQPKCWwKBlxD  Yzi+KmovJmIYFOCHtDCXkB
```

and if you try to read or create a file inside the volume you might get the `Required key not available` error.

No [definitive solution](https://gist.github.com/FreddieOliveira/efe850df7ff3951cb62d74bd770dce27#gistcomment-3606119) was discovered so far, but a workaround is to `cat` the files from within the host to give the container temporary access to them. You can cat an individual file by: 
```
$ sudo cat ~/Documents/docker-share/file.pdf >/dev/null
``` 
or all of them by:

```
$ sudo find ~/Documents/docker-share -exec cat {} >/dev/null \;
```

## 3.2. GUI

Yes, it's possible to run GUI programs inside a container! There's basically two ways of accomplishing it in a simple manner:

### 3.2.1. X11 Forwarding

#### Description

This method has the advantage of not making necessary the installation nor configuration of any additional programs inside the container; all you'll have to do is to [setup the X inside termux](https://wiki.termux.com/wiki/Graphical_Environment) and share its sockets with the container.

This is advisable to be used when you intend to run various containers with GUI, since you'll only have to install and setup a VNC once in the host, instead of doing it for each container. This will save storage space and time.

#### Steps

The first step is to enable the X11 repository in termux, this will allow installation of graphical interface related programs, like the VNC server we'll be using.

```
$ pkg install x11-repo
```

Then install a VNC server in termux:

```
$ pkg install tigervnc
```

> **Note:** These installations steps need to be executed only once.

Now, just run it:

```
$ vncserver -noxstartup -localhost
```

> **Note:** It's advisable to pass the `-geometry HEIGHTxWEIGHT` flag substituting HEIGHT and WEIGHT by your phone's screen resolution or some multiple of it.

> **Note:** The very first time you run it, you'll be prompted to setup a password. Note that passwords are not visible when you are typing them and it's maximal length is 8 characters. If you don't wanna use a passwd, use the `-SecurityTypes none` flag.

If everything is okay, you will see this message:

```
New 'localhost:1 ()' desktop is localhost:1
```

It means that X (vnc) server is available on display 'localhost:1'. Finally, export the DISPLAY environment variable according to that value:

```
$ export DISPLAY=:1
```

Now that the VNC server is configured and running in the host, start the container sharing the X related files as volumes:

```
$ sudo docker run -ti \
    --net="host" \
    --dns="8.8.8.8" \
    -e DISPLAY=$DISPLAY \
    -v $TMPDIR/.X11-unix:/tmp/.X11-unix \
    -v $HOME/.Xauthority:/root/.Xauthority \
    ubuntu
```

> **Note:** If by any reason you forget to export the DISPLAY before starting the container, you can still export it from inside it.

You'll now be able to launch GUI programs from inside the container, e.g.:

```
# echo 'APT::Sandbox::User "root";' > /etc/apt/apt.conf
# apt update
# apt install x11-apps
# xeyes
```

To check the GUI, you'll need to install a VNC client app in your Android phone, like [VNC Viewer](https://play.google.com/store/apps/details?id=com.realvnc.viewer.android) (developed by RealVNC Limited). Unfortunately it's not open source, but it's a good and intuitive VNC client for Android.

> **Note:** There's also an open source alternative developed by @pelya called [XServer XSDL](https://github.com/pelya/xserver-xsdl), which will not be covered by this guide (for now).

After installing the VNC Viewer app, open it and setup a new connection using 127.0.0.1 (or localhost) as the IP, 5901 as the port (the port is calculated as 5900 + {display number}) and when/if prompted, type the password choosen when running vnctiger for the first time.

### 3.2.2. VNC server within the container

#### Description

This method is very similar to the previous, with the difference that the X server will be installed inside the container instead of in the termux host.

The advantages are:

1. you aren't changing your host system by installing softwares on it (like the VNC server);
2. security, since you won't be sharing your host's X (this is only relevant when you are not the one running the container).

The main disadvantage is that you'll need to install and config the VNC server for each container you'd run a GUI program, thus making these containers big and time consuming to setup.

#### Steps

First, start a container:

```
$ sudo docker run -ti \
    --net="host" \
    --dns="8.8.8.8" \
    ubuntu
```

Now, a VNC server needs to be installed and configured inside the container. You can choose between TigerVNC or x11vnc:

##### TigerVNC

The same VNC server used above in termux. To install it:

```
# echo 'APT::Sandbox::User "root";' > /etc/apt/apt.conf
# apt update
# apt install tigervnc-standalone-server
```

Next, start it with:

```
# vncserver -noxstartup -localhost -SecurityTypes none
```

Here we disabled password (`-SecurityTypes none`) because using it causes things to crash as described in this issue report https://github.com/TigerVNC/tigervnc/issues/800

If everything is okay, you will see this message:

```
New 'localhost:1 (root)' desktop at :1 on machine localhost
```

Export the DISPLAY environment variable according to that value:

```
# export DISPLAY=:1
```

From now on, you can already run GUI programs and access them using the VNC Viewer client as already described in the end of [X11 Forwarding](#321-x11-forwarding) steps.

##### x11vnc

Install the x11vnc and the virtual fake X (since x11vnc can't emulate a X11 by itself):

```
# echo 'APT::Sandbox::User "root";' > /etc/apt/apt.conf
# apt update
# apt install x11vnc xvfb
```

Now, start it:

```
# x11vnc -nopw -forever -noshm -create
```

If everything is okay, you will see this message:

```
The VNC desktop is:      localhost:0
PORT=5900
```

This will open a xterm terminal which can be acessed by the VNC Viewer client as already described in the end of [X11 Forwarding](#321-x11-forwarding) steps. From that terminal you can open the desired GUI program.

## 3.3. Steam (work in progress)

I'm not talking about running the useless steam app for Android, but about running the Desktop version and play the games inside a docker container. Yes, you read it right, it's possible to play your Steam games on Android!

(ACTUALLY NOT YET, BECAUSE I DIDN'T MANAGE TO GET OPENGL TO WORK, THAT'S WHY THIS IS A WORK IN PROGRESS. TO CONTRIBUTE OR STAY UP TO DATE ABOUT THE PROGRESS CHECK https://github.com/ptitSeb/box86/issues/249)

To do so, we'll use an awesome x86 emulator for ARM developed by @ptitSeb called [box86](https://github.com/ptitSeb/box86).

But first, you need to enable `System V IPC` under `General Setup` in the kernel config and recompile it again. That's because the steam binary needs some semaphore functions and will crash in case it can't use them.

Next, we hit a problem: box86 can only be compiled by a 32 bit toolchain. But, in fact, this can be easily circumvented by using a 32 bit container:

```
$ sudo docker run -ti \
    --net="host" \
    --dns="8.8.8.8" \
    -e DISPLAY=$DISPLAY \
    -w /root \
    -v $TMPDIR/.X11-unix:/tmp/.X11-unix \
    -v $HOME/.Xauthority:/root/.Xauthority \
    --platform=linux/arm \
    arm32v7/ubuntu
```

> **Note:** if your system is 32 bit already (run `uname -m` to check), you don't need to specify the `--platform=linux/arm` flag and can simply use `ubuntu` instead of `arm32v7/ubuntu`.

Now that we are inside the container, let's install the tools we're gonna use, as well as the steam .deb installer:

```
# echo 'APT::Sandbox::User root;' >> /etc/apt/apt.conf
# apt update
# apt install wget binutils xterm libvdpau1 libappindicator1 libnm0 libdbusmenu-gtk4
```

Install steam:

```
# wget https://steamcdn-a.akamaihd.net/client/installer/steam.deb
# ar x steam.deb
# mkdir steam
# tar xf data.tar.xz -C steam
# find steam -type d -exec sh -c 'mkdir -p /${0#*/}' {} \;
# find steam \! -type d -exec sh -c 'mv $0 /${0#*/}' {} \;
# patch /usr/lib/steam/bin_steam.sh bin_steam.sh.patch
# rm -rf steam* *.tar* bin_steam.sh.patch
# steam
```

Steam will fail with a bunch of errors, but that's expected. The important thing is that it installed the necessary files under `~/.local/share/Steam`, one of them being the steam binary. Finish the installation by adding it to the path:

```
# ln -sf /root/.local/share/Steam/ubuntu12_32/steam /usr/bin/steam
```

Now, we need to install the i386 version of some libs required by steam. For this, we're going to download them directly from Ubuntu packages. That's because if we instead simply apt install them we would be getting the arm32 version.

# 4. Attachments

## 4.1. kernel patches

- kernel/Makefile

```patch
diff --git a/kernel/Makefile b/kernel/Makefile
index d5c1115..2dea801 100644
--- a/kernel/Makefile
+++ b/kernel/Makefile
@@ -121,7 +121,7 @@ $(obj)/configs.o: $(obj)/config_data.h
# config_data.h contains the same information as ikconfig.h but gzipped.
# Info from config_data can be extracted from /proc/config*
targets += config_data.gz
-$(obj)/config_data.gz: arch/arm64/configs/lavender_stock-defconfig FORCE
+$(obj)/config_data.gz: $(KCONFIG_CONFIG) FORCE
    $(call if_changed,gzip)

    filechk_ikconfiggz = (echo "static const char kernel_config_data[] __used = MAGIC_START"; cat $< | scripts/basic/bin2c; echo "MAGIC_END;")
```

- net/netfilter/xt_qtaguid.c

```patch
--- orig/net/netfilter/xt_qtaguid.c     2020-05-12 12:13:14.000000000 +0300
+++ my/net/netfilter/xt_qtaguid.c       2019-09-15 23:56:45.000000000 +0300
@@ -737,7 +737,7 @@
{
        struct proc_iface_stat_fmt_info *p = m->private;
        struct iface_stat *iface_entry;
-       struct rtnl_link_stats64 dev_stats, *stats;
+       struct rtnl_link_stats64 *stats;
        struct rtnl_link_stats64 no_dev_stats = {0};  
@@ -745,13 +745,8 @@
        current->pid, current->tgid, from_kuid(&init_user_ns, current_fsuid()));
        iface_entry = list_entry(v, struct iface_stat, list);
+       stats = &no_dev_stats; 
-       if (iface_entry->active) {
-               stats = dev_get_stats(iface_entry->net_dev,
-                                     &dev_stats);
-       } else {
-               stats = &no_dev_stats;
-       }
        /*
         * If the meaning of the data changes, then update the fmtX
         * string.
```

## 4.2. docker-cli patches

- [vendor/github.com/containerd/containerd/platforms/database.go](https://github.com/termux/termux-root-packages/files/5793950/database.go.patch.txt)
- [scripts/docs/generate-man.sh](https://github.com/termux/termux-root-packages/files/5793951/generate-man.sh.patch.txt)
- [man/md2man-all.sh](https://github.com/termux/termux-root-packages/files/5793952/md2man-all.sh.patch.txt)
- [cli/config/config.go](https://github.com/termux/termux-root-packages/files/5793948/config.go.patch.txt)

## 4.3. dockerd patches 
- [cmd/dockerd/daemon.go](https://raw.githubusercontent.com/termux/termux-root-packages/29ca852ba95ae76b03189adbf68309fc217be7dd/packages/docker/daemon.go.patch)

## 4.4. containerd patches

- [runtime/v1/linux/bundle.go](https://github.com/termux/termux-root-packages/files/5793939/bundle.go.patch.txt)
- [runtime/v2/shim/util_unix.go](https://github.com/termux/termux-root-packages/files/5793946/util_unix.go.patch.txt)
- [Makefile](https://raw.githubusercontent.com/termux/termux-root-packages/master/packages/containerd/Makefile.patch)
- [platforms/database.go](https://github.com/termux/termux-root-packages/files/5793940/database.go.patch.txt)
- [vendor/github.com/cpuguy83/go-md2man/v2/md2man.go](https://github.com/termux/termux-root-packages/files/5793944/md2man.go.patch.txt)

# 5. Aknowledgements

I'd like to thank the Termux Dev team for this wonderful app and @xeffyr for discovering about the bug in `net/netfilter/xt_qtaguid.c` and sharing the patch, as well as all the conversation we had [here](https://github.com/termux/termux-root-packages/issues/60) that led to docker finally working.

Also @yjwong, for figuring out how to use the bridge network driver.

# 6. Final notes

If you are a docker developer reading this, please consider adding an official support for Android. Look above the possibilities it opens for a smartphone. If you are not a docker developer, consider supporting this by showing interest [here](https://github.com/moby/moby/issues/41111). If we annoy the devs enough, this may become official (of they may simply unsubscribe from the thread and let it rot in the Issues section ¯\\_(ツ)\_/¯ ).
