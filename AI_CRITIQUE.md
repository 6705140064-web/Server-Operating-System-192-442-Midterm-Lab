# AI Usage and Critique

**Course:** Server Operating System (192-442)  
**Lab group:** lab01  
**Submission:** Individual (confirm with instructor that solo submission is permitted)  
**Student:** [Add your name]

## 3a. AI-use declaration

- **ChatGPT (OpenAI):** Used to interpret checker output, suggest diagnostic commands, explain DNS/DHCP symptoms, and help draft documentation based on terminal output I supplied.
- **Other AI tools:** [List every other AI tool you actually used and how. If you used none, write “None”; do not claim to have tested another AI unless you really did.]

ChatGPT did not directly access my live KillerCoda environment. Its suggestions were based on the command output I provided, and I verified repairs by running commands in the lab and rerunning the checker.

## 3b. My diagnosis before AI comparison

I first ran the checker and received **11.0 / 25**. I inspected the DNS zone configuration, zone files, listening sockets, firewall rules, and DHCP DNS option. After making changes, I reran the checker and received **25.0 / 25**.

## 3c. AI comparison

The assignment requires at least two different AI tools. I used ChatGPT during troubleshooting. Complete the AI #2 column after giving the broken scenario to a second AI tool. Paste or accurately summarize its real response, and test each suggestion against the lab. Do not invent an answer or mark an untested suggestion correct.

| Fault | Did I find it? | ChatGPT | AI #2 | Was each AI right? How did I verify? |
|---|---|---|---|---|
| Forward zone referenced `/etc/bind/db.MISSING` | Yes | Suggested inspecting `named.conf.local` and correcting the zone-file path | [Enter the second AI's actual suggestion] | ChatGPT's diagnosis matched the file-not-found output. `named-checkconf -z` subsequently showed both company zones loaded. Evaluate AI #2 using the same evidence. |
| Firewall dropped UDP DNS port 53 | Yes | Suggested checking firewall rules after verifying BIND status and listening sockets | [Enter the second AI's actual suggestion] | The INPUT chain showed a UDP port 53 DROP rule. After removing it, the final checker confirmed client DNS reachability. Evaluate AI #2 using the same evidence. |
| `api` A record used `192.168.19.1` instead of `192.168.19.40` | Yes | Suggested correcting the A record to the expected IP | [Enter the second AI's actual suggestion] | The final checker reported all four A records correct. Evaluate AI #2 using the same evidence. |
| DHCP advertised `8.8.8.8` instead of `192.168.19.10` | Yes | Suggested changing the DHCP DNS option to the company resolver | [Enter the second AI's actual suggestion] | The final checker confirmed DHCP advertised the company DNS and the client resolved `www` through its leased resolver. Evaluate AI #2 using the same evidence. |

### Verification approach

I did not treat an AI answer as proof. I checked the live system with commands including `named-checkconf -z`, `ss -lntup`, `iptables -L INPUT -n -v`, and `grep`, then reran the provided checker. The final score of **25.0 / 25** is evidence that all tests passed after the repairs.

## 3d. Verdict

ChatGPT helped organize the troubleshooting process and explain how the observed symptoms related to possible causes. The command output was essential: the missing zone-file message identified the wrong path, the firewall listing showed a UDP port 53 DROP rule consistent with DNS timeouts, the zone file showed the incorrect `api` address, and the DHCP configuration showed the wrong advertised resolver.

The troubleshooting also showed why a service being “active” does not prove it works from the client. BIND was running and listening, but DNS requests timed out until the firewall issue was fixed. Likewise, a client can obtain a DHCP lease and still fail to resolve company names if DHCP advertises the wrong DNS server.

ChatGPT could not directly inspect the live lab, so its recommendations had to be checked against real command output and the checker. **Add a specific example from AI #2 here**, including what it recommended, whether the evidence supported it, and what the checker showed. My conclusion is that AI is useful for generating hypotheses and organizing a diagnostic plan, but the administrator must verify every proposed fix.
