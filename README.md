# openstack-home

## Topology

This private cloud consists of :
- 1 physical host (compute01) with 12 cores and 64GB RAM
- 1 physical host (compute02) with 32 cores and 128GB RAM
- 1 virtual machine (deployment) with 1 core and 2GB RAM

We also have a common network 192.168.0.0/24 with DHCP

## Preparation

### Prerequisites for all machines

- Setup Ubuntu Noble 24.04 on all hosts
- Create a user with passwordless sudo on all machines
````bash
echo "$USER ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/$USER
````
- Edit hosts file on all machines to be able to contact physical hosts using hostname (see [Hosts file](etc/hosts))

### Create veth interfaces

- On each physical host, download the veth configuration file
````bash
sudo wget -P /etc/systemd/network/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/systemd/network/25-veth-b1b2.netdev
````
- Restart network service
````bash
sudo systemctl restart systemd-networkd
````

### Setup Networking

- On each physical host, remove the existing netplan configuration
````bash
sudo rm /etc/netplan/00-installer-config.yaml
````
- Download the appropriate netplan configuration file
````bash
sudo wget -P /etc/netplan/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/netplan/10-$HOSTNAME-network.yaml
````
- Apply the configuration
````bash
sudo netplan apply
````

### SSH configuration
- On deployment, upload your custom SSH key pair and ensures the proper access rights are set
````bash
chmod 400 .ssh/id_rsa*
````
- Initiate connection to both compute nodes from deployment
````bash
ssh-copy-id algueron@compute01
ssh-copy-id algueron@compute02
````

### Setup Docker
- On each machine, setup Docker
````bash
sudo apt install -y docker.io
````

## Kolla Ansible setup

### Setup Python
- On deployment, install Python system dependencies
````bash
sudo apt install -y git python3-dev libffi-dev gcc libssl-dev
````
- Install venv
````bash
sudo apt install -y python3-venv
````
- Create a virtual env
````bash
mkdir ~/kolla
python3 -m venv ~/kolla/venv
source ~/kolla/venv/bin/activate
````
- Ensure you're using the latest version of PIP
````bash
pip install -U pip
````

### Setup Ansible

- Install Ansible. Kolla Ansible requires at least Ansible 2.16 and supports up to 2.17.
````bash
pip install 'ansible-core>=2.16,<2.17.99'
````

### Setup Kolla Ansible

- Install Kolla Ansible. We're using the 2024.2 release.
````bash
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2024.2
````
- Create Kolla Ansible configuration directory
````bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
mkdir /etc/kolla/host_vars
````
- Install Kolla Ansible dependencies
````bash
kolla-ansible install-deps
````

## Kolla Ansible configuration

### Password generation

- Download [passwords configuration file](etc/kolla/passwords.yml)
````bash
wget -P /etc/kolla/ <URL REDACTED>
````

### Ansible configuration

- Download [Ansible configuration file](etc/kolla/ansible.cfg)
````bash
wget -P /etc/kolla/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/kolla/ansible.cfg
````

### Kolla Ansible global configuration

- Download [globals configuration file](etc/kolla/globals.yml)
````bash
wget -P /etc/kolla/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/kolla/globals.yml
````

### Inventory configuration

- Download [inventory file](etc/kolla/multinode)
````bash
wget -P /etc/kolla/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/kolla/multinode
````

## Openstack Deployment

### Bootstrap

- Bootstrap servers with kolla deploy dependencies
````bash
kolla-ansible bootstrap-servers -i /etc/kolla/multinode
````
Note: It may fail due to "docker.service: Start request repeated too quickly.". Just re-run the command which should now be fine.

### Certificates Generation

- Generate certificates for Octavia
````bash
kolla-ansible octavia-certificates -i /etc/kolla/multinode
````

### Pre-checks

- Do pre-deployment checks for hosts
````bash
kolla-ansible prechecks -i /etc/kolla/multinode
````

### Deployment

- Finally proceed to actual OpenStack deployment
````bash
kolla-ansible deploy -i /etc/kolla/multinode
````

### Administrator credentials

- OpenStack requires a clouds.yaml file where credentials for the admin user are set. To generate this file :
````bash
kolla-ansible post-deploy -i /etc/kolla/multinode
````

## Cloud Bootstraping

### Openstack CLI Setup

