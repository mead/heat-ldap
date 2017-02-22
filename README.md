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

## architecture
This environment has the following topology:
1. A deployment node (ie; where you normally run openstack client from).  That's
where you run the heat commands in order to create the cluster.
1. Once the heat stack is up, a login node is created; what's where you run ceph-deploy from
1. And finally, once the ceph cluster is up, mons and storage nodes and all the rest.
1. The ceph cluster and deployment nodes etc all use public ip addresses.
1. The security groups still need work and may break cluster comms.  Yes this may not work.

The number of mons and storage nodes and the number of OSDs per node and the
size of each OSD volume etc is all configurable and defined in the heat environment
file.  As is the default image used for nodes and which ssh keys to use and so
on.  It's not all in there, security groups are still hardcoded in the template
(help please!) but it works ok.

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
1. a functioning ssh-agent.  if you don't know what that is, you'll need to.
1. understand openstack images, default usernames etc

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
1. no really.  have a look at heat_ceph_environment.yaml; it's critical.
1. then create the stack:
. `openstack stack create -t heat_ceph_template.yaml -e heat_ceph_environment.yaml ceph`
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
1. Have a look at the instances and volumes under the dashboard.  I find it helps.
1. Delete the stack and try again
..`openstack stack delete ceph`
..`openstack stack create -t heat_ceph_template.yaml -e heat_ceph_environment.yaml ceph`
1. cry?
1. Talk to your usual support channels.  You may need to modify the heat template(s)
for your environment.

# Setup Ceph cluster (ansible and ceph-deploy)
From here on most work happens from the login node of the ceph cluster.  Unless
otherwise advised, all commands now happen there.

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

## connect to the (new) login node
1. determine the ip address of the login-node.  Caution if you have multiple ceph stacks ;)
..`openstack server list | grep login`
..or `openstack stack ceph show | grep login`
1. note the default username of the glance image you're using
1. ssh to the login node making sure to enable ssh-agent
.. eg: `ssh -A ubuntu@xxx.xxx.xxx.xxx`

## manually :( bootstrap ansible
Ansible on the login node needs to know about your instances.  In an ideal world
/etc/ansible/hosts would be populated automatically, but it's not yet (help please!).
Until then we make it manually; apologies for that.
1. connect to the deployment node (ie where you're running the openstack client)
1. generate a list of the storage nodes
.. `openstack server list | grep storage` etc, and build up the file.  eg;
```
[storage]
130.56.253.179
130.56.253.175
..

[mons]
130.56.253.164
130.56.253.160
..
```
etc
1. test ansible connectivity:
. `ansible all -s -o 'hostname'`
1. type yes alot.  or fix ansible config to not require that..

## bootstrap ceph-deploy
1. connect to your login node as ubuntu
1. `git clone git@github.com:hooliowobbits/heat-ceph.git`
1. `cd heat-ceph`
1. `ansible-playbook ansible_bootstrap_ceph_deploy.yaml`

what just happened?  not much, but the login-node should now have a ceph user
and so should all the ceph nodes.  also the authorized_keys for ubuntu has been
copied over for ceph, so you should now exit and login as ceph
1. exit ssh
1. login again with `ssh -A ceph@x.x.x.x`
1. `ansible all -s -o -a 'whoami'`
1. `pip install ceph-deploy`

from here on in, all work should be done as ceph, and from the login node.

## setup name resolution
nasty hack;
1. `openstack  server list`
1. paste into google spreadsheets..
1. handcraft /etc/hosts on login node

## run cepy-deploy and make things happen!!
out of scope ;)

see http://docs.ceph.com/docs/master/rados/deployment/ceph-deploy-new/

# Things i don't like (because they aren't working nicely)
1. Security groups are duplicated and embedded in multiple templates.
1. Need better way to get cluster name resolution.
