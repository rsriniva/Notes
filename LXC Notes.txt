LXC Notes
=========

START, STOP & ATTACH
---------------------

# Create a "p1" container using the "ubuntu" template and the same version of Ubuntu
# and architecture as the host. Pass "-- --help" to list all available options.

sudo lxc-create -t ubuntu -n p1

# Start the container (in the background)
sudo lxc-start -n p1 -d

# Enter the container in one of those ways## Attach to the container's console (ctrl-a + q to detach)
sudo lxc-console -n p1

## Spawn bash directly in the container (bypassing the console login), requires a >= 3.8 kernel
sudo lxc-attach -n p1

## SSH into it
sudo lxc-info -n p1
ssh ubuntu@<ip from lxc-info>

# Stop the container in one of those ways
## Stop it from within
sudo poweroff

## Stop it cleanly from the outside
sudo lxc-stop -n p1

## Kill it from the outside
sudo lxc-stop -n p1 -k

# List all containers
stgraber@castiana:~$ sudo lxc-ls -f
NAME    STATE    IPV4        IPV6                                    AUTOSTART     
---------------------------------------------------------------------------------
p1      RUNNING  10.0.3.128  2607:f2c0:f00f:2751:216:3eff:feb1:4c7f  YES (ubuntu)
p2      RUNNING  10.0.3.165  2607:f2c0:f00f:2751:216:3eff:fe3a:f1c1  YES

# Freeze a container
sudo lxc-freeze -n <container name>


NETWORKING
----------

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

With this setup my container will have 3 interfaces, eth0 will be the usual veth device in the lxcbr0 bridge, eth1 will be the host’s eth2 moved inside the container (it’ll disappear from the host while the container is running) and virt0 will be another veth device in the virbr0 bridge on the host.

Those last two interfaces don’t have a mac address or network flags set, so they’ll get a random mac address at boot time (non-persistent) and it’ll be up to the container to bring the link up.


