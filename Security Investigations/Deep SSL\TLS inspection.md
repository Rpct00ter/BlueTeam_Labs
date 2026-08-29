# Lab 7 — Certificate Operations (Deep SSL/TLS Inspection)

This repository showcases my hands-on implementation of **Full SSL/TLS Deep Inspection (Man-in-the-Middle)** on a **FortiGate Next-Generation Firewall (NGFW)**, integrated with centralized threat and event logging on **FortiAnalyzer**.

---

## 📌 Project Objective & Use Case
Over 90% of modern web traffic utilizes HTTPS encryption. While this protects user privacy, it creates a massive security blind spot: firewall security engines (such as Antivirus, IPS, and Web Filtering) cannot scan encrypted payloads. 

In this lab, I configured FortiGate to act as a **secure intermediary (Man-in-the-Middle proxy)** to decrypt HTTPS sessions, scan payloads for threats, enforce strict certificate revocation checks, and seamlessly re-encrypt the traffic before forwarding it to its destination. I also configured granular security exemptions to bypass decryption for sensitive or trusted destinations.

---

## ⚙️ Architectural Workflow

Before deep inspection, the firewall only has visibility up to the SSL handshake (Server Name Indication / SNI). With **Full SSL Inspection**, the traffic flow is split into two secure segments:

\\[\text{Client} \quad \xrightarrow{\text{HTTPS (Signed by FortiGate CA)}} \quad \textbf{FortiGate NGFW (Inspects Payload)} \quad \xrightarrow{\text{HTTPS (Signed by Web Server CA)}} \quad \text{Web Server}\\]

---

## Step 1: Configuring the Custom SSL Inspection Profile

To maintain control over how the FortiGate manages certificate validation anomalies, I created a custom SSL/SSH inspection profile:

1. Navigated to **Security Profiles → SSL/SSH Inspection** and selected **Create New**.
2. Configured the profile to intercept **Multiple Clients Connecting to Multiple Servers**.
3. Selected **Full SSL Inspection** as the inspection method.
4. Set the firewall to use the local CA certificate **Fortinet_CA_SSL** to sign the dynamically generated site certificates.
5. Configured the **Invalid SSL certificates** action to **Custom** to allow granular control over how expired or untrusted certificates are handled.

/new line
[picture of something <Screenshot of the FortiGate GUI in Security Profiles -> SSL/SSH Inspection displaying the "Custom_Full_Inspection" profile configuration with Full SSL Inspection enabled, Fortinet_CA_SSL selected, and Invalid SSL certificates set to Custom>]
/new line

---

## Step 2: Applying the Decryption Profile to the Edge Firewall Policy

An SSL inspection profile must be bound to a security policy to actively inspect traffic.

1. Navigated to **Policy & Objects → Firewall Policy** and edited the outbound internet access policy (`Internet`).
2. Under **Security Profiles**, enabled **Web Filter** and bound my newly created **Custom_Full_Inspection** profile.
3. Configured **Log Allowed Traffic** to **All Sessions** to capture detailed connection metadata.

/new line
[picture of something <Screenshot of the Firewall Policy configuration page under Security Profiles, highlighting the SSL Inspection profile set to Custom_Full_Inspection and Web Filter set to default>]
/new line

---

## Step 3: Simulating the Trust Issue (The MITM Verification)

Before importing the FortiGate CA certificate into the client workstation's trust store, I attempted to visit a secure HTTPS site (`https://www.goto.com`).

* **Expected Result:** The browser stopped the connection, throwing an untrusted issuer security warning.
* **Technical Explanation:** Because the FortiGate intercepts the handshake, it generates a replacement certificate for `www.goto.com` on-the-fly and signs it using its local `Fortinet_CA_SSL` certificate. Since the browser does not yet have this CA in its root trust store, it flags the certificate as untrusted.

