# Network Delay Measurement Tool - SDN/Mininet Project

## 🎯 Project Objective

Implement an SDN-based solution using **Mininet** and **POX OpenFlow Controller** to measure and analyze network latency (RTT) in a multi-hop topology. The project demonstrates:
- Controller-switch interaction via OpenFlow protocol
- Dynamic flow rule installation (match-action)
- Network behavior observation under varying delay conditions
- RTT analysis and correlation with link delays

---

## 📋 Problem Statement

Measure how network latency changes in a multi-hop linear topology when artificial delays are introduced on links. Compare observed RTT values with theoretical calculations to validate the Mininet simulation accuracy.

**Key Question:** How does RTT scale with per-link delay in a linear topology?

---

## 🏗️ Topology Design

### Linear Topology (3 switches)
```
Host1 ─── Switch1 ─── Switch2 ─── Switch3 ─── Host3
(h1)       (s1)         (s2)         (s3)       (h3)
  ↑                                              ↑
  └──────── 4 Links (Each with configurable delay) ────┘
```

**Why Linear Topology?**
- Simple path for RTT measurement
- Predictable hop count (4 links total)
- Easy to calculate theoretical RTT
- Ideal for delay analysis

---

## 🛠️ Tools & Technologies

| Component | Version | Purpose |
|-----------|---------|---------|
| **Mininet** | Latest | Network emulation platform |
| **POX Controller** | 0.7.0 | OpenFlow SDN controller |
| **Python** | 3.6+ | Controller logic & scripting |
| **Linux (WSL/Ubuntu)** | 20.04 LTS | Operating system |
| **ping** | - | RTT measurement utility |
| **Wireshark** | - | Network packet analysis (optional) |

---

## 📦 Project Structure

```
network-delay-measurement/
├── controller.py              # POX OpenFlow controller
├── run_experiment.sh         # Bash script to execute tests
├── README.md                 # This file
├── RESULTS.md               # Experimental results & analysis
└── delay1.png             # Proof of execution
└── delay2.png
└── rtt_vs_delay.png

```

---

## 🚀 Setup & Execution

### Prerequisites
```bash
# Install Mininet (Ubuntu/WSL)
sudo apt-get install mininet

# Install POX controller
cd ~
git clone https://github.com/noxrepo/pox.git
cd pox

# Install required Python dependencies
sudo apt-get install python3-pip
pip install pox
```

### Step 1: Start POX Controller
```bash
cd ~/pox
./pox.py forwarding.l2_learning
# OR run custom controller:
./pox.py py.controller
```

### Step 2: Launch Mininet Topology (In another terminal)

**Experiment 1: 5ms Link Delay**
```bash
sudo mn --controller=remote,port=6633 --topo linear,3 --link tc,delay=5ms
```

**Experiment 2: 20ms Link Delay**
```bash
sudo mn --controller=remote,port=6633 --topo linear,3 --link tc,delay=20ms
```

### Step 3: Run RTT Measurement (Inside Mininet CLI)

```mininet
mininet> h1 ping -c 5 h3
```

### Step 4: Observe Results
```mininet
mininet> h1 ping -c 5 h3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=41.5 ms
...
--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss
rtt min/avg/max/mdev = 40.7/41.5/42.6/0.650 ms
```

---

## 📊 Experimental Results

### Test Case 1: 5 ms Delay per Link

| Parameter | Value |
|-----------|-------|
| Delay per Link | 5 ms |
| Total Links (h1→h3) | 4 |
| One-way Delay | 4 × 5 = 20 ms |
| **Theoretical RTT** | **2 × 20 = 40 ms** |
| **Observed RTT** | **≈ 41.5 ms** |
| **Error** | **3.75%** ✅ |

### Test Case 2: 20 ms Delay per Link

| Parameter | Value |
|-----------|-------|
| Delay per Link | 20 ms |
| Total Links (h1→h3) | 4 |
| One-way Delay | 4 × 20 = 80 ms |
| **Theoretical RTT** | **2 × 80 = 160 ms** |
| **Observed RTT** | **≈ 161 ms** |
| **Error** | **0.625%** ✅ |

### RTT Formula
```
RTT = 2 × (Number of Links × Delay per Link)
RTT = 2 × (4 × d) = 8d  [for our linear topology]
```

### Key Observations
- ✅ **Linear Relationship:** RTT increases proportionally with link delay
- ✅ **Accuracy:** Observed values match theoretical calculations (error < 4%)
- ✅ **Consistency:** Multiple ping runs show stable RTT values
- ✅ **Multi-hop Impact:** 4 hops multiply the delay effect significantly
- ⚠️ **First Packet Overhead:** Initial ping slightly higher due to ARP resolution

---

## 🔌 SDN Controller Logic

### OpenFlow Packet_In Handling

```python
Event: Packet_In (Switch → Controller)
  ├─ Parse Ethernet frame
  ├─ Learn: MAC → Port mapping
  ├─ Install flow rule if destination known
  ├─ Flood if destination unknown
  └─ Send packet out

Flow Rule Installation
  ├─ Match: Source MAC + Destination MAC
  ├─ Action: Forward to learned port
  ├─ Priority: 100
  └─ Timeouts: 10s (idle), 30s (hard)
```

### Controller Behavior

1. **Learning Phase:** First packet triggers packet_in → controller learns path
2. **Forwarding Phase:** Subsequent packets follow installed flow rules
3. **Timeout:** Rules expire after 10s inactivity (automatic cleanup)

---

## 📈 Performance Analysis

### Latency Breakdown

