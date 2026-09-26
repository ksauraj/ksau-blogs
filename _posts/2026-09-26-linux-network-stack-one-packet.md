---
layout: blog
title: "The Linux Network Stack, One Packet at a Time"
date: "2026-09-26"
excerpt: "What actually happens between your process calling send() and bytes leaving the wire. The ring buffer, softirqs, sockets and the queue of queues - the journey of one packet through the Linux network stack."
tags: ["linux", "networking", "kernel", "sockets", "observability", "devops"]
---

Every developer has sent data over a network. Very few can say what happened
between the moment their process called `write()` and the moment bytes
appeared on the other side of the world. There is a kernel-sized black box in
between, and it does a surprising amount of work.

This post opens that box. We follow a single packet through the Linux network
stack - from a userspace socket, down into the kernel, across the NIC and out
onto the wire, and then the same journey in reverse for the reply. We keep it
concrete: real queues, real kernel subsystems, real reasons each step exists.

No kernel experience needed. If you have run `ping`, seen a `tcpdump` output,
or wondered why your load balancer needs multiple cores to keep up - this is
the post for you.

## The one idea to keep in your head

The network stack is a set of **queues** with **workers** on both sides.
Packets do not teleport. They sit in a queue, wait for a worker to pick them
up, get processed, and move to the next queue.

At every boundary there is the same shape: something produces packets into a
queue, something consumes them, and if the producer is faster than the
consumer, the queue grows and packets wait (or get dropped). The whole art of
network performance is keeping those queues short.

> Keep this sentence: **a network packet is a job that moves through a
> pipeline of queues, and each step either processes it or queues it.**

## The journey, mapped out

Here is the full route a packet takes outbound, before we walk each leg:

```mermaid
graph LR
    APP["your app<br/>(process)"] -->|"send()"| SOCK["socket"]
    SOCK -->|"copy skb"| TCP["TCP layer"]
    TCP -->|"route + segment"| IP["IP / routing"]
    IP -->|"build skb"| QDISC["qdisc (tx queue)"]
    QDISC -->|"dequeue"| DRIVER["NIC driver"]
    DRIVER -->|"DMA ring"| NIC["NIC hardware"]
    NIC -->|"wire"| NET["internet"]
```

That is the outbound path. Each arrow is a real step where a queue or a data
structure changes hands. We now walk each one.

## The socket: your process's door to the kernel

It all starts with a **socket**. A socket is a file descriptor - a number your
process can `read()` and `write()` to - that the kernel has wired up to a
network connection instead of a file on disk.

When your app calls `send(fd, data, len, 0)`, here is what happens in the
kernel:

1. The kernel looks up the file descriptor and finds the socket structure.
2. It allocates an **skb** - a `struct sk_buff`, the kernel's name for a
   network packet. This is the fundamental unit of network I/O in Linux.
3. It copies your userspace `data` into the skb's data area.
4. It hands the skb to the protocol layer - TCP for a TCP socket.

The socket layer is your process's only window into the network. Everything
after this point happens in kernel context, on behalf of your process, but
outside your process's address space.

## The TCP layer: segmentation, windows and the state machine

If your socket is a TCP socket (nearly all web traffic is), the next stop is
the TCP layer. This is where the famous TCP behavior lives.

The key concept is the **MTU** (Maximum Transmission Unit) - the largest
packet the link can carry, usually 1500 bytes on Ethernet. Your app may call
`send()` with 64 KB of data, but the kernel cannot put 64 KB in one Ethernet
frame. So the TCP layer **segments** the data into chunks that fit:

```text
your 64 KB write()  →  ~44 packets of ~1460 bytes each
```

Each segment gets a sequence number, a destination port, a window size, and
gets queued for transmission. If the other side does not acknowledge them fast
enough, the sender's **congestion window** shrinks and it sends more slowly.
That slow start / congestion avoidance dance is the core of TCP and it all
happens here, invisibly, inside the kernel.

The kernel does one more thing here: it attaches a **socket buffer** (`skb`)
with all the TCP headers filled in use, ready for the next layer. It then asks
the routing layer: *which interface does this destination use, and what is the
next hop?*

## The IP and routing layer: picking the exit

The routing layer answers the "which way out" question. It looks up the
destination IP in the **routing table** (`ip route`) and decides:

- which **interface** the packet leaves through (`eth0`, `eth1`, ...)
- which **gateway** receives it next, if the destination is not local
- what **source address** to use

This is also where the kernel may apply **iptables / nftables** rules in the
`OUTPUT` and `POSTROUTING` chains - the layer where NAT masquerading and
firewall policy live. The skb is now fully addressed and leaves the routing
layer, heading for the transmit queue.

