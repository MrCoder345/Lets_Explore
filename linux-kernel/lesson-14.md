# Lesson 14 — Networking Stack (Socket → TCP/IP → NIC)

## 1. Networking Architecture

In our previous lessons, we learned how the kernel reads data from local disks. But how does the kernel send and receive data across the globe?

The Linux networking stack is a masterpiece of software engineering. It implements the standard OSI/TCP/IP network models in a highly layered, modular way.

Here is the high-level architecture:

1. **Application Layer:** User-space programs calling `send()` or `recv()`.
2. **Socket Layer:** The generic kernel interface bridging user-space and network protocols.
3. **Transport Layer (L4):** TCP and UDP (Port numbers, reliability).
4. **Network Layer (L3):** IP (IP addresses, routing).
5. **Data Link Layer (L2):** Ethernet (MAC addresses, framing).
6. **Physical Layer (L1):** The physical Network Interface Card (NIC) and cables.

## 2. Socket Layer

The problem: User-space programs need a standard way to talk to *any* network protocol (IPv4, IPv6, Bluetooth, Unix Domain Sockets) without changing their code.

The solution: **The BSD Socket Interface**.
In Linux, everything is a file. When you call the `socket()` system call, the kernel creates a `struct socket`. This structure provides the standard VFS file interface (so you can use `read()` and `write()` on it).

Beneath `struct socket` lies `struct sock`. While `socket` is the generic file abstraction, `struct sock` is the network-specific abstraction. It holds the socket's state, receive/transmit queues, and pointers to the specific protocol (TCP/UDP) operations.

**Directory path:** `net/socket.c` and `net/core/sock.c`

## 3. TCP (Transmission Control Protocol)

TCP provides a reliable, ordered, and error-checked stream of data.

To achieve this, the kernel implements a massive, complex **State Machine** (ESTABLISHED, TIME_WAIT, CLOSE_WAIT). The TCP layer handles:

- **Segmenting:** Breaking massive data streams into network-sized chunks (Maximum Segment Size - MSS).
- **Retransmission:** Starting timers and re-sending data if acknowledgments (ACKs) are not received.
- **Congestion Control:** Algorithms like CUBIC or BBR that calculate exactly how fast to send data without overwhelming the internet routers.

**Directory path:** `net/ipv4/tcp.c`

## 4. UDP (User Datagram Protocol)

UDP is "fire and forget." It has no state, no reliability, and no congestion control.
Because of this, the UDP layer in the kernel is incredibly thin. It simply adds an 8-byte header (Source Port, Destination Port, Length, Checksum) and immediately passes the data down to the IP layer.

**Directory path:** `net/ipv4/udp.c`

## 5. IP Layer (Internet Protocol)

The IP layer handles **logical addressing** and **fragmentation**.

- It slaps an IPv4 or IPv6 header onto the data.
- It decrements the TTL (Time To Live). If TTL hits 0, it drops the packet to prevent infinite internet loops.
- If a packet is larger than the network's Maximum Transmission Unit (MTU), the IP layer fragments it into smaller pieces and reassembles them on the receiving end.

**Directory paths:** `net/ipv4/ip_output.c` and `net/ipv4/ip_input.c`

## 6. Routing

Before the IP layer can send a packet, it must know *where* to send it. Does it go to the local loopback interface (`lo`), or out to the internet via the default gateway?

The kernel maintains a **FIB (Forwarding Information Base)**. The routing subsystem (`net/ipv4/route.c`) queries this table. The result of a routing lookup attaches a `dst_entry` (Destination Entry) to the packet, giving it strict instructions on which physical network device (`net_device`) must transmit it.

## 7. Netfilter (The Firewall)

Linux has a built-in framework for packet mangling, NAT (Network Address Translation), and firewalls called **Netfilter** (the backend for `iptables` and `nftables`).

Netfilter places **Hooks** at strategic points in the networking stack:

- `NF_INET_PRE_ROUTING`: Right after the packet arrives.
- `NF_INET_LOCAL_IN`: Right before it is delivered to a local socket.
- `NF_INET_POST_ROUTING`: Right before it hits the physical wire.

At any hook, Netfilter can evaluate rules and decide to `NF_DROP` (silently kill) or `NF_ACCEPT` (allow) the packet.

## 8. `sk_buff` (Socket Buffer)

This is the single most important data structure in Linux networking.

**The Problem:** As a packet travels down the stack, TCP adds a header, IP adds a header, and Ethernet adds a header. If we allocated a new block of memory and copied the payload every time we added a header, performance would be terrible.

**The Solution:** The `struct sk_buff`. The kernel allocates a single, large block of memory once. The `sk_buff` contains pointers that dynamically move to reserve space for headers without moving the actual data!

```text
[ Headroom (Empty) ] [ TCP Header ] [ Payload Data ] [ Tailroom (Empty) ]
^                    ^                               ^                    ^
head                 data                            tail                 end
```

- `skb_reserve()`: Moves `data` and `tail` forward to leave blank space at the start (Headroom) for future headers.
- `skb_push()`: Moves the `data` pointer backward to write a new header (e.g., when moving from TCP down to IP).
- `skb_pull()`: Moves the `data` pointer forward to strip a header (e.g., when receiving an IP packet and passing it up to TCP).

**Directory path:** `include/linux/skbuff.h`

## 9. NAPI (New API)

Historically, every time a network packet arrived, the NIC fired a hardware interrupt. On a 10 Gigabit network receiving 1,000,000 packets per second, this caused an **Interrupt Storm**. The CPU spent 100% of its time just context-switching into the interrupt handler, completely freezing the OS (Livelock).

**NAPI** solves this by combining Interrupts with Polling:

1. First packet arrives -> Hardware Interrupt fires.
2. The kernel disables further NIC hardware interrupts!
3. The kernel schedules a SoftIRQ (Bottom Half - Lesson 8) to `napi_poll()`.
4. The CPU actively polls the NIC's hardware ring-buffer, pulling thousands of packets off the wire in a single batch.
5. When the queue is empty, the kernel re-enables hardware interrupts.

## 10. NIC Driver (`struct net_device`)

Just like the VFS uses `struct file_operations`, the networking stack uses `struct net_device` to represent a physical or virtual network interface (`eth0`, `wlan0`, `docker0`).

Drivers populate `net_device_ops` with function pointers like:

- `.ndo_start_xmit`: The function the kernel calls to physically transmit a packet.
- `.ndo_set_mac_address`: To change the MAC address.

**Directory path:** `drivers/net/ethernet/`

## 11. Packet Receive Flow (Rx)

Let's trace a packet arriving from the internet:

1. **Hardware:** NIC receives electrons, validates the MAC address, and uses DMA to copy the packet directly into RAM.
2. **Hard IRQ:** NIC fires an interrupt. The Top Half masks interrupts and schedules NAPI.
3. **SoftIRQ (NAPI):** `net_rx_action()` runs. It calls the driver's `napi_poll()` function to harvest the `sk_buff`s from RAM.
4. **Core Stack:** The driver calls `netif_receive_skb()`. This is the entry point to the protocol stack.
5. **IP Layer:** Checks the IP header, verifies the checksum, consults routing, and passes it to L4.
6. **TCP Layer:** Validates sequence numbers, acknowledges data, and adds the payload to the socket's receive buffer.
7. **Wake Up:** TCP calls `wake_up()` on the socket's Wait Queue (Lesson 11).
8. **User Space:** The sleeping `recv()` system call wakes up, copies the data from the `sk_buff` to the user's buffer via `copy_to_user()`, and returns!

## 12. Packet Transmit Flow (Tx)

Let's trace a packet leaving the server:

1. **User Space:** `send(fd, buffer, length)` is called.
2. **Socket/TCP:** The kernel allocates an `sk_buff`, reserves headroom, copies the user data into it, and uses `skb_push()` to prepend the TCP header.
3. **IP Layer:** `skb_push()` is used again to prepend the IP header. Routing determines the outgoing `net_device`.
4. **Traffic Control (qdisc):** The packet enters a Queuing Discipline (like `fq_codel`). This layer handles QoS (Quality of Service) and rate-limiting.
5. **Driver:** The kernel calls `dev_queue_xmit()`, which eventually triggers the driver's `ndo_start_xmit()`.
6. **Hardware:** The driver maps the `sk_buff` memory for DMA and rings the hardware doorbell. The NIC pulls the packet from RAM and sends it out the wire.

## 13. Important Structs

- `struct socket`: The VFS-facing socket representation.
- `struct sock`: The network-facing socket state machine.
- `struct sk_buff`: The ultimate packet buffer (metadata + data pointers).
- `struct net_device`: The abstraction of a network interface card.
- `struct napi_struct`: The scheduling context for the NAPI polling mechanism.

