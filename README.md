<div align="center">

# 📦 EIGRP Manual Route Summarization

### `ip summary-address eigrp` · 192.1.64.0/22 · Null0 Discard Route · Smaller Routing Tables

![Cisco](https://img.shields.io/badge/Cisco-IOS%20Routers-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Protocol](https://img.shields.io/badge/Protocol-EIGRP%20AS%201-blue?style=for-the-badge)
![Summary](https://img.shields.io/badge/Summary-192.1.64.0%2F22-brightgreen?style=for-the-badge)
![Mask](https://img.shields.io/badge/Mask-255.255.252.0-purple?style=for-the-badge)
![Routers](https://img.shields.io/badge/Routers-3-yellow?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA%20%2F%20CCNP-red?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)

**Many specific routes in → one summary route out**

</div>

---

## 📖 Table of Contents

[Overview](#-project-overview) · [Lab Snapshot](#-lab-snapshot) · [Before vs After](#-before-vs-after-summarization) · [Topology](#️-network-topology) · [IP Table](#-ip-addressing-table) · [Configuration](#️-configuration) · [Verification](#-verification--stage-by-stage) · [Subnet Math](#-why-19216422-subnet-math) · [Null0](#-the-automatic-null0-route) · [Command Breakdown](#-command-breakdown) · [Troubleshooting](#-troubleshooting-guide) · [Advanced](#-advanced-topics) · [Takeaways](#-key-takeaways) · [Challenges](#-challenges)

---

## 📌 Project Overview

As networks grow, routing tables grow with them. Every extra prefix means more memory, more update traffic and more CPU work. **Route summarization** fixes this by advertising **one summary route instead of many specific ones**.

This lab configures **manual EIGRP summarization** on **Router3**. It owns several networks in the `192.1.x.0/24` range and advertises them to the rest of the network as a single **`192.1.64.0/22`** summary:

```bash
ip summary-address eigrp 1 192.1.64.0 255.255.252.0
```

> 🎯 **Result:** Router1 (two hops away) sees **one `/22` summary** instead of three separate `/24` routes — and also shows exactly what happens to a route that falls **outside** the summary range (`192.1.68.0/24`).

---

## 📸 Lab Snapshot

| Item | Value |
|:-----|:------|
| 🌐 Routing protocol | **EIGRP AS 1** |
| 🖧 Routers | Router1 · Router2 · Router3 (linear topology) |
| 🔗 Links | 1.1.1.0/24 (R1–R2) · 2.1.1.0/24 (R2–R3), FastEthernet |
| 🔁 Loopbacks | 10.1.1.1/32 · 20.1.1.1/32 · 30.1.1.1/32 |
| 📌 Static routes (on Router3) | 192.1.65.0/24 · 192.1.66.0/24 · 192.1.67.0/24 · 192.1.68.0/24 → `2.1.1.1` |
| 📦 Summary | **192.1.64.0/22** (`255.255.252.0`) |
| 🕳️ Auto-created on Router3 | `D 192.1.64.0/22 is a summary … Null0` |
| 🔍 Verified from | Router3 (originator) and Router1 (remote) via Telnet |

---

## 🔄 Before vs After Summarization

> *"Before" is the expected result without the summary command. "After" is captured in the screenshots.*

| | ❌ Without summary (expected) | ✅ With summary (captured) |
|:--|:----------------------------|:---------------------------|
| What Router1 learns for 192.1.x | 192.1.65.0/24 · 192.1.66.0/24 · 192.1.67.0/24 · 192.1.68.0/24 | **192.1.64.0/22** + 192.1.68.0/24 |
| Number of routes | 4 | **2** |
| Route code | `D EX` (all four) | `D` (summary) + `D EX` (outside range) |
| Flaps of one /24 | Update sent to every router | **Hidden behind the summary** |
| Routing table size | Larger | **Smaller** |

```
 Router3 knows:                         Router1 sees:
 ┌───────────────────────┐              ┌──────────────────────────────┐
 │ 192.1.65.0/24  ──┐    │              │                              │
 │ 192.1.66.0/24  ──┼────┼─ summary ──► │ 192.1.64.0/22  (1 route)     │
 │ 192.1.67.0/24  ──┘    │              │                              │
 │ 192.1.68.0/24  ───────┼─ outside ──► │ 192.1.68.0/24  (stays alone) │
 └───────────────────────┘              └──────────────────────────────┘
```

---

## 🎯 Objectives

- ✅ Configure **manual EIGRP summarization** with `ip summary-address eigrp`
- ✅ Calculate the correct **summary prefix and mask** for a group of networks
- ✅ Verify the **summary route on the originating router** (Router3)
- ✅ Verify the **summary route on a remote router** (Router1)
- ✅ Understand the **automatic Null0 discard route**
- ✅ See what happens to a network **outside** the summary range
- ✅ Decode EIGRP **metrics** from FastEthernet delay values

---

## 🗺️ Network Topology

```
   L1 10.1.1.1/32            L2 20.1.1.1/32                 L3 30.1.1.1/32
        │                          │                              │
   ┌─────────┐ fa0/0     fa0/0 ┌─────────┐ fa3/0     fa3/0 ┌─────────┐
   │ Router1 ├─────────────────┤ Router2 ├─────────────────┤ Router3 │
   └─────────┘ .1    1.1.1.0/24 .2└──────┘ .1    2.1.1.0/24 .2└────┬───┘
                                                                   │
                                   ip summary-address eigrp 1 192.1.64.0 255.255.252.0
                                   ip route 192.1.65.0 255.255.255.0 2.1.1.1
                                   ip route 192.1.66.0 255.255.255.0 2.1.1.1
                                   ip route 192.1.67.0 255.255.255.0 2.1.1.1
                                   ip route 192.1.68.0 255.255.255.0 2.1.1.1
```

> 📷 Add your screenshot: `![Topology](images/topology.png)`

---

## 📋 IP Addressing Table

| Device | Interface | IP Address | Connected To |
|:------:|:---------:|:----------:|:-------------|
| **Router1** | Loopback1 (L1) | 10.1.1.1/32 | — |
| **Router1** | Fa0/0 | 1.1.1.1/24 | Router2 Fa0/0 |
| **Router1** | Fa3/0 | 100.100.150.1/24 | Connected network *(seen in route table)* |
| **Router2** | Loopback | 20.1.1.1/32 | — |
| **Router2** | Fa0/0 | 1.1.1.2/24 | Router1 Fa0/0 |
| **Router2** | Fa3/0 | 2.1.1.1/24 | Router3 Fa3/0 |
| **Router3** | Loopback1 (L3) | 30.1.1.1/32 | — |
| **Router3** | Fa3/0 | 2.1.1.2/24 | Router2 Fa3/0 |
| **Router3** | Fa0/0 | 100.100.100.1/24 | Connected network *(seen in route table)* |

---

## ⚙️ Configuration

> 💡 Basic EIGRP AS 1 (network statements for the links and loopbacks) was already running. This lab **adds the static routes and the summary** on Router3.

### 🔹 Step 1 — Static routes on Router3

```bash
ip route 192.1.65.0 255.255.255.0 2.1.1.1
ip route 192.1.66.0 255.255.255.0 2.1.1.1
ip route 192.1.67.0 255.255.255.0 2.1.1.1
ip route 192.1.68.0 255.255.255.0 2.1.1.1
```

### 🔹 Step 2 — Make EIGRP advertise them

```bash
router eigrp 1
 redistribute static
```

### 🔹 Step 3 — Configure the manual summary (interface toward Router2)

```bash
interface FastEthernet3/0
 ip summary-address eigrp 1 192.1.64.0 255.255.252.0
```

> ⚠️ Assumed from the topology and outputs: the redistribution and the interface are inferred, since `192.1.68.0/24` appears on Router1 as `D EX`. Match these lines to your own running-config.

---

## ✅ Verification — Stage by Stage

### 🟡 Stage 1: Router3 (the Originator)

```
R3# show ip route
     1.0.0.0/24 is subnetted, 1 subnets
D       1.1.1.0 [90/30720] via 2.1.1.1, 00:02:34, FastEthernet3/0
     2.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       2.1.1.0/24 is directly connected, FastEthernet3/0
L       2.1.1.2/32 is directly connected, FastEthernet3/0
     10.0.0.0/32 is subnetted, 1 subnets
D       10.1.1.1 [90/158720] via 2.1.1.1, 00:02:34, FastEthernet3/0
     20.0.0.0/32 is subnetted, 1 subnets
D       20.1.1.1 [90/156160] via 2.1.1.1, 00:02:34, FastEthernet3/0
     30.0.0.0/32 is subnetted, 1 subnets
C       30.1.1.1 is directly connected, Loopback1
     100.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
C       100.100.100.0/24 is directly connected, FastEthernet0/0
L       100.100.100.1/32 is directly connected, FastEthernet0/0
D       100.100.150.0/24 [90/33280] via 2.1.1.1, 00:02:34, FastEthernet3/0
D       192.1.64.0/22 is a summary, 00:02:36, Null0        ◄── 📦 SUMMARY (auto-created Null0 route)
S       192.1.65.0/24 [1/0] via 2.1.1.1                     ◄── specific routes still exist locally
S       192.1.66.0/24 [1/0] via 2.1.1.1
S       192.1.67.0/24 [1/0] via 2.1.1.1
S       192.1.68.0/24 [1/0] via 2.1.1.1
```

🧠 **What this shows**

- Router3 **creates the summary** `192.1.64.0/22` and points it to **Null0**
- The **four specific static routes still exist** on Router3 — summarization only changes what is *advertised*
- Router3 keeps using the **more specific** routes for real forwarding (longest prefix match)

---

### 🟢 Stage 2: Router1 (the Remote Router) ⭐

```
R1# show ip route
     1.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       1.1.1.0/24 is directly connected, FastEthernet0/0
L       1.1.1.1/32 is directly connected, FastEthernet0/0
     2.0.0.0/24 is subnetted, 1 subnets
D       2.1.1.0 [90/30720] via 1.1.1.2, 00:01:17, FastEthernet0/0
     10.0.0.0/32 is subnetted, 1 subnets
C       10.1.1.1 is directly connected, Loopback1
     20.0.0.0/32 is subnetted, 1 subnets
D       20.1.1.1 [90/156160] via 1.1.1.2, 00:01:17, FastEthernet0/0
     30.0.0.0/32 is subnetted, 1 subnets
D       30.1.1.1 [90/158720] via 1.1.1.2, 00:01:12, FastEthernet0/0
     100.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
D       100.100.100.0/24 [90/33280] via 1.1.1.2, 00:01:12, FastEthernet0/0
C       100.100.150.0/24 is directly connected, FastEthernet3/0
L       100.100.150.1/32 is directly connected, FastEthernet3/0
D       192.1.64.0/22 [90/33280] via 1.1.1.2, 00:01:12, FastEthernet0/0    ◄── 📦 ONE summary route
D EX    192.1.68.0/24 [170/33280] via 1.1.1.2, 00:01:12, FastEthernet0/0   ◄── ⚠️ outside the /22
```

🧠 **What this shows**

- Router1 gets **one `D 192.1.64.0/22`** instead of three separate `/24` routes (65, 66, 67)
- The summary is a **normal internal EIGRP route** (`D`, AD **90**)
- `192.1.68.0/24` is **not covered** by the `/22`, so it is advertised **individually** as an external route (`D EX`, AD **170**)

---

### 🔍 The Key Observation: What the Summary Covers

| Static route on Router3 | Inside `192.1.64.0/22`? | Seen on Router1 as |
|:------------------------|:-----------------------:|:-------------------|
| 192.1.65.0/24 | ✅ Yes | *(hidden — covered by the summary)* |
| 192.1.66.0/24 | ✅ Yes | *(hidden — covered by the summary)* |
| 192.1.67.0/24 | ✅ Yes | *(hidden — covered by the summary)* |
| **192.1.68.0/24** | ❌ **No** | `D EX 192.1.68.0/24 [170/33280]` |

> 💡 **Lesson:** a summary must cover **every** network you want to hide. To include `192.1.68.0/24` too, use a wider summary such as **`192.1.64.0/21`** (covers 64 – 71).

---

### 🧾 Evidence Summary

| # | Screenshot | What it proves |
|:-:|:-----------|:---------------|
| 1 | Topology + command box | 3-router chain, summary command, 4 static routes |
| 2 | Router3 `show ip route` | `192.1.64.0/22 is a summary … Null0` + 4 static routes |
| 3 | Router1 `show ip route` | One `D 192.1.64.0/22` + separate `D EX 192.1.68.0/24` |

### 📷 Add These Screenshots to Complete the Lab

```bash
show ip route 192.1.64.0 255.255.252.0   # on Router1 — details of the summary
show ip eigrp topology                   # summary entry on Router3
show ip protocols                        # "Address Summarization: 192.1.64.0/22 for FastEthernet3/0"
ping 192.1.66.1 source loopback1         # forwarding follows the summary
# BEFORE shot: run show ip route on Router1 before adding the summary → 4 separate routes
```

---

## 🧮 Why `192.1.64.0/22`? (Subnet Math)

A `/22` mask is **255.255.252.0** — a block of **4** values in the third octet.

```
Third octet:   64  = 0100 0000
Mask 252       252 = 1111 1100
                      └────┘ network bits (first 6) · host bits (last 2) → block size = 4

Summary block:  192.1.64.0  →  192.1.67.255
                third octet:  64 · 65 · 66 · 67
```

| Third octet | Binary | In `/22`? |
|:-----------:|:------:|:---------:|
| 64 | 0100 00**00** | ✅ (network address) |
| 65 | 0100 00**01** | ✅ |
| 66 | 0100 00**10** | ✅ |
| 67 | 0100 00**11** | ✅ |
| **68** | 0100 01**00** | ❌ (starts the next block) |

> 📏 **Rule of thumb:** the summary must start on a block boundary and cover a power-of-two count of networks. 64–67 fits `/22`; 64–71 fits `/21`.

---

## 🕳️ The Automatic Null0 Route

```
D       192.1.64.0/22 is a summary, 00:02:36, Null0
```

When Router3 creates a summary, it **automatically installs a route to `Null0`** for that summary prefix.

| Why it exists | Explanation |
|:--------------|:------------|
| 🔁 **Loop prevention** | If traffic arrives for an address in the summary that has **no specific route**, it is **dropped** instead of looping back out |
| 🎯 **Longest match still wins** | Real networks (`192.1.65.0/24` …) match first because they're **more specific** |
| 🏷️ **Low AD** | EIGRP summary routes use **AD 5**, so the discard route is trusted over normal EIGRP routes |

---

## 🧮 Metric Decode (FastEthernet Values)

Using the standard EIGRP formula — bandwidth term **100** (FastEthernet = 100,000 Kbps) plus the sum of interface delays (**10** per FastEthernet hop):

```
Metric = 256 × ( 10⁷ ÷ 100,000  +  total delay )
```

| Route on Router1 | Metric | ÷ 256 | = 100 + delay | Delay hops |
|:-----------------|:------:|:-----:|:-------------:|:----------:|
| `2.1.1.0` | 30720 | 120 | 100 + **20** | 2 × FastEthernet |
| `100.100.100.0/24` | 33280 | 130 | 100 + **30** | 3 × FastEthernet |
| `192.1.64.0/22` (summary) | 33280 | 130 | 100 + **30** | 3 × FastEthernet |

> 💡 The **summary inherits the lowest metric** of the specific routes it covers — here, the same 33280.

---

## 🧩 Command Breakdown

```
ip summary-address eigrp 1 192.1.64.0 255.255.252.0
│  │               │     │  │          │
│  │               │     │  │          └── summary mask (/22)
│  │               │     │  └───────────── summary network address
│  │               │     └──────────────── EIGRP AS number
│  │               └────────────────────── protocol
│  └────────────────────────────────────── applied PER INTERFACE
└───────────────────────────────────────── interface-level command
```

| Fact | Detail |
|:-----|:-------|
| Where is it configured? | On the **outgoing interface** (toward the neighbors that should receive the summary) |
| Does it hide the specific routes? | Yes — **only on that interface**; locally they remain |
| Creates a Null0 route? | ✅ Yes, automatically |
| Needs at least one component route? | ✅ Yes — the summary appears only while at least one specific route exists |

---

## 🧰 Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|:--------|:-------------|:----|
| Summary not visible on the remote router | Command on the **wrong interface** | Apply it on the interface **toward** the neighbor |
| Summary never appears | No component route inside the range | Make sure at least one specific route exists in EIGRP |
| Specific routes still seen remotely | Summary range doesn't cover them | Recalculate the prefix (e.g., `/22` → `/21`) |
| One network leaks as `D EX` | Falls outside the summary block | Widen the summary or summarize separately |
| Traffic to a missing subnet disappears | The Null0 discard route (**by design**) | Add the missing route or accept the drop |
| Wrong summary address | Not on a block boundary | Use a network address aligned to the mask |

---

## 🚀 Advanced Topics

### 🔹 Auto-summary vs manual summary

| | Auto-summary | Manual summary |
|:--|:-------------|:---------------|
| Command | `auto-summary` | `ip summary-address eigrp …` |
| Boundary | Classful network boundary | **Any prefix you choose** |
| Control | Low | **High** |
| Modern use | Disabled by default (`no auto-summary`) | ✅ Preferred |

### 🔹 Summarize to a default route

```bash
interface FastEthernet3/0
 ip summary-address eigrp 1 0.0.0.0 0.0.0.0
```
> Sends a **default route** to the neighbor and hides all specifics.

### 🔹 Faster convergence & smaller query scope

Summaries **stop queries** from travelling through the whole network — an unreachable specific route inside the summary doesn't trigger a flood of EIGRP queries beyond the summarizing router.

### 🔹 Leak a specific route through a summary

```bash
route-map LEAK permit 10
 match ip address prefix-list KEEP-66
interface FastEthernet3/0
 ip summary-address eigrp 1 192.1.64.0 255.255.252.0 leak-map LEAK
```

---

## 🎓 Key Takeaways

1. 🔸 **Summarization** replaces many specific routes with **one** advertisement
2. 🔸 `ip summary-address eigrp` is **configured per interface**, not globally
3. 🔸 `192.1.64.0/22` covers **192.1.64.0 – 192.1.67.255** (third octet 64 – 67)
4. 🔸 Router3 keeps its **specific routes**; neighbors see only the **summary**
5. 🔸 A **Null0 route** is created automatically to prevent loops
6. 🔸 Networks **outside** the range (192.1.68.0/24) are **not** summarized
7. 🔸 The summary uses the **lowest metric** of its components
8. 🔸 Smaller tables = **less memory, less traffic, fewer flaps, smaller query scope**
9. 🔸 Always verify from **both** the summarizing router and a **remote** router

---

## 🧪 Challenges

1. Change the summary to `192.1.64.0/21` — does `192.1.68.0/24` disappear from Router1?
2. Remove one static route — does the summary stay while other components remain?
3. Remove **all** static routes — does the summary disappear?
4. Apply the summary on the **wrong interface** — what does Router1 see?
5. Summarize the loopbacks (`10.1.1.1`, `20.1.1.1`, `30.1.1.1`) — is it even possible with one prefix?
6. Add a **leak-map** to keep one specific route visible

---

## 📁 Repository Structure

```
📦 EIGRP-Summarization
 ┣ 📜 README.md
 ┣ 📂 images
 ┃ ┣ 🖼️ topology.png
 ┃ ┣ 🖼️ 01-router3-summary-null0.png
 ┃ ┗ 🖼️ 02-router1-summary-received.png
 ┗ 📂 lab-file
   ┗ 📦 EIGRP-Summarization (project file)
```

---

## 🔗 EIGRP Lab Series

| # | Lab |
|:-:|:----|
| 1 | EIGRP Successor – Single Path |
| 2 | EIGRP Successor – 2 Paths (Equal-Cost Load Balancing) |
| 3 | EIGRP Unequal-Cost Load Balancing & Variance |
| 4 | EIGRP MD5 Authentication |
| 5 | EIGRP Metric Calculation |
| 6 | EIGRP Redistribution (RIP + Static) |
| 7 | **EIGRP Manual Route Summarization** ◄── *you are here* |

`EIGRP` · `Route Summarization` · `ip summary-address` · `Null0` · `Subnetting` · `CIDR` · `Routing Table Optimization`

---

## 👨‍💻 Author

**Maaz Khan**
CCNA-Certified Network & NOC Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-maazkhanms-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/maazkhanms)
[![GitHub](https://img.shields.io/badge/GitHub-maazkhanms-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/maazkhanms)

---

<div align="center">

⭐ **If this lab helped you, please star the repository!** ⭐

*Part of my ongoing networking lab series — routing, switching, security & more.*

</div>
