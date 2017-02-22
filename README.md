# How to make a Ceph cluster on OpenStack
## What this is and isn't
Is: A virtual ceph cluster is a great way to learn ceph.  
Not: Something you should use for production workloads.  There be dragons..

## disclaimer and about heat
Openstack Heat has the ability to do magical (and dangerous) things to your openstack
environment.  in particular it can call and will call many of the openstack APIs
and like anything from the internet, you should really understand what this does
before running it.  i promise i won't intentionally harm your environment but
that is no guarantee that something bad will not happen.  be careful.  

`$ openstack stack delete ceph`  
`Are you sure you want to delete this stack(s) [y/N]? y`

## Installation prerequisites
1. an openstack environment :) and tacit approval to lean on it a bit
1. a functioning current openstack client:  
.. `virtualenv openstack`  
.. `. openstack/bin/activate`  
.. `pip install python-openstackclient`  
.. `pip install python-heatclient`  
1. an openstack tennancy with sufficient instance and volume quota.  ideally empty.
1. an open-rc.sh file for the tennancy above
1. a deployment node (I use an instance running 16.04) or laptop etc to work from.  

## testing
1. `openstack stack list` should work but show no existing stacks
1. `openstack server list` should work too

leave this session open as we'll be using it in a moment

## create cluster via heat template
1. ssh to your deployment node if required
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
1. wait, then check stack deployment status with
1. `openstack stack show ceph`

everything from here assumes the stack built correctly.  Once built you shouldn't
need to revisit the stack again, it should just work.

## Troubleshooting stack builds
Working out why stuff didn't spawn is as fun as it usually is with Openstack.
Heat issues many requests and stresses the environment somewhat.  If there are
any issues building single instances you're not going to have fun with Heat ;)
You should try/use any/all of the following until you get CREATE_COMPLETE
1. Check the build errors that appear under status(?).
1. `openstack server list` to see what the status of the build is.
1. Have a look at the instances and volumes under the dashboard.
1. Delete the stack and try again
..`openstack stack delete ceph`
..`openstack stack create -t heat_ceph_template.yaml -e heat_ceph_environment.yaml ceph`

# Setup ceph cluster
From here on most work happens from the login node of the ceph cluster.  

## Setup cluster comms
1. determine the ip address of the login-node.  Caution if you have multiple ceph stacks ;)
..`openstack server list | grep login`
..or `openstack stack ceph show | grep login`
1. ssh to the login node, where username and ip address are appropriate to your image and instance

### bootstrap ansible
1. generate a list of the storage nodes
.. `openstack server list | grep storage`

etc and build up /etc/ansible/hosts on the login node.  I'd like this to be automated.

# EVERYTHING


1. @storage-nodes = `heat output-show ceph storage-nodes | grep -E -o '[a-z0-9.]+'`
1. @mons = `heat output-show ceph mons | grep -E -o '[a-z0-9.]+'`
1. $salt-master-ip = `heat output-show ceph salt-master | grep -E -o '[a-z0-9.]+'`
1. ssh ubuntu@$salt-master-ip
1. vi /etc/ansible/hosts
1. enter @mons under a [mons] heading, same with storage-nodes
1. `ansible all -o -s -a 'whoami'`

salt-minions seem to require a restart before reporting in
1. `ansible all -s -s -a 'service salt-minion restart'`
1. `sudo salt-key`
1. `sudo salt-key --accept-all`
1. `sudo salt '*' test.ping`

## go yay!

## think about ceph stuff
