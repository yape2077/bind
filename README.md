# Technical Implementation Guide: BIND9 Authoritative DNS Server

**Target Domain:** `example.com` (and `www.example.com`, `ns1.example.com`)  
**Server IP:** `192.168.1.1/24`  
**Subnet:** `192.168.1.0/24`  
**Reverse Lookup Zone:** `1.168.192.in-addr.arpa`  
**Integration:** Pairs with Nginx SSL Virtual Host on `192.168.1.1`  
**Status:** Validated & Production Ready  

---

## 1. Architecture & DNS Topology

The BIND9 service provides internal authoritative name resolution for the local domain `example.com` while serving as a recursive resolver with upstream public forwarders for general internet queries.

```
       [ Client Queries (Windows 10 / VM Clients) ]
                            │
                            │ Port 53 (UDP/TCP)
                            ▼
        [ BIND9 DNS Service (192.168.1.1) ]
       ┌────────────────────┴────────────────────┐
       ▼                                         ▼
[ Authoritative Zones ]              [ Recursive Forwarding ]
 ├── example.com (Forward)            ├── Upstream 1: 8.8.8.8
 └── 1.168.192.in-addr.arpa (Reverse) └── Upstream 2: 1.1.1.1
       │
       ▼
 [ Returns: 192.168.1.1 ]
       │
       ▼
[ Routes Client to Nginx (Port 80/443) ]
```

---

## 2. Environment Specifications

| Parameter | Value |
| :--- | :--- |
| **DNS Server Software** | BIND 9 (`named`) |
| **Server Hostname** | `ns1.example.com` |
| **Server IPv4** | `192.168.1.1/24` |
| **Local Network Subnet** | `192.168.1.0/24` |
| **Forward Zone** | `example.com` |
| **Reverse Zone** | `1.168.192.in-addr.arpa` |
| **Upstream DNS** | `8.8.8.8`, `1.1.1.1` |
| **Protocols & Ports** | Port 53 UDP / Port 53 TCP |

---

## 3. Step-by-Step Installation & Configuration

### Step 3.1: Install BIND9 and DNS Utilities

Execute system updates and install the BIND9 suite along with diagnostic tools:

```bash
sudo apt update
sudo apt install -y bind9 bind9utils bind9-doc dnsutils
```

Verify service registration:

```bash
systemctl is-enabled named
# Expected: enabled
```

---

### Step 3.2: Configure Global Options & Access Control (`named.conf.options`)

Define the Access Control List (ACL) to restrict recursion to trusted local clients, specify listening interfaces, and add public DNS forwarders.

Edit the options configuration file:

```bash
sudo nano /etc/bind/named.conf.options
```

Replace the entire content with the following block:

```text
acl "trusted" {
    127.0.0.1;
    192.168.1.0/24;
    localhost;
    localnets;
};

options {
    directory "/var/cache/bind";

    // Bind strictly to IPv4 local addresses
    listen-on port 53 { 127.0.0.1; 192.168.1.1; };
    listen-on-v6 { none; };

    // Access control policies
    allow-query { trusted; };
    allow-recursion { trusted; };
    allow-query-cache { trusted; };

    // Upstream DNS Forwarders (Fallback for non-local domains)
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };

    dnssec-validation auto;
    auth-nxdomain no;
};
```

---

### Step 3.3: Declare Forward and Reverse Zones (`named.conf.local`)

Open `/etc/bind/named.conf.local` to define zone declarations:

```bash
sudo nano /etc/bind/named.conf.local
```

Add the following zone stanzas:

