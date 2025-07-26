---
title: "Deploying Trove DBaaS with OpenStack 2025.1 and Kolla-Ansible"
layout: post
date: 2025-07-26 15:35
image: /assets/images/markdown.jpg
headerImage: false
tag:
- trove
- openstack
star: true
category: blog
author: vurmil
description: Trove is the Database-as-a-Service (DBaaS) component of OpenStack, allowing users to provision and manage databases on demand through the OpenStack API. This guide covers how to deploy Trove with OpenStack 2025.1 using Kolla-Ansible.
---

**Title: Deploying Trove DBaaS with OpenStack 2025.1 and Kolla-Ansible**

Trove is the Database-as-a-Service (DBaaS) component of OpenStack, allowing users to provision and manage databases on demand through the OpenStack API. This guide covers how to deploy Trove with OpenStack 2025.1 using Kolla-Ansible.

I have deployed OpenStack using Kolla-Ansible, including core services like Keystone, Nova, Neutron, Cinder, Glance, and Horizon. I wanted to add Trove as a DBaaS solution for my users.

My environment consists of three controller nodes and several compute nodes. The OpenStack deployment uses VXLAN overlay networks with a bonded 10G NIC. I’ve created a dedicated VLAN for the Trove management network, which is used by Trove guest instances to communicate with RabbitMQ and the Trove conductor.

I only assign an IP from this DBaaS VLAN to the controller nodes (not compute nodes). Below is my sample netplan configuration to give you an idea:

# VxLAN for 10G Bond

bond10.11:  
id: 11  
link: bond10  
dhcp4: false  
dhcp6: false  
mtu: 9000  
addresses: \[172.16.8.14/22\]

# LBaaS (Octavia)

bond10.12:  
id: 12  
link: bond10  
dhcp4: false  
dhcp6: false  
mtu: 1500  
addresses: \[172.17.8.14/22\]

# DBaaS (Trove)

bond10.15:  
id: 15  
link: bond10  
dhcp4: false  
dhcp6: false  
mtu: 1500  
addresses: \[172.19.8.14/22\]

My OpenStack API runs on a separate network from the Trove management network. To allow communication, I configured host routes to the Trove subnet from within the OpenStack subnet.

For example:

In the subnet definition for the OpenStack management network, I added the following host route:

Destination CIDR: 172.19.8.0/22  
Next Hop: 172.16.8.14

This makes the OpenStack API network aware of how to reach the DBaaS network.

Trove guest instances must also reach RabbitMQ, Trove conductor, and Trove API. I use host routes and NAT to achieve this.

In addition, I ensured the guest image has appropriate metadata:

* *   Distro: Ubuntu 22.04
*     
* *   Root partition: 10 GB
*     
* *   Includes cloud-init
*     
* *   Guest agent pre-installed and enabled
*     
* *   rabbitmq, trove-conductor, and trove-api reachable via routes
*     

You can build your own image or use diskimage-builder (DIB) provided by OpenStack.

Once the image is ready, upload it to Glance and tag it accordingly:

openstack image set --tag trove ubuntu-22.04-trove  
openstack image set --tag mysql ubuntu-22.04-trove

In your `globals.yml`, make sure you include the following:

enable\_trove: "yes"  
trove\_custom\_config: "/etc/kolla/config/trove"

I used the following config overrides:

In `/etc/kolla/config/trove/trove.conf`:

\[DEFAULT\]  
control\_exchange = trove  
transport\_url = rabbit://openstack:<password>@<vip>:5672/  
network\_driver = trove.network.neutron.NeutronDriver  
taskmanager\_manager = trove.taskmanager.manager.Manager  
trove\_api\_workers = 4  
api\_paste\_config = /etc/trove/api-paste.ini  
debug = true  
log\_file = trove.log  
use\_syslog = False

\[database\]  
connection = mysql+pymysql://trove:<password>@<vip>/trove

\[nova\]  
region\_name = RegionOne  
endpoint\_type = internalURL  
service\_name = nova

\[neutron\]  
region\_name = RegionOne  
endpoint\_type = internalURL  
service\_name = neutron

\[cinder\]  
region\_name = RegionOne  
endpoint\_type = internalURL  
service\_name = cinder

\[glance\]  
region\_name = RegionOne  
endpoint\_type = internalURL  
service\_name = glance

\[heat\]  
region\_name = RegionOne  
endpoint\_type = internalURL  
service\_name = heat

\[keystone\_authtoken\]  
auth\_url = http://<vip>:5000  
project\_domain\_name = Default  
user\_domain\_name = Default  
project\_name = service  
username = trove  
password = <password>  
auth\_type = password

To deploy Trove:

kolla-ansible -i /etc/kolla/multinode deploy --tags trove

Then, register Trove service in Keystone:

openstack service create --name trove --description "Database as a Service" database  
openstack endpoint create --region RegionOne database public http://<vip>:8779/v1.0/%(tenant\_id)s  
openstack endpoint create --region RegionOne database internal http://<vip>:8779/v1.0/%(tenant\_id)s  
openstack endpoint create --region RegionOne database admin http://<vip>:8779/v1.0/%(tenant\_id)s

To test Trove, create a datastore and version:

openstack datastore version create mysql 5.7 mysql ubuntu-22.04-trove --image-tags trove mysql --active

Then create an instance:

openstack database instance create mysql1 --flavor m1.small --size 5 --datastore mysql --datastore-version 5.7 --nic net-id=<uuid>

Check logs in the Trove containers:

docker exec -it trove\_api tail -f /var/log/kolla/trove/trove-api.log  
docker exec -it trove\_taskmanager tail -f /var/log/kolla/trove/trove-taskmanager.log  
docker exec -it trove\_conductor tail -f /var/log/kolla/trove/trove-conductor.log

Make sure security groups and Neutron config allow DHCP and metadata services. Also, ensure RabbitMQ is reachable from the Trove guest instances.

If you’re using SELinux or AppArmor, verify nothing is being blocked.

Conclusion:

Trove can be a powerful addition to your OpenStack cloud, offering managed databases to your users. With proper networking, image preparation, and service configuration, deploying Trove with Kolla-Ansible is straightforward and stable. Make sure you align networking, image metadata, and service credentials to ensure everything works end-to-end.
