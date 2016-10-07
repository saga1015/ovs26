# ovs26

Build DPDK 16.07 and OVS-DPDK 2.6 from git repo on Ubuntu 16.04 running as a OpenStack compute node.

------------------------------------------------------------------------
Stop existing ovs and services
------------------------------------------------------------------------
service nova-compute stop
service neutron-openvswitch-agent stop
service openvswitch-switch stop

-------------------------------------------------------------------------
Deinstall existing ubuntu dpdk 2.2 and ovs-dpdk 2.5
-------------------------------------------------------------------------
sudo apt remove openvswitch-switch-dpdk

Before removing the dpdk package make a copy of /lib/dpdk/dpdk-init script because we need it later when integrating our dpdk into dpdk startup. Removing thge dpdk package removes this script.

sudo apt remove dpdk

------------------------------------------------------------------------
Ubuntu 16.04: DPDK 16.07 + OVS-DPDK 2.6 build prerequisite packages
------------------------------------------------------------------------
apt install unzip
apt install make
apt install gcc
apt install clang
apt install openssl
apt install libcap-ng
apt install autoconf
apt install libtool
apt install pyftpdlib
apt install python3-pyftpdlib
apt install clibcurl4-openssl-dev pkg-config libssl-dev libsslcommon2-dev
apt install pkg-config libssl-dev libsslcommon2-dev
apt install libcap-ng-dev

------------------------------------------------------------------------
DPDK 16.07 build
------------------------------------------------------------------------
cd /usr/src
wget http://dpdk.org/browse/dpdk/snapshot/dpdk-16.07.zip
unzip dpdk-16.07.zip

Now you need to modify the dpdk.org code for vhost-owner permissions and another fix to chmod the vhu sockets correctly (steps 1 and 2 below). This is only required to make it work with Ubuntu 16.04. Note that the Ubuntu dpdk and openvswitch source packages will have these fixes already in the repository but the openvswitch package curently does not build because of several issues.

1. Canonicals vhost-owner code 
Patch the dpdk 16.07 source with this patch:
https://git.launchpad.net/~ubuntu-server/dpdk/tree/debian/patches/fix-vhost-user-socket-permission.patch?h=ubuntu-yakkety-dpdk16.07

2. Fix to chmod vhost-user sockets, apply this patch:
https://bugs.launchpad.net/ubuntu/+source/dpdk/+bug/1625542/+attachment/4748866/+files/dpdk_vhost_permission_tikoehle.patch


export DPDK_DIR=/usr/src/dpdk-16.07
cd $DPDK_DIR
export DPDK_TARGET=x86_64-native-linuxapp-gcc
export DPDK_BUILD=$DPDK_DIR/$DPDK_TARGET
make install T=$DPDK_TARGET DESTDIR=install

Reference:
https://github.com/openvswitch/ovs/blob/branch-2.6/INSTALL.DPDK.md


------------------------------------------------------------------------
OVS-DPDK 2.6 build
------------------------------------------------------------------------
cd /usr/src
git clone https://github.com/openvswitch/ovs.git
export OVS_DIR=/usr/src/ovs
cd $OVS_DIR
./boot.sh
./configure --with-dpdk=$DPDK_BUILD --localstatedir=/var --runstatedir=/var/run
make install

Installs into /usr/local/sbin/

Reference:
https://github.com/openvswitch/ovs/blob/branch-2.6/INSTALL.DPDK.md


-------------------------------------------------------------------------
Create and initialize the ovs db: start ovsdb-server manually
------------------------------------------------------------------------
mv /etc/openvswitch /etc/openvswitch.off
mkdir -p /etc/openvswitch
ovsdb-tool create /etc/openvswitch/conf.db /usr/local/share/openvswitch/vswitch.ovsschema

/usr/local/sbin/ovsdb-server /etc/openvswitch/conf.db -vconsole:emer -vsyslog:err -vfile:info \
--remote=punix:/var/run/openvswitch/db.sock \
--remote=db:Open_vSwitch,Open_vSwitch,manager_options \
--private-key=db:Open_vSwitch,SSL,private_key \
--certificate=db:Open_vSwitch,SSL,certificate \
--bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert \
--no-chdir --log-file=/var/log/openvswitch/ovsdb-server.log \
--pidfile=/var/run/openvswitch/ovsdb-server.pid --detach --monitor

ovs-vsctl --no-wait init                                 # do this once

root@compute29:/usr/src/ovs# which ovs-vsctl
/usr/local/bin/ovs-vsctl

ovs-vsctl show


