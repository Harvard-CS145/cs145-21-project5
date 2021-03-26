# Project 5: Flowlet-based Load Balancing

## Objectives

- Understand the concept of flowlets and detect flowlets in real-traffic
- Understand the benefits of flowlet based load balancing

## Getting Started

Project 5 will inherit your `p4src/l3fwd.p4`, `topology/p4app_fat.json` (Fattree with k=4), and `controller/controller_fat_l3.py` files from project 3.

## Introduction

We have implemented ECMP in Project 3. But one drawback of ECMP is that it doesn't work well for large flows. It is possible for ECMP to hash two large flows onto the same path resulting in congestion. The purpose of this project is to divide large flows into smaller *flowlets* and run load balancing based on those flowlets (instead of flows). Flowlet switching leverages the burstness of TCP flows to achieve a better load balancing. TCP flows tend to come in bursts (because TCP is window based). Every time there is gap which is big enough (e.g., 200us) between packets from the same flow, flowlet switching will rehash the flow to another path (by hashing an flowlet ID value together with the 5-tuple). For more information about flowlet switching check out this [paper](https://www.usenix.org/system/files/conference/nsdi17/nsdi17-vanini.pdf).

## Part One: Flowlet Switching

In this part, you are expected to implement flowlet switching using ECMP routing algorithm. 
You need to create an initial `p4src/flowlet_switching.p4` based on `p4src/l3fwd.p4`, and modify it to implement flowlet switching. 
You also need to create an initial `controller/controller_flowlet.py` based on `controller/controller_fat_l3.py`. 
The basic logic of flowlet switching is as follows: 

1. Implement two hash tables to store flowlet states (use large hash table size, e.g, 8192, so that you do not need to handle hash collision). In this part, you only need to maintain the timestamp of the last packet belonging to the flowlet, as well as the flowlet ID. You can use `standard_metadata.ingress_global_timestamp` to get the current timestamp (in micro-second) in the P4 switch. 
2. Use a fixed number to define the flowlet gap (we suggest **200us**). Whenever the difference between the current timestamp and the last timestamp is larger than the gap, then you should treat the packet as the starting packet of a new flowlet.
3. For each new flowlet, assign it with a random flowlet ID. A large flow can have many flowlets sometimes even over a thousand. Register with of 16 bits should be fine for storing a flowlet ID. Anything larger also should not be an issue. You can use `random(val, (bit<32>)0, (bit<32>)65000)` to get a random number from 0 to 65000, and the value is written to `val`.
4. Use hash function to compute hash value for a combination of five tuples and the flowlet ID. Then use the hash value as the new ecmp group ID (due to modulo, this new ID might be the same as the old one; but overall, flowlets are distributed among all path evenly). This new ecmp group ID will determine which port you will forward this packet to. 
5. Consider whether or not to modify the controller code.

## Hints
* You can use registers in P4 to implement hash table; below is an example: 
    ```
    struct metadata {
        bit<48> interval;
        bit<16> flow_index;
    }

    control MyIngress(inout headers hdr, inout metadata meta,
                      inout standard_metadata_t standard_metadata) {
        
        /* timestamp is 48bit */
        register<bit<48>>(8192) last_seen;

        action get_inter_packet_gap(){
            bit<48> last_pkt_ts;
            
            /* Get the time the previous packet was seen */
            last_seen.read(last_pkt_ts, (bit<32>)meta.flow_index);
            
            /* Calculate the time interval */
            meta.interval = standard_metadata.ingress_global_timestamp – last_pkt_ts;
            
            /* Update the register with the new timestamp */
            last_seen.write((bit<32>)meta.flow_index, standard_metadata.ingress_global_timestamp);
        }

        apply {
            if (hdr.ipv4.isValid()){
                /* call hash() to calculate meta.flow_index */

                get_inter_packet_gap();

                /* operations based on meta.interval */
            }
        }
    ```

### Testing
Your code should work for the following testing. These are also good steps for debugging.
1. Testing connectivity.
Run `pingall`.
2. Testing ECMP
Run `sudo python3 tests/validate_ecmp.py`. 
3. Testing Flowlet ECMP
Run `sudo python3 tests/validate_flowlet.py`. The script will send one flow intermittently, and check whether the dataplane distributes each flowlet into different paths.

