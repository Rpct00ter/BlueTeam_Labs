# Lab 10 — IPS and Application Control (Threat Mitigation & Traffic Steering)

This repository showcases my hands-on implementation of an **Intrusion Prevention System (IPS)** to protect a web server from known web exploits, and **Application Control** to identify, block, and steer application-layer traffic on a **FortiGate Next-Generation Firewall (NGFW)**, integrated with monitoring on **FortiAnalyzer**.

---

## Project Objective & Use Case
A standard stateful firewall only controls traffic based on ports and protocols. However, modern threats hide inside legitimate ports (like HTTP port 80 or HTTPS port 443), and evasive applications can run over non-standard ports. 

In this lab, I deployed deep packet inspection to protect an internal Apache web server from live vulnerability scans and exploits. I then configured Application Control to override and block high-bandwidth applications while maintaining a granular exception to allow specific business-critical sites.

---

## Part 1: Intrusion Prevention System (IPS) Implementation

### Step 1: Configuring a Targeted IPS Sensor

To protect an internal web server, I built a highly specific IPS profile. Rather than enabling thousands of irrelevant signatures (which degrades firewall performance), I tuned the sensor to focus exclusively on relevant threats.

1. Navigated to **Security Profiles → Intrusion Prevention** and selected **Create New** to build a sensor named `WEBSERVER`.
2. Created custom filters targeting signatures matching:
   * **Severity (SEV):** Medium, High, and Critical.
   * **Target (TGT):** Server.
   * **Application (App):** Apache.
3. Saved the sensor, matching all active signatures matching these strict parameters.

<img width="860" height="353" alt="image" src="https://github.com/user-attachments/assets/54cbd66d-68ed-46a0-87e9-fd65a7dbbe1f" />


---

### Step 2: Creating the Destination NAT (VIP) and Security Policy

To expose the web server to the external network and apply security inspection:

1. Navigated to **Policy & Objects → Virtual IPs** and created a VIP named `VIP-WEB-SERVER` to map the external IP `100.65.0.200` to the internal server `10.0.11.50`.
2. Created a new firewall policy named `Web_Server_Access_IPS`:
   * **Incoming Interface:** `port2` (WAN)
   * **Outgoing Interface:** `port4` (Internal LAN)
   * **Destination:** `VIP-WEB-SERVER`
   * **Inspection Mode:** Flow-based
   * **NAT:** Disabled (since the VIP natively handles the destination translation).
3. Under **Security Profiles**, enabled the custom **IPS** profile (`WEBSERVER`) and **SSL Inspection** (`certificate-inspection`).

<img width="479" height="823" alt="image" src="https://github.com/user-attachments/assets/72cc2b56-4903-4da1-881c-ffa5372afb18" />


---

### Step 3: Emulating Web Exploits

To simulate an external attacker scanning and attempting to exploit our protected web server:

1. Opened an SSH session to the external Linux workstation provided by Fortinet in lab environment (`LINUX-ISP`).
2. Executed a live web vulnerability scan using the Nikto command-line tool targeting the public VIP address:
   ```bash
   nikto.pl -host 100.65.0.200
   ```
3. Let the attack script execute, mimicking real-world exploit signatures targeted at HTTP/Apache.

---

### Step 4: Security Log Analysis & Threat Intelligence Triage

On **FortiAnalyzer**, I pivoted to **Log View → Security → Intrusion Prevention** to analyze the blocked events.

1. Correlated the real-time security events detailing the detected attack signatures, severity levels, and client IPs.
2. Opened an individual threat log and selected the **Reference Link** to pivot to the **FortiGuard Labs Threat Encyclopedia** to verify the CVE, severity details, and vendor mitigation recommendations.

<img width="1666" height="517" alt="image" src="https://github.com/user-attachments/assets/e1c7d8b5-34bd-4d88-b681-44d0ce344ebb" />


---

### Step 5: Advanced IPS Command-Line Troubleshooting

To manage and troubleshoot the IPS engine directly from the CLI, I used the following diagnostics on the FortiGate:

```bash
# 1. Check the general health, memory usage, and status of the IPS engines
diagnose test application ipsmonitor 1

# 2. Force the firewall into "Bypass Mode" (used to quickly restore traffic if IPS engines freeze or overload)
diagnose test application ipsmonitor 5

# 3. Soft-restart all active IPS engines (which also resets bypass mode back to 'disabled')
diagnose test application ipsmonitor 99
```

---

## Part 2: Application Control & Traffic Steering

Unlike port filtering, Application Control uses IPS protocol decoders to identify the exact application generating traffic, even if it runs over non-standard or encrypted ports.

### Step 1: Implementing a Bandwidth Block (Filter Override)

1. Navigated to **Security Profiles → Application Control** and edited the default profile.
2. Under **Application and Filter Overrides**, created a new **Filter** override:
   * **Filter Behavior:** `Excessive-Bandwidth` (to target bandwidth-heavy programs).
   * **Action:** Block.
3. Applied the default Application Control profile and a **Deep SSL Inspection** profile to the outbound firewall policy. *Note: Deep inspection is mandatory here to decrypt HTTPS payloads and identify application signatures inside encrypted traffic.*

📌/new line
[picture of something <Screenshot of the FortiGate Application Control profile configuration showing the Filter Overrides section with an active block rule targeting the 'Excessive-Bandwidth' behavior filter>]
/new line

---

### Step 2: Custom Block Pages

To verify the block, I visited `http://abc.go.com` (which matched the blocked Excessive-Bandwidth behavior filter).

1. Enabled **Replacement Messages for HTTP-based Applications** in the Application Control profile.
2. Refreshed the client browser to verify that the firewall intercepted the HTTP connection and served a customized **Application Blocked** replacement page.

<img width="523" height="568" alt="image" src="https://github.com/user-attachments/assets/528a9250-887f-46f8-8a4c-c9eb1ef2f2e9" />

---

### Step 3: Configuring a Specific Application Override (The Ordering Rule)

In an enterprise environment, a broad block filter must often have selective exceptions. I created an override to allow `ABC.Com` specifically:

1. Under the default Application Control profile overrides, added a new **Application** override.
2. Searced for and selected `ABC.Com` and set the action to **Allow**.
3. **The Critical Rule (Top-to-Bottom Evaluation):** Moved the `ABC.Com` Allow rule **ABOVE** the broader `Excessive-Bandwidth` Block rule. Since Application Control overrides evaluate from top to bottom, this ensures the firewall allows `abc.com` before the bandwidth filter blocks it.

<img width="483" height="258" alt="image" src="https://github.com/user-attachments/assets/f21fe8d7-73e6-4d3c-ad52-0aac59e4606c" />


**Result:** Visiting `http://abc.go.com` succeeded because the specific allow exception matched first.

---

### Step 4: Tracking Applications via FortiView

On **FortiAnalyzer**, I investigated application-layer logs and traffic distribution:

1. Navigated to **FortiAnalyzer → FortiView → Applications & Websites** to view bandwidth, session counts, and byte counters by application.
2. Drill-down analysis of `ABC.Com` sessions verified the traffic bypassed the bandwidth filter and registered an "Allow" action.

<img width="1593" height="392" alt="image" src="https://github.com/user-attachments/assets/326ed435-e3a8-45fe-a499-b82ef269ae2a" />


---

## 🧠 Key Takeaways for SOC Roles
* **Targeted IPS Tuning:** Creating targeted IPS sensors based on severe vulnerabilities, Server-Tgt, and specific applications (Apache) minimizes firewall CPU overhead and filters out unnecessary false positives.
* **Application Decoders:** App Control uses deep packet analysis to look past port numbers, blocking bandwidth-heavy evasion apps even if they use unexpected ports.
* **Override Logic:** Specific overrides must always be ordered above broad filters to guarantee exception handling.
* **CLI Emergency Diagnostics:** Mastering the `ipsmonitor` command line utility is crucial for real-time performance troubleshooting and managing active bypass behaviors under high network load.
