# Cisco ISE 3.4 — Two Node Deployment with External Root CA Certificate

> **Author:** Anurag Mishra — Network Implementation Engineer  
> **Published:** September 2024  
> **LinkedIn Article:** [Cisco ISE 3.4 Two Node Deployment with External CA](https://www.linkedin.com/pulse/cisco-ise-34-two-node-deployment-external-ca-anurag-mishra-nybkc/)

---

## Lab Topology
<img width="786" height="443" alt="image" src="https://github.com/user-attachments/assets/f9dd979f-4644-4b58-b23e-f18c7cb21192" />


![ISE Two Node Deployment Topology](images/topology.png)

> **Topology Overview:** ISE-01 and ISE-02 are connected via ServerFarm switch alongside the CA/FTP server and AD/DHCP server. A Core switch connects the access layer with MAB and Dot1x test endpoints. FirePower (FTD) handles internet edge. VLANs 10–13 segment traffic across the environment.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Components Used](#2-components-used)
3. [Node Role Assignment](#3-node-role-assignment)
4. [Network & IP Addressing](#4-network--ip-addressing)
5. [Prerequisites](#5-prerequisites)
6. [Phase 1 — Windows Server CA Setup](#6-phase-1--windows-server-ca-setup)
7. [Phase 2 — DNS Configuration](#7-phase-2--dns-configuration)
8. [Phase 3 — Download Root CA Certificate](#8-phase-3--download-root-ca-certificate)
9. [Phase 4 — Primary Node (ISE-01) Configuration](#9-phase-4--primary-node-ise-01-configuration)
10. [Phase 5 — Secondary Node (ISE-02) Registration](#10-phase-5--secondary-node-ise-02-registration)
11. [Phase 6 — Verification](#11-phase-6--verification)
12. [Important Notes & Lessons Learned](#12-important-notes--lessons-learned)
13. [Two-Node Limitations — PAN Failover](#13-two-node-limitations--pan-failover)

---

## 1. Introduction

This document describes the best practices and step-by-step procedures to deploy a **Cisco ISE 3.4 two-node cluster** registered with an **external Root CA certificate** from a Windows Server Certificate Authority.

In a two-node deployment, both ISE nodes carry **all three personas** — PAN (Policy Administration), PSN (Policy Service), and MnT (Monitoring & Troubleshooting) — providing full redundancy on a minimal footprint. The external CA certificate ensures that inter-node communication (replication, registration) is trusted via a signed certificate rather than the default self-signed certificate.

---

## 2. Components Used

| Component | Version / Detail |
|---|---|
| Cisco ISE | 3.4 |
| Windows Server (CA + AD + DHCP) | 2016 |
| ISE Deployment Type | Physical (Two-Node) |
| Certificate Type | External Root CA (Windows CA) |
| Domain | `hydent.com` |

---

## 3. Node Role Assignment

| Node | Hostname | IP Address | Roles |
|---|---|---|---|
| **Primary** | `ise-01.hydent.com` | 10.10.10.11 | Primary PAN + Secondary MnT + PSN |
| **Secondary** | `ise-02.hydent.com` | 10.10.10.12 | Secondary PAN + Primary MnT + PSN |
| CA / FTP Server | `ca-server.hydent.com` | 10.10.10.10 | Root CA + FTP |
| AD / DHCP Server | `ad.hydent.com` | 10.10.10.20 | Active Directory + DHCP |

> 💡 **Role Explanation:**
> - **PAN (Policy Administration Node):** GUI administration, policy configuration, and replication control
> - **PSN (Policy Service Node):** Handles RADIUS/TACACS+ authentication requests from network devices
> - **MnT (Monitoring & Troubleshooting Node):** Collects logs, RADIUS live logs, and reports

---

## 4. Network & IP Addressing

| VLAN | Purpose |
|---|---|
| VLAN 10 | Management / ISE + Servers |
| VLAN 11 | Corporate Users (Dot1x) |
| VLAN 12 | Guest / MAB Devices |
| VLAN 13 | Quarantine / Remediation |

---

## 5. Prerequisites

Before starting deployment, verify all of the following:

- [ ] Each ISE node has a designated persona planned
- [ ] **NTP sync** configured on all ISE nodes pointing to the same NTP server
- [ ] **DNS forward and reverse lookup zones** created for all ISE FQDNs
- [ ] All ISE nodes running the **same IOS version and same patch level**
- [ ] **Root CA** installed on Windows Server and accessible at `<CA-IP>/certsrv`
- [ ] Console and GUI access available to both ISE nodes
- [ ] Ports open between ISE nodes:

```
TCP 443   — Admin GUI / ERS API
TCP 8905  — ISE Node Communication (NAD)
TCP 8443  — Guest Portal / Sponsor Portal
TCP 1521  — Database replication (Oracle)
TCP/UDP 1812, 1813 — RADIUS Auth/Acct
TCP 49    — TACACS+
UDP 123   — NTP
TCP/UDP 53 — DNS
```

### ISE Node Initial CLI Setup (if not done)

```bash
# On each ISE node via console
configure
 hostname ise-01
 ip domain-name hydent.com
 ip name-server 10.10.10.20
 ntp server 10.10.10.20
 clock timezone IST 5 30
end

# Verify DNS resolution from ISE CLI
nslookup ise-01.hydent.com
nslookup ise-02.hydent.com
```

---

## 6. Phase 1 — Windows Server CA Setup

Install the **Active Directory Certificate Services (AD CS)** role on Windows Server 2016 to act as the Root CA.

**Steps on Windows Server:**

1. Open **Server Manager** → Add Roles and Features
2. Select **Active Directory Certificate Services**
3. Select **Certification Authority** role service
4. Choose **Enterprise CA** → **Root CA** → **Create a new private key**
5. Set CA name (e.g., `hydent-RootCA`)
6. Set validity period (10 years recommended)
7. Complete installation

> ✅ Once installed, the CA is accessible at: `http://10.10.10.10/certsrv`

---

## 7. Phase 2 — DNS Configuration

DNS forward and reverse lookup zones must be configured so ISE nodes can resolve each other by FQDN. Registration **will fail** if DNS is not correct.

### Forward Lookup Zone Entries

![DNS Forward Lookup Zone] <img width="791" height="435" alt="image" src="https://github.com/user-attachments/assets/99c3673e-415d-4173-bac2-7aade4927ef8" />


| Record Type | Name | Value |
|---|---|---|
| A Record | ise-01 | 10.10.10.11 |
| A Record | ise-02 | 10.10.10.12 |

### Reverse Lookup Zone Entries

![DNS Reverse Lookup Zone]<img width="741" height="367" alt="image" src="https://github.com/user-attachments/assets/655ceb04-127e-4065-b78c-6a67ea725402" />


| Record Type | IP | PTR Record |
|---|---|---|
| PTR | 10.10.10.11 | ise-01.hydent.com |
| PTR | 10.10.10.12 | ise-02.hydent.com |

### Verify DNS Resolution — NSLookup

![NSLookup Verification](images/dns-nslookup.png)

```bash
# Run from Windows Server or any domain-joined machine
nslookup ise-01.hydent.com
nslookup ise-02.hydent.com
nslookup 10.10.10.11
nslookup 10.10.10.12
```

> ⚠️ **Both forward AND reverse lookups must resolve correctly before proceeding.**

---

## 8. Phase 3 — Download Root CA Certificate

The Root CA certificate must be downloaded and imported into **both** ISE nodes before registration.

**Step 1:** Open browser → navigate to:
```
http://10.10.10.10/certsrv
```

![CA Server Portal](images/ca-server-portal.png)

**Step 2:** Click **"Download a CA certificate, certificate chain, or CRL"**

![Download CA Certificate](images/ca-download.png)

**Step 3:** Select **Base 64** encoding → Click **"Download CA Certificate"**

![Base64 Download](images/ca-base64.png)

> 💡 Save this file as `rootca.cer` — you will import it into both ISE nodes

---

## 9. Phase 4 — Primary Node (ISE-01) Configuration

### Step 9.1 — Make ISE-01 the Primary PAN

1. Log in to **ISE-01** GUI: `https://ise-01.hydent.com`
2. Navigate to **Administration → Deployment**

![ISE Deployment Page](images/ise-deployment.png)

3. Click on **ISE-01** → Edit General Settings

![Edit ISE Node](images/ise-edit-node.png)

4. Click **"Make Primary"** → Click **Save**

![Make Primary](images/ise-make-primary.png)

> ✅ ISE-01 is now the **Primary PAN**

---

### Step 9.2 — Import Root CA into ISE-01 Trusted Store

1. Navigate to **Administration → System → Certificates → Trusted Certificates**
2. Click **Import**

![Trusted Certificates](images/ise-trusted-cert.png)

3. Browse and select `rootca.cer`
4. Check the following boxes:
   - ✅ Trust for authentication within ISE
   - ✅ Trust for client authentication and Syslog
   - ✅ Trust for authentication of Cisco Services
5. Click **Submit**

![Import Root CA](images/ise-import-rootca.png)

6. Verify the Root CA appears with the correct expiry date

![CA Imported Successfully](images/ise-ca-imported.png)

---

### Step 9.3 — Generate CSR (Certificate Signing Request) on ISE-01

1. Navigate to **Administration → System → Certificates → Certificate Signing Requests**
2. Click **Generate Certificate Signing Requests (CSR)**

![Generate CSR](images/ise-generate-csr.png)

3. Fill in the CSR parameters:

| Field | Value |
|---|---|
| Certificate(s) will be used for | Multi-use (wildcard) |
| Common Name (CN) | `*.hydent.com` |
| Organizational Unit | Network Team |
| Organization | Your Company Name |
| City | Hyderabad |
| State | Telangana |
| Country | IN |
| Subject Alternative Name | `ise-01.hydent.com` |

> 💡 Using a **wildcard certificate** (`*.hydent.com`) allows the same certificate to cover both ISE nodes and portal FQDNs — saves time vs generating individual certs per node.

![CSR Parameters](images/ise-csr-params.png)

4. Click **Generate** → Click **Export** to download the CSR file

![Export CSR](images/ise-export-csr.png)

---

### Step 9.4 — Sign the CSR with Windows CA

1. Navigate to: `http://10.10.10.10/certsrv`
2. Click **"Request a certificate"**

![Request Certificate](images/ca-request-cert.png)

3. Click **"Advanced certificate request"**

![Advanced Certificate Request](images/ca-advanced-request.png)

4. Open the downloaded CSR file → Copy its entire content → Paste into the **"Saved Request"** field → Click **Submit**

![Submit CSR](images/ca-submit-csr.png)

5. Download the signed certificate (**Base 64 encoded**)

> 💡 The CA signs the CSR and returns a signed `.cer` certificate — this is what gets bound to ISE

---

### Step 9.5 — Bind the Signed Certificate to ISE-01

1. Go back to ISE → **Administration → System → Certificates → Certificate Signing Requests**
2. Select the CSR entry → Click **Bind Certificate**

![Bind Certificate](images/ise-bind-cert.png)

3. Browse and upload the signed `.cer` file
4. Provide a **friendly name** (e.g., `ISE-01-Wildcard-2034`)
5. Check services to enable:
   - ✅ Admin
   - ✅ EAP Authentication
   - ✅ RADIUS DTLS
   - ✅ Portal
6. Click **Submit**

> ⚠️ **ISE services will restart** after binding. This takes 3–5 minutes. Normal behaviour.

![Certificate Bound](images/ise-cert-bound.png)

7. After restart, verify ISE-01 GUI loads with the new signed certificate — browser should show **no certificate warning**

![ISE New Certificate](images/ise-new-cert.png)

---

## 10. Phase 5 — Secondary Node (ISE-02) Registration

### Step 10.1 — Import Root CA into ISE-02

> ⚠️ **This step is critical.** ISE-02 must trust the same Root CA **before** registration, otherwise the certificate handshake during registration will fail.

1. Log in to **ISE-02** GUI: `https://ise-02.hydent.com`
2. Navigate to **Administration → System → Certificates → Trusted Certificates**
3. Click **Import** → Upload `rootca.cer`
4. Check all trust boxes → Click **Submit**

![ISE-02 Import Root CA](images/ise02-import-rootca.png)

---

### Step 10.2 — Register ISE-02 from ISE-01

> All registration is done **from the Primary PAN (ISE-01)**

1. On **ISE-01** → Navigate to **Administration → Deployment**
2. Click **Register**

![Register Node](images/ise-register.png)

3. Enter ISE-02 details:

| Field | Value |
|---|---|
| Hostname / IP | `ise-02.hydent.com` |
| Username | admin |
| Password | `<ISE-02 admin password>` |

4. Click **Next**

![Register ISE-02](images/ise-register-ise02.png)

5. A popup will appear asking to **import the certificate** — Click **"Import Certificate and Proceed"**

![Import Cert Popup](images/ise-cert-popup.png)

---

### Step 10.3 — Assign Roles to ISE-02

After clicking Next, assign the personas for ISE-02:

| Role | Setting |
|---|---|
| Administration | Secondary |
| Policy Service | ✅ Enabled |
| Monitoring | Primary |

> 💡 **Why Primary MnT on ISE-02?** This gives you MnT redundancy — if ISE-01 (Secondary MnT) has an issue, ISE-02 (Primary MnT) continues collecting RADIUS logs without interruption.

![ISE-02 Role Assignment](images/ise02-roles.png)

6. Click **Submit**

---

### Step 10.4 — Monitor Synchronization

After submitting, ISE-02 will show **"Registering"** → then **"Syncing"** status. This is normal and takes 5–15 minutes depending on existing policy data.

![Syncing Status](images/ise-syncing.png)

> ⚠️ Do **not** make any policy changes during sync. Wait until both nodes show **"Connected"**

---

### Step 10.5 — Registration Complete

Once sync completes, both nodes will show **"Connected"** with their respective roles.

**ISE-01 Final State:**

![ISE-01 Final](images/ise01-final.png)

**ISE-02 Final State:**

![ISE-02 Final](images/ise02-final.png)

---

## 11. Phase 6 — Verification

### GUI Verification

```
Administration → Deployment
```

Both nodes should show:
- ✅ Status: **Connected**
- ✅ Sync Status: **Completed**
- ✅ Correct roles assigned

### CLI Verification on ISE Nodes

```bash
# Check ISE application status
show application status ise

# Check ISE version
show version

# Check NTP sync
show ntp

# Check interface and IP
show interface GigabitEthernet 0
show ip route

# Verify DNS resolution
nslookup ise-01.hydent.com
nslookup ise-02.hydent.com

# Check ISE processes
show application status ise | include running
```

### Expected Output — `show application status ise`

```
ISE PROCESS NAME                       STATE            PID
-----------------------------------------------------------
Database Listener                      running          1234
Database Server                        running          1235
Application Server                     running          1236
Profiler Database                      running          1237
ISE Indexing Engine                    running          1238
AD Connector                           running          1239
M&T Session Database                   running          1240
M&T Log Collector                      running          1241
```

### Verify Replication Health from GUI

```
Administration → Deployment → Select Node → Sync Status
```

Should show: **"Node is in sync with Primary"**

---

## 12. Important Notes & Lessons Learned

| # | Issue | Root Cause | Fix |
|---|---|---|---|
| 1 | Registration fails immediately | DNS not resolving ISE-02 FQDN from ISE-01 | Add A and PTR records on DNS server |
| 2 | Certificate error during registration | Root CA not imported on ISE-02 before registration | Always import Root CA on secondary **before** registering |
| 3 | Services don't restart after cert bind | ISE in degraded state | Manually restart: `application stop ise` then `application start ise` |
| 4 | Sync stuck at "Registering" | Clock skew between nodes | Ensure NTP is in sync on both nodes — max 1 min drift allowed |
| 5 | CSR wildcard not accepted | Wrong template used on CA | Use **Web Server** template on Windows CA for ISE CSR signing |
| 6 | GUI unreachable after cert bind | Wrong services checked during bind | Re-bind cert and ensure **Admin** checkbox is ticked |

---

## 13. Two-Node Limitations — PAN Failover

> This is an important real-world consideration raised by the community.

In a two-node ISE deployment, **PAN Auto Failover is NOT available.** This means:

- If **ISE-01 (Primary PAN)** goes down, administrators **must manually promote** ISE-02 to Primary PAN
- There is no automatic sub-minute failover like in a 3+ node deployment
- **PSN failover** still works automatically via NAD configuration (multiple RADIUS servers)
- Only **PAN administration** is affected — endpoint authentication continues via PSN on ISE-02

### Manual Promotion Steps (when Primary PAN is down)

```
1. Log in to ISE-02 GUI: https://ise-02.hydent.com
2. Administration → Deployment → Click ISE-02 → Edit
3. Click "Promote to Primary"
4. Confirm the action
5. ISE-02 is now Primary PAN
```

### To Enable PAN Auto Failover — Add a Third Node

```
Administration → System → Deployment → PAN Failover
- Requires minimum 3 nodes (Primary PAN + Secondary PAN + Health Check Node)
- Set failover interval and polling thresholds
```

> 💡 For **production deployments with strict HA requirements**, always plan for a minimum 3-node deployment to enable PAN Auto Failover.

---

## References

- [Cisco ISE 3.4 Installation Guide](https://www.cisco.com/c/en/us/td/docs/security/ise/3-4/install_guide/b_ise_InstallationGuide34.html)
- [Cisco ISE Certificate Management](https://www.cisco.com/c/en/us/td/docs/security/ise/3-4/admin_guide/b_ISE_admin_3_4/b_ISE_admin_34_certificates.html)
- [ISE Two-Node Deployment Best Practices](https://community.cisco.com/t5/security-knowledge-base/ise-deployment-scenarios/ta-p/3641277)
- [Windows Server CA Configuration](https://docs.microsoft.com/en-us/windows-server/networking/core-network-guide/cncg/server-certs/install-the-certification-authority)

---

## Author

**Anurag Mishra** — Network Implementation Engineer | 6 Years Experience  
📍 Hyderabad, India  
🔧 Cisco ISE | FTD/FMC | Catalyst 9K | Aruba CX | FortiGate | Python & Ansible  
🔗 [LinkedIn](https://www.linkedin.com/in/anuragmishra6)

---

> **Disclaimer:** All IP addresses and domain names used in this document are from a lab environment. Sanitize all values before using in production. Screenshots are from a controlled lab deployment.
