# How to make a Ceph cluster on OpenStack
## What this is and isn't
Is: A virtual ceph cluster is a great way to learn ceph.
Not: Something you should use for production workloads.  There be dragons..

## disclaimer and about heat
heat has the ability to do magical (and dangerous) things to your openstack
environment.  in particular it can call and will call many of the openstack APIs
and like anything from the internet, you should really understand what this does
before running it.  i promise i won't intentionally harm your environment.

`$ openstack stack delete ceph`
`Are you sure you want to delete this stack(s) [y/N]? y`

## prerequisites
1. an openstack environment :) and tacit approval to lean on it a bit
1. a functioning current openstack client:
.. `virtualenv openstack`
.. `. openstack/bin/activate`
.. `pip install python-openstackclient`
.. `pip install python-heatclient`
1. an openstack tennancy with sufficient instance and volume quota
1. ideally an empty tennancy, although it's not compulsory; just safer ;)
1. a deployment node or laptop etc to work from

testing
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

everything from here assumes the stack built correctly.  If it didn't check the
built errors that appear under status(?)

# EVERYTHING HERE NOT VERIFIED YET

## setup cluster comms
bootstrap ansible
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