**Note**: If your topology is making port 1 and 2 of agg switches sending packets "up" (ie, for agg switches, port 1 and 2 connect to core switches and port 3 and 4 connect to tor switches), you may fail the `validate_flowlet` test. All you need to do here is to change the `file tests/iperf_flowlet.sh'. In this file, you can see 4 consecutive commands using tcpdump, and please change "eth3" to "eth1" and change "eth4" to "eth2" for the 4 commands.


## Part Two: Compare Performance

In this part, you are expected to compare the performance of the ECMP algorithm before and after using flowlet switching. You will use a specific trace named `apps/trace/flowlet.trace` (generated using `python apps/trace/generate_trace.py --mchost=1-16 --iperfhost=1-16 --length=60 --file=apps/trace/flowlet.trace`), which consists of persistent flows, while each flow is comprised of many small flowlets. Please proceed with the following steps:

1. Run ECMP algorithm without flowlet switching. Run your codes in project 3 (ie, `sudo p4run --conf topology/p4app_fat.json` to run your `p4src/l3fwd.p4`, then start your controller by `python controller/controller_fat_l3.py`), and use `apps/send_traffic.py` script to send the traffic. 
    ```
    sudo python ./apps/send_traffic.py ./apps/trace/flowlet.trace 1-16 60 ecmp_logs
    ```
    You can check out the iperf throughput values from `ecmp_logs` directory: each float value in `ecmp/\*_iperf.log' is the throughput (unit is bps) for a specific flow. 
2. Run ECMP algorithm with flowlet switching in the current project (ie, `sudo p4run --conf topology/p4app_fat_flowlet.json` to run your `p4src/flowlet_switching.p4`, then start your controller by `python controller/controller_flowlet.py`). Use `apps/send_traffic.py` to send the traffic. 
    ```
    sudo python ./apps/send_traffic.py ./apps/trace/flowlet.trace 1-16 60 flowlet_logs
    ```
    You can check out the iperf throughput values of this run from `flowlet_logs` directory.
3. Draw CDF figures of iperf throughputs for both step 1 and step 2.
4. Use pcap files to gather the following information:
    - Find out the largest flow in the traffic.
    - There are four paths for each flowlet to choose in our topology. Within the largest flow, each flowlet could choose a different path. What's the percentage of flowlets of the largest flow on each of the four paths?

### Parsing Pcap Files

For each packet arriving in or leaving an interface, the packet will be recorded in the pcap files. You can check pcap files in the `pcap` directory after running the traffic. The name of each pcap file is in this format: `{sw_name}-{intf_name}_{in/out}.pcap`. For example, if the pcap file is `a1-eth1_in.pcap`, the file records all packets **leaving** the `eth1` interface of switch `a1`. If the pcap file is `t2-eth3_out.pcap`, the file records all packets **arriving in** the `eth3` interface of switch `t2`.

Pcap files are in binary format, so you cannot directly read the packets within those files. You need to use `tcpdump` to parse those files.

```
tcpdump -enn -r [pcap file] > res.txt
```

Then you can get a human-readable file `res.txt` containing the information of each packet. Within this file, each line represents one packet. For example 

```
13:29:40.413988 00:00:00:09:11:00 > 00:00:00:00:09:00, ethertype IPv4 (0x0800), length 9514: 10.0.0.5.41456 > 10.0.0.1.5001: Flags [.], seq 71136:80584, ack 1, win 74, options [nop,nop,TS val 4116827540 ecr 1502193499], length 9448
```

Each field represents timestamp, src MAC address, dst MAC address, ethernet type, packet size, src IP address/TCP port, dst IP address/TCP port, TCP flags, sequence number, ACK number, etc.

For more information about pcap, please refer to [Pcap for Tcpdump page](https://www.tcpdump.org/pcap.html).

**Note**: 
- To get reliable performance numbers for this experiment (and all future experiments that need to measure throughput and latency), you'd better check your VM CPU usage and ensure it's low. You can get reduce CPU usage by removing unnecessarily running applications in your VM.

## Extra Credit 

One critical parameter in flowlet switching is the flowlet timeout, which impacts the performance of flowlet switching a lot. Can you explore the impact of different timeout values based on this flowlet [paper](https://www.usenix.org/system/files/conference/nsdi17/nsdi17-vanini.pdf) and testing by yourself. 
You can draw a figure with differnet flowlet timeout values as x-axis, and corresponding iperf average throughput as y-axis. Write down your findings and embed the figure in your `report.md`. 

## Submission and Grading

### What to Submit

You are expected to submit the following documents:

1. Code: The main P4 code should be in `p4src/flowlet_switching.p4`, while you can also use other file to define headers or parsers, in order to reduce the length of each P4 file; The controller code should be in `controller_flowlet.py` which fills in table entries when launching those P4 switches.

2. report/report.md: You should describe how you implement flowlet switching and provide a detailed report on performance analysis as described above in `report.md`. You might include your findings and figure if you choose to explore different flowlet timeout value.

### Grading

The total grades is 100:

- 30: For your description of how you program in `report.md`.
- 70: We will check the correctness of your solutions for flowlet switching using the specific trace.
- **10**: Extra credit for exploring different flowlet timeout value. 
- Deductions based on late policies


### Survey

Please fill up the survey when you finish your project.

[Survey link](https://docs.google.com/forms/d/e/1FAIpQLScfl58RVYXSFxOI2twPta5rdvtxHL0Yn7PsJnzowjR5AS_Fwg/viewform?usp=sf_link)
