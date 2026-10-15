# Incident Log — NovaPeak Network Services

**Course:** Server Operating System (192-442)  
**Lab group:** lab01  
**Submission:** Individual (confirm with instructor that solo submission is permitted)  
**Domain:** `novapeak.lab`  
**DNS server:** `192.168.19.10`  
**Final checker result:** 25.0 / 25.0

> This log is based on terminal output shared during troubleshooting. Before submitting, compare it with your actual `/root/session.log`, add any missing real outputs, and ensure the session file contains the work you performed.

## Fault 1 — Forward DNS zone referenced a missing file

### Symptom
`named-checkconf -z` reported that the forward zone could not load.

### Proof
Command:
```bash
named-checkconf -z
```

Observed output:
```text
zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
zone novapeak.lab/IN: not loaded due to errors.
/_default/novapeak.lab/IN: file not found
zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101
```

The file listing showed `/etc/bind/db.novapeak.lab` existed. The `novapeak.lab` zone in `/etc/bind/named.conf.local` referenced `/etc/bind/db.MISSING`.

### Hypothesis
The forward zone referenced an incorrect master-file path.

### Fix
Changed the forward zone's file path in `/etc/bind/named.conf.local` from:
```text
file "/etc/bind/db.MISSING";
```
to:
```text
file "/etc/bind/db.novapeak.lab";
```

### Confirmation
After the change, `named-checkconf -z` reported:
```text
zone novapeak.lab/IN: loaded serial 2024010101
zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101
zone localhost/IN: loaded serial 2
zone 127.in-addr.arpa/IN: loaded serial 1
zone 0.in-addr.arpa/IN: loaded serial 1
zone 255.in-addr.arpa/IN: loaded serial 1
```

## Fault 2 — Firewall dropped UDP DNS queries

### Symptom
DNS queries timed out even though BIND was running and listening on port 53.

### Proof
Commands:
```bash
ss -lntup | grep ':53'
iptables -L INPUT -n -v
```

The socket listing showed BIND listening on `127.0.0.1:53` and `192.168.19.10:53`. The INPUT chain showed:
```text
Chain INPUT (policy ACCEPT ...)
21 1788 DROP udp -- * * 0.0.0.0/0 0.0.0.0/0 udp dpt:53
```

The query returned:
```text
;; communications error to 127.0.0.1#53: timed out
;; no servers could be reached
```

### Hypothesis
The INPUT rule dropping UDP destination port 53 was preventing DNS requests from being answered.

### Fix
Removed the matching firewall rule:
```bash
iptables -D INPUT -p udp --dport 53 -j DROP
```

### Confirmation
The final checker reported:
```text
[   2/2 ] C1   named is authoritative for novapeak.lab and reachable from the client.
```

> If available, add the post-fix firewall listing and a successful `dig` output from your actual session log.

## Fault 3 — Incorrect A record for `api`

### Symptom
The forward zone contained an incorrect IP address for `api`.

### Proof
Command:
```bash
cat /etc/bind/db.novapeak.lab
```

Observed record before correction:
```text
api IN A 192.168.19.1
```

The checker expected `api.novapeak.lab` to resolve to `192.168.19.40`.

### Hypothesis
The `api` A record pointed to the wrong IP address.

### Fix
Changed the record to:
```text
api IN A 192.168.19.40
```

### Confirmation
The final checker reported:
```text
[ 3.0/3 ] C2   all four A records correct
```

## Fault 4 — DHCP advertised the wrong DNS resolver

### Symptom
The client obtained a lease but was given public DNS `8.8.8.8`, rather than the company's DNS server.

### Proof
Command:
```bash
grep -n 'domain-name-servers' /etc/dhcp/dhcpd.conf
```

Observed output before correction:
```text
8:      option domain-name-servers 8.8.8.8;
```

### Hypothesis
The DHCP DNS option was misconfigured, so the client would not use the company's resolver for company hostnames.

### Fix
Changed the DHCP option to:
```text
option domain-name-servers 192.168.19.10;
```

Then validated and restarted DHCP:
```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
systemctl restart isc-dhcp-server
```

### Confirmation
The final checker reported:
```text
[   3/3 ] C9   DHCP advertises the company DNS (192.168.19.10)
[   3/3 ] C11  client resolves www via the leased resolver
```

## Final verification

Command:
```bash
python3 /root/grading/check_lab.py --group lab01 --phase build
```

Observed final output:
```text
[   2/2 ] C4   named-checkconf -z clean; named-checkzone clean
[   2/2 ] C5   dhcpd -t clean; service running
[   2/2 ] C1   named is authoritative for novapeak.lab and reachable from the client.
[ 3.0/3 ] C2   all four A records correct
[   2/2 ] C3   reverse lookup of 192.168.19.10 returns ns1.
[   3/3 ] C6   client obtained a lease: 192.168.19.104
[ 2.0/2 ] C7   pool+netmask correct
[   2/2 ] C8   gateway advertised correctly
[   3/3 ] C9   DHCP advertises the company DNS (192.168.19.10)
[   1/1 ] C10  company domain advertised
[   3/3 ] C11  client resolves www via the leased resolver
--------------------------------------------------
TOTAL: 25.0 / 25
```

All checker tests passed after the repairs.
