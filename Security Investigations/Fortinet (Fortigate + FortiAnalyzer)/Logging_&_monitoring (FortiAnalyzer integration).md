# Lab 2 — Centralized Logging and Monitoring (FortiAnalyzer Integration)

This repository showcases my hands-on implementation of **centralized log management** by integrating a **FortiGate Next-Generation Firewall (NGFW)** with **FortiAnalyzer**. 

---

## 📌 Project Objective & Use Case
For any Security Operations Center (SOC) team, visibility across the network edge is paramount. While standalone firewalls generate immense amounts of log data, storing and searching logs locally consumes valuable firewall system resources and makes multi-vector threat correlation nearly impossible. 

In this lab, I configured the FortiGate NGFW to securely stream its logs in **Real Time** to a centralized log management and analytical platform (FortiAnalyzer). I then authorized the device within a specific **Administrative Domain (ADOM)**, simulated network traffic to generate logs, and performed log analysis using both real-time and historical views.

---

## ⚙️ Log Management Architecture
Centralizing logging shifts the burden of log storage, database indexing, and analytical querying from the firewall to the dedicated management platform:

\\[\textbf{FortiGate NGFW} \quad \xrightarrow{\text{Generates Real-Time Security Logs}} \quad \textbf{FortiAnalyzer (Collects, Indexes, & Analyzes)} \quad \rightarrow \quad \textbf{SOC L1/L2 Alert Triage}\\]

---

## Step 1: Configuring FortiAnalyzer Log Forwarding on the FortiGate

To establish a secure logging connection from the firewall to the centralized collector:

1. Logged in to the FortiGate GUI and navigated to **Security Fabric → Fabric Connectors**.
2. Edited the **Logging & Analytics** widget and enabled **FortiAnalyzer**.
3. Configured the connector settings:
   * **Server IP:** `10.0.13.125`
   * **Upload Option:** **Real Time** (ensures immediate visibility for SOC analysts)
   * **Allow access to FortiGate REST API:** Enabled
   * **Verify FortiAnalyzer certificate:** Enabled (secures communication against MITM)
4. Clicked **OK** and accepted the remote FortiAnalyzer serial number.

<img width="1919" height="905" alt="image" src="https://github.com/user-attachments/assets/20edba86-7688-4d45-94be-2b87f10c7ff3" />


---

## Step 2: Device Authorization & ADOM Alignment on FortiAnalyzer

When a FortiGate first connects to a FortiAnalyzer, it is placed in an unauthorized state for security. I completed the registration workflow on the receiver side:

1. Logged in to the **FortiAnalyzer** GUI.
2. Navigated to **Device Manager** and opened the **Unauthorized Devices** notification list.
3. Selected `HQ-NGFW-1` and clicked **Authorize**.
4. Assigned the device to the **root ADOM** (Administrative Domain). 

> 💡 **ADOM (Administrative Domain) Concept:** Logical containers used to organize, group, and isolate log databases for different devices or business units. This is critical in multi-tenant or large enterprise SOC environments to enforce role-based access control (RBAC).

<img width="1919" height="905" alt="image" src="https://github.com/user-attachments/assets/8a8aed58-0e34-4c31-bd25-b0277a687876" />


---

## Step 3: Verifying the Connection

To confirm the successful secure channel and log flow:

1. Navigated to **Device Manager → All Logging Devices** on FortiAnalyzer and verified that the FortiGate appeared with an active, connected status.
2. On FortiGate, verified under **Security Fabric → Fabric Connectors → Logging & Analytics** that a **green connection arrow** appeared next to FortiAnalyzer, indicating active log reception.

/new line
[picture of something <Screenshot of the FortiGate Logging & Analytics connector dashboard displaying the active connection status with the green up-arrow pointing to FortiAnalyzer>]
/new line

---

## Step 4: Simulating Network Traffic & Generating Log Entries

To validate log generation and collection without executing a real or dangerous attack, I leveraged FortiOS CLI diagnostic testing utilities:

1. Opened an SSH session to the FortiGate.
2. Executed the CLI diagnostic command to generate artificial traffic and event logs:
   ```bash
   diagnose log test
   ```
3. Ran this command multiple times to populate the FortiAnalyzer database.

> ⚠️ **SOC Warning:** The `diagnose log test` command generates simulated, harmless traffic and security anomalies. It is used strictly for pipeline validation and does not indicate an active compromise.


---

## Step 5: Triage and Log Analysis on FortiAnalyzer

With logs successfully forwarded and indexed, I conducted log investigation on FortiAnalyzer:

1. Navigated to **Log View → Logs → Fortinet Logs** and selected the **FortiGate** icon.
2. Selected **Traffic** logs to view outbound and inbound session logs.
3. Leveraged **Real-Time Log View** to watch current incoming logs stream to the console (perfect for live incident response or verification).
4. Utilized **Historical Log View** to perform retrospective queries on indexed data (essential for security forensic investigations).

<img width="1919" height="905" alt="image" src="https://github.com/user-attachments/assets/e1e1504d-0ea2-40ba-b89f-b07e026cb209" />


---

## 🧠 Key Takeaways for SOC Roles

* **Analytics vs. Archive Logs:** Understanding how FortiAnalyzer handles logs is crucial for compliance and performance. 
  - **Analytics Logs:** Structured and fully indexed in a database. They are optimized for fast searching, ad-hoc threat hunting, and generating structured compliance reports.
  - **Archive Logs:** Raw compressed log files used for offline, long-term storage to meet compliance requirements with minimal storage footprint.
* **Connection Troubleshooting:** If logs stop flowing, a SOC analyst should immediately check if the **FDS/collector IP is correct**, confirm **SSL certificate verification is intact**, verify that the device is **properly authorized in the target ADOM**, and look for the **green arrow status** on the FortiGate dashboard.
