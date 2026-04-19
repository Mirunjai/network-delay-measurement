# Network Delay Measurement Tool - SDN/Mininet Project

## 🎯 Project Objective

This project implements an SDN system using Mininet and POX to measure how network latency (RTT) behaves when delays are introduced across multiple hops. We:
- Set up a multi-hop linear topology with 3 switches
- Introduce configurable delays on links
- Measure RTT and compare with theoretical values
- Validate that Mininet accurately simulates real network behavior

---

## 📋 Problem Statement

Real networks experience latency due to physical distance, intermediate devices, and link characteristics. This project answers: **How does RTT scale when delay is introduced across network hops?**

We test with two scenarios (5ms and 20ms per-link delay) and validate our measurements against theory.

---

## 🏗️ Topology Design

### Linear Topology (3 switches)
```
h1 ─── s1 ─── s2 ─── s3 ─── h3
```

**What we have:**
- 2 hosts (h1, h3)
- 3 switches (s1, s2, s3)
- 4 total links (h1→s1, s1→s2, s2→s3, s3→h3)
- Each link has a configurable delay

**Why this topology?** It's simple, predictable, and lets us easily calculate expected RTT. The fixed 4-hop path shows how delay accumulates across multiple hops.

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
└── screenshots/             # Proof of execution
    ├── topology_setup.png
    ├── experiment_5ms.png
    ├── experiment_20ms.png
    └── rtt_graph.png
```

---

## 🚀 Setup & Execution

**Prerequisites:**
```bash
sudo apt-get install mininet
cd ~
git clone https://github.com/noxrepo/pox.git
```

**Step 1: Start POX Controller (Terminal 1)**
```bash
cd ~/pox
./pox.py py.controller
# Wait for "INFO:core:POX ... is up."
```

**Step 2: Launch Mininet (Terminal 2)**

For 5ms delay:
```bash
sudo mn --controller=remote,port=6633 --topo linear,3 --link tc,delay=5ms
```

For 20ms delay:
```bash
sudo mn --controller=remote,port=6633 --topo linear,3 --link tc,delay=20ms
```

**Step 3: Run Measurements (Inside Mininet CLI)**
```mininet
mininet> h1 ping -c 5 h3
```

**Sample output:**
```
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=41.5 ms
...
rtt min/avg/max/mdev = 40.7/41.5/42.6/0.650 ms
```

---

## 📊 Experimental Results

### Test 1: 5ms Delay per Link

| Parameter | Value |
|-----------|-------|
| Delay per link | 5 ms |
| Number of hops | 4 |
| One-way delay | 4 × 5 = 20 ms |
| Theoretical RTT | 2 × 20 = **40 ms** |
| Observed RTT | **41.5 ms** |
| Error | 3.75% ✅ |

### Test 2: 20ms Delay per Link

| Parameter | Value |
|-----------|-------|
| Delay per link | 20 ms |
| Number of hops | 4 |
| One-way delay | 4 × 20 = 80 ms |
| Theoretical RTT | 2 × 80 = **160 ms** |
| Observed RTT | **161 ms** |
| Error | 0.625% ✅ |

### The Formula
```
RTT = 2 × (Number of hops × Delay per link)

