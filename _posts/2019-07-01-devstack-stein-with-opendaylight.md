---
layout: post
title:  "Devstack Stein setup with OpenDaylight"
date:   2019-07-01 15:40:00
categories: devstack openstack opendaylight
---

Steps to setup a devstack (stein) single node setup with OpenDaylight controller as Neutron networking plugin -
1. Download Ubuntu 16.04 server ISO.
2. In virtualbox go to *File > Host Network Manager*. The Create a new host only network (#2). Enable DHCP.
3. Create a vm in virtualbox. Provide 4 CPU, 10 GB RAM. Enable 3 network adapters NAT, Hostonly, Hostonly #2
4. Start VM and mount the Ubuntu ISO. Install Ubuntu server.
5. Update software after installation, apt-get update and apt-get upgrade
6. Edit `/etc/network/interfaces` file and add 2nd and 3rd nic details (auto, dhcp)
7. Add stack user - `sudo useradd -s /bin/bash -d /opt/stack -m stack`
8. Give sudo permission to stack user - `echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack`
9. Restart VM
10. Become stack user - `sudo su - stack`
11. Download stein release devstack - `git clone https://git.openstack.org/openstack-dev/devstack -b stable/stein`
12. `cd devstack`
Create `local.conf` file with following content -

{% highlight ini %}
[[local|localrc]]
# This will fetch the latest ODL snapshot
ODL_RELEASE=oxygen-latest

# Default is V2 driver, uncomment below line to use V1
#ODL_V2DRIVER=False

# Default is psuedo-port-binding-controller
#ODL_PORT_BINDING_CONTROLLER=


# Set here which ODL openstack service provider to use
# These are core ODL features
ODL_NETVIRT_KARAF_FEATURE=odl-neutron-service,odl-restconf-all,odl-aaa-authn,odl-dlux-core,odl-mdsal-apidocs

# Set DLUX Karaf features needed for the ODL GUI at http://<ODL_IP>:8181/index.html
ODL_NETVIRT_KARAF_FEATURE+=,odl-dluxapps-nodes,odl-dluxapps-topology,odl-dluxapps-yangui,odl-dluxapps-yangvisualizer

# Set L2 Karaf features needed for the ODL GUI at http://<ODL_IP>:8181/index.html
ODL_NETVIRT_KARAF_FEATURE+=,odl-l2switch-switch,odl-l2switch-switch-ui,odl-ovsdb-hwvtepsouthbound-ui,odl-ovsdb-southbound-impl-ui,odl-netvirt-ui

# Set OpenFlow Karaf features needed for the ODL GUI at http://<ODL_IP>:8181/index.html
ODL_NETVIRT_KARAF_FEATURE+=,odl-openflowplugin-flow-services-ui

# odl-netvirt-openstack is used for new netvirt
ODL_NETVIRT_KARAF_FEATURE+=,odl-netvirt-openstack

# optional feature neutron-logger to log changes of neutron yang models
ODL_NETVIRT_KARAF_FEATURE+=,odl-neutron-logger

# Switch to using the ODL's L3 implementation
ODL_L3=True

# Set Host IP here. It is externally reachable network, set
# below param to use ip from a different network
#HOST_IP=$(ip route get 8.8.8.8 | awk '{print $NF; exit}')
# Replace with your 1st hostonly interface ip
HOST_IP=192.168.56.108

# public network connectivity
#Q_USE_PUBLIC_VETH=True
#Q_PUBLIC_VETH_EX=enp0s3
#Q_PUBLIC_VETH_INT=enp0s3
# Replace with your 2nd hostonly interface name
ODL_PROVIDER_MAPPINGS=public:enp0s9

# Enable debug logs for odl ovsdb
ODL_NETVIRT_DEBUG_LOGS=True

#Q_USE_DEBUG_COMMAND=True

DEST=/opt/stack/
# move DATA_DIR outside of DEST to keep DEST a bit cleaner
DATA_DIR=/opt/stack/data

ADMIN_PASSWORD=techno
MYSQL_PASSWORD=${ADMIN_PASSWORD}
RABBIT_PASSWORD=${ADMIN_PASSWORD}
SERVICE_PASSWORD=${ADMIN_PASSWORD}
SERVICE_TOKEN=supersecrettoken

enable_service dstat
enable_service g-api
enable_service g-reg
enable_service key
enable_service mysql
enable_service n-api
enable_service n-cond
enable_service n-cpu
enable_service n-crt
enable_service n-novnc
enable_service n-sch
enable_service placement-api
enable_service placement-client
enable_service q-dhcp
enable_service q-meta
enable_service q-svc
enable_service rabbit
enable_service tempest

# These can be enabled if storage is needed to do
# any feature or testing for integration
#disable_service c-api
#disable_service c-vol
#disable_service c-sch
#disable_service etcd3

SKIP_EXERCISES=boot_from_volume,bundle,client-env,euca

# Screen console logs will capture service logs.
SYSLOG=False
SCREEN_LOGDIR=/opt/stack/new/screen-logs
LOGFILE=/opt/stack/new/devstacklog.txt
VERBOSE=True
FIXED_RANGE=10.1.0.0/20
FLOATING_RANGE=172.24.5.0/24
PUBLIC_NETWORK_GATEWAY=172.24.5.1
FIXED_NETWORK_SIZE=4096
VIRT_DRIVER=libvirt

export OS_NO_CACHE=1

# Additional repositories need to be cloned can be added here.
#LIBS_FROM_GIT=

# Enable MySql Logging
DATABASE_QUERY_LOGGING=True

# set this until all testing platforms have libvirt >= 1.2.11
# see bug #1501558
EBTABLES_RACE_FIX=True

enable_plugin networking-odl https://git.openstack.org/openstack/networking-odl
{% endhighlight %}
