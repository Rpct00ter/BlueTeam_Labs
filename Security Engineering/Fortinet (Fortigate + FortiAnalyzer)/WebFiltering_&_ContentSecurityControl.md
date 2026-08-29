# Lab 9 - Web Filtering & Content Security Control

This repository showcases my hands-on implementation of **Category-Based Web Filtering**, **Local Web Rating Overrides**, **User-Authenticated Security Bypasses**, and **Static URL Filtering** on a **FortiGate Next-Generation Firewall (NGFW)**, integrated with centralized monitoring on **FortiAnalyzer**.

---

## Project Objective & Use Case
Unrestricted web access in an enterprise environment exposes the organization to malware, data exfiltration, legal liability, and lost productivity. 

In this lab, I designed and configured a multi-layered web filtering policy to control inbound and outbound web traffic. The objective was to enforce acceptable use policies (blocking social media), apply soft-repressive controls (warning users), implement role-based overrides (requiring authentication to access restricted sites), and configure local URL exceptions.

---

## Web Filtering Evaluation Order
When a client requests a URL, the FortiGate processes the web security profiles using a strict evaluation hierarchy:

\\[\text{Incoming URL Request} \;\rightarrow\; \textbf{1. Static URL Filter} \;\rightarrow\; \textbf{2. Web Rating Override} \;\rightarrow\; \textbf{3. FortiGuard Category Action}\\]

---

## Step 1: Licensing & Inspection Mode Selection

Before applying content restrictions, I verified service availability and aligned the firewall's inspection architecture:

1. Navigated to the **Dashboard** and checked the **Licenses** widget to verify the **Web Filter** service was licensed and actively communicating with the **FortiGuard Distribution Servers (FDS)**.
2. Navigated to **Policy & Objects → Firewall Policy** and edited the outbound internet policy to utilize **Proxy-based** inspection mode, which allows full web traffic processing through the proxy engine.
<img width="339" height="240" alt="image" src="https://github.com/user-attachments/assets/4f1061fb-a77a-48ed-bfb3-8e1a6dfd8027" />


---

## Step 2: Configuring FortiGuard Category-Based Actions

Using the global FortiGuard categorization engine, I restricted or monitored specific web behaviors:

1. Navigated to **Security Profiles → Web Filter** and edited the `default` profile.
2. Under **General Interest - Personal**, located **Social Networking** (representing domains like `www.facebook.com`), right-clicked, and set the action to **Block**.
3. Under **Bandwidth Consuming**, located **Internet Telephony**, set the action to **Warning**, and configured a **Warning Interval** of **5 minutes**.

<img width="1918" height="903" alt="image" src="https://github.com/user-attachments/assets/d901ef5d-f7c5-4071-9c0b-ad33717ec495" />


### **Verification:**
* **Block Action:** Attempting to access `www.facebook.com` on the client machine resulted in an immediate **Web Page Blocked** security replacement page served by the FortiGate.
* **Warning Action:** Visiting `voice.google.com` presented a warning page. Clicking **Proceed** granted access to the site for the configured 5-minute interval.

<img width="509" height="581" alt="image" src="https://github.com/user-attachments/assets/aca1cf3d-5f7a-4914-898e-15c4ce780907" />

---

## Step 3: Implementing local Web Rating Overrides

When an organization's local policy differs from FortiGuard’s global classification, Web Rating Overrides are utilized:

1. Navigated to **Security Profiles → Web Rating Overrides** and clicked **Create New**.
2. Entered `www.bing.com` and locally re-categorized it under the **Security Risk** category with the sub-category **Malicious Websites**.

<img width="551" height="578" alt="image" src="https://github.com/user-attachments/assets/ed05c474-373c-43b4-b854-71ae9bbdb3a8" />


---

## Step 4: Configuring Authenticate Actions & Role-Based Access

Rather than flatly blocking a category, I implemented an authentication bypass to allow verified users temporary access to restricted categories:

1. Created a local firewall user (`user1` / `Fortinet1!`) and assigned them to the **Override_Permissions** user group.
2. Edited the Web Filter profile and changed the action for the **Malicious Websites** category from *Block* to **Authenticate**.
3. Set the **Warning Interval** to **5 minutes** and mapped the authentication privilege to the **Override_Permissions** group.

<img width="934" height="115" alt="image" src="https://github.com/user-attachments/assets/769c767a-ecae-4ee3-8989-7eaab8f8fef9" />


### **Verification:**
Visiting `www.bing.com` (which was overridden to *Malicious Websites*) now prompted an authentication challenge page. Entering the credentials for `user1` successfully authorized a 5-minute bypass, registering a **passthrough** action in the security logs.

---

## Step 5: Enforcing Static URL Filters

Static URL filtering allows the perimeter administrator to block or permit explicit URL matches locally before any remote FortiGuard database check is run:

1. Edited the Web Filter profile and enabled **URL Filter** under the **Static URL Filter** section.
2. Added a new entry for `www.bing.com`, set the type to **Simple**, and set the action to **Block**.
3. Converted the firewall policy and Web Filter profile feature set to **Flow-based** mode to test behavior differences.

<img width="589" height="228" alt="image" src="https://github.com/user-attachments/assets/a667bca3-3b5f-4564-b146-4df83f5c43fc" />


### **The Critical Log Behavior Difference:**
Because the static URL filter is evaluated **first**, matches are blocked immediately. Consequently, no FortiGuard rating query is sent to the FDS, resulting in a **Web Filter Log entry where the "Category" field is completely empty**.

---

## Step 6: Web Filtering Incident Logs on FortiAnalyzer

For security event analysis, I pivoted to **FortiAnalyzer** under **Log View → Logs → Fortinet Logs → FortiGate → Security → Web Filter** to inspect the traffic.

I verified three major log behaviors:
1. **Blocked Log:** Traced blocks for `www.facebook.com` with the category marked as *Social Networking*.
2. **Passthrough Log:** Traced the successful authentication session for `www.bing.com` showing user association and a `passthrough` action status.
3. **Empty Category Log:** Identified the static block for `www.bing.com`, verifying that the Category field remained blank because the static engine dropped the traffic before categorization.

<img width="1001" height="270" alt="image" src="https://github.com/user-attachments/assets/8c6c207e-cf88-4f91-a7d7-3dd90782bde2" />


---

## 🛠️ CLI Diagnostics & Troubleshooting

To monitor rating server connections and resolve latency issues from the command-line interface, I utilized the following diagnostics:

```bash
# Display the connection status and response times of FortiGuard Distribution Servers (FDS)
get webfilter status

# Verify real-time rating server RTT and see which FDS server is currently active
diagnose debug rating
```

---

## 🧠 Key Takeaways for SOC Roles
* **Log Triage (Static vs Category):** Understanding that an empty category field in a web filter log is normal behavior for local static blocks avoids wasting time troubleshooting FDS database issues.
* **Policy Auditing:** Leveraging the difference between **Allow** (permits traffic silently) and **Monitor** (permits and generates explicit threat logs) is critical for adjusting corporate web security posture and baseline analysis.
* **Optimal Inspection:** Since web filtering operates on the HTTP Host header or TLS Server Name Indication (SNI) handshake, **Certificate Inspection** is highly efficient and sufficient for standard category-based filtering, avoiding the overhead of full decryption.
