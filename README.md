# 🔍 TCP Packet Analysis with Wireshark

A hands-on deep dive into a real TCP connection — from the three-way handshake to RST teardown — using Wireshark and a live `.pcap` trace file.

This repository documents the full analysis: inspecting TCP segment headers, tracing the connection lifecycle, visualizing throughput with IO Graphs, and observing behaviors like slow start, delayed ACKs, and flow control — all from real captured traffic.

---

## 📌 What This Covers

- Opening and navigating a `.pcap` file in Wireshark
- Reading TCP header fields: Seq, ACK, Flags, Window, Options
- Observing the three-way handshake (SYN → SYN-ACK → ACK)
- TCP connection options: MSS, Window Scale, SACK, Timestamps
- Connection teardown: FIN vs RST
- Measuring throughput with IO Graphs (slow start, steady state)
- Delayed ACKs and flow control in the data transfer phase

---

## 🛠 Tools Used

| Tool | Purpose |
|---|---|
| Wireshark 4.x | Packet capture analysis |
| Kali Linux | Analysis environment |
| `.pcap` trace file | Pre-recorded TCP session over HTTP |

---

## 📡 The Trace

| Role | IP Address | Port |
|---|---|---|
| Client | 192.168.1.122 | Ephemeral (60643) |
| Web Server | 64.238.147.113 | 80 (HTTP) |

Total packets captured: **1172**
Session type: **HTTP file download over TCP**

---

## 🤝 Three-Way Handshake

The connection opened with the classic SYN → SYN-ACK → ACK sequence.

![TCP Header expanded in Wireshark middle panel](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/tlp7m1sl8io333a8kxkl.png)

| Packet | Direction | Flags | Seq | Ack |
|---|---|---|---|---|
| 1 (t=0.000000) | Client → Server | SYN | 0 | — |
| 2 (t=0.088010) | Server → Client | SYN, ACK | 0 | 1 |
| 3 (t=0.088080) | Client → Server | ACK | 1 | 1 |

**RTT measured from trace: 88 ms**

Wireshark filter to isolate handshake:
```
tcp.flags.syn == 1
```

![SYN filter applied in Wireshark](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/jgx8a4p4sdhkvplbidbq.png)

---

## 🧱 TCP Segment Structure

```
|  Source Port (2B)  |  Dest Port (2B)  |
|       Sequence Number (4B)            |
|    Acknowledgement Number (4B)        |
| Header Len + Flags (2B) | Window (2B) |
|   Checksum (2B)  |  Urgent Ptr (2B)  |
|       Options (variable, if any)      |
|            Payload (N bytes)          |
```

![Raw hex bytes panel in Wireshark](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/kdhmpy0u1zwuvss0gvr6.png)

Notable behaviors observed:
- Header Length and Flags are packed into 2 bytes — Wireshark splits and labels them
- Options were present on most packets and always padded to a multiple of 4 bytes
- Pure ACK segments had no payload and were only ~66 bytes total

---

## ⚙️ Connection Options (Negotiated on SYN)

| Option | Purpose |
|---|---|
| MSS (Maximum Segment Size) | Max segment the receiver can accept |
| Window Scale | Extends window field beyond 65535 bytes |
| SACK Permitted | Selective acknowledgement for loss recovery |
| Timestamps | RTT estimation on each packet |
| NOP / End of Option List | 4-byte alignment padding |

---

## 📊 IO Graph — Throughput Over Time

**Statistics → IO Graph** with these settings:
- X-Axis: Tick interval = `0.1 sec`, Pixels per tick = `10`
- Y-Axis: Unit = `Bits/Tick`
- Graph 1: `tcp.srcport == 80` (download)
- Graph 2: `tcp.dstport == 80` (upload / ACK traffic)

![IO Graph showing download and ACK traffic](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/f1fs4ex395kk691b6u4d.png)

| Direction | Rate |
|---|---|
| Download (steady state) | ~2.5 Mbps / 250 packets/sec |
| Upload (ACK only) | ~60,000 bits/sec / 120 packets/sec |
| Payload efficiency | ~95% (1368 of 1434 bytes per packet) |

The initial ramp-up visible in the graph is **TCP slow start** — the protocol probing for available bandwidth before committing to full rate.

---

## 🔄 Data Transfer Observations

- **Delayed ACKs** — one ACK sent per ~2 data packets received, halving ACK overhead
- **Increasing Seq on server packets** — as download progresses, server's sequence numbers rise
- **Flat Seq on client packets** — client sends no new data after the initial HTTP GET
- **Window Size** stayed well above zero throughout — no flow control stalls occurred

---

## 🔌 Connection Teardown

This trace ended with an abrupt **RST (Reset)** rather than a graceful FIN exchange:

| Packet | Direction | Flags | Seq | Ack |
|---|---|---|---|---|
| Last | Client → Server | RST | 146 | 1056827 |

RST requires no acknowledgement — the connection closes immediately on both sides.

Wireshark filters for teardown:
```
tcp.flags.reset == 1
tcp.flags.fin == 1
```

---

## 🗂 Observation Table

| Obs. | Packet | Time | What I Saw |
|---|---|---|---|
| a | 1 | 0.000000 | Client sends SYN — connection initiated |
| b | 2 | 0.088010 | Server replies SYN-ACK |
| c | 3 | 0.088080 | Client sends ACK — handshake complete |
| d | 4 | 0.088579 | Client sends HTTP GET request |
| e | 5 | 0.177819 | Server ACKs the GET |
| f | 6 | 0.178321 | Server sends HTTP response with PSH+ACK |
| g | 7 | 0.178388 | Client ACKs first data segment |
| h | 8 | 0.189114 | Server sends next TCP data segment |
| i | 9 | 0.266705 | Server continues streaming |
| j | 10 | 0.266787 | Client sends delayed ACK |

---

## ⚠️ Common Mistakes

| Mistake | Reality |
|---|---|
| Confusing Seq and ACK | Seq = my position; ACK = what I expect from you |
| Trusting Wireshark Seq numbers as absolute | They are relative by default |
| Using TCP Stream Graph for receiver-side traces | Use IO Graph instead |
| Ignoring Window field | Zero window = connection stall |
| Expecting one ACK per packet | Delayed ACK halves the ACK count |
| Treating RST and FIN the same | RST = hard cut; FIN = graceful symmetric close |

---

## 📖 Full Write-Up

The detailed article with step-by-step explanation, filters, and analysis is published on my website:

👉 [How I Used Wireshark to Dissect a Real TCP Connection — From Handshake to Teardown](https://dev.to/almahmudkhalif/how-i-used-wireshark-to-dissect-a-real-tcp-connection-from-handshake-to-teardown-9de)

---

## 🌐 Connect With Me

- **Dev.to** → [dev.to/almahmudkhalif](https://dev.to/almahmudkhalif)
- **LinkedIn** → [linkedin.com/in/almahmudkhalif](https://linkedin.com/in/almahmudkhalif)
- **GitHub** → [github.com/almahmudkhalif](https://github.com/almahmudkhalif)
