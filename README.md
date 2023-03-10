# openstack-home

## Topology

This private cloud consists of :
- 1 physical host (compute01) with 12 cores and 64GB RAM
- 1 physical host (compute02) with 32 cores and 128GB RAM
- 1 virtual machine (deployment) with 1 core and 2GB RAM

We also have 2 networks :
- Management network : 192.168.0.0/24 with DHCP
- Provider network : 172.16.0.0/12 without DHCP

Physical hosts must be connected to both networks, deployment node only needs Management network.

## Preparation

### Prerequisites for all machines

- Setup Ubuntu Jammy 22.04 on all hosts
- Create a user with passwordless sudo on all machines
````bash
echo "$USER ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/$USER
````
- Edit hosts file on all machines to be able to contact physical hosts using hostname (see [Hosts file](etc/hosts))

### Setup Promiscuous mode on Provider Network

- On each physical host, get the list of network interfaces
````bash
ip addr
````
- Create a service bridge-promisc to enable promiscuous mode (replace enp4s0 and enp7s0 with appropriate interfaces)
[bridge-promisc.service](etc/systemd/system/bridge-promisc.service)
- Reload the services definitions
````bash
sudo systemctl daemon-reload
````
- Starts the service
````bash
sudo systemctl start bridge-promisc
````
- Activate the service at reboot
````bash
sudo systemctl enable bridge-promisc
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
- Install PIP
````bash
sudo apt install -y python3-pip
````
- Ensure you're using the latest version of PIP
````bash
sudo pip install -U pip
````

### Setup Ansible

- Install Ansible. Kolla Ansible requires at least Ansible 4 and supports up to 5.
````bash
sudo pip3 install 'ansible>=4,<6'
````

### Setup Kolla Ansible

- Install Kolla Ansible. We're using the Zed release.
````bash
sudo pip3 install git+https://opendev.org/openstack/kolla-ansible@stable/zed
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
wget -P /etc/kolla/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/kolla/passwords.yml
````
- Generate random passwords
````bash
kolla-genpwd
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
- Download [compute01 configuration file](etc/kolla/host_vars/compute01)
````bash
wget -P /etc/kolla/host_vars/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/kolla/host_vars/compute01
````
- Download [compute02 configuration file](etc/kolla/host_vars/compute02)
````bash
wget -P /etc/kolla/host_vars/ https://raw.githubusercontent.com/Algueron/openstack-home/main/etc/kolla/host_vars/compute02
````

## Openstack Deployment

### Bootstrap

- Bootstrap servers with kolla deploy dependencies
````bash
kolla-ansible -i /etc/kolla/multinode bootstrap-servers
````
Note: It may fail due to "docker.service: Start request repeated too quickly.". Just re-run the command which should now be fine.

### Certificates Generation

- Generate certificates for Octavia
````bash
kolla-ansible octavia-certificates
````

### Pre-checks

- Do pre-deployment checks for hosts
````bash
kolla-ansible -i /etc/kolla/multinode prechecks
````

### Deployment

- Finally proceed to actual OpenStack deployment
````bash
kolla-ansible -i /etc/kolla/multinode deploy
````

### Administrator credentials

- OpenStack requires a clouds.yaml file where credentials for the admin user are set. To generate this file :
````bash
kolla-ansible -i /etc/kolla/multinode post-deploy
````

## Cloud Bootstraping

### Openstack CLI Setup

- On deployment node, install Openstack CLI :
````bash
sudo pip3 install python-openstackclient -c https://releases.openstack.org/constraints/upper/zed
````
- Install Octavia extension
````bash
sudo pip3 install python-octaviaclient -c https://releases.openstack.org/constraints/upper/zed
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
openstack subnet create --project admin --subnet-range "172.16.0.0/12" --dhcp --gateway "172.16.0.1" --ip-version 4 --network public-net public-subnet
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
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
````
- Upload the image to Glance
````bash
openstack image create --disk-format qcow2 --container-format bare   --public --file jammy-server-cloudimg-amd64.img ubuntu-server-22.04
````

### Flavors creation

- Create the default flavors
````bash
openstack flavor create t2.nano --vcpus 1 --ram 512 --disk 5 --public
openstack flavor create t2.micro --vcpus 1 --ram 1024 --disk 5 --public
openstack flavor create t2.small --vcpus 1 --ram 2048 --disk 10 --public
openstack flavor create t2.medium --vcpus 2 --ram 4096 --disk 20 --public
openstack flavor create t2.large --vcpus 2 --ram 8192 --disk 40 --public
openstack flavor create t2.xlarge --vcpus 4 --ram 16384 --disk 80 --public
openstack flavor create t2.2xlarge --vcpus 8 --ram 32768 --disk 120 --public
````
## Octavia bootstraping

### Amphora image creation

- On deployment node, install system dependencies
````bash
sudo apt install -y debootstrap qemu-utils git kpartx
````
- Acquire the Octavia source code
````bash
git clone https://opendev.org/openstack/octavia -b stable/zed
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

- Retrieve the ID of Octavia subnet
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
- On compute01 node, create the bridge for the Octavia worker
````bash
INTERFACE=o-hm0
MGMT_PORT_ID=xxx
MGMT_PORT_MAC=yyyy
sudo docker exec -it openvswitch_vswitchd ovs-vsctl -- --may-exist add-port ${OVS_BRIDGE:-br-int} $INTERFACE -- set Interface $INTERFACE type=internal -- set Interface $INTERFACE external-ids:iface-status=active -- set Interface $INTERFACE external-ids:attached-mac=$MGMT_PORT_MAC -- set Interface $INTERFACE external-ids:iface-id=$MGMT_PORT_ID -- set Interface $INTERFACE external-ids:skip_cleanup=true
````
