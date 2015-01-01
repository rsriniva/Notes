## START, STOP & ATTACH

### Create a "p1" container using the "ubuntu" template and the same version of Ubuntu and architecture as the host. 
```bash
sudo lxc-create -t ubuntu -n p1
```

### Start the container (in the background)
```bash
sudo lxc-start -n p1 -d
```

### Enter the container in one of those ways## Attach to the container's console (ctrl-a + q to detach)
```bash
sudo lxc-console -n p1
```

### Spawn bash directly in the container (bypassing the console login), requires a >= 3.8 kernel
```bash
sudo lxc-attach -n p1
```

### SSH into it
```bash
sudo lxc-info -n p1
ssh ubuntu@<ip from lxc-info>
```

### Stop the container in one of those ways
### Stop it from within
```bash
sudo poweroff
```

### Stop it cleanly from the outside
```bash
sudo lxc-stop -n p1
```

### Kill it from the outside
```bash
sudo lxc-stop -n p1 -k
```

### List all containers
```bash
stgraber@castiana:~$ sudo lxc-ls -f
NAME    STATE    IPV4        IPV6                                    AUTOSTART     
---------------------------------------------------------------------------------
p1      RUNNING  10.0.3.128  2607:f2c0:f00f:2751:216:3eff:feb1:4c7f  YES (ubuntu)
p2      RUNNING  10.0.3.165  2607:f2c0:f00f:2751:216:3eff:fe3a:f1c1  YES
```

### Freeze a container
```bash
sudo lxc-freeze -n <container name>
```

## NETWORKING
```bash
lxc.network.type = veth
lxc.network.hwaddr = 00:16:3e:3a:f1:c1
lxc.network.flags = up
lxc.network.link = lxcbr0
lxc.network.name = eth0

lxc.network.type = veth
lxc.network.link = virbr0
lxc.network.name = virt0

lxc.network.type = phys
lxc.network.link = eth2
lxc.network.name = eth1
```

With this setup my container will have 3 interfaces, eth0 will be the usual veth device in the lxcbr0 bridge, eth1 will be the host’s eth2 moved inside the container (it’ll disappear from the host while the container is running) and virt0 will be another veth device in the virbr0 bridge on the host.

Those last two interfaces don’t have a mac address or network flags set, so they’ll get a random mac address at boot time (non-persistent) and it’ll be up to the container to bring the link up.

## Exchanging data with a container

what about having the container access and write data to the host?
Well, let’s say we want to have our host’s /var/cache/lxc shared with “p1″, we can edit /var/lib/lxc/p1/fstab and append:

```bash
/var/cache/lxc var/cache/lxc none bind,create=dir
```

This line means, mount “/var/cache/lxc” from the host as “/var/cache/lxc” (the lack of initial / makes it relative to the container’s root), mount it as a bind-mount (“none” fstype and “bind” option) and create any directory that’s missing in the container (“create=dir”).

Now restart “p1″ and you’ll see /var/cache/lxc in there, showing the same thing as you have on the host. Note that if you want the container to only be able to read the data, you can simply add “ro” as a mount flag in the fstab.

## LXC Storage

LXC supports a variety of storage backends (also referred to as backingstore).
It defaults to “none” which simply stores the rootfs under
/var/lib/lxc/<container>/rootfs but you can specify something else to lxc-create or lxc-clone with the -B option.

Currently supported values are:
directory based storage (“none” and “dir)

This is the default backingstore, the container rootfs is stored under
/var/lib/lxc/<container>/rootfs

The --dir option (when using “dir”) can be used to override the path.

btrfs

With this backingstore LXC will setup a new subvolume for the container which makes snapshotting much easier.
lvm

This one will use a new logical volume for the container.
The LV can be set with --lvname (the default is the container name).
The VG can be set with --vgname (the default is “lxc”).
The filesystem can be set with --fstype (the default is “ext4″).
The size can be set with --fssize (the default is “1G”).
You can also use LVM thinpools with --thinpool
overlayfs

This one is mostly used when cloning containers to create a container based on another one and storing any changes in an overlay.

When used with lxc-create it’ll create a container where any change done after its initial creation will be stored in a “delta0″ directory next to the container’s rootfs.
zfs

Very similar to btrfs, as I’ve not used either of those myself I can’t say much about them besides that it should also create some kind of subvolume for the container and make snapshots and clones faster and more space efficient.

### Cloning containers

All those backingstores only really shine once you start cloning containers.
For example, let’s take our good old “p1″ Ubuntu container and let’s say you want to make a usable copy of it called “p4″, you can simply do:

```bash
sudo lxc-clone -o p1 -n p4
```

And there you go, you’ve got a working “p4″ container that’ll be a simple copy of “p1″ but with a new mac address and its hostname properly set.

Now let’s say you want to do a quick test against “p1″ but don’t want to alter that container itself, yet you don’t want to wait the time needed for a full copy, you can simply do:

```bash
sudo lxc-clone -o p1 -n p1-test -B overlayfs -s
```

And there you go, you’ve got a new “p1-test” container which is entirely based on the “p1″ rootfs and where any change will be stored in the “delta0″ directory of “p1-test”.
The same “-s” option also works with lvm and btrfs (possibly zfs too) containers and tells lxc-clone to use a snapshot rather than copy the whole rootfs across.

### Snapshotting

So cloning is nice and convenient, great for things like development environments where you want throw away containers. But in production, snapshots tend to be a whole lot more useful for things like backup or just before you do possibly risky changes.

In LXC we have a “lxc-snapshot” tool which will let you create, list, restore and destroy snapshots of your containers.
Before I show you how it works, please note that “lxc-snapshot” currently doesn’t appear to work with directory based containers. With those it produces an empty snapshot, this should be fixed by the time LXC 1.0 is actually released.

So, let’s say we want to backup our “p1-lvm” container before installing “apache2″ into it, simply run:

```bash
echo "before installing apache2" > snap-comment
sudo lxc-snapshot -n p1-lvm -c snap-comment
```

At which point, you can confirm the snapshot was created with:

```bash
sudo lxc-snapshot -n p1-lvm -L -C
```

Now you can go ahead and install “apache2″ in the container.

If you want to revert the container at a later point, simply use:

```bash
sudo lxc-snapshot -n p1-lvm -r snap0
```

Or if you want to restore a snapshot as its own container, you can use:

```bash
sudo lxc-snapshot -n p1-lvm -r snap0 p1-lvm-snap0
```

And you’ll get a new “p1-lvm-snap0″ container which will contain a working copy of “p1-lvm” as it was at “snap0″.

