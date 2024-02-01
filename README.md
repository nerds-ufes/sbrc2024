# SBRC2024

Official repository for the paper "Sliced WANs for Data-Intensive Science: Deployment Experiences and Performance Analysis" to [SBRC 2024](https://sbrc.sbc.org.br/2024/).


## Abstract

The task of transferring massive data sets in Data-Intensive Science (DIS) systems, such as those generated from high-energy experiments at CERN in the EU and at Sirius synchrotron light source in Brazil, often rely on physical WAN infrastructure for network connectivity that is provided by various National Research and Education Networks (NRENs), including ESnet, Géant, Internet2, RNP, among others. Sliced WANs bring a new paradigm for infrastructure yet to be exploited by DIS, but a realistic study of these particular systems poses a significant challenge due to their complexity, scale, and the number of factors affecting the data transport. In this paper, we take the first steps in addressing these challenges by deploying and evaluating a virtual infrastructure for data transport within a representative national-scale WAN. Our approach here encompasses two main aspects: i) Evaluating the performance of TCP congestion control algorithms (BBR versus Cubic) when only a single path is available for the data transfer; and ii) Assessing the performance of flow completion times (related to the management of bandwidth allocation) for sets of interdependent transfers environment provided by a network slice.

## Slice Deployment and Experimentation with the FABRIC Testbed

Import the FABlib API.

```python
from fabrictestbed_extensions.fablib.fablib import FablibManager as fablib_manager

try:
    fablib = fablib_manager()
    
    fablib.show_config()
except Exception as e:
    print(f"Exception: {e}")
```

Submit a slice request. In addition, the progress of your slice’s build process will be printed.

```python
from ipaddress import ip_address, IPv4Address, IPv6Address, IPv4Network, IPv6Network

slice_name = 'Coast-to-Coast'
os_name = 'default_rocky_8'
model_name = 'NIC_ConnectX_5'

try:
    # Creates a new slice
    slice = fablib.new_slice(name=slice_name)
    
    # Creates the subnetworks
    subnet1 = IPv4Network("192.168.1.0/24")
    subnet2 = IPv4Network("192.168.2.0/24")
    subnet3 = IPv4Network("192.168.3.0/24")
    subnet4 = IPv4Network("192.168.4.0/24")
    
    # Lists of available ips
    net1_available_ips = list(subnet1)[1:]
    net2_available_ips = list(subnet2)[1:]
    net3_available_ips = list(subnet3)[1:]
    net4_available_ips = list(subnet4)[1:]
    
    # Creates l2 overlay networks
    net1 = slice.add_l2network(name='net1', subnet=subnet1)
    net2 = slice.add_l2network(name='net2', subnet=subnet2)
    net3 = slice.add_l2network(name='net3', subnet=subnet3)
    net4 = slice.add_l2network(name='net4', subnet=subnet4)
    
    # Adds H1, R1, R2, R3 and H2 nodes to the slice
    h1 = slice.add_node(name='h1', site='SEAT', image=os_name)
    [h1n1, h1n2] = h1.add_component(model=model_name, name='nic1').get_interfaces()
    h1n1.set_mode('config')
    net1.add_interface(h1n1)
    h1n1.set_ip_addr(net1_available_ips.pop(0))
    
    r1 = slice.add_node(name='r1', site='SALT', image=os_name)
    [r1n1, r1n2] = r1.add_component(model=model_name, name='nic1').get_interfaces()
    r1n1.set_mode('config')
    net1.add_interface(r1n1)
    r1n1.set_ip_addr(net1_available_ips.pop(0))
    r1n2.set_mode('config')
    net2.add_interface(r1n2)
    r1n2.set_ip_addr(net2_available_ips.pop(0))
    
    r2 = slice.add_node(name='r2', site='STAR', image=os_name)
    [r2n1, r2n2] = r2.add_component(model=model_name, name='nic1').get_interfaces()
    r2n1.set_mode('config')
    net2.add_interface(r2n1)
    r2n1.set_ip_addr(net2_available_ips.pop(0))
    r2n2.set_mode('config')
    net3.add_interface(r2n2)
    r2n2.set_ip_addr(net3_available_ips.pop(0))
    
    r3 = slice.add_node(name='r3', site='NEWY', image=os_name)
    [r3n1, r3n2] = r3.add_component(model=model_name, name='nic1').get_interfaces()
    r3n1.set_mode('config')
    net3.add_interface(r3n1)
    r3n1.set_ip_addr(net3_available_ips.pop(0))
    r3n2.set_mode('config')
    net4.add_interface(r3n2)
    r3n2.set_ip_addr(net4_available_ips.pop(0))
    
    h2 = slice.add_node(name='h2', site='MASS', image=os_name)
    [h2n1, h2n2] = h2.add_component(model=model_name, name='nic1').get_interfaces()
    h2n1.set_mode('config')
    net4.add_interface(h2n1)
    h2n1.set_ip_addr(net4_available_ips.pop(0))
    
    # Submit Slice Request
    slice.submit()
    
except Exception as e:
    print(f"Exception: {e}")

```

Connectivity test of directly connected nodes. Note that it is not necessary to instantiate H2.
```python
try:
    slice = fablib.get_slice(name=slice_name)

    h1 = slice.get_node('h1')
    r1 = slice.get_node('r1')
    r2 = slice.get_node('r2')
    r3 = slice.get_node('r3')
    
    stdout, stderr = h1.execute(f'ping 192.168.1.2 -c 4') # h1 -> r1
    stdout, stderr = r1.execute(f'ping 192.168.2.2 -c 4') # r1 -> r2
    stdout, stderr = r2.execute(f'ping 192.168.3.2 -c 4') # r2 -> r3
    stdout, stderr = r3.execute(f'ping 192.168.4.2 -c 4') # r3 -> h2
    
except Exception as e:
    print(f"Exception: {e}")
```