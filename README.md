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
- Acquire admin credentials
````bash
. /etc/kolla/admin-openrc.sh
````

### External Network creation

- Create the Provider network
````bash
openstack network create --share --enable --project admin --external --provider-network-type flat --provider-physical-network physnet1 provider01
````
- Create the Provider subnet
````bash
openstack subnet create --project admin --subnet-range "172.16.0.0/12" --dhcp --gateway "172.16.0.1" --ip-version 4 --network provider01 provider-subnet
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
- Add the rule to allow ICMP
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
openstack flavor create c1.medium --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create c1.xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create c3.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create c3.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create c3.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create c3.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create c3.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create c4.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create c4.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create c4.8xlarge --id auto --vcpus 36 --ram 36864 --ephemeral 5 --public
openstack flavor create c4.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create c4.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create c5.18xlarge --id auto --vcpus 72 --ram 73728 --ephemeral 5 --public
openstack flavor create c5.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create c5.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create c5.9xlarge --id auto --vcpus 36 --ram 36864 --ephemeral 5 --public
openstack flavor create c5.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create c5.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create cc2.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create cg1.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create cr1.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create d2.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create d2.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create d2.8xlarge --id auto --vcpus 36 --ram 36864 --ephemeral 5 --public
openstack flavor create d2.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create f1.16xlarge --id auto --vcpus 64 --ram 65536 --ephemeral 5 --public
openstack flavor create f1.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create g2.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create g2.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create g3.16xlarge --id auto --vcpus 64 --ram 65536 --ephemeral 5 --public
openstack flavor create g3.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create g3.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create hi1.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create hs1.8xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create i2.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create i2.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create i2.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create i2.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create i3.16xlarge --id auto --vcpus 64 --ram 65536 --ephemeral 5 --public
openstack flavor create i3.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create i3.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create i3.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create i3.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create i3.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create m1.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create m1.medium --id auto --vcpus 1 --ram 1024 --ephemeral 5 --public
openstack flavor create m1.small --id auto --vcpus 1 --ram 1024 --ephemeral 5 --public
openstack flavor create m1.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create m2.2xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create m2.4xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create m2.xlarge --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create m3.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create m3.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create m3.medium --id auto --vcpus 1 --ram 1024 --ephemeral 5 --public
openstack flavor create m3.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create m4.10xlarge --id auto --vcpus 40 --ram 40960 --ephemeral 5 --public
openstack flavor create m4.16xlarge --id auto --vcpus 64 --ram 65536 --ephemeral 5 --public
openstack flavor create m4.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create m4.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create m4.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create m4.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create p2.16xlarge --id auto --vcpus 64 --ram 65536 --ephemeral 5 --public
openstack flavor create p2.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create p2.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create r3.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create r3.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create r3.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create r3.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create r3.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create r4.16xlarge --id auto --vcpus 64 --ram 65536 --ephemeral 5 --public
openstack flavor create r4.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create r4.4xlarge --id auto --vcpus 16 --ram 16384 --ephemeral 5 --public
openstack flavor create r4.8xlarge --id auto --vcpus 32 --ram 32768 --ephemeral 5 --public
openstack flavor create r4.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create r4.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create t1.micro --id auto --vcpus 1 --ram 1024 --ephemeral 5 --public
openstack flavor create t2.2xlarge --id auto --vcpus 8 --ram 8192 --ephemeral 5 --public
openstack flavor create t2.large --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create t2.medium --id auto --vcpus 2 --ram 2048 --ephemeral 5 --public
openstack flavor create t2.micro --id auto --vcpus 1 --ram 1024 --ephemeral 5 --public
openstack flavor create t2.nano --id auto --vcpus 1 --ram 1024 --ephemeral 5 --public
openstack flavor create t2.small --id auto --vcpus 1 --ram 1024 --ephemeral 5 --public
openstack flavor create t2.xlarge --id auto --vcpus 4 --ram 4096 --ephemeral 5 --public
openstack flavor create x1.16xlarge --id auto --vcpus 64 --ram 65536 --ephemeral 5 --public
openstack flavor create x1.32xlarge --id auto --vcpus 128 --ram 131072 --ephemeral 5 --public
````
