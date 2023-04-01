# openstack-helper
Collection of useful scripts for Openstack Management

## Open Virtual Network (OVN)

### Get Northbound database

````bash
export NORTHDB=$(cat /etc/kolla/neutron-server/ml2_conf.ini | grep ovn_nb_connection | awk '{print $3}')
sudo docker exec -it ovn_controller ovn-nbctl --db=$NORTHDB show
````
### Get Southbound database

````bash
export SOUTHDB=$(cat /etc/kolla/neutron-server/ml2_conf.ini | grep ovn_sb_connection | awk '{print $3}')
sudo docker exec -it ovn_controller ovn-sbctl --db=$SOUTHDB list datapath_binding
````

## Virtual Machines

### Retrieve the hypervisor host of a virtual machine

````bash
. admin-openrc.sh
export OS_PROJECT=myproject
export VM_NAME=myvm

# Get VM ID
VM_ID=$(openstack server list --project $OS_PROJECT --name $VM_NAME -f value -c ID)

# Get VM Hypervisor
VM_HYPERVISOR=$(openstack server show -c OS-EXT-SRV-ATTR:hypervisor_hostname -f value $VM_ID)
````

## Retrieve the OVS tap device name of a virtual machine

````bash
. admin-openrc.sh
export OS_PROJECT=myproject
export VM_NAME=myvm

# Get VM ID
VM_ID=$(openstack server list --project $OS_PROJECT --name $VM_NAME -f value -c ID)

# Retrieve the port of the VM
PORT_ID=$(openstack port list --project $OS_PROJECT --server $VM_ID -f value -c ID)

# Generates the tap device name
OVS_VM_TAP_ID=tap${PORT_ID:0:11}
````