For the 20ms delay case:
```
Total RTT = 161 ms
  ├─ Propagation delay (4 links × 20ms each × 2 directions) = 160 ms
  └─ Processing overhead (controller + switch) ≈ 1 ms
```

### Throughput Observations

Tested with `iperf3`:
```bash
mininet> iperf h1 h3
------------------------------------------------------------
Client connecting to 10.0.0.3, TCP port 5201
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-10.00  sec  25.6 MBytes  21.5 Mbits/sec
------------------------------------------------------------
```

The 20ms delay reduces bandwidth due to TCP retransmission timeout and congestion window effects.

---

## ✅ Validation & Testing

### Test Scenarios

#### Scenario 1: Normal Operation
```bash
mininet> h1 ping -c 5 h3
Results: 0% packet loss, stable RTT
Status: ✅ PASS
```

#### Scenario 2: Multiple Hosts
```bash
mininet> pingall
h1 -> h2 h3 
h2 -> h1 h3 
h3 -> h1 h2 
Ping: 100% complete
Status: ✅ PASS
```

#### Scenario 3: Flow Table Verification
```bash
mininet> s1 dpctl dump-flows
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=12.345s, table=0, n_packets=5
 idle_timeout=10, hard_timeout=30
 priority=100, dl_src=00:00:00:00:00:01, dl_dst=00:00:00:00:00:03
 actions=output:2
Status: ✅ PASS
```

---

## 🔧 Code Explanation

### Key Functions in `controller.py`

#### `launch()`
- POX entry point
- Registers event handlers for OpenFlow events
- Initializes controller instance

#### `packet_in_handler(event)`
- Called when switch sends packet_in message
- Learns MAC addresses from packet source
- Installs flow rules for known destinations
- Floods packet if destination unknown

#### `install_flow_rule(connection, src_mac, dst_mac, out_port)`
- Creates OpenFlow flow modification message (ofp_flow_mod)
- Sets match criteria (source + destination MAC)
- Specifies action (forward to port)
- Configures timeouts for rule cleanup

### Design Choices

| Choice | Reason |
|--------|--------|
| Learning Switch | Simple, demonstrates MAC learning |
| Linear Topology | Predictable delay, easy validation |
| 5ms & 20ms Delays | Cover low-latency and high-latency cases |
| 10s Idle Timeout | Balance between state freshness and controller load |
| POX Controller | Lightweight, easy to understand SDN concepts |

---

## 📊 Expected Output

### Console Output (POX Controller)
```
INFO:core:POX 0.7.0 (gar) is up.
INFO:openflow.of_01:[00-00-00-00-00-01 1] connected
INFO:openflow.of_01:[00-00-00-00-00-02 2] connected
INFO:openflow.of_01:[00-00-00-00-00-03 3] connected
INFO:controller:Packet_in: Switch 1, Port 1, Src: 00:00:00:00:00:01, Dst: 00:00:00:00:00:03
INFO:controller:Flow rule installed: 00:00:00:00:00:01 -> 00:00:00:00:00:03 via port 2
```

### Ping Output (Mininet)
```
mininet> h1 ping -c 5 h3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=41.6 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=41.5 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=41.4 ms
64 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=41.5 ms
64 bytes from 10.0.0.3: icmp_seq=5 ttl=64 time=41.3 ms

--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0%packet loss, time 407ms
rtt min/avg/max/mdev = 41.3/41.4/41.6/0.127 ms
```

---

## 🎓 SDN Concepts Demonstrated

### 1. Controller-Switch Communication (OpenFlow)
- Switches connect to controller on TCP port 6633
- OpenFlow 1.0 messages (packet_in, flow_mod, packet_out)
- Asynchronous event-driven architecture

### 2. Flow Rule Design
- Match fields: dl_src, dl_dst (Layer 2)
- Actions: output to port
- Priorities and timeouts for rule management

### 3. Network Programmability
- Dynamic flow installation based on packet arrivals
- Flexible forwarding logic implemented in Python
- Separation of control plane (controller) and data plane (switches)

### 4. Performance Metrics
- RTT (Round Trip Time) for latency measurement
- Throughput via iperf for bandwidth analysis
- Flow table statistics for forwarding verification

---

## 🐛 Troubleshooting

| Issue | Solution |
|-------|----------|
| **Controller connection fails** | Ensure POX is running on correct port (6633) |
| **High RTT unexpectedly** | Check for network congestion; verify delays are applied via `tc` |
| **Packets flooding forever** | Reduce timeout values; check topology connectivity |
| **Flow rules not installed** | Verify controller code is executing; check POX logs |
| **Ping fails** | Confirm hosts can reach controller; test with `mininet> pingall` |

---

## 📚 References

- [POX OpenFlow Controller Documentation](https://github.com/noxrepo/pox)
- [Mininet Tutorial](http://mininet.org/walkthrough/)
- [OpenFlow 1.0 Specification](https://opennetworking.org/)
- [Linux Traffic Control (tc) Guide](https://man7.org/linux/man-pages/man8/tc.8.html)
- Kurose & Ross, *Computer Networking* (7th Edition) - SDN Chapter

---

## 📝 Author & Submission

- **Student:** Mirunjai Suresh Kumar
- **USN:** PES1UG24AM161
- **Course:** Computer Networks (PES University)
- **Date:** April 2025
- **Repository:** [GitHub Link](https://github.com/Mirunjai/network-delay-measurement/)

---

## 📄 License

This project is for educational purposes as part of PES University coursework. Free to use and modify for learning purposes.

---

**Last Updated:** April 2025  
**Status:** ✅ Complete & Ready for Submission
