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
echo "myuser ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/myuser
````
- Edit hosts file on all machines to be able to contact physical hosts using hostname (see [Hosts file](etc/hosts))

### Setup Promiscuous mode on Provider Network

- On each physical host, get the list of network interfaces
````bash
ip addr
````
- Create a service bridge-promisc to enable promiscuous mode (replace enp7s0 with appropriate interface)
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