```text
// Forward Lookup Zone
zone "example.com" {
    type master;
    file "/etc/bind/zones/db.example.com";
};

// Reverse Lookup Zone (192.168.1.0/24)
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

---

### Step 3.4: Create the Zone Database Files

1. Create a dedicated storage directory for custom zones:

   ```bash
   sudo mkdir -p /etc/bind/zones
   ```

2. **Create the Forward Zone File (`db.example.com`):**

   ```bash
   sudo nano /etc/bind/zones/db.example.com
   ```

   Paste the following record set:

   ```text
   $TTL    604800
   @       IN      SOA     ns1.example.com. admin.example.com. (
                                 2026102301     ; Serial (YYYYMMDDNN)
                                     604800     ; Refresh (7 days)
                                      86400     ; Retry (1 day)
                                    2419200     ; Expire (28 days)
                                     604800 )   ; Negative Cache TTL (7 days)
   ;
   ; Name Servers
   @       IN      NS      ns1.example.com.

   ; Name Server A Record
   ns1     IN      A       192.168.1.1

   ; Host Records
   @       IN      A       192.168.1.1
   www     IN      A       192.168.1.1
   ```

3. **Create the Reverse Zone File (`db.192.168.1`):**

   ```bash
   sudo nano /etc/bind/zones/db.192.168.1
   ```

   Paste the PTR mapping records:

   ```text
   $TTL    604800
   @       IN      SOA     ns1.example.com. admin.example.com. (
                                 2026102301     ; Serial (YYYYMMDDNN)
                                     604800     ; Refresh (7 days)
                                      86400     ; Retry (1 day)
                                    2419200     ; Expire (28 days)
                                     604800 )   ; Negative Cache TTL (7 days)
   ;
   ; Name Servers
   @       IN      NS      ns1.example.com.

   ; PTR Pointer Records (192.168.1.1 -> Hostnames)
   1       IN      PTR     example.com.
   1       IN      PTR     ns1.example.com.
   1       IN      PTR     www.example.com.
   ```

---

### Step 3.5: Syntax Validation, Firewall, and Service Restart

1. **Verify configuration syntax:**

   ```bash
   sudo named-checkconf
   ```
   *(If clean, returns 0 with no error output).*

2. **Verify forward and reverse zone integrity:**

   ```bash
   sudo named-checkzone example.com /etc/bind/zones/db.example.com
   sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1
   ```
   *Expected Output for both:*
   ```text
   zone example.com/IN: loaded serial 2026102301
   OK
   ```

3. **Open firewall ports (Port 53 TCP and UDP):**

   ```bash
   sudo ufw allow 53/tcp comment 'Allow DNS TCP'
   sudo ufw allow 53/udp comment 'Allow DNS UDP'
   sudo ufw reload
   ```

4. **Restart and activate the BIND9 service:**

   ```bash
   sudo systemctl restart named
   sudo systemctl status named
   ```

---

## 4. Client-Side DNS Configuration (Windows 10 / 11)

To route client DNS queries through the newly deployed BIND9 server:

### Method A: Via GUI (Network Connections)
1. Press `Win + R`, type `ncpa.cpl`, and hit **Enter**.
2. Right-click the active network adapter (e.g., *Ethernet0*) > **Properties**.
3. Select **Internet Protocol Version 4 (TCP/IPv4)** > click **Properties**.
4. Check **Use the following DNS server addresses**:
   * **Preferred DNS server:** `192.168.1.1`
   * **Alternate DNS server:** Leave empty (forces local resolution).
5. Click **OK** > **Close**.

### Method B: Via PowerShell (Run as Administrator)
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses ("192.168.1.1")
Clear-DnsClientCache
```

> **Important:** Disable **Secure DNS (DNS-over-HTTPS)** inside Chrome/Edge/Firefox under `Settings -> Privacy and Security -> Security` so queries query your local BIND9 instance rather than external public resolvers.

---

## 5. Verification & Testing Matrix

Run these diagnostic commands to ensure complete forward, reverse, and recursive resolution:

| Test Type | Execution Command | Expected Output | Status |
| :--- | :--- | :--- | :--- |
| **Local Forward Lookup** | `dig @127.0.0.1 example.com +short` | `192.168.1.1` | Pass |
| **Subdomain Resolution** | `dig @192.168.1.1 www.example.com +short` | `192.168.1.1` | Pass |
| **Reverse PTR Lookup** | `dig @192.168.1.1 -x 192.168.1.1 +short` | `example.com.` | Pass |
| **Upstream Recursion** | `dig @192.168.1.1 google.com +short` | Valid public IP list | Pass |
| **Windows Client Test** | `nslookup example.com` | `Address: 192.168.1.1` | Pass |

---

## 6. Common Troubleshooting Scenarios

* **`connection timed out; no servers could be reached`:**
  * Ensure UFW allows UDP port 53 (`sudo ufw allow 53/udp`).
  * Verify `listen-on` in `/etc/bind/named.conf.options` includes `192.168.1.1;`.
* **`query (cache) 'example.com/A/IN' denied`:**
  * The client IP is outside the permitted ACL. Add the client subnet to the `acl "trusted"` block in `named.conf.options`.
* **Changes to zone file do not take effect:**
  * Increment the **Serial** number in the SOA record (e.g., from `2026102301` to `2026102302`) and reload: `sudo rndc reload`.