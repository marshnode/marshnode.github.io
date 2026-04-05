# GL-iNet Flint 2 Router Hardening & Troubleshooting

**Device:** GL-iNet Flint 2 (GL-MT6000)  
**Firmware:** OpenWrt  
**Category:** Network Security | Home Lab | Firewall Configuration  
**Status:** Completed

---

## Overview

This project documents the security hardening of a GL-iNet Flint 2 router running OpenWrt firmware. The goal was to apply security best practices to a home network device — restricting admin access, encrypting DNS traffic, and implementing custom firewall rules to reduce the attack surface.

The project also includes a real-world troubleshooting case study where a misconfigured firewall rule caused a full DHCP failure across the network, requiring systematic diagnosis and recovery.

---

## Objectives

- Restrict router admin panel to HTTPS only
- Encrypt all DNS queries using DNS over HTTPS (DoH)
- Enable DNS rebinding protection to prevent DNS-based attacks
- Implement custom firewall rules to control inbound and outbound traffic
- Document and recover from a self-caused network misconfiguration

---

## Environment

| Component | Details |
|-----------|---------|
| Device | GL-iNet Flint 2 (GL-MT6000) |
| Firmware | OpenWrt (latest stable) |
| Admin Access | HTTPS via 192.168.8.1 |
| DNS Provider | Cloudflare (1.1.1.1 / 1.0.0.1) |
| Network | Home LAN with multiple connected devices |

---

## Hardening Steps

### 1. Enforcing HTTPS-Only Admin Access

By default, the GL-iNet admin panel is accessible over both HTTP and HTTPS. Allowing unencrypted HTTP access means admin credentials could be intercepted on the local network.

**What I did:**
- Logged into the admin panel at `192.168.8.1`
- Navigated to **System → Administration**
- Disabled HTTP access and enforced HTTPS only
- Verified that navigating to `http://192.168.8.1` no longer loaded the panel and redirected to the HTTPS version

**Why it matters:** Even on a home network, enforcing HTTPS ensures admin credentials are never transmitted in plaintext. This is especially important if the network has multiple users or guest devices.

---

### 2. Configuring Cloudflare DNS over HTTPS (DoH)

Standard DNS queries are sent in plaintext over UDP port 53, meaning your ISP or anyone on the network can see every domain you look up. DNS over HTTPS encrypts these queries inside HTTPS traffic.

**What I did:**
- Navigated to **Network → DNS** in the OpenWrt admin panel
- Disabled the default ISP-assigned DNS servers
- Configured Cloudflare's DoH endpoints:
  - Primary: `https://1.1.1.1/dns-query`
  - Secondary: `https://1.0.0.1/dns-query`
- Enabled the DoH client within OpenWrt
- Verified DNS resolution was functioning correctly after changes

**Why it matters:** Encrypting DNS traffic prevents ISP-level DNS snooping and protects against DNS hijacking attacks where malicious actors redirect traffic by intercepting DNS queries.

---

### 3. Enabling DNS Rebinding Protection

DNS rebinding is an attack technique where a malicious website causes a victim's browser to make requests to internal network resources (like a router admin panel) by manipulating DNS responses.

**What I did:**
- Located the DNS rebinding protection setting under **Network → DNS**
- Enabled the rebinding protection option in OpenWrt
- Configured the router to reject DNS responses that resolve public domain names to private IP ranges (RFC 1918 addresses: 10.x.x.x, 172.16.x.x, 192.168.x.x)

**Why it matters:** Without this protection, an attacker could craft a malicious website that tricks the browser into attacking devices on the local network — including the router itself.

---

### 4. Custom Firewall Rules

After completing the above hardening steps, I implemented additional firewall rules through the OpenWrt firewall configuration to further restrict unnecessary traffic and tighten the network perimeter.

**Rules implemented:**
- Blocked unsolicited inbound traffic from WAN to LAN
- Restricted admin panel access to specific trusted devices on the LAN
- Configured logging for dropped packets to assist with future monitoring
- Set default policies: WAN input = DROP, WAN forward = DROP, LAN = ACCEPT

---

## Incident Report: Self-Caused DHCP Failure

### What Happened

While implementing a custom firewall rule intended to restrict inbound traffic, I accidentally introduced a rule that blocked DHCP traffic on the LAN interface. Within minutes of applying the rule, connected devices began losing their IP addresses and were unable to obtain new leases from the router. The network went down completely.

### Impact

- All devices on the LAN lost internet connectivity
- New devices were unable to connect to the network
- Router admin panel was briefly inaccessible due to IP loss on the client machine

### Diagnosis Process

**Step 1 — Identify the symptom**  
Devices were showing "No IP address" or self-assigning 169.254.x.x APIPA addresses, which immediately indicated a DHCP failure rather than a routing or WAN issue.

**Step 2 — Isolate the cause**  
I connected a device directly to the router via ethernet and attempted to manually request an IP using a static assignment, which succeeded. This confirmed the router itself was functional but DHCP leasing was broken.

**Step 3 — Access the router**  
I assigned a static IP address manually on my laptop (192.168.8.x range) to regain access to the admin panel at 192.168.8.1 without needing a DHCP lease.

**Step 4 — Review firewall rules**  
Inside the OpenWrt firewall configuration, I reviewed the rules I had just applied. I identified a rule that was incorrectly blocking UDP port 67 and 68 — the ports used by DHCP for server and client communication respectively.

**Step 5 — Fix and verify**  
I removed the offending firewall rule, restarted the DHCP service, and verified that devices began obtaining IP addresses correctly. Full network connectivity was restored within minutes.

### Root Cause

The firewall rule was written too broadly. The intent was to block unsolicited inbound traffic from external sources, but the rule inadvertently applied to internal LAN traffic as well, blocking the DHCP broadcast packets that devices use to request IP addresses.

### Fix Applied

Rewrote the firewall rule with a narrower scope, explicitly specifying the WAN interface as the target rather than applying the rule globally. Added an explicit ACCEPT rule for DHCP traffic on the LAN interface to prevent similar issues in the future.

---

## Key Lessons Learned

1. **Always scope firewall rules precisely** — a rule that is too broad can have unintended consequences across multiple services and interfaces.

2. **Know your recovery path before you apply changes** — having a plan to access the router via static IP assignment saved significant time during recovery.

3. **DHCP runs on UDP 67/68** — blocking these ports on the wrong interface will silently kill all dynamic IP assignment on the network.

4. **Test in stages** — applying multiple firewall rules at once makes it harder to identify which rule caused a problem. Apply and verify one rule at a time.

5. **Document as you go** — the troubleshooting process here was faster because I knew exactly what changes I had made and in what order.

---

## Skills Demonstrated

- OpenWrt firewall configuration and management
- DNS over HTTPS (DoH) setup and verification
- DNS rebinding attack mitigation
- Network troubleshooting methodology (symptom → isolation → root cause → fix)
- DHCP protocol understanding (UDP 67/68, lease process)
- Static IP assignment for emergency router recovery
- Security hardening principles applied to consumer/prosumer networking hardware

---

## References

- [OpenWrt Firewall Documentation](https://openwrt.org/docs/guide-user/firewall/firewall_configuration)
- [Cloudflare DNS over HTTPS](https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/)
- [GL-iNet Flint 2 Documentation](https://docs.gl-inet.com/router/en/4/guides/gl-mt6000/)
- [DNS Rebinding Attack — OWASP](https://owasp.org/www-community/attacks/DNS_Rebinding)
