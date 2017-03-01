# How to make a Ceph cluster on OpenStack

# TLDR;
1. `virtualenv openstack`  
1. `. openstack/bin/activate`  
1. `pip install python-openstackclient python-heatclient ansible`
1. `. your-tennancy-openrc.sh`
1. `git clone git@github.com:hooliowobbits/heat-ceph.git && cd heat-ceph`
1. `openstack stack create -t heat_ceph_template.yaml -e heat_ceph_environment.yaml ceph`
1. `ansible-playbook ansible_bootstrap_ceph_deploy.yaml`
1. `openstack stack output show ceph login-node-public-ip`
1. `ssh -A ceph@xxx.xxx.xxx.xxx` where xxx.xxx.xxx.xxx is the public ip from above
1. `ceph-deploy .. `

# Full instructions
## What this is and isn't
Is: A virtual ceph cluster is a great way to learn ceph.  
Not: Something you should use for production workloads.  There be dragons.

## disclaimer and about heat
Openstack Heat has the ability to do magical (and dangerous) things to your openstack
environment.  in particular it can call and will call many of the openstack APIs
and like anything from the internet, you should really understand what this does
before running it.  i promise i won't intentionally harm your environment but
that is no guarantee that something bad will not happen.  be careful.  

`$ openstack stack delete ceph`  
`Are you sure you want to delete this stack(s) [y/N]? y`

## architecture
This environment has the following topology:
1. A deployment node That's where you run the heat and ansible commands in order to create and bootstrap the cluster.
1. The stack consists of a single login node, a configurable amount of storage, mon and mds nodes.

The number of mons and storage nodes and the number of OSDs per node and the
size of each OSD volume etc is all configurable and defined in the heat environment
file.  As is the default image used for nodes and which ssh keys to use and so
on.  Security groups are hardcoded in the template.  Network uses public ip.

## Installation prerequisites
1. an openstack environment :) and tacit approval to lean on it a bit
1. a functioning current openstack client installed in a virtualenv:  
.. `virtualenv openstack`  
.. `. openstack/bin/activate`  
.. `pip install python-openstackclient python-heatclient ansible`  
1. an openstack tennancy with sufficient instance and volume quota.  ideally empty.
1. an open-rc.sh file for the tennancy above
1. a deployment node (I use an instance running 16.04) or laptop etc to work from.  
1. a functioning ssh-agent.  if you don't know what that is, you'll need to.
1. understanding of openstack images, default usernames etc

## testing
1. ssh to your deployment node if required
1. `openstack stack list` should work but show no existing stacks
1. `openstack server list` should work too

leave this session open as we'll be using it in a moment