For our topology: RTT = 2 × (4 × d) = 8d
```

### What We Found
- RTT increases linearly with delay ✓
- Results match theory within 4% error ✓
- Multi-hop networks multiply the delay effect ✓
- First ping may be slightly higher (ARP + flow setup) ⚠️

---

## 🔌 SDN Controller Logic

### How It Works

The POX controller uses a learning switch approach:

1. **First packet arrives** → controller receives `packet_in` event
2. **Controller learns** the source MAC address and port
3. **Controller checks** if it knows the destination
   - If yes: installs flow rule and forwards
   - If no: floods to all ports (discovery phase)
4. **Next packets** use the installed flow rule (no controller overhead)

### Flow Rule Details

**What gets installed:**
- Match: Source MAC + Destination MAC
- Action: Forward to the learned port
- Priority: 100
- Timeout: 10 seconds idle, 30 seconds hard

This is efficient because after the first packet, subsequent packets are forwarded in-switch without bothering the controller.

---

## 📈 Performance Analysis

### Latency Breakdown (20ms Case)

```
Total RTT = 161 ms
├─ Propagation delay = 160 ms (4 hops × 20ms × 2 directions)
└─ Processing overhead ≈ 1 ms (controller + switch operations)
```

### Key Insight

The delay is mostly propagation (as expected). The controller processing is negligible, confirming that SDN doesn't add significant overhead once flow rules are installed.

### Bandwidth Impact

With high delay (20ms), TCP performance drops because:
- Longer round-trip times
- Wider congestion windows needed
- More retransmission timeouts

This is why delay-sensitive applications (VoIP, gaming) need careful network design.

---

## ✅ Validation & Testing

We tested three scenarios to confirm everything works:

### Test 1: Basic Connectivity
```mininet
mininet> h1 ping -c 5 h3
```
**Result:** 0% packet loss, stable RTT ✓

### Test 2: Multiple Hosts
```mininet
mininet> pingall
```
**Result:** All hosts can reach each other ✓

### Test 3: Flow Table Check
```mininet
mininet> s1 dpctl dump-flows
```
**Result:** Flow rules installed correctly with proper match/action fields ✓

All tests passed, confirming the controller and topology work as expected.

---

## 🔧 Code Explanation

### What controller.py Does

**Main functions:**

1. **`launch()`** - Entry point that registers the controller with POX
2. **`start_switch(event)`** - Called when a switch connects
3. **`packet_in_handler(event)`** - Processes packets from switches
4. **`install_flow_rule()`** - Creates and installs OpenFlow rules

### Design Approach

We chose a **learning switch** because:
- Simple to understand and implement
- Shows the core SDN concept (dynamic flow rules)
- Easy to validate with ping tests
- Scales to show multi-hop effects

### Why This Works

The learning switch learns paths on-demand. The first packet triggers the controller, which then installs a rule. Future packets use the rule directly, avoiding controller overhead. This demonstrates the efficiency of SDN.

---

## 📊 Expected Output

### From POX Controller (Terminal 1)
```
INFO:core:POX 0.7.0 (gar) is up.
INFO:openflow.of_01:[00-00-00-00-00-01 1] connected
INFO:controller:Packet_in: Switch 1, Port 1, Src: 00:00:00:00:00:01
INFO:controller:Flow rule installed via port 2
```

### From Mininet (Terminal 2)
```
mininet> h1 ping -c 5 h3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=41.6 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=41.5 ms
...
rtt min/avg/max/mdev = 41.3/41.5/41.6/0.127 ms
```

That's it. Clean output, clear results.

---

## 🎓 SDN Concepts Demonstrated

### 1. Controller-Switch Communication
- Switches connect to controller via TCP (port 6633)
- Switches send `packet_in` events when they don't know what to do
- Controller sends `flow_mod` commands to install rules
- This is OpenFlow protocol in action

### 2. Flow Rules
A flow rule says: "If you see packets that match X, then do Y"
- **Match:** Source MAC = h1, Destination MAC = h3
- **Action:** Forward to port 2
- **Benefit:** No need to ask controller every time

### 3. Network Programmability
Instead of predefined routing, SDN lets you write custom logic in Python. You control packet forwarding rules in real-time. This is powerful for:
- Load balancing
- Security policies
- Traffic engineering
- Research and testing

### 4. Performance Metrics
We measured:
- **RTT** - How long for a packet to make a round trip
- **Consistency** - Do results repeat reliably?
- **Accuracy** - Do measurements match theory?

---

## 🐛 Troubleshooting

| Problem | Fix |
|---------|-----|
| POX won't connect | Make sure you started POX first and it's on port 6633 |
| RTT is too high | Check that delay is applied with `tc` (Linux Traffic Control) |
| Ping fails | Make sure controllers and switches are running; try `pingall` first |
| Flow rules missing | Verify controller code is running; check POX logs |
| Network won't start | Kill any old Mininet: `sudo pkill -f mininet` |

---

## 📚 References

- POX Controller: https://github.com/noxrepo/pox
- Mininet: http://mininet.org
- OpenFlow Spec: https://opennetworking.org/
- Linux tc Command: `man tc`

---

## 📝 Author & Submission

**Student:** Mirunjai Suresh Kumar  
**SRN:** PES1UG24AM161  
**University:** PES University, Bangalore  
**Program:** CSE (AIML) - Class 4C  
**Date:** April 2025

---

**Status:** ✅ Complete and Ready for Submission