- On deployment node, install Openstack CLI :
````bash
pip install python-openstackclient
````
- Install Octavia extension
````bash
pip install python-octaviaclient
````
- Acquire admin credentials
````bash
. /etc/kolla/admin-openrc.sh
````

### External Network creation

- Create the Provider network
````bash
openstack network create --share --enable --project admin --external --provider-network-type flat --provider-physical-network physnet1 public-net
````
- Create the Provider subnet
````bash
openstack subnet create --project admin --subnet-range "192.168.0.0/24" --dhcp --gateway "192.168.0.1" --ip-version 4 --allocation-pool "start=192.168.0.101,end=192.168.0.219" --dns-nameserver "192.168.1.15" --network public-net public-subnet
````

### Security Groups creation

- Create a security group to allow ICMP
````bash
openstack security group create --project admin --stateful allow-icmp
````
- Add the rule to allow ICMP
````bash
openstack security group rule create --remote-ip "0.0.0.0/0" --protocol icmp --ingress --project admin allow-icmp
````
- Create a security group to allow SSH
````bash
openstack security group create --project admin --stateful allow-ssh
````
- Add the rule to allow SSH
````bash
openstack security group rule create --remote-ip "0.0.0.0/0" --protocol tcp --dst-port 22 --ingress --project admin allow-ssh
````

### SSH Keypair creation

- Create your keypair using your public SSH key
````bash
openstack keypair create --public-key .ssh/id_rsa.pub --type ssh my-key
````

### Images creation

- On deployment node, download the jammy ubuntu image
````bash
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
````
- Upload the image to Glance
````bash
openstack image create --disk-format qcow2 --container-format bare   --public --file noble-server-cloudimg-amd64.img ubuntu-server-24.04
````

### Flavors creation

- Create the default flavors
````bash
openstack flavor create t2.nano --vcpus 1 --ram 512 --disk 20 --public
openstack flavor create t2.micro --vcpus 1 --ram 1024 --disk 20 --public
openstack flavor create t2.small --vcpus 1 --ram 2048 --disk 20 --public
openstack flavor create t2.medium --vcpus 2 --ram 4096 --disk 20 --public
openstack flavor create t2.large --vcpus 2 --ram 8192 --disk 20 --public
openstack flavor create t2.xlarge --vcpus 4 --ram 16384 --disk 20 --public
openstack flavor create t2.2xlarge --vcpus 8 --ram 32768 --disk 20 --public
````
## Octavia (LBaaS) bootstraping

### Amphora image creation

- On deployment node, install system dependencies
````bash
sudo apt install -y debootstrap qemu-utils git kpartx
````
- Acquire the Octavia source code
````bash
git clone https://opendev.org/openstack/octavia -b stable/2024.2
````
- Install diskimage-builder
````bash
sudo pip3 install diskimage-builder
````
- Create the Amphora image
````bash
octavia/diskimage-create/diskimage-create.sh
````
- Source octavia user openrc
````bash
. /etc/kolla/octavia-openrc.sh
````
- Register the image in Glance
````bash
openstack image create amphora-x64-haproxy.qcow2 --container-format bare --disk-format qcow2 --private --tag amphora --file amphora-x64-haproxy.qcow2 --property hw_architecture='x86_64' --property hw_rng_model=virtio
````

### Network configuration

Kolla Ansible generates a Octavia tenant network as a VXLAN Neutron network. However, this network will manifest itself as a local VLAN tag on the integration bridge of each node on which a port is connected to this network. For Octavia to work properly, the Controller itself needs physical access to the Octavia network, which is why we need to retrieve the VLAN tag for the Octavia network, and then plug the Health Manager onto this VLAN.