/new line
[picture of something <Screenshot of Firefox displaying an 'Insecure Connection' warning when visiting an HTTPS site, with the certificate details expanded showing 'Issuer: Fortinet' instead of the site's public CA>]
/new line

---

## Step 4: Installing and Trusting the FortiGate CA Certificate

To establish trust between the internal endpoints and the intercepting firewall, the root CA must be pushed to clients.

1. Logged in to the FortiGate GUI, navigated to **System → Certificates**, and downloaded the **Fortinet_CA_SSL** local CA certificate.
2. Imported the certificate directly into the client browser's **Authorities trust store**, checking the box to **Trust this CA to identify websites**.
3. Restarted the browser session.

/new line
[picture of something <Screenshot of Firefox Certificate Manager under the 'Authorities' tab, displaying the imported and trusted 'Fortinet_CA_SSL' certificate with trust settings enabled>]
/new line

### Verification:
I re-visited `https://www.goto.com`. The site loaded seamlessly without warnings. Inspecting the secure lock icon verified that the certificate was signed by our internal FortiGate CA, proving the firewall successfully decrypted and inspected the payload in transit.

/new line
[picture of something <Screenshot of the browser visiting the HTTPS website successfully with the secure lock pad clicked, showing the connection is verified by 'Fortinet' and indicating full decryption is occurring>]
/new line

---

## Step 5: Advanced Security — Managing Certificate Revocation (CRL & OCSP)

A compromised or retired certificate must be blocked. I configured the FortiGate to actively check certificate revocation status using both local lists and online queries:

### A. Certificate Revocation List (CRL) Configuration
1. Found the CRL Distribution Point URI inside the target Certificate Authority details.
2. Navigated to **System → Certificates** and imported the CRL via **Create/Import → CRL**, pointing the HTTP server URL directly to the CA's published list.

### B. Online Certificate Status Protocol (OCSP) CLI Configuration
For real-time status checking, I leveraged the FortiGate CLI to enable strict OCSP checks:

```bash
# Enable strict real-time OCSP validation checks on VPN and client certs
config vpn certificate setting
    set ocsp-option certificate
    set ocsp-status enable
    set strict-ocsp-check enable
end
```
*(Reference:)*

---

## Step 6: Testing Perimeter Blocks on Invalid Certificates

With deep inspection active, the firewall enforces the configured security behaviors on untrusted destinations.

1. Navigated to a test domain with an expired SSL certificate: `https://expired.badssl.com/`.
2. **Result:** The connection was immediately terminated by the FortiGate's security engine, indicating the perimeter blocked an expired certificate transaction.

/new line
[picture of something <Screenshot of Firefox showing a connection blocked screen when attempting to access expired.badssl.com, verifying that the firewall blocked the session before rendering the invalid handshake>]
/new line

---

## Step 7: Centralized Security Log Analysis on FortiAnalyzer

To verify threat capture, I analyzed the security transactions on **FortiAnalyzer** under **Log View → Logs → Fortinet Logs → FortiGate → Security → SSL**.

The logs detailed:
* **The Action:** Blocked / Terminated.
* **The Reason:** Certificate validation failure (expired certificate).
* **The Client/Destination IPs** and policy details.

/new line
[picture of something <Screenshot of FortiAnalyzer SSL Logs showing the blocked event for expired.badssl.com, highlighting the source IP, destination IP, action set to 'Block', and reason column>]
/new line

---

## Step 8: Configuring Deep Inspection Address Exemptions

Certain categories (such as Finance, Health) or specific applications break when subjected to SSL decryption. To accommodate this, I configured a bypass exception:

1. Navigated to **Security Profiles → SSL/SSH Inspection** and edited our `Custom_Full_Inspection` profile.
2. Under **Exempt from SSL Inspection → Addresses**, created a destination address override for the target IP/subnet (`badssl` representing `104.154.89.105/32`).
3. Saved the profile.

/new line
[picture of something <Screenshot of the SSL/SSH Inspection profile editor, pointing to the Exempt from SSL Inspection section with the 'badssl' address object added as an active bypass exception>]
/new line

### Verification:
I visited `https://expired.badssl.com/` again. The site was now accessible because the firewall bypassed full SSL inspection on that traffic. (The browser still displayed its native security warning, as the original site certificate remained expired and the firewall was no longer acting as the proxy to override/block it).

---

## 🧠 Key Takeaways for SOC Roles
* **MITM Visibility:** Deep packet inspection is mandatory for edge antivirus, web control, and IPS signature matching over encrypted protocols.
* **PKI Governance:** Configured Certificate Revocation Lists (CRL) and Online Certificate Status Protocol (OCSP) to ensure perimeter identity validation.
* **Performance & Privacy Tuning:** Designed target bypass exceptions to satisfy regulatory compliance requirements (e.g., privacy laws) and prevent application failures.
