# OSPF-SHA-256-Authentication
Secure OSPF routing updates between three routers using **HMAC-SHA-256** authentication via a **key chain**. This prevents unauthorized routers from injecting fake OSPF routes into the network.
OSPF SHA-256 Authentication
Status: Completed
Date: October 2026
Time: ~45 min
Course: CCNA 3 — Enterprise Networking, Security, and Automation

Goal
Secure OSPF routing updates between three routers using HMAC-SHA-256 authentication via a key chain. This prevents unauthorized routers from injecting fake OSPF routes into the network.

Topology
https://./topology.png

Addressing
R1 G0/0/0 — 10.1.1.1/30
R1 G0/0/1 — 192.168.1.1/24
R2 G0/0/0 — 10.1.1.2/30
R2 G0/0/1 — 10.2.2.2/30
R3 G0/0/0 — 10.2.2.1/30
R3 G0/0/1 — 192.168.3.1/24
PC-A — 192.168.1.3/24, gateway 192.168.1.1
PC-C — 192.168.3.3/24, gateway 192.168.3.1

What is OSPF SHA-256 Authentication
OSPF by default does not authenticate routing updates. An attacker could connect and inject false routes — this is called OSPF spoofing.

SHA-256 authentication solves this:

Every OSPF packet carries an HMAC-SHA-256 hash computed from packet contents plus a shared secret key

The receiver computes its own hash and compares

If the key does not match, the packet is discarded and adjacency is not formed

Why key chains instead of MD5:

Stronger hashing (SHA-256 vs MD5)

Key rotation support

Industry best practice

Part 1 — Basic Configuration
Step 1: Cable the Network
R1 G0/0/0 to R2 G0/0/0
R2 G0/0/1 to R3 G0/0/0
R1 G0/0/1 to S1 F0/5, S1 F0/6 to PC-A
R3 G0/0/1 to S3 F0/5, S3 F0/18 to PC-C

Step 2: Configure R1
cisco
enable
configure terminal
hostname R1

interface g0/0/0
 ip address 10.1.1.1 255.255.255.252
 no shutdown
exit

interface g0/0/1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit

no ip domain-lookup

router ospf 1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.1.1.0 0.0.0.3 area 0
 passive-interface g0/0/1
exit
Step 3: Configure R2
cisco
enable
configure terminal
hostname R2

interface g0/0/0
 ip address 10.1.1.2 255.255.255.252
 no shutdown
exit

interface g0/0/1
 ip address 10.2.2.2 255.255.255.252
 no shutdown
exit

no ip domain-lookup

router ospf 1
 network 10.1.1.0 0.0.0.3 area 0
 network 10.2.2.0 0.0.0.3 area 0
exit
Step 4: Configure R3
cisco
enable
configure terminal
hostname R3

interface g0/0/0
 ip address 10.2.2.1 255.255.255.252
 no shutdown
exit

interface g0/0/1
 ip address 192.168.3.1 255.255.255.0
 no shutdown
exit

no ip domain-lookup

router ospf 1
 network 10.2.2.0 0.0.0.3 area 0
 network 192.168.3.0 0.0.0.255 area 0
 passive-interface g0/0/1
exit
Step 5: Verify OSPF Before Authentication
cisco
show ip ospf neighbor
show ip route
On PC-A:

text
ping 192.168.3.3
All neighbors should be in FULL state and ping should succeed.

Part 2 — Configure SHA-256 Authentication
Step 1: Create Key Chain on All Three Routers
On R1, R2, R3:

cisco
key chain NetAcad
 key 1
  key-string NetSeckeystring
  cryptographic-algorithm hmac-sha-256
 exit
exit
Key chain name is NetAcad.
Key ID is 1.
Key string is NetSeckeystring.
Algorithm is hmac-sha-256.

All three must match on all routers: chain name, key ID, key-string.

Step 2: Apply Key Chain to Interfaces
On R1 (G0/0/0 only):

cisco
interface g0/0/0
 ip ospf authentication key-chain NetAcad
exit
On R2 (both interfaces):

cisco
interface g0/0/0
 ip ospf authentication key-chain NetAcad
exit
interface g0/0/1
 ip ospf authentication key-chain NetAcad
exit
On R3 (G0/0/0 only):

cisco
interface g0/0/0
 ip ospf authentication key-chain NetAcad
exit
Adjacency drops temporarily when enabling auth on one side only:

text
%OSPF-5-ADJCHG: Process 1, Nbr 10.2.2.2 on GigabitEthernet0/0/0
from FULL to DOWN, Neighbor Down: Dead timer expired
This is normal. It recovers once both sides are configured.

Verification
Step 1: Verify OSPF Interface Authentication
cisco
R1# show ip ospf interface g0/0/0
Expected output (key lines):

text
Cryptographic authentication enabled
Sending SA: Key 1, Algorithm HMAC-SHA-256 - key chain NetAcad
Step 2: Verify Key Chain
cisco
R1# show key chain
Expected:

text
Key-chain NetAcad:
    key 1 -- text "NetSeckeystring"
        cryptographic-algorithm hmac-sha-256
Step 3: Verify Neighbors
cisco
R2# show ip ospf neighbor
Both neighbors should be in FULL state:

text
Neighbor ID     Pri   State           Dead Time   Address         Interface
192.168.3.1     1     FULL/DR         00:00:37    10.2.2.1        GigabitEthernet0/0/1
192.168.1.1     1     FULL/DR         00:00:37    10.1.1.1        GigabitEthernet0/0/0
Step 4: Verify Routing Table
cisco
R3# show ip route
Expected: OSPF routes to 10.1.1.0/30 and 192.168.1.0/24.

Step 5: Verify End-to-End Connectivity
On PC-A:

text
ping 192.168.3.3
Success means everything works.

Screenshots
show ip ospf interface — screenshots/ospf-interface.png
show key chain — screenshots/key-chain.png
show ip ospf neighbor — screenshots/ospf-neighbor.png
ping PC-A to PC-C — screenshots/ping.png

Key Nuances
Key chain name, key ID, and key-string must match on all routers. Otherwise adjacency will not form.

Adjacency temporarily drops when enabling auth on one side. This is expected. It recovers once the other side is configured.

SHA-256 is stronger than MD5. MD5 is deprecated in modern networks.

Key chain applies only to OSPF-enabled interfaces, not to passive interfaces.

show ip ospf interface is the main verification command. Look for Cryptographic authentication enabled.

Key chains support multiple keys for smooth rotation. Devices accept both, send with the newest.

passive-interface g0/0/1 stays as is. Authentication does not change that.

show key chain verifies key chain config directly.

Command Summary
Create key chain — key chain NetAcad
Add key — key 1
Set password — key-string NetSeckeystring
Set algorithm — cryptographic-algorithm hmac-sha-256
Apply to interface — ip ospf authentication key-chain NetAcad
Verify auth — show ip ospf interface g0/0/0
Verify key chain — show key chain
Verify neighbors — show ip ospf neighbor

Files
configs/R1.txt — R1 running-config
configs/R2.txt — R2 running-config
configs/R3.txt — R3 running-config

License
MIT. Free to use.

Author
Fariza

GitHub: https://github.com/hhhxnln-stack