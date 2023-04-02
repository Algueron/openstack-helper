# openstack-helper
Collection of useful scripts for Openstack Management

## Open Virtual Network (OVN)

### Get Northbound database content

````bash
export NORTHDB=$(cat /etc/kolla/neutron-server/ml2_conf.ini | grep ovn_nb_connection | awk '{print $3}')
sudo docker exec -it ovn_controller ovn-nbctl --db=$NORTHDB show
````
### Get Southbound database content

````bash
export SOUTHDB=$(cat /etc/kolla/neutron-server/ml2_conf.ini | grep ovn_sb_connection | awk '{print $3}')
sudo docker exec -it ovn_controller ovn-sbctl --db=$SOUTHDB list datapath_binding
````

### Simulate the path of a pack from a VM a to another VM b

````bash
. admin-openrc.sh
export SOUTHDB=$(cat /etc/kolla/neutron-server/ml2_conf.ini | grep ovn_sb_connection | awk '{print $3}')
openstack port set --name ap $(openstack port list --server a -f value -c ID)
openstack port set --name bp $(openstack port list --server b -f value -c ID)
AP_MAC=$(openstack port show -f value -c mac_address ap)
BP_MAC=$(openstack port show -f value -c mac_address bp)

sudo docker exec -it ovn_controller ovn-trace --db=$SOUTHDB n1 'inport == "ap" && eth.src == '$AP_MAC' && eth.dst == '$BP_MAC
````

## Open vSwitch (OVS)

### Display OVS devices

````bash
sudo docker exec openvswitch_vswitchd ovs-vsctl show
````

### Simulate the path of a pack from a VM a to another VM b

````bash
. admin-openrc.sh
openstack port set --name ap $(openstack port list --server a -f value -c ID)
openstack port set --name bp $(openstack port list --server b -f value -c ID)
AP_MAC=$(openstack port show -f value -c mac_address ap)
BP_MAC=$(openstack port show -f value -c mac_address bp)
AP_PORT=$(sudo docker exec openvswitch_vswitchd ovs-vsctl --bare --columns=ofport find  interface external-ids:attached-mac=\"$AP_MAC\")

sudo docker exec openvswitch_vswitchd ovs-appctl ofproto/trace br-int in_port=$AP_PORT,dl_src=$AP_MAC,dl_dst=$BP_MAC
````

### Trace physical routes for real packets

````bash
sudo docker exec -it openvswitch_vswitchd watch ovs-dpctl dump-flows
````

### Create a fake port to listen traffic

````bash
OVS_BRIDGE=br-int
OVS_PORT=tapf9b87f34-cf

# Create a fake Ethernet card snooper0
sudo ip link add name snooper0 type dummy

# Bring snooper0 up
sudo ip link set dev snooper0 up

# Connect snooper0 to the br-int OVS bridge
sudo docker exec openvswitch_vswitchd ovs-vsctl add-port $OVS_BRIDGE snooper0

# Create a mirror of a port of br-int on snooper0
sudo docker exec openvswitch_vswitchd ovs-vsctl -- set Bridge $OVS_BRIDGE mirrors=@m  -- --id=@snooper0 get Port snooper0  -- --id=@$OVS_PORT get Port $OVS_PORT -- --id=@m create Mirror name=mymirror select-dst-port=@$OVS_PORT select-src-port=@$OVS_PORT output-port=@snooper0 select_all=1

# Listen SSH traffic on snooper0
sudo tcpdump -i snooper0 tcp port 22 and host 192.168.0.102 and 172.16.1.2
````

### Delete a fake port

````bash
OVS_BRIDGE=br-int
OVS_PORT=tapf9b87f34-cf

sudo docker exec openvswitch_vswitchd ovs-vsctl clear Bridge $OVS_BRIDGE mirrors
sudo docker exec openvswitch_vswitchd ovs-vsctl del-port $OVS_BRIDGE snooper0
sudo ip link delete dev snooper0
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

### Retrieve the OVS tap device name of a virtual machine

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