## 14. Important APIs

- `alloc_skb()`: Allocates a new socket buffer.
- `skb_reserve()`, `skb_push()`, `skb_pull()`, `skb_put()`: Pointer manipulation for zero-copy header addition/removal.
- `netif_receive_skb()`: Hands a received packet from the driver to the kernel stack.
- `dev_queue_xmit()`: Hands a transmit packet from the stack down to the driver.
- `napi_schedule()`: Triggers the SoftIRQ polling mechanism.

## 15. Source Files to Explore

- `net/core/dev.c`: The core network device layer (`netif_receive_skb`, `dev_queue_xmit`).
- `net/ipv4/tcp.c`: The massive TCP implementation.
- `include/linux/skbuff.h`: The `sk_buff` struct and its inline pointer manipulation functions.
- `include/linux/netdevice.h`: The `net_device` struct and driver API.
- `drivers/net/ethernet/intel/e1000/`: A great, readable example of a standard Gigabit Ethernet driver.

## 16. Mental Model

- **Transmit (The Assembly Line):** User data is placed on a conveyor belt (`sk_buff`). As it rolls down the factory (TCP -> IP -> Ethernet), robot arms (`skb_push`) bolt on headers. Finally, the shipping dock (`net_device`) loads it onto a truck (DMA to the NIC).

- **Receive (The Mailroom):** A massive mailbag arrives (DMA). The clerk stops answering the phone (Disables Interrupts) and sorts the entire bag rapidly (NAPI Poll). They rip open the outer envelopes (`skb_pull`), read the inner addresses (IP Routing), and finally place the letters into the specific P.O. Boxes (Socket Receive Queues), ringing a bell (`wake_up`) to alert the waiting customers.

## 17. Summary

The Linux networking stack is a highly optimized, layered architecture. To prevent memory bottlenecks, it uses `sk_buff` structures to manipulate packets via pointer arithmetic (zero-copy) rather than physically moving data. To prevent CPU starvation during heavy traffic, it uses NAPI to dynamically switch from interrupt-driven receiving to high-speed polling. The entire stack elegantly connects user-space Sockets down through complex protocols to object-oriented NIC drivers.

## 18. Exercises

1. Open `include/linux/skbuff.h` and find `struct sk_buff`. Locate the `head`, `data`, `tail`, and `end` pointers.
2. In the same file (`skbuff.h`), search for `static inline void *skb_push`. Observe how it simply subtracts from `skb->data` and increments `skb->len`.
3. Open `include/linux/netdevice.h` and find `struct net_device_ops`. Scroll through the function pointers. Find `.ndo_start_xmit` and `.ndo_set_mac_address`.

## 19. Questions

1. When an application calls `send()` and the `skb_push()` function is used to add the TCP header, what happens if the kernel forgot to call `skb_reserve()` when it first allocated the `sk_buff`?

2. Why does NAPI wait for an interrupt to fire for the *first* packet, instead of just continuously polling the NIC 100% of the time?

3. Netfilter hooks allow firewalls to drop packets. Based on the Receive Flow, does the IP layer process the routing tables *before* or *after* the `NF_INET_PRE_ROUTING` hook evaluates the packet?

---

## 💡 Answer Key (For Your Reference)

**Q1:** The `skb_push()` function would overwrite the actual payload data. Without headroom, the `data` pointer is already at the start of the payload. Moving it backward (which `skb_push` does) would point to memory before the allocated buffer, causing memory corruption and likely a kernel panic or subtle data corruption. This is why `skb_reserve()` is mandatory before pushing headers.

**Q2:** Polling the NIC 100% of the time would consume **100% of one CPU core** even when the network is idle. This wastes power and CPU cycles that could be used for other tasks. The hybrid approach (interrupt for first packet, then poll while traffic is heavy) provides the best of both worlds: zero CPU usage during idle periods, and maximum throughput during heavy traffic.

**Q3:** `NF_INET_PRE_ROUTING` runs **before** IP routing decisions. This is crucial for firewalls and NAT: the hook must evaluate and potentially drop the packet *before* the kernel decides which local socket to deliver it to (for incoming packets) or which interface to send it out (for forwarded packets). This prevents malicious packets from being routed or delivered before they can be filtered.
