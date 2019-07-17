## Project
middleware-ceph provides a template for the installation of Ceph onto an OSN
pod. This effort, in part, provided the bottom layer of a functional OSN data
stack used for early discovery and discussions. It is likely that this project
will be deprecated overtime as it is replaced with a permanent solution more
fully integrated into the OSN deployment model.

This project was derived from early work in the project 'ansible-osn'. That
project was later divided to form middleware-ceph and globus-connect-server. The
purpose of the split was to allow development in parallel of both middleware
components. This was an early consideration before the creation of
ceph-ansible-root which now supercedes this repo.

The supplied configuration will install the Ceph monitor and radosgw on each of
the 3 monitor nodes and the osd services on each of the 5 data nodes. To do so,
it provides 4 roles, located in `./roles/`, which may be reused:  

#### ceph-common  
Installs the Ceph components common to all Ceph services from
the RPM repository https://download.ceph.com/rpm-mimic/el7 and generates key
rings needed for all services other than the radosgw. The common configuration
file `ceph.conf` can be found in this role.

#### ceph-mon  
Generate the Ceph monitor map and starts the monitor service.

#### ceph-osd  
Initializes all hard disks configured for use as OSD data devices
and starts all OSD processes. All disks are created with `ceph-volume` as
lvm devices using Ceph Bluestore.

#### ceph-rgw  
Installs the radosgw, generates the radosgw keyring and ensure
that the process is running.

## Getting Started
This Ansible configuration was developed for execution from a mock command
center node, citadel, using a SSH-push model from the local user 'ansible'
home directory to install the JHU pod. The pod consists of 3 monitor nodes
and 5 data nodes. Each pod node is required to permit passwordless SSH
access from ansible@citidel and the local ansible account must have
passwordless sudo capabilities.

## Prerequisites
* The command center node is required to have Ansible 2.4.2 or newer.
* SSH public key authentication must be enabled for the user 'ansible' on each
of the pod nodes.
  * ansible@citadel:~/.ssh/osn-ansible <--- Private SSH key
  * On each pod node: ~ansible/.ssh/authorized_keys <--- Public SSH key
* The ansible user must be able to perform passwordless sudo on the pod nodes:
  * /etc/sudoers

  > \#\# Allows people in group wheel to run all commands  
  > \#%wheel ALL=(ALL)       ALL
  >
  > \#\# Same thing without a password  
  > %wheel  ALL=(ALL)       NOPASSWD: ALL

  * /etc/group
  > `wheel:x:10:ansible`

## Configuration
#### ansible.cfg  
 `remote_user`:  Change this to the account local to the pod nodes  
 `private_key_file`: Set to the location of your SSH private key 

#### group_vars/ceph.yml  
 `ceph_network`: Set this to the cluster internal network.  

 `ceph_fsid`: UUID that identifies this Ceph storage instance. The configured
  UUID is not special, it was created with uuidgen. This value will be reused
  with each installation and therefore should like change between production
  Ceph installations.  

 `ceph_osd_devices`: Hard disks found on each data node that are to be
  initailized as OSD devices. This disks on this list must be on all data nodes.  

#### group_vars/jhu.yml  
 `ceph_monitor_hosts`: Short hostname of the Ceph monitor nodes. This variable
  assists with keeping `ceph_monitor_addrs` organized.
 `ceph_monitor_addrs`: List of interfaces on the monitor nodes which reside on
  on the Ceph cluster network. One interface per monitor node.

#### inventory  
  This file is a standard Ansible definition of target hosts. See site.yml for
  how hosts are mapped to roles.

#### site.yml  
  Maps hosts to roles. All `ceph` nodes will have the `ceph-common` role
  applied. All hosts containing the midfix `-mon` will have the monitor and
  radosgw services installed. All hosts with the midfix `-data` will have the
  OSD services installed.

#### state  
  This is the location of intermediate products of the initial Ansible run.
  These files (ex. keyrings) are generated remotely on a single node and 
  copied locally so that they can be distributed to remaining nodes during
  the initial Ansible run and used for recovery in subsequent Ansible runs.

## Deployment
  The following command will perform the installation of Ceph on the target Pod
  and ensure that Ceph services are operational:  

  > ansible-playbook site.yml

