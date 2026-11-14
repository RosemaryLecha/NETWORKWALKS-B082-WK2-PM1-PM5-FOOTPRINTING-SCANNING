# PENETRATION TESTING REPORT: Footprinting & Network Scanning

**W2-PM-FINAL | CYBERSECURITY | NETWORKWALKS**

| Field | Detail |
|---|---|
| Pentester Name (Cybersecurity Professional) | Rose Mary Lecha Ngwa Mbenoh |
| Program/Batch | B082-Networkwalks |
| Date | 14 November 2026 |
| Modules completed | W2-PM1 (Multiple Kali Tools), W2-PM5 (Zenmap Scanning) |
| Client/Target | 1. Networkwalks (networkwalks.com, public website and DNS records) <br> 2. My own virtual lab network (Kali VM) |
| Permission secured from client? | Yes |
| Phases covered | Phase 1: Reconnaissance & Footprinting <br> Phase 2: Scanning & Network Discovery <br> Phase 3-5: In Progress |

> The full report is available as a PDF in this repository: [W2-PM-FINAL_Pentest_Report.pdf](W2-PM-FINAL_Pentest_Report.pdf)

---

## 1. Liability Disclaimer

I have performed these activities only on the systems and devices where I had secured written permission or the devices/systems that I own myself. All these materials are for education and research purposes only. Do not use anything from here to break the law. The instructor, the authors and Networkwalks are not responsible for what you do with this knowledge. Every action you take is your own responsibility. Misuse can lead to criminal charges, heavy fines, loss of your job and a permanent record. In most countries unauthorised access is a crime even when nothing is damaged.

---

## 2. Introduction

This project covers footprinting the networkwalks.com domain with multiple Kali Linux tools (W2-PM1) and scanning my own lab network with Zenmap (W2-PM5). One module is the footprinting phase and the other is the scanning phase, so together they show how an attacker moves from gathering public information to mapping live hosts on a network.

Everything was done in a Kali Linux virtual machine. For each step below I list the command, what I observed, a screenshot as evidence, and why the finding matters from an attacker's point of view. Testing stayed inside the authorized scope: the networkwalks.com public website and DNS records, and my own lab network. No exploitation, brute-forcing or denial-of-service activity was performed.

---

## 3. Tools Used

| Tool | Purpose |
|---|---|
| Kali Linux (VM) | Operating system used for all activities |
| WHOIS | Domain registration details (registrar, dates, name servers) |
| whatweb | Fingerprint web technologies (server, CMS, plugins, IP) |
| nslookup | Resolve the domain name to its IP address |
| curl -I | Read the HTTP response headers |
| wafw00f | Detect a Web Application Firewall |
| dnsrecon | Enumerate DNS records (NS, MX, SPF, TXT, SRV) |
| Zenmap (Nmap GUI) | Find live hosts, IPs and MAC addresses on the local subnet |
| ip a | Identify the VM's IP address, subnet and MAC address |

---

## 4. Activities Performed

### 4.1 Footprinting & Reconnaissance (W2-PM1)

Setup: I created a working folder and checked that the target was reachable. Each tool's output was saved to a text file with `tee`.

```bash
mkdir ~/W2-PM1 && cd ~/W2-PM1
ping -c 2 networkwalks.com
```

![Working folder and ping](screenshots/01_ping_target.png)

#### Step 1: WHOIS

```bash
whois networkwalks.com | tee 1_whois.txt
```

**Observed:** The domain is registered with GoDaddy.com, LLC. It was created on 2019-11-06 and expires on 2027-11-06. Name servers are NS6135/NS6136.HOSTGATOR.COM, and the registrar record also lists NS29/NS30.DOMAINCONTROL.COM. The domain has four client lock statuses (transfer, update, renew, delete prohibited). The registrant is hidden behind a privacy service (Domains By Proxy, LLC), and DNSSEC is unsigned.

**Why it matters:** It shows the registrar, hosting and DNS providers and the age of the domain. The privacy service keeps the owner's details hidden.

![WHOIS registry record](screenshots/02_whois_registry.png)
![WHOIS registrar record](screenshots/03_whois_registrar.png)

#### Step 2: WhatWeb

```bash
whatweb networkwalks.com | tee 2_whatweb.txt
```

**Observed:** HTTP redirects to HTTPS (301, then 200 OK). The site runs on Apache with WordPress 7.1.4 and the WordPress Download Manager plugin 3.3.76, plus jQuery 3.7.1, Bootstrap and Google Tag Manager. It also exposed a contact email address and an HttpOnly cookie named `__wpdm_client`.

**Why it matters:** Exact CMS and plugin versions can be matched against known issues.

*Note: the first run returned a temporary "no address" DNS error. Running the command again worked.*

![WhatWeb output](screenshots/04_whatweb.png)

#### Step 3: Nslookup

```bash
nslookup networkwalks.com | tee 3_nslookup.txt
```

**Observed:** The domain resolves to `192.232.216.135` (non-authoritative answer from resolver 8.8.8.8).

**Why it matters:** It gives the real server IP, the starting point for network-level scanning in an authorized test.

#### Step 4: Curl

```bash
curl -I https://networkwalks.com | tee 4_curl.txt
```

**Observed:** `HTTP/1.1 200 OK` from Apache. The headers show the WordPress REST API entry point (`/wp-json/`), caching headers (`X-nginx-cache`, `X-Endurance-Cache-Level`), a Permissions-Policy and Referrer-Policy, and a cookie set with the Secure and HttpOnly flags.

**Why it matters:** Headers confirm the technology stack and point to further endpoints to review.

![Nslookup and curl output](screenshots/05_nslookup_curl.png)

#### Step 5: Wafw00f

```bash
wafw00f networkwalks.com | tee 5_wafw00f.txt
```

**Observed:** The site is behind a **ModSecurity (SpiderLabs)** Web Application Firewall.

**Why it matters:** A WAF helps block basic attacks, but being able to fingerprint it reveals part of the security architecture.

![Wafw00f output](screenshots/06_wafw00f.png)

#### Step 6: DNSRecon

```bash
dnsrecon -d networkwalks.com | tee 6_dnsrecon.txt
```

**Observed:**
- Name servers `ns6135.hostgator.com` and `ns6136.hostgator.com`, both reporting BIND version 9.16.23-RH
- Mail server `mail.networkwalks.com` on the same IP as the website
- An SPF record ending in `~all` (soft fail)
- A Google site-verification TXT record
- 8 cPanel autodiscover SRV records
- No answer to the DNSSEC query, which matches the unsigned DNSSEC seen in WHOIS

**Why it matters:** It maps the DNS and mail footprint and discloses the DNS software version.

![DNSRecon output](screenshots/07_dnsrecon.png)

---

### 4.2 Network Scanning with Zenmap (W2-PM5)

I identified my VM's address with `ip a`: interface `eth0` has `10.0.0.2/24`, so the subnet to scan was `10.0.0.0/24`. I started Zenmap with `sudo zenmap` (root is needed to read MAC addresses), entered the subnet as the target, and chose the **Ping scan** profile, which runs:

```bash
nmap -sn 10.0.0.0/24
```

The scan checked 256 addresses and found **3 live hosts** in 7.02 seconds.

| Host IP | MAC Address | Vendor / Notes |
|---|---|---|
| 10.0.0.1 | 52:54:00:12:35:00 | QEMU virtual NIC (appears to be the virtual gateway) |
| 10.0.0.2 | 08:00:27:5a:87:bc | My Kali VM (MAC taken from `ip a`; Nmap does not list a MAC for the machine it runs on) |
| 10.0.0.3 | 08:00:27:E2:54:2F | Oracle VirtualBox virtual NIC (another VM) |

The MAC vendors show that all three devices are virtual, so this is a virtual lab network.

I then opened the **Topology** tab, enabled the legend and saved the graphic as a PDF (see `zenmap_topology.pdf`).

![Zenmap ping scan results](screenshots/08_zenmap_ping_scan.png)
![Zenmap window](screenshots/09_zenmap_window.png)
![Zenmap topology legend](screenshots/10_zenmap_topology_legend.png)

---

## 5. Troubleshooting: dnsrecon crashed on Kali

**Problem:** `dnsrecon -d networkwalks.com` failed immediately with:

```
AttributeError: module 'urllib.request' has no attribute 'FancyURLopener'
```

**Cause:** A known incompatibility. The installed dnsrecon (1.3.1) uses a function that newer Python versions removed. The crash happened before any scan traffic was sent.

**Solution:** I upgraded the package, then re-ran the same command.

```bash
sudo apt install --only-upgrade dnsrecon
dnsrecon -d networkwalks.com | tee 6_dnsrecon.txt
```

This upgraded dnsrecon to 1.6.0 and the scan ran successfully.

---

## 6. Risk Analysis / Impact

| # | Risk / Finding | Evidence / Observation | Potential Impact | Risk Level |
|---|---|---|---|---|
| 1 | Web technology and version information exposed | WhatWeb identified WordPress 7.1.4 and WP Download Manager 3.3.76 | Exact versions can be matched against known issues | Medium |
| 2 | DNS software version disclosed | DNSRecon showed BIND 9.16.23-RH on both name servers | Helps attackers research the DNS software | Medium |
| 3 | Server IP address identifiable | Nslookup resolved the domain; web and mail share one IP | Shows the network location of web and mail services | Low |
| 4 | HTTP technical information exposed | Curl headers exposed `/wp-json/` and caching details | May assist fingerprinting and enumeration | Low |
| 5 | WAF technology identifiable | Wafw00f identified ModSecurity (SpiderLabs) | Reveals part of the security architecture | Low |
| 6 | Email and DNS configuration | SPF ends in `~all` (soft fail); DNSSEC unsigned | Spoofed email may be harder to reject; DNS answers are not signed | Low |
| 7 | Contact email exposed | WhatWeb found a contact email address | Could be used for phishing or spam | Low |
| 8 | Multiple live hosts on the local network | Zenmap found 3 live hosts | Unknown devices on a network should be identified and verified | Low |

These are observations, not confirmed vulnerabilities. Only information gathering and host discovery were performed. No exploitation or vulnerability validation was done, so further authorized testing would be needed to confirm any actual vulnerability.

**Positive observations:** the domain is locked; WHOIS privacy protection is on; a ModSecurity WAF is active; HTTP redirects to HTTPS; and the session cookie uses the Secure and HttpOnly flags.

---

## 7. Recommendations

1. **Review publicly exposed technology information.** Reduce version disclosure where possible.
2. **Keep software updated.** Update WordPress, plugins and server software and check current security advisories.
3. **Review HTTP headers and the REST API.** Remove unnecessary technical details and restrict the REST API where it is not needed.
4. **Hide DNS software versions.** Configure DNS servers so they do not reveal their BIND version.
5. **Strengthen DNS and email settings.** Consider enabling DNSSEC and a stricter SPF policy (`-all`) once all legitimate mail sources are confirmed. Enable auto-renewal for the domain.
6. **Keep the WAF enabled and tuned.**
7. **Scan your own networks regularly** to know which devices are active.
8. **Investigate unknown devices** found during scanning.
9. **Maintain network documentation** (topology and device lists).
10. **Test only with authorization.** Reconnaissance and scanning should only target systems where permission has been given.

---

## 8. Conclusion

In Week 2 of my Cybersecurity & Ethical Hacking internship I completed practical work on footprinting, reconnaissance and network scanning. With six Kali tools I collected information about networkwalks.com: WHOIS for registration details, WhatWeb for web technologies, Nslookup for the IP, Curl for HTTP headers, Wafw00f for the firewall, and DNSRecon for DNS, mail and service records. I also troubleshot a dnsrecon crash caused by a known compatibility bug and fixed it by upgrading the package.

With Zenmap I identified my network configuration, found 3 live hosts on my lab network, collected their IP and MAC addresses, and created a network topology.

The main lessons: a lot can be learned about an environment before any exploitation, findings must be documented clearly (what was done, found, what it means, the risk, and the fix), and scanning must always stay within an authorized scope.

---

## Evidence

- Screenshots: [`screenshots/`](screenshots/)
- Full PDF report: [`W2-PM-FINAL_Pentest_Report.pdf`](W2-PM-FINAL_Pentest_Report.pdf)
- Zenmap topology graphic: `zenmap_topology.pdf` (add your saved file to this repo)

---

**Author:** Rose Mary Lecha Ngwa Mbenoh, Cybersecurity Professional B082
**Program:** Cybersecurity program at Networkwalks | Week 02 | Repository: GitHub