## create cluster via heat template
1. git clone this repo
.. `git clone git@github.com:hooliowobbits/heat-ceph.git`
1. `cd heat-ceph`
1. have a look at heat_ceph_environment.yaml and sanitise for your environment
1. `openstack stack create -t heat_ceph_template.yaml -e heat_ceph_environment.yaml ceph`
```
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| id                  | bed657bf-b172-47e9-af0e-2602fef80ceb |
| stack_name          | ceph                                 |
| description         | No description                       |
| creation_time       | 2017-02-22T01:46:08                  |
| updated_time        | None                                 |
| stack_status        | CREATE_IN_PROGRESS                   |
| stack_status_reason |                                      |
+---------------------+--------------------------------------+`
```
1. `openstack stack show ceph`

everything from here assumes the stack built correctly.  Once built you shouldn't
need to revisit the stack again, it should just work.

## Troubleshooting stack builds
Working out why stuff didn't spawn is as fun as it usually is with Openstack.
Heat issues many requests and stresses the environment somewhat.  If there are
any issues building single instances you're not going to have fun with Heat ;)
You should try/use any/all of the following until you get CREATE_COMPLETE
1. Check stack_status and stack_status_reason
1. `openstack server list` to see what the status of the build is.
1. Have a look at the instances and volumes under the dashboard.  I find it helps.
1. Delete the stack and try again
..`openstack stack delete ceph`
..`openstack stack create -t heat_ceph_template.yaml -e heat_ceph_environment.yaml ceph`
1. Talk to your usual support channels.  You may need to modify the heat template(s)
for your environment.

## so what did we get??
```
$ openstack server list
+----------------------------------+----------------+--------+-------------------+----------------------------------+
| ID                               | Name           | Status | Networks          | Image Name                       |
+----------------------------------+----------------+--------+-------------------+----------------------------------+
| 32302178-90d5-4217-b882-bca477a9 | storage-1-euk  | ACTIVE | nci=x.x.x.x       | NeCTAR Ubuntu 16.04 LTS (Xenial) |
| 2de1                             |                |        |                   | amd64                            |
| 367ea7e6-6e94-4919-9b9f-         | mds-1-euk      | ACTIVE | nci=x.x.x.x       | NeCTAR Ubuntu 16.04 LTS (Xenial) |
| aabb66418f2b                     |                |        |                   | amd64                            |
| 0073c756-cba4-4479-a737-db9e1354 | mon-1-euk      | ACTIVE | nci=x.x.x.x       | NeCTAR Ubuntu 16.04 LTS (Xenial) |
| a9d0                             |                |        |                   | amd64                            |
| 619eaa80-5415-4ae8-b81a-         | storage-0-euk  | ACTIVE | nci=x.x.x.x       | NeCTAR Ubuntu 16.04 LTS (Xenial) |
| c209b76577ae                     |                |        |                   | amd64                            |
| 722c3afa-                        | login-node-euk | ACTIVE | nci=x.x.x.x       | NeCTAR Ubuntu 16.04 LTS (Xenial) |
| ea69-4a5e-a084-5669cca6c3a2      |                |        |                   | amd64                            |
| 1434f1f1-74a5-4dcf-a11a-         | mds-0-euk      | ACTIVE | nci=x.x.x.x       | NeCTAR Ubuntu 16.04 LTS (Xenial) |
| 98d57f405676                     |                |        |                   | amd64                            |
| 08cf0493-8294-42c3-ae3b-         | mon-0-euk      | ACTIVE | nci=x.x.x.x       | NeCTAR Ubuntu 16.04 LTS (Xenial) |
| 76d89bbfcc11                     |                |        |                   | amd64                            |
+----------------------------------+----------------+--------+-------------------+----------------------------------+
```

## name resolution?
all nodes have hosts files populated with every node member, this is done by ansible.

## node naming
all nodes are named in the form <type>-<serial_number>-<random_suffix_for_this_cluster>

# Setup Ceph cluster

## Bootstrap the cluster with ansible
Ansible allows us to collect to all cluster nodes and make them ready.  ansible
normally uses a static inventory file, but in this case inventory.py cleverly
 connects to openstack to determine the cluster nodes.  You should have a look
 at ansible_bootstrap_ceph_deploy.yaml so that you know what it's doing.
1. `ansible-playbook ansible_bootstrap_ceph_deploy.yaml`

If that worked, then the cluster is now ready for ceph.  from here on in,
 all work should be done as the ceph user and from the login node.  

## connect to the (new) login node
1. determine the ip address of the login-node
..`openstack stack output show ceph login-node-public-ip`
1. ssh to the login node as ceph and making sure to enable ssh-agent
.. eg: `ssh -A ceph@xxx.xxx.xxx.xxx`
1. `sudo apt-get install ceph-deploy`

## run cepy-deploy and make things happen!!
Exact steps may differ depending on your ceph release, below assumes jewel.  
Nominally the steps are as follows.  Note that node names are randomly generated
but all follow the same pattern.  
1. `cat /etc/hosts| grep mon`
1. `ceph-deploy install mon-{0,1,2}-euk`
1. `ceph-deploy new mon-{0,1,2}-euk`
1. `ceph-deploy mon create mon-{0,1,2}-euk`
1. `ceph-deploy gatherkeys mon-2-euk`
1. `ceph-deploy disk zap storage-{0,1,2,3,4,5,6}-euk:/dev/vd{b,c,d,e}`
see https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/1.3/html/installation_guide_for_red_hat_enterprise_linux/storage_cluster_quick_start

## Troubleshooting ceph install
I've found that if you purge packages with `ceph-deploy purge mon-{0,1,2}-euk` that reinstallation doesn't work.
You may need to delete the cluster and start again.

# Things i don't like (because they aren't working nicely)
1. Security groups are duplicated and embedded in multiple templates.
1. Need better way to get cluster name resolution.
