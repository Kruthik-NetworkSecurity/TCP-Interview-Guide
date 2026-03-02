# 🔗 TCP Deep Dive — Interview Preparation Guide

> A senior-level technical reference for TCP flags, options, and flow control.  
> Built from real enterprise experience supporting Citrix NetScaler, Cisco ASA, and enterprise network environments.

---

## 📋 Table of Contents

1. [TCP Flags](#1-tcp-flags)
2. [MTU vs MSS — The Math](#2-mtu-vs-mss--the-math)
3. [TCP Options — MSS, WSCALE, SACK, Timestamps](#3-tcp-options--mss-wscale-sack-timestamps)
4. [MSS Clamping on NetScaler](#4-mss-clamping-on-netscaler)
5. [Window Scaling & The 64KB Limit](#5-window-scaling--the-64kb-limit)
6. [SACK — Selective Acknowledgment](#6-sack--selective-acknowledgment)
7. [Flow Control vs Congestion Control](#7-flow-control-vs-congestion-control)
8. [Retransmission & Nagle's Algorithm](#8-retransmission--nagles-algorithm)
9. [NetScaler Full Proxy Architecture](#9-netscaler-full-proxy-architecture)
10. [Interview Cheat Sheet](#10-interview-cheat-sheet)

---

## 1. TCP Flags

### SYN (Synchronize)
The SYN flag initiates the three-way handshake and synchronizes Initial Sequence Numbers (ISN).

> **Interview angle:** If you see a SYN with no response in a Wireshark trace, it points to a **firewall drop** or **host offline**. The client will keep retrying using **Exponential Backoff** to avoid overwhelming the network.

---

### ACK (Acknowledgment)
Once the initial handshake is complete, the ACK bit is effectively set to `1` for the rest of the session. It tells the sender exactly which byte the receiver wants next.

> **Interview angle:** If ACKs stop arriving, the sender assumes data was lost and retransmits. Consistent missing ACKs → look for packet loss or asymmetric routing.

---

### PSH (Push)
PSH tells the receiving stack to deliver data to the application immediately without waiting to fill a full segment buffer.

> **Real-world example:** SSH interactive sessions — every keystroke carries PSH=1 so the letter appears on screen instantly without lag.

---

### URG (Urgent)
URG is an emergency interrupt. It tells the receiver to skip buffered data and process the urgent data immediately.

> **Real-world example:** Pressing `Ctrl+C` while viewing a large file — URG=1 signals the server to stop the stream immediately rather than finishing the 2,000-line output.

---

### FIN (Finish)
FIN initiates a graceful 4-way TCP teardown.

> **NetScaler Pro-Tip:** NetScaler TCP health monitors send a `FIN ACK` after receiving `SYN+ACK` to close the probe. Many backend servers respond with `RST` instead of the full 4-way teardown because the connection carried no data — **this is still a successful health check.** Enable **Non-Graceful Monitor** if you want to suppress RST errors in backend logs.

---

### RST (Reset)
RST is a hard connection kill — used for errors or mismatches.

| Scenario | What Happens |
|---|---|
| **Port mismatch** | NetScaler targets port 443, backend only listens on 80 → backend sends RST |
| **Protocol mismatch** | NetScaler sends SSL `Client Hello` but backend expects plain HTTP → backend gets confused and sends RST |
| **Firewall policy** | Firewall silently drops packets, then sends RST after its timeout expires |

> **NetScaler Pro-Tip:** Enable **SSL Offloading** — NetScaler handles TLS on port 443 for clients and talks plain HTTP to the backend. Backend never sees encrypted traffic.

---

## 2. MTU vs MSS — The Math

This is a favourite interview question at senior level.

| Layer | Header | Size |
|---|---|---|
| Ethernet (L2) | MTU — the full "envelope" | **1500 bytes** |
| IP Header (L3) | Minimum | **20 bytes** |
| TCP Header (L4) | Minimum | **20 bytes** |
| **MSS** | Actual payload | **1460 bytes** |

```
MSS = MTU - IP Header - TCP Header
MSS = 1500 - 20 - 20 = 1460 bytes
```

### TCP Header — The Options Field
The standard TCP header is **20 bytes**, but it can grow up to **60 bytes** when Options are added.

The MSS Option structure (always in the SYN packet):
| Field | Size | Value |
|---|---|---|
| Kind | 1 byte | `2` (identifies as MSS) |
| Length | 1 byte | `4` (total option length) |
| Value | 2 bytes | e.g., `1460` |

> **Interview Tip:** If you see a **Data Offset > 20 bytes** in a packet capture, TCP Options are present. A Data Offset of 32 bytes means 12 bytes of Options are in use.

---

## 3. TCP Options — MSS, WSCALE, SACK, Timestamps

Think of TCP Options as "special instructions" agreed upon before data transfer begins.

| Option | Analogy | Purpose |
|---|---|---|
| **MSS** | Size of the box | Prevents oversized packets hitting a low bridge and fragmenting |
| **WSCALE** | Number of trucks on the road | Fills the "long fat pipe" — allows more data in flight |
| **SACK** | Only resend the missing truck | Fixes packet loss without retransmitting already-received data |
| **Timestamps** | GPS expiration date on each truck | Prevents old delayed packets corrupting new data (PAWS) |

---

## 4. MSS Clamping on NetScaler

### The Problem — Black Hole Scenario
1. Client MTU: 1500 → MSS: 1460
2. Backend MTU: 1500 → MSS: 1460
3. Path goes through **IPsec or GRE tunnel** → adds 60 bytes overhead → real path MTU: **1440**

Small packets (like login requests) work fine. Large packets (like database results pages) get fragmented or silently dropped. This is called a **PMTU Black Hole**.

### The NetScaler Solution — MSS Clamping

Configure a **TCP Profile** on the NetScaler:
- NetScaler intercepts the SYN packet
- Even if the server advertises MSS 1460, NetScaler **rewrites it to 1380** before passing to the client
- Both sides now automatically send smaller packets that fit through the tunnel

```
# NetScaler TCP Profile for MSS Clamping
add ns tcpProfile <PROFILE_NAME> -mss 1380
bind lb vserver <VS_NAME> -tcpProfileName <PROFILE_NAME>
```

### MSS is NOT Negotiated — It's Advertised

> **Key Interview Point:** MSS is a **unilateral advertisement**, not a negotiated value. The server doesn't reject the connection if MSS values differ. It accepts and uses the **smaller of the two values** for the session.

The handshake step-by-step:
1. **SYN** (NetScaler → Server): "My MSS is 1460"
2. **SYN-ACK** (Server → NetScaler): "Accepted. My MSS is 1380"
3. **Result**: Both sides use **1380** for the session (the lower value per RFC 793)

**Where does it actually fail?** Not during the handshake — during **data transfer**, if one side ignores the agreed MSS and sends a larger segment. The receiver silently drops it → sender waits for an ACK that never comes → retransmission storm.

---

## 5. Window Scaling & The 64KB Limit

### The Problem
Original TCP used a **16-bit Window Size** field → maximum **65,535 bytes (64KB)**.

On a 10Gbps fiber link between New York and London, 64KB is exhausted in microseconds. The sender then sits idle waiting for ACKs to travel across the ocean. A 10Gbps link behaves like a 10Mbps link.

### The Solution — WSCALE Option

WSCALE adds a **multiplier** during the SYN handshake:

```
NetScaler SYN → WSCALE: 7
Multiplier = 2^7 = 128

Window Size field says: 1,000
Real window = 1,000 × 128 = 128,000 bytes
```

WSCALE option structure (3 bytes — unlike MSS which is 4 bytes):
| Field | Value |
|---|---|
| Kind | `3` |
| Length | `3` |
| Shift Count | The exponent (e.g., `7`) |

### Classic Interview Scenario

**Interviewer:** "I have a 10Gbps circuit between two data centers but file transfers are capped at exactly 5Mbps. Why?"

**Answer:** An intermediate firewall or IPS is **stripping the WSCALE option** from the SYN handshake. Both hosts fall back to the legacy 64KB limit. Due to the inter-DC latency, the sender spends most of its time waiting for ACKs rather than sending data. Verify with a packet trace — WSCALE present in SYN but missing in SYN-ACK confirms the issue.

---

## 6. SACK — Selective Acknowledgment

### The Problem — Legacy "Go-Back-N"
You send 10 packets. Packet #3 is lost but #4–#10 arrive safely.

Without SACK: Receiver says "I have up to #2, resend from #3" → sender resends **#3 through #10** even though the receiver already has #4–#10. Massive bandwidth waste.

### The Solution — SACK
Receiver says: "I'm missing #3 **but I have #4–#10**. Only send me #3."

Only the one missing 1460-byte segment is retransmitted.

### SACK Negotiation
Must be permitted during handshake:
1. **SYN** → includes `SACK-Permitted` option
2. **SYN-ACK** → includes `SACK-Permitted` option
3. If both agree → receiver can send **SACK Blocks** identifying specific received ranges

SACK Block structure (Kind: `5`, Length: variable):
- **Left Edge**: Start sequence number of received data
- **Right Edge**: End sequence number of received data

### Classic Interview Scenario

**Interviewer:** "Over a satellite link with 1% packet loss, throughput drops 80%. Why?"

**Answer:** If SACK is disabled or stripped by a firewall, 1% packet loss triggers legacy cumulative ACKs. Every lost packet forces the entire window to be resent, creating a retransmission storm. Check the TCP Profile on NetScaler to confirm SACK is enabled, and verify both sides negotiate `SACK-Permitted` in the SYN handshake via Wireshark.

---

## 7. Flow Control vs Congestion Control

These are often confused in interviews. They solve different problems.

| Concept | Who Controls It | What It Protects | Key Signal |
|---|---|---|---|
| **Flow Control** | The **Receiver** | Prevents sender overwhelming receiver | Zero Window advertisement |
| **Congestion Control** | The **Network path** | Prevents sender overwhelming routers | Packet loss / RTT increase |

### Flow Control — Zero Window
When the receiver's application is slow (e.g., database writing to disk), its buffer fills up. It sends a **Zero Window** advertisement: "Stop sending — my buffer is full."

### Congestion Control — Slow Start
TCP doesn't know the network capacity at connection start. It begins by sending a small amount (typically 10 segments) and doubles on each successful ACK: `2 → 4 → 8 → 16...`

Once it hits a limit or detects loss, it reduces the sending rate (**Congestion Avoidance**).

> **Senior-level phrase:** "TCP uses algorithms like **CUBIC** or **BBR** to find the sweet spot — maximum throughput without causing router drops."

---

## 8. Retransmission & Nagle's Algorithm

### Retransmission Timeout (RTO)
When a sender transmits, it starts an **RTO timer**. If it expires before an ACK arrives, the packet is assumed lost and retransmitted.

**Exponential Backoff:** 1s → 2s → 4s → 8s (doubles each time)

> **Troubleshooting Tip:** RTOs in a trace are **much worse** than Fast Retransmits — the connection completely freezes for 1+ second. Fast Retransmit triggers after 3 duplicate ACKs and recovers in milliseconds.

### Nagle's Algorithm vs Delayed ACK — The Conflict

| Mechanism | Behaviour |
|---|---|
| **Nagle's Algorithm** | Sender waits to fill a full MSS (1460 bytes) before sending |
| **Delayed ACK** | Receiver waits up to 200ms before sending ACK, hoping to piggyback it on a response |

**The conflict:** Sender waits for a full buffer. Receiver waits before sending ACK. Result: **200ms artificial latency on every packet.**

**The Fix:** On NetScaler, disable Nagle's Algorithm using `TCP_NODELAY` for interactive applications like Citrix Virtual Apps and Desktops.

```
# Disable Nagle's Algorithm in TCP Profile
set ns tcpProfile <PROFILE_NAME> -nagle DISABLED
bind lb vserver <VS_NAME> -tcpProfileName <PROFILE_NAME>
```

---

## 9. NetScaler Full Proxy Architecture

NetScaler operates as a **Full Proxy** — it terminates the client connection and establishes a completely new connection to the backend server.

```
Client ←——— Client-Side TCP ———→ NetScaler VIP
                                        |
                               NetScaler SNIP
                                        |
NetScaler SNIP ←——— Server-Side TCP ———→ Backend Server
```

### Why This Matters for TCP Options

TCP Options (MSS, WSCALE, SACK) are **negotiated independently** on each side.

**Practical example:**
- **Client-Side:** Client and NetScaler both support WSCALE → agree on multiplier (e.g., WSCALE: 8) → fast upload/download to NetScaler
- **Server-Side:** Old backend doesn't support WSCALE → falls back to 64KB legacy window

**Result:** NetScaler buffers data internally, bridging a "modern TCP" client to a "legacy TCP" backend simultaneously. This is called **TCP Optimization** or **Connection Multiplexing**.

> **Interview answer:** "Because the NetScaler is a Full Proxy, it manages two independent TCP stacks. It can negotiate a large Window Scale with the client while simultaneously maintaining a standard 64KB window with a legacy backend. It acts as a buffer, receiving data quickly from the client and feeding it to the server at the pace the server can handle."

---

## 10. Interview Cheat Sheet

```
TCP FLAGS
─────────────────────────────────────────────────
SYN      → Request to start (handshake initiation)
ACK      → "Got it!" (set to 1 for entire session after handshake)
PSH      → "Send now!" (no buffering — used in SSH, interactive apps)
URG      → Emergency interrupt (skip buffer, process immediately)
FIN      → Graceful close (4-way teardown)
RST      → Hard kill (port mismatch, protocol mismatch, firewall)

TCP OPTIONS
─────────────────────────────────────────────────
MSS      → Max segment size | 4 bytes | Kind=2 | Sent in SYN
WSCALE   → Window multiplier | 3 bytes | Kind=3 | Sent in SYN
SACK     → Selective ACK    | Variable | Kind=5 | Sent in SYN
Timestamp→ PAWS protection  | 10 bytes | Kind=8 | Sent in SYN

KEY FORMULAS
─────────────────────────────────────────────────
MSS  = MTU(1500) - IP(20) - TCP(20) = 1460 bytes
WSCALE multiplier = 2^(shift count)
Max window without WSCALE = 2^16 - 1 = 65,535 bytes (64KB)

NETSCALER SPECIFIC
─────────────────────────────────────────────────
Full Proxy      → Two independent TCP stacks (client-side + server-side)
MSS Clamping    → Rewrite MSS in SYN to prevent PMTU black holes
TCP_NODELAY     → Disables Nagle's (fixes 200ms latency on interactive apps)
Non-Graceful    → NetScaler sends RST first (cleans up backend logs)
SSL Offload     → Client=HTTPS(443), Backend=HTTP(80)

TROUBLESHOOTING QUICK REFERENCE
─────────────────────────────────────────────────
SYN no response        → Firewall drop or host down
RST from server        → Port/protocol mismatch
Zero Window            → Receiver buffer full (app slow)
RTO in trace           → Packet loss + 1s+ freeze
200ms latency spikes   → Nagle + Delayed ACK conflict
5Mbps on 10Gbps link   → WSCALE stripped by firewall
80% drop on 1% loss    → SACK disabled
Large packets fail     → MTU Black Hole — enable MSS clamping
```

---

## About

Built by **Kruthik K V** — Network Security Engineer with 3.7+ years of enterprise TAC experience at Citrix (CSG) and HCL Tech.

📧 kruthik.39t@gmail.com
🔗 [LinkedIn](https://www.linkedin.com/in/kruthik-k-v-3115ba21a)
🔗 [GitHub Profile](https://github.com/Kruthik-NetworkSecurity)
🌐 [NetScaler Field Guide](https://github.com/Kruthik-NetworkSecurity/NetScaler)
🦈 [Wireshark Playbook](https://github.com/Kruthik-NetworkSecurity/Wireshark-Playbook)
