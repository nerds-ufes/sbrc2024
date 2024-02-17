# SBRC2024

This repository contains the test artifacts and the scripts used to produce them for the paper "Sliced WANs for Data-Intensive Science: Deployment Experiences and Performance Analysis" to [SBRC 2024](https://sbrc.sbc.org.br/2024/).


## Abstract

The task of transferring massive data sets in Data-Intensive Science (DIS) systems, such as those generated from high-energy experiments at CERN in the EU and at Sirius synchrotron light source in Brazil, often rely on physical WAN infrastructure for network connectivity that is provided by various National Research and Education Networks (NRENs), including ESnet, Géant, Internet2, RNP, among others. Sliced WANs bring a new paradigm for infrastructure yet to be exploited by DIS, but a realistic study of these particular systems poses a significant challenge due to their complexity, scale, and the number of factors affecting the data transport. In this paper, we take the first steps in addressing these challenges by deploying and evaluating a virtual infrastructure for data transport within a representative national-scale WAN. Our approach here encompasses two main aspects: **i)** Evaluating the performance of TCP congestion control algorithms (BBR versus Cubic) when only a single path is available for the data transfer; and **ii)** Assessing the performance of flow completion times (related to the management of bandwidth allocation) for sets of interdependent transfers environment provided by a network slice.

## Contents

- [Data transfer performance over a coast-to-coast single path](#data-transfer-performance-over-a-coast-to-coast-single-path)
- [Data transfer performance over multipath](#data-transfer-performance-over-multipath)

## Slice Deployment and Experimentation with the FABRIC Testbed

### Data transfer performance over a coast-to-coast single path

Import the FABlib API.

```python
from fabrictestbed_extensions.fablib.fablib import FablibManager as fablib_manager

try:
    fablib = fablib_manager()
    
    fablib.show_config()
except Exception as e:
    print(f"Exception: {e}")
```

The slice sets a coast-to-coast linear topology with 5 nodes (`H1`, `R1`, `R2`, `R3`, and `H2`) deployed, respectively, geographically located at the following sites: `SEAT` (Seattle), `SALT` (Salt Lake City), `STAR` (StarLight), `NEWY` (New York) and `MASS` (UMass), as shown in Fig. 1.
Submit a slice request. In addition, the progress of your slice’s build process will be printed.

```python
from ipaddress import ip_address, IPv4Address, IPv6Address, IPv4Network, IPv6Network

slice_name = 'Linear'
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
    
    # Lists of available ips for each subnet
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

Connectivity test of directly connected nodes. Note that it is not necessary to instantiate `H2` host/DTN.
```py
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

>[!IMPORTANT]
>Install [iPerf3](https://iperf.fr/) (v. 3.5.0) and...
>```py
>!ansible-playbook -i ansible/inventory.yml -l [EXPERIMENT]  ansible/setup.yml
>```

This configuration to apply CPU Pinning and Non-Uniform Memory Access (NUMA) is especially interesting in systems with multiple processors or sockets. This enforces the allocation of all virtual CPUs (vCPUs) assigned to the VM on the specific NUMA node containing the relevant components (e.g. network interfaces). In a VM, aligning vCPUs and memory with the physical NUMA nodes helps reduce the time it takes for processors to access the memory associated with their node. Note that a system restart across all nodes is required for the configuration changes to be implemented.
```py
try:
    slice = fablib.get_slice(name = slice_name)
    
    for node in slice.get_nodes():
        # Pin all vCPUs for VM to same Numa node as the component
        node.pin_cpu(component_name='nic1')
        
        # Pin memmory for VM to same Numa node as the components
        node.numa_tune()
    
        # Reboot the VM
        node.os_reboot()
    
except Exception as e:
    print(f"Exception: {e}")
```

Static routing configuration so that H1 can reach H2.
```py
try:
    slice = fablib.get_slice(name=slice_name)
    
    h1 = slice.get_node('h1')
    r1 = slice.get_node('r1')
    r2 = slice.get_node('r2')
    r3 = slice.get_node('r3')
    h2 = slice.get_node('h2')
    
    stdout, stderr = h1.execute(f'sudo ip a add 192.168.1.1/24 dev eth1')
    stdout, stderr = r1.execute(f'sudo ip a add 192.168.1.2/24 dev eth1')
    stdout, stderr = r1.execute(f'sudo ip a add 192.168.2.1/24 dev eth2')
    stdout, stderr = r2.execute(f'sudo ip a add 192.168.2.2/24 dev eth1')
    stdout, stderr = r2.execute(f'sudo ip a add 192.168.3.1/24 dev eth2')
    stdout, stderr = r3.execute(f'sudo ip a add 192.168.3.2/24 dev eth1')
    stdout, stderr = r3.execute(f'sudo ip a add 192.168.4.1/24 dev eth2')
    stdout, stderr = h2.execute(f'sudo ip a add 192.168.4.2/24 dev eth1')
    
    stdout, stderr = h1.execute(f'sudo ip route add 192.168.4.0/24 via 192.168.1.2')
    stdout, stderr = r1.execute(f'sudo ip route add 192.168.4.0/24 via 192.168.2.2')
    stdout, stderr = r2.execute(f'sudo ip route add 192.168.4.0/24 via 192.168.3.2')
    
    stdout, stderr = r2.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.2.1')
    stdout, stderr = r3.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.3.1')
    stdout, stderr = h2.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.4.1')
    
    stdout, stderr = h1.execute(f'ping 192.168.4.2 -c 4')
    
except Exception as e:
    print(f"Exception: {e}")
```

Code used to investigate how to tune the slice network parameters comparing BBR vs. Cubic in terms of throughput and flow-completion time.
```py
try:
    slice = fablib.get_slice(name=slice_name)
    
    h1 = slice.get_node('h1')
    h2 = slice.get_node('h2')
    r1 = slice.get_node('r1')

    size, target = '1G', '192.168.4.2'
    algs = ['cubic','bbr']
    delays = ['0', '60', '120', '180', '240', '320', '380', '440']
    buffers = ['100K', '250K', '500K', '1M','5M', '10M', '25M', '50M']
    rates = ['300']

    r1.execute(f'sudo tc qdisc add dev eth1 root handle 1: htb default 10;\
                 sudo tc class add dev eth1 parent 1: classid 1:10 htb rate {rates[0]}Mbit;\
                 sudo tc qdisc add dev eth1 parent 1:10 netem delay 0ms;\
                 sudo tc qdisc add dev eth2 root handle 1: htb default 10;\
                 sudo tc class add dev eth2 parent 1: classid 1:10 htb rate {rates[0]}Mbit;\
                 sudo tc qdisc add dev eth2 parent 1:10 tbf rate {rates[0]}Mbit buffer {rates[0]} limit {rates[0]}')
    h2.execute(f'iperf3 -s -p 5201 -i 2 -D')
    for alg in algs:
        h1.execute(f'sudo sysctl -w net.ipv4.tcp_congestion_control={alg}', quiet=True)
        h2.execute(f'sudo sysctl -w net.ipv4.tcp_congestion_control={alg}', quiet=True)
        for delay in delays:
            r1.execute(f'sudo tc qdisc change dev eth1 parent 1:10 netem delay {delay}ms')
            for buffer in buffers:
                for rate in rates:
                    r1.execute(f'sudo tc qdisc change dev eth2 parent 1:10 tbf rate {rate}Mbit buffer {buffer} limit {buffer}')
                    h1.execute(f'iperf3 -c {target} -p 5201 -i 2 -n {size} -Z -C {alg} -J | tee > bw-{rate}-sz-{size}-de-{delay}-bf-{buffer}-{alg}.json')
    h2.execute(f'sudo pkill iperf3')
    r1.execute(f'sudo tc qdisc del dev eth1 root')
    r1.execute(f'sudo tc qdisc del dev eth2 root')
except Exception as e:
    print(f"Exception: {e}")
```

Copy files from remote node to local enviroment (GitLab). This approach uses the Linux `ls` command to select files to copy. Replace `[FILE_NAME]` with the name of the file (or a chunk of the files name using `*`) you want to copy. 
```py
try:
    slice = fablib.get_slice(name=slice_name)
    h1 = slice.get_node('h1')

    stdout,stderr  = h1.execute(f'ls | grep "[FILE_NAME]-*"', quiet=True)
    
    files = [i for i in stdout.split()]
    
    for file in files:
        h1.download_file(local_file_path = f'{file}', remote_file_path = f'{file}')
        
except Exception as e:
    print(f"Exception: {e}")
```

Another way to copy the measurements (outputs) obtained by iPerf3 is to use the same parameters of the repetition loops used in the experiment automation. Replace the `size`, `algs`, `delays`, `buffers` and `rates` parameters with the values needed to select the desired files.
```py
try:
    slice = fablib.get_slice(name=slice_name)
    h1 = slice.get_node('h1')
    
    size = '1G'
    algs = ['cubic', 'bbr']
    delays = ['0', '60', '120', '180', '240', '320', '380', '440']
    buffers = ['100K', '250K', '500K', '1M', '5M', '10M', '25M', '50M']
    rates = ['300']
    
    for algorithm in algorithms:
        for delay in delays:
            for buffer in buffers:
                for rate in rates:
                    h1.download_file(local_file_path = f'bw-{rate}-sz-{size}-de-{delay}-bf-{buffer}-{algorithm}.json', remote_file_path = f'bw-{rate}-sz-{size}-de-{delay}-bf-{buffer}-{algorithm}.json')
except Exception as e:
    print(f"Exception: {e}")
```

Thread experiment.
```py
try:
    slice = fablib.get_slice(name=slice_name)
    
    h1 = slice.get_node('h1')
    h2 = slice.get_node('h2')
    
    target = '192.168.4.2'
    algorithms = ['cubic','bbr']
    sizes = ['1G', '5G', '10G']
    threads = ['1', '2', '3', '5', '10','15', '20', '30']
    
    h2.execute(f'iperf3 -s -p 5201 -i 2 -D')
    
    for i in range(1,11):
        for algorithm in algorithms:
            h1.execute(f'sudo sysctl -w net.ipv4.tcp_congestion_control={algorithm}', quiet=True)
            h2.execute(f'sudo sysctl -w net.ipv4.tcp_congestion_control={algorithm}', quiet=True)
            for size in sizes:
                for thread in threads:
                    print(f'thread={thread}, size={size}, alg={algorithm}')
                    h1.execute(f'iperf3 -c {target} -p 5201 -i 2 -n {size} -Z -C {algorithm} -J -P {thread} | tee > th-{thread}-sz-{size}-{algorithm}-{i}.json')
    h2.execute(f'sudo pkill iperf3')
    
except Exception as e:
    print(f"Exception: {e}")
```

### Data transfer performance over multipath

As each experiment is normally run in different `.ipynb` files, it is necessary to import the FABlib API again.
```py
from fabrictestbed_extensions.fablib.fablib import FablibManager as fablib_manager

try:
    fablib = fablib_manager()
    
    fablib.show_config()
except Exception as e:
    print(f"Exception: {e}")
```

The slice is a multipath topology composed by 7 nodes (H1, R1, R2, R3, R4, R5, and H2), respectively, geographically located in the following sites: `USCD` (USCD), `LOSA` (Los Angeles), `SEAT` (Seattle), `SALT` (Salt Lake City), `STAR` (StarLight), `DALL` (Dallas), and `MICH` (University of Michigan), as shown in Fig. 2 (Left) and (Right). The Red, Green and Blue paths have respectively the average RTT of: `82.2 ms`, `78.6 ms` and `62.5 ms`.

```py
from ipaddress import ip_address, IPv4Address, IPv6Address, IPv4Network, IPv6Network

slice_name = 'Multipath'
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
    subnet5 = IPv4Network("192.168.5.0/24")
    subnet6 = IPv4Network("192.168.6.0/24")
    subnet7 = IPv4Network("192.168.7.0/24")
    subnet8 = IPv4Network("192.168.8.0/24")
    
    # Lists of available ips for each subnet
    net1_available_ips = list(subnet1)[1:]
    net2_available_ips = list(subnet2)[1:]
    net3_available_ips = list(subnet3)[1:]
    net4_available_ips = list(subnet4)[1:]
    net5_available_ips = list(subnet5)[1:]
    net6_available_ips = list(subnet6)[1:]
    net7_available_ips = list(subnet7)[1:]
    net8_available_ips = list(subnet8)[1:]
    
    # Creates l2 overlay networks
    net1 = slice.add_l2network(name='net1', subnet=subnet1)
    net2 = slice.add_l2network(name='net2', subnet=subnet2)
    net3 = slice.add_l2network(name='net3', subnet=subnet3)
    net4 = slice.add_l2network(name='net4', subnet=subnet4)
    net5 = slice.add_l2network(name='net5', subnet=subnet5)
    net6 = slice.add_l2network(name='net6', subnet=subnet6)
    net7 = slice.add_l2network(name='net7', subnet=subnet7)
    net8 = slice.add_l2network(name='net8', subnet=subnet8)

    # Adds H1, R1, R2, R3, R4, R5 and H2 nodes to the slice
    h1 = slice.add_node(name='h1', site='UCSD', image=os_name)
    [h1n1, h1n2] = h1.add_component(model=model_name, name='nic1').get_interfaces()
    h1n1.set_mode('config')
    net1.add_interface(h1n1)
    h1n1.set_ip_addr(net1_available_ips.pop(0))
    
    r1 = slice.add_node(name='r1', site='LOSA', image=os_name)
    [r1n1, r1n2] = r1.add_component(model=model_name, name='nic1').get_interfaces()
    [r1n3, r1n4] = r1.add_component(model=model_name, name='nic2').get_interfaces()
    r1n1.set_mode('config')
    net1.add_interface(r1n1)
    r1n1.set_ip_addr(net1_available_ips.pop(0))
    r1n2.set_mode('config')
    net2.add_interface(r1n2)
    r1n2.set_ip_addr(net2_available_ips.pop(0))
    r1n3.set_mode('config')
    net6.add_interface(r1n3)
    r1n3.set_ip_addr(net6_available_ips.pop(0))
    r1n4.set_mode('config')
    net7.add_interface(r1n4)
    r1n4.set_ip_addr(net7_available_ips.pop(0))
    
    r2 = slice.add_node(name='r2', site='SEAT', image=os_name)
    [r2n1, r2n2] = r2.add_component(model=model_name, name='nic1').get_interfaces()
    r2n1.set_mode('config')
    net2.add_interface(r2n1)
    r2n1.set_ip_addr(net2_available_ips.pop(0))
    r2n2.set_mode('config')
    net3.add_interface(r2n2)
    r2n2.set_ip_addr(net3_available_ips.pop(0))
    
    r3 = slice.add_node(name='r3', site='SALT', image=os_name)
    [r3n1, r3n2] = r3.add_component(model=model_name, name='nic1').get_interfaces()
    [r3n3, r3n4] = r3.add_component(model=model_name, name='nic2').get_interfaces()
    r3n1.set_mode('config')
    net3.add_interface(r3n1)
    r3n1.set_ip_addr(net3_available_ips.pop(0))
    r3n2.set_mode('config')
    net4.add_interface(r3n2)
    r3n2.set_ip_addr(net4_available_ips.pop(0))
    r3n3.set_mode('config')
    net7.add_interface(r3n3)
    r3n3.set_ip_addr(net7_available_ips.pop(0))
    
    r4 = slice.add_node(name='r4', site='STAR', image=os_name)
    [r4n1, r4n2] = r4.add_component(model=model_name, name='nic1').get_interfaces()
    [r4n3, r4n4] = r4.add_component(model=model_name, name='nic2').get_interfaces()
    r4n1.set_mode('config')
    net4.add_interface(r4n1)
    r4n1.set_ip_addr(net4_available_ips.pop(0))
    r4n2.set_mode('config')
    net5.add_interface(r4n2)
    r4n2.set_ip_addr(net5_available_ips.pop(0))
    r4n3.set_mode('config')
    net8.add_interface(r4n3)
    r4n3.set_ip_addr(net8_available_ips.pop(0))
    
    r5 = slice.add_node(name='r5', site='DALL', image=os_name)
    [r5n1, r5n2] = r5.add_component(model=model_name, name='nic1').get_interfaces()
    r5n1.set_mode('config')
    net5.add_interface(r5n1)
    r5n1.set_ip_addr(net5_available_ips.pop(0))
    r5n2.set_mode('config')
    net6.add_interface(r5n2)
    r5n2.set_ip_addr(net6_available_ips.pop(0))
    
    h2 = slice.add_node(name='h2', site='MICH', image=os_name)
    [h2n1, h2n2] = h2.add_component(model=model_name, name='nic1').get_interfaces()
    h2n1.set_mode('config')
    net8.add_interface(h2n1)
    h2n1.set_ip_addr(net8_available_ips.pop(0))
    
    # Submit Slice Request
    slice.submit()
    
except Exception as e:
    print(f"Exception: {e}")
```

Connectivity test of directly connected nodes.
```py
try:
    slice = fablib.get_slice(name=slice_name)

    h1 = slice.get_node('h1')
    r1 = slice.get_node('r1')
    r2 = slice.get_node('r2')
    r3 = slice.get_node('r3')
    r4 = slice.get_node('r4')
    r5 = slice.get_node('r5')
    h2 = slice.get_node('h2')
    
    stdout, stderr = h1.execute(f'ping 192.168.1.2 -c 4') # h1 -> r1
    stdout, stderr = r2.execute(f'ping 192.168.2.1 -c 4') # r1 -> r2
    stdout, stderr = r2.execute(f'ping 192.168.3.2 -c 4') # r2 -> r3
    stdout, stderr = r3.execute(f'ping 192.168.4.2 -c 4') # r3 -> r4
    stdout, stderr = r4.execute(f'ping 192.168.5.2 -c 4') # r4 -> r5
    stdout, stderr = r5.execute(f'ping 192.168.6.1 -c 4') # r5 -> r1
    stdout, stderr = r1.execute(f'ping 192.168.7.2 -c 4') # r1 -> r3
    stdout, stderr = h2.execute(f'ping 192.168.8.1 -c 4') # h2 -> r4
except Exception as e:
    print(f"Exception: {e}")
```

Adding static routes.
```py
try:
    slice = fablib.get_slice(name = slice_name)
    
    h1 = slice.get_node('h1')
    r1 = slice.get_node('r1')
    r2 = slice.get_node('r2')
    r3 = slice.get_node('r3')
    r4 = slice.get_node('r4')
    r5 = slice.get_node('r5')
    h2 = slice.get_node('h2')
    
    # Red Path Config
    stdout, stderr = h1.execute(f'sudo ip route add 192.168.8.0/24 via 192.168.1.2')
    stdout, stderr = r1.execute(f'sudo ip route add 192.168.8.0/24 via 192.168.2.2')
    stdout, stderr = r2.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.2.1')
    stdout, stderr = r2.execute(f'sudo ip route add 192.168.8.0/24 via 192.168.3.2')
    stdout, stderr = r3.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.3.1')
    stdout, stderr = r3.execute(f'sudo ip route add 192.168.8.0/24 via 192.168.4.2')
    stdout, stderr = r4.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.4.1')
    stdout, stderr = h2.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.8.1')
    stdout, stderr = h1.execute(f'ping 192.168.8.2 -c 4')
    
    # Green Path Config
    stdout, stderr = r1.execute(f'sudo ip route add 192.168.8.0/24 via 192.168.7.2 tos 0x04')
    stdout, stderr = r3.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.7.1 tos 0x04')
    stdout, stderr = r3.execute(f'sudo ip route add 192.168.8.0/24 via 192.168.4.2 tos 0x04')
    stdout, stderr = r4.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.4.1 tos 0x04')
    stdout, stderr = h1.execute(f'ping 192.168.8.2 -c 4 -Q 0x04')
    
    # Blue Path Config
    stdout, stderr = r1.execute(f'sudo ip route add 192.168.8.0/24 via 192.168.6.2 tos 0x08')
    stdout, stderr = r5.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.6.1')
    stdout, stderr = r5.execute(f'sudo ip route add 192.168.8.0/24 via 192.168.5.1')
    stdout, stderr = r4.execute(f'sudo ip route add 192.168.1.0/24 via 192.168.5.2 tos 0x08')
    stdout, stderr = h1.execute(f'ping 192.168.8.2 -c 4 -Q 0x08')
    
except Exception as e:
    print(f"Exception: {e}")
```

>[!IMPORTANT]
>Install [iproute-tc](https://www.mankier.com/package/iproute-tc) (v. ), [iPerf3](https://iperf.fr/) (v. 3.5.0), nano (v.), tcpdump and traceroute (v.) on every node of the topology.
>```py
>!ansible-playbook -i ansible/inventory.yml -l [EXPERIMENT]  ansible/setup.yml
>```

We set the capacity of crossing links (l1 = R3 ⇐⇒ R4 and l2 = R1 ⇐⇒ R5) seeking for the point that achieves maximal performance for transferring the flows. We refer to an assignment of capacity values to the links as a slice configuration S1. For example, in the configuration S1, we set the links l1 = R3 ⇐⇒ R4 = 50M bps and l2 = R1 ⇐⇒ R5 = 100 from the topology of Fig. 8 (Right), so that the ratio between the crossing links are r = l1/l2 = 0.5.

```py
import time

try:
    slice = fablib.get_slice(name = slice_name)
    
    h1 = slice.get_node('h1')
    h2 = slice.get_node('h2')
    r1 = slice.get_node('r1')
    r3 = slice.get_node('r3')
    r5 = slice.get_node('r5')
    
    size, target = '1G', '192.168.8.2'
    ports = ['5201', '5202', '5203', '5204', '5205', '5206']
    rates = ['200mbit', '100mbit']
    threads = []

    tcr1 = r1.execute_thread(f'sudo tc qdisc add dev eth1 root handle 1: htb default 10;\
                               sudo tc class add dev eth1 parent 1: classid 1:10 htb rate {size}')
    
    threads.insert(0, tcr1)

    for i, port in enumerate(ports):
        threads.insert(i+1, h2.execute_thread(f'iperf3 -s -p {port} -i 2 -D'))
    
    flow1 = h1.execute_thread(f'iperf3 -c {target} -p {ports[0]} -i 2 -n {size} -J -Z | tee > flow1.json')
    flow2 = h1.execute_thread(f'iperf3 -c {target} -p {ports[1]} -i 2 -n {size} -J -Z -S 0x04 | tee > flow2.json')
    flow3 = h1.execute_thread(f'iperf3 -c {target} -p {ports[2]} -i 2 -n {size} -J -Z -S 0x08 | tee > flow3.json')

    threads.extend([flow1, flow2, flow3])
    
    time.sleep(10)
    
    tcr3 = r3.execute_thread(f'sudo tc qdisc add dev eth2 root handle 1: htb default 10;\
                               sudo tc class add dev eth2 parent 1: classid 1:10 htb rate {rates[0]}')
    
    tcr5 = r5.execute_thread(f'sudo tc qdisc add dev eth1 root handle 1: htb default 10;\
                               sudo tc class add dev eth1 parent 1: classid 1:10 htb rate {rates[0]}')

    threads.extend([tcr3, tcr5])
    
    flow4 = h1.execute_thread(f'iperf3 -c {target} -p {ports[3]} -i 2 -n {size} -J -Z | tee > flow4.json')
    flow5 = h1.execute_thread(f'iperf3 -c {target} -p {ports[4]} -i 2 -n {size} -J -Z -S 0x04 | tee > flow5.json')
    flow6 = h1.execute_thread(f'iperf3 -c {target} -p {ports[5]} -i 2 -n {size} -J -Z -S 0x08 | tee > flow6.json')

    threads.extend([flow4, flow5, flow6])
    
    print(f"Joining Threads")
    for thread in threads:
        stdout, stderr = thread.result()
        print(f"Error: ", stdout, stderr)
    
    r1.execute(f'sudo tc qdisc del dev eth1 root')
    r3.execute(f'sudo tc qdisc del dev eth2 root')
    r5.execute(f'sudo tc qdisc del dev eth1 root')
    
except Exception as e:
    print(f"Exception: {e}")
```