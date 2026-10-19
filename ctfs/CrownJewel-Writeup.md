# The Crown Jewel -- TryHackMe
 
**Category:** Network Forensics / Log Analysis
**Tools used:** Wireshark, grep, Pluma
**Difficulty:** Easy
 
## Summary
 
Incident response room with a pcap and a network_logs file covering an internal compromise. Attacker got in via ARP spoofing, ran a C2 beacon on a non standard port, grabbed plaintext creds off an internal login page, hit jira with a scripted exploit, and exfiltrated data over DNS. Main thing this room tests is switching between network_logs (searchable but incomplete) and the raw pcap (complete but you gotta dig manually) depending on what each one can actually show you.
 
## Process
 
Started in network_logs, filtered post requests around the flagged incident timestamp to find the origin. no matching entries at that timestamp, so whatever happened wasnt getting logged at all.
 
Went into wireshark and filtered the two nearest timestamps around the gap. found a tcp conversation with no matching entry in network_logs, confirmed that activity just wasnt visible to the logger. followed the stream, got 0 turns, no payload. checked the flags, it was just a bare connection attempt with no data exchanged. ruled that one out but kept the ip.
 
Searched for tcp across the network logs, found one instance (occuring at the time of the attack, meaning the logs were not sorted as I previously assumed). Used that IP across the whole pcap and got four instances of the same conversation, one flagged black and red ("bad tcp", turned out to just be a capture gap, not actually malicious). followed that stream and got one turn with 7 bytes of payload: id=beacon&ver=1 / 200 ok, coming off port 8080. c2 beacon check in, implant phoning home and server acking it.
 
From there just worked through the rest of the questions bouncing between the two sources depending what each one had:
 
- **arp spoofing mac:** network_logs had nothing on link layer stuff so this was all wireshark. filtered arp.opcode == 2 to only get replies, not requests (requests broadcast by design and are just noise). found gratuitous arp replies claiming to be the gateway (10.10.10.1), all coming from one mac, 00:0c:29:11:22:33, a vmware oui so obviously the attacker vm.
- **jira user agent:** network_logs had the field (agent=) but thousands of entries, no way to read through that manually. piped it through grep/sort/uniq -c to get a frequency count. every normal ua (curl, python-requests, browser strings, go-http-client) showed up hundreds of times. one value, CVE-202X-EXPLOIT, showed up exactly once. obvious outlier.
- **plaintext creds:** network_logs had http metadata but no request bodies, so grepping for password just matched random substrings in session tokens, false positive. had to go back to wireshark, filter http.request.method == "POST", follow the stream directly. found username=dev_user&password=SecretPassword! in the raw body, a field network_logs never captured in the first place.
- **exfil domain/protocol:** spotted this in a dns log line while scanning for something else, a query for a long random looking subdomain under exfil-domain.xyz. confirmed with grep "exfil-domain.xyz" network_logs | wc -l that it was a repeated pattern, lots of queries each with a different random subdomain. classic dns tunneling, data chunked and encoded into subdomain labels since dns rarely gets blocked outbound.
- **arp spoofing attack count:** went back to the arp filter and found the attacker mac wasnt just impersonating the gateway, it was also claiming to be 10.10.10.100. bidirectional poisoning, spoof the gateway to the victim and the victim to the gateway at the same time. packet count came out to 45 but the room wanted 90, so the scoring was counting something per direction or per exchange instead of per packet. didnt bother reverse engineering the exact logic, just moved on.

## Root Cause
 
network_logs was only a partial application layer view, it only recorded whatever its scope was configured to capture, no visibility into arp, non standard ports, or request bodies. the pcap was the actual ground truth and the logs were a lossy summary of it. trusting the logs alone would have missed the c2 channel, the arp spoofing, and the creds entirely.
 
## Lessons
 
Main takeaway here was wireshark mechanics more than the attack chain itself:
 
- follow tcp stream rebuilds the full conversation off one packet, and the content instantly tells you if youre looking at c2, exfil, or nothing.
- 0 turns on a followed stream means the handshake happened but no data moved, good for ruling stuff out but not a finding on its own.
- black/red coloring flags anomalies like retransmissions or "acked unseen segment" but its usually just a capture gap, not something malicious.
- arp.opcode == 2 gets you replies instead of the much noisier broadcast requests, gratuitous arp replies are a strong spoofing signal.
- when a log file is too big to read by hand, pull the field you care about and run sort | uniq -c | sort -rn, the anomaly is usually the one sitting at count 1.
- logs and pcaps arent redundant, one is searchable but incomplete, the other is complete but needs manual digging. need both.

# Answers:
## No Flag for this Room

- 10.10.10.100
- 1.1.1.1:8080
- 00:0c:29:11:22:33
- CVE-202X-EXPLOIT
- 90
- username=dev_user&password=SecretPassword!
- exfil-domain.xyz
- DNS