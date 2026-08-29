# Lab 14 — Diagnostics, Performance, and Packet Flow Troubleshooting

This repository showcases my hands-on ability to troubleshoot network-layer and physical-layer issues on a **FortiGate Next-Generation Firewall (NGFW)** using  CLI diagnostics, real-time packet sniffing, and the FortiOS **Debug Flow** engine.

---

## Project Objective & Use Case
In this lab, I diagnosed a simulated network outage where an internal client could not reach an external destination. I utilized a bottom-up troubleshooting methodology (Physical \\(\rightarrow\\) Packet Capture \\(\rightarrow\\) Routing Engine \\(\rightarrow\\) Security Policy \\(\rightarrow\\) NAT) to isolate the root cause and restore secure connectivity.

---

## Step 1: Establishing a Performance & Resource Baseline

Before debugging individual packet flows, I audited the overall health of the firewall to ensure the issue wasn't caused by a CPU bottleneck or memory exhaustion:

1. Checked general system statistics using `get system status` to confirm the firmware branch, HA mode, and hostname.
2. Ran `get system performance status` to log overall CPU, memory utilization, average network usage, and session setup rates.
3. Monitored active processes using `diagnose sys top 1` to identify any resource-hungry daemons.


---

## Step 2: Isolating the Boundary with the Packet Sniffer

To determine if the connection issue was occurring outside or inside the firewall, I initiated a continuous ping from the client workstation and ran the FortiGate packet sniffer:

```bash
# Capture ICMP traffic on any interface matching the destination IP
diagnose sniffer packet any "icmp and host 100.65.0.254" 4
```


* **Observation:** The sniffer captured inbound packets on `port4` (the internal LAN interface) but showed no corresponding outbound packets leaving the WAN interface (`port2`).
* **Technical Verdict:** This proved that the client's traffic successfully reached the FortiGate, meaning the issue lay entirely within the firewall's internal processing engines.


---

## Step 3: Deep Packet Tracing using Debug Flow

To reveal exactly *why* the firewall was discarding the packets, I configured the **Debug Flow** engine to trace the session lifecycle:

1. Navigated to **Network → Diagnostics → Debug Flow**.
2. Set a filter for the destination IP (`100.65.0.254`) and protocol (`ICMP`).
3. Started the debug trace to capture the firewall's step-by-step decision matrix.

* **The Discovery:** The trace confirmed that the routing engine successfully resolved the exit path (`find a route: ... via port2`). However, the packet was immediately discarded with the error message: `Denied by forward policy check (policy 0)`.
* **The Root Cause:** **Policy 0** represents the firewall's implicit deny policy. This verified that while a valid route existed, there was no matching firewall policy configured to allow ICMP traffic.

<img width="1919" height="908" alt="debug1" src="https://github.com/user-attachments/assets/e50bc827-2093-47ca-9e64-30b5d48df9aa" />
<img width="1284" height="904" alt="debug2" src="https://github.com/user-attachments/assets/dedc62b9-b001-4404-a30b-4504b3d17d4b" />


---

## Step 4: Policy Remediation and NAT Verification

To resolve the blockage, I updated the outbound internet policy and cleared the active sessions to ensure clean state tracking:

1. Navigated to **Policy & Objects → Firewall Policy** and edited the active outbound rule.
2. Added the permitted service to not only allow **HTTP**, but also **ICMP**.


---

## 🧠 Key Takeaways for SOC Roles
* **Sniffer vs. Debug Flow:** The packet sniffer answers **"Are packets arriving?"**, while Debug Flow answers **"What is the firewall doing with those packets, and why?"**. 
* **Policy 0 is the Clue:** Seeing `policy 0` in logs or debug streams always indicates a firewall policy mismatch or an **implicit deny**, never a routing issue.
* **System Resource Triage:** Using `get system performance status` paired with `diagnose sys top` allows L1/L2 analysts to quickly identify whether performance degradation is caused by a hardware limitation or a specific runaway process.

---