-------------------------------------------------------------------------
Start vswitchd once manually to configure OVS EAL args for dpdk
-------------------------------------------------------------------------
Jumbo frame mbuf size requires to increase socket mem from "2048,2048" to "4096,4096". The args to configure coremask, socket-mem, vhost-owner, .. are now with ovs 2.6 directly passed to the dpdk EAL via the ovs db and /etc/default/openvswitch-switch is no longer needed. In fact you have to disable DPDK_OPTS in /etc/default/openvswitch-switch.

ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="4096,4096"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-lcore-mask=0x300
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-extra="--vhost-owner libvirt-qemu:kvm --vhost-perm 0664"
ovs-vsctl get Open_vSwitch . other_config

export DB_SOCK=/var/run/openvswitch/db.sock
/usr/local/sbin/ovs-vswitchd unix:$DB_SOCK -vconsole:emer -vsyslog:err -vfile:info --mlockall --no-chdir \
--log-file=/var/log/openvswitch/ovs-vswitchd.log \
--pidfile=/var/run/openvswitch/ovs-vswitchd.pid \
--detach \
--monitor

Now after starting the vswitchd you should see lot of EAL messages and check there should be no errors. If everything is good then switch it off.

-------------------------------------------------------------------------
Stop db and vswitchd 
-------------------------------------------------------------------------
ovs-appctl -t ovs-vswitchd exit
ovs-appctl -t ovsdb-server exit


-------------------------------------------------------------------------
Integrate DPDK 16.07 and OVS-DPDK 2.6 into existing startup services on Ubuntu 16.04
-------------------------------------------------------------------------

1. dpdk
ln -s /usr/src/dpdk-16.07/install/share/dpdk/tools/dpdk-devbind.py /sbin/dpdk_nic_bind
systemctl unmask dpdk.service
Restory your backup copy of /lib/dpdk/dpdk-init
systemctl restart dpdk.service
dpdk_nic_bind --status

Should have the interface now dpdk bound, otherwise vfio_pci module is not loaded or modprobe it manually or shutdown -r now.

2. ovs-dpdk
mkdir -p /usr/lib/openvswitch-switch-dpdk
cp /usr/local/sbin/ovs-vswitchd /usr/lib/openvswitch-switch-dpdk/ovs-vswitchd-dpdk
cp /usr/local/sbin/ovs-vswitchd /usr/lib/openvswitch-switch/ovs-vswitchd
cp /usr/local/sbin/ovsdb-server /usr/sbin/ovsdb-server
cp /usr/local/share/openvswitch/vswitch.ovsschema /usr/share/openvswitch/vswitch.ovsschema

/etc/default/openvswitch-switch
#DPDK_OPTS

Copy the new 2.6 bins to /usr/bin/ otherwise L2 agent will not find it.
cp /usr/local/bin/ovs* /usr/bin/

systemctl unmask openvswitch-switch.service
systemctl start openvswitch-switch.service

-------------------------------------------------------------------------
Test
-------------------------------------------------------------------------
Add dpdk br and port and configure port for jumbo frame:
ovs-vsctl add-br br-data -- set bridge br-data datapath_type=netdev
ovs-vsctl add-port br-data dpdk0 -- set interface dpdk0 type=dpdk -- set Interface dpdk0 mtu_request=9000

Check the bridge is netdev enabled and not system:
ovs-vsctl list bridge br-data

Check this copy of vswitchd is really dpdk enabled:
ovs-vsctl get Open_vSwitch . iface_types

Restart nova / neutron services:
service nova-compute start
service neutron-openvswitch-agent start

-------------------------------------------------------------------------
Mitaka Jumbo Frame Size
-------------------------------------------------------------------------
When nova booting a vm you need to manually set the mtu size for the vhu port, for example:
ovs-vsctl set Interface vhu6ef65d50-d7 mtu_request=9000

Check that all vhost-user sockets were created with the correct ownership:
ls -la /var/run/openvswitch


-------------------------------------------------------------------------
Canonicals dpdk and ovs source packages
-------------------------------------------------------------------------
I admit the ovs-dpdk integration in Ubuntu 16.04, as described above, is a crude hack. There are currently some issues to build the openvswitch package with a dpdk enabled vswitchd:

https://bugs.launchpad.net/ubuntu/+source/openvswitch-dpdk/+bug/1629271

The dpdk 16.07 source package builds just fine, creates all .deb packages and installs correctly without any dependency issues.

The current latest Ubuntu dpdk and ovs source packages are:

https://launchpad.net/ubuntu/+source/dpdk/16.07-0ubuntu5
https://launchpad.net/ubuntu/+source/openvswitch/2.6.0-0ubuntu2

