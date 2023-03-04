# openstack-home

## Topology

This private cloud consists of :
- 1 physical host (compute01) with 12 cores and 64GB RAM
- 1 physical host (compute02) with 32 cores and 128GB RAM
- 1 virtual machine (deployment) with 1 core and 2GB RAM

## Prerequisites

- Setup Ubuntu Jammy 22.04 on all hosts
- Create a user with passwordless sudo on all machines
````bash
echo "my user ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/myuser
````