## The qdisc: the queue that shapes your traffic

Before a packet can reach the NIC driver, it sits in a **qdisc** (queueing
discipline). This is a userspace-visible, tunable queue - you have seen it
with `tc` (traffic control).

By default, most interfaces use a simple FIFO queue (`pfifo_fast`). But the
qdisc is where the kernel does its shaping: `tc` can add a `htb` or `fq_codel`
qdisc that reorders, drops, or rate-limits packets. This is the queue that
determines a packet's **latency** under load - the one that makes `ping` spike
when the link is saturated.

The qdisc hands packets to the NIC driver one at a time, when the driver says
it can accept more.

## The NIC driver and the ring buffer

Now we are at the bottom. The **NIC driver** owns two ring buffers - one for
transmit (TX) and one for receive (RX) - and the **DMA ring** is the actual
hardware interface.

The driver allocates a set of **descriptors** (small structures pointing at
kernel memory where the packet data lives) and hands them to the NIC. The NIC
uses **DMA** (Direct Memory Access) to read the packet data straight out of
RAM and onto the wire - the CPU does not copy the bytes one at a time. The
hardware does the copy in parallel while the CPU does other work.

This is the key to high-performance networking: the CPU sets up the descriptor,
the hardware does the bulk copy, and the CPU only gets interrupted when the
packet is fully gone.

```text
CPU    →  builds skb, fills descriptor, tells NIC "go"
NIC    →  DMA-reads bytes out of RAM onto the wire
NIC    →  interrupts CPU "done"
```

## The receive path: the same story, reversed

Now the reply comes back. The receive path is the mirror image, with one
important difference: the CPU gets interrupted a lot more.

1. The NIC receives bytes and **DMA-writes** them into the RX ring buffer -
   kernel memory it has been told about in advance.
2. The NIC raises a **hard IRQ**. The kernel's interrupt handler does the
   minimum - it acknowledges the hardware and schedules a **softirq**.
3. A **softirq** (software interrupt, running in kernel context but allowed to
   be deferred and preempted) runs on a CPU core and pulls packets out of the
   RX ring.
4. The softirq walks the packet up: IP reassembly, TCP delivery, and finally
   waking your process by putting the data into the socket's receive queue.
5. Your process's `recv()` returns with the data.

The receive path is why people say network performance is about **interrupt
mitigation**. Each incoming packet is an interrupt, and a machine drowning in
interrupts spends all its time being interrupted instead of doing work. Modern
drivers use **NAPI** (New API) - they switch from interrupt mode to *polling*
mode when packets arrive faster than a threshold, letting the CPU process many
packets per interrupt.

## Why this matters: three practical lessons

Here is where the theory pays rent, with three real situations you will meet.

### Lesson 1: Why one core gets saturated

Because softirqs process packets, and softirqs run on a CPU core, a high
packet rate can pin one core at 100% handling interrupts - while the other
cores sit idle. This is the classic "one core on fire" load balancer problem.
The fix is usually **RPS** (Receive Packet Steering) or a multi-queue NIC
(`ethtool -L`) so incoming packets spread across cores.

### Lesson 2: Why tuning the NIC ring matters

The RX ring has a finite size. If the softirq cannot drain it fast enough, the
ring fills and the NIC starts dropping packets (`ethtool -S` shows `rx_dropped`
climbing). Raising the ring size (`ethtool -G eth0 rx 4096`) gives the kernel
more headroom, at the cost of memory and latency. This is a concrete lever you
can pull today.

### Lesson 3: Why `tc` and qdiscs add latency

Every qdisc is a queue, and queues add delay. A default FIFO queue holds
packets in order, so a burst of bulk transfers queues a small interactive
packet behind them - that is your `ping` spike. `fq_codel` (the fair queue
controlled delay) exists precisely to give small interactive flows priority
and keep latency low. That is why modern distros default to it.

## The one mental model to keep

Put it all together and the stack is simple:

> A packet is a **job** that moves through a **pipeline of queues**. Your
> socket produces it, the kernel's protocol layers process it, the qdisc
> queues it, the NIC's DMA ring carries it onto the wire - and on the way back,
> interrupts and softirqs feed it up to your socket.

Each leg is a place where work happens, where queues can grow, and where you
have a tuning knob. The stack is not a magic black box; it is a set of
well-named queues with well-understood behavior.

Now, when you see `rx_dropped` climb or one core peg out under traffic, you
know exactly which queue is overflowing and which knob to turn. That is the
whole point of understanding the stack - not to memorize layers, but to know
where the work actually happens.