- On deployment node, copy the Octavia credentials to the network controller
````bash
scp /etc/kolla/octavia-openrc.sh $USER@compute01:
````
- On Controller node, install Openstack client
````bash
sudo pip3 install python-openstackclient -c https://releases.openstack.org/constraints/upper/zed
````
- Acquire Octavia service credentials
````bash
source octavia-openrc.sh
````
- Retrieve the Load Balancer subnet GUID
````bash
SUBNET_ID=$(openstack subnet show lb-mgmt-subnet -f value -c id)
````
- Create a port for Octavia Health Manager
````bash
MGMT_PORT_ID=$(openstack port create --security-group lb-health-mgr-sec-grp --device-owner Octavia:health-mgr --host=compute01 -c id -f value --network lb-mgmt-net --fixed-ip subnet=$SUBNET_ID,ip-address=10.1.0.2 octavia-health-manager-listen-port)
````
- Get the MAC Address of Octavia Health Manager port
````bash
MGMT_PORT_MAC=$(openstack port show -c mac_address -f value $MGMT_PORT_ID)
````
- Retrieve the GUID of the DCHP port
````bash
DHCP_PORT_ID=$(openstack port list --network lb-mgmt-net --device-owner network:dhcp -c id -f value)
````
- Generate the corresponding Open vSwitch Port
````bash
OVS_DHCP_PORT_ID=tap${DHCP_PORT_ID:0:11}
````
- Get the VLAN tag of this port from OVS DB
````bash
OVS_TAG=$(sudo docker exec openvswitch_vswitchd ovsdb-client dump  unix:/var/run/openvswitch/db.sock Open_vSwitch Port name tag | tr -d '\r' | grep $OVS_DHCP_PORT_ID | awk '{print $2'})
````
- Generate the name of the OVS link for the Health Manager
````bash
OVS_MGMT_PORT_ID=lb-${MGMT_PORT_ID:0:11}
````
- Create an OVS link for the Health Manager on br-int
````bash
sudo docker exec -it openvswitch_vswitchd ovs-vsctl add-port br-int $OVS_MGMT_PORT_ID -- set interface $OVS_MGMT_PORT_ID type=internal -- set port $OVS_MGMT_PORT_ID tag=$OVS_TAG -- set port $OVS_MGMT_PORT_ID other-config:hwaddr=$MGMT_PORT_MAC
````
- Edit the netplan configuration to add the link
````YAML
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp4s0:
[...]
    lb-d43756e0-f2:
      addresses:
      - 10.1.0.2/24
      routes:
      - to: 10.1.0.1/24
        via: 10.1.0.1
````
- Apply the changes
````bash
sudo netplan apply
````
- On deployment node, change the Health Manager configuration
````bash
wget -P /etc/kolla/config/octavia/compute01/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/kolla/config/compute01/octavia.conf
wget -P /etc/kolla/config/octavia/compute02/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/kolla/config/compute02/octavia.conf
````
- Apply Octavia changes
````bash
kolla-ansible -i /etc/kolla/multinode reconfigure -t octavia
````
- Restart Neutron Open vSwitch Agent
````bash
sudo docker restart neutron_openvswitch_agent
````
- Restart Octavia Health Manager
````bash
sudo docker restart octavia_health_manager
````


### Health Manager Security Rules

For some unknown reason, the lb-health-mgr-sec-grp does not have a rule allowing UDP 5555, so let's add it
````bash
openstack security group rule create --remote-ip "0.0.0.0/0" --protocol udp --dst-port 5555 --ingress --project service lb-health-mgr-sec-grp
````

## On reboot

- If a machine reboots, use the following script to make Octavia work
````bash
source octavia-openrc.sh
SUBNET_ID=$(openstack subnet show lb-mgmt-subnet -f value -c id)
MGMT_PORT_ID=$(openstack port list --device-owner Octavia:health-mgr --host=$HOSTNAME -c id -f value --network lb-mgmt-net)
MGMT_PORT_MAC=$(openstack port show -c mac_address -f value $MGMT_PORT_ID)
OVS_MGMT_PORT_ID=lb-${MGMT_PORT_ID:0:11}
OVS_TAG=$(sudo docker exec openvswitch_vswitchd ovsdb-client dump  unix:/var/run/openvswitch/db.sock Open_vSwitch Port name tag | tr -d '\r' | grep $OVS_MGMT_PORT_ID | awk '{print $2'})
sudo docker exec -it openvswitch_vswitchd ovs-vsctl del-port br-int $OVS_MGMT_PORT_ID
sudo docker exec -it openvswitch_vswitchd ovs-vsctl add-port br-int $OVS_MGMT_PORT_ID -- set interface $OVS_MGMT_PORT_ID type=internal -- set port $OVS_MGMT_PORT_ID tag=$OVS_TAG -- set port $OVS_MGMT_PORT_ID other-config:hwaddr=$MGMT_PORT_MAC
sudo docker restart neutron_openvswitch_agent
sudo docker restart octavia_health_manager
````
