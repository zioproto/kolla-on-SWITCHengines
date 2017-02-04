
# Get started with Kolla

It is about a week that I am playing with Kolla. I think sharing my notes is good for the Openstack operators community.

Up to `stable/netwon` Kolla is a single project that lives in the git repository:

 - https://github.com/openstack/kolla

In the current master (Ocata not yet released), Kolla is split in two repositories

 - https://github.com/openstack/kolla
 - https://github.com/openstack/kolla-ansible

So in the current master you will not find the directory with the ansible roles, because that directory is now in the new repository.

There is also a `kolla-kubernetes` repo, but I still did not look at that.

My first goal is to deploy Openstack on top of Openstack with Kolla. I will use SWITCHengines that is Openstack Mitaka, and I will try to deploy Openstack Newton.

To get started you need a Operator Seed node, that is the machine where you actually install Kolla, and from where you run the `kolla-ansible` command.

I used Ubuntu Xenial for all my testing. Ubuntu has not yet packages for Kolla. Instead of just installing with `pip` a lot of python stuff and coming up with a deployment that is hard to reproduce, on ```#ubuntu-server``` I got this tip to use http://snapcraft.io.


There are already some openstack tools packaged with snapcraft:

 - https://github.com/openstack-snaps

I looked at what was already done and I tried to package a snap for Kolla my self:

 - https://github.com/zioproto/snap-kolla

It worked quite fast, but I needed to write a couple of Kolla patches:

 - https://review.openstack.org/#/c/426703/
 - https://review.openstack.org/#/c/427139/

Also, because I had a lot of permission issues, I had to introduce this ugly patch to run all ansible things as `sudo`:

 - https://github.com/zioproto/kolla/commit/51cc9ce1cc9d1c17410ef813930a65ac8b3dce4b

At the beginning I tried to fix in a elegant way and add only where necessary the `Become: True`, but my work collided with somebody already working on that:

 - https://blueprints.launchpad.net/kolla-ansible/+spec/ansible-specific-task-become

I hope that all this dirty workarounds will be gone by `stable/ocata`. Apart from this small glitches everything worked pretty good.

For docker, I used this repo on Xenial:
`deb http://apt.dockerproject.org/repo ubuntu-xenial main`

# Understanding HA

Kolla comes with HA built-in. The key idea is to have as frontend two servers sharing a public VIP with VRRP protocol. These frontend nodes run HA proxy in active-backup way. HA-proxy then loadbalances the requests for the API services and for DB and rabbitmq to two or more controller nodes in the backend.

In the standard setup the frontend nodes are called `network` because they act also as neutron network nodes. The nodes in the backend are called `controllers`.


# Run the playbook

To get started source your openstack config and get a tenant with enough quota and run this ansible playbook:
```
cp vars.yaml.template vars.yaml
vim vars.yaml # add your custom config
export ANSIBLE_HOST_KEY_CHECKING=False
source ~/opestack-config
ansible-playbook main.yaml
```
The ansible playbook will create the necessary VMs, will hack the `/etc/hosts` of all VMs so that they look all reachable to each other with names, and it will install kolla on the operator-seed node using my snap package.

To have the frontend VMs share a vip, I used the approach I found on this blog:

 - https://blog.codecentric.de/en/2016/11/highly-available-vips-openstack-vms-vrrp/

The playbook will configure all the openstack networking needed for our tests, and will configure kolla on the operator node.

Now you can ssh to the operator node and start configuring kolla. For this easy example make sure on `/etc/kolla/passwords.yaml` you have at least something written for the following values:
```
database_password:
rabbitmq_password: 
rabbitmq_cluster_cookie:
haproxy_password:
```

If you want you can also just type `kolla-genpwd` and this will enter some password for all the fields in the file.

Now let's get ready to run ansible:

```
export ANSIBLE_HOST_KEY_CHECKING=False
kolla-ansible -i inventory/mariadb bootstrap-servers
kolla-ansible -i inventory/mariadb pull
kolla-ansible -i inventory/mariadb deploy
```

This example inventory that I have put at the path `/home/ubuntu/inventory/mariadb` is a very simplified inventory that will just deploy `mariadb` and `rabbitmq`. Check what I disabled in `/etc/kolla/globals.yml`

# Check what is working

With the command:
```
openstack floating ip list | grep 192.168.22.22
```

You can check the public floating IP applied to the VIP. Check the openstack security groups applied to the frontend VMs. If the necessary ports are open you should be able to access the mysql on port 3306, and the haproxy admin panel on port 1984. The password are the ones on the password.yml file and the username for haproxy is openstack.

# TODO

I will update this file with more steps :) Pull requests are welcome !
