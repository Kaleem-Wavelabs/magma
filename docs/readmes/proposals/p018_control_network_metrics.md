---
id: p018_control_network_metrics
title: Export Control Path Network Metrics
hide_title: true
---

# Proposal: Export Control Path Network Metrics

Author(s): @waqaraqeel

Last updated: 07/13/2021

Discussion at
[https://github.com/magma/magma/issues/8028](https://github.com/magma/magma/issues/8028).

## Context & scope

Services running on the access gateway communicate with the orc8r and generate
control traffic. We do not have visibility into the amount, patterns, and
service-level breakdown of this traffic. As we try to minimize control plane
bandwidth consumption for our deployment partners, especially those with
satellite backhaul, visibility into bandwidth consumption is crucial. This
proposal details a **design for collecting and exporting control plane network
metrics from the access gateway to the orchestrator and NMS**.

### Goals

1. On the access gateway, record byte and packet counts grouped by service,
destination IP address, and destination port.

2. Export these counters as Prometheus metrics on the NMS UI.

3. Minimize performance penalty on the gateway for network metrics collection.

4. Minimize required infrastructure changes for metrics export and for
deployment.


### Non-goals

1. Change collection/export methods for existing data-path metrics.

## Proposal

To collect relevant metrics, i.e., byte and packet counters, we propose using
an [eBPF](https://ebpf.io/what-is-ebpf/) program. eBPF is a modern Linux kernel
feature that allows running sandboxed programs inside the kernel without having
to change kernel source code or loading privileged/risky kernel modules. The
Linux kernel verifies eBPF programs to ensure safety and termination, and
provide certain performance guarantees [1].

We will use the [BCC toolkit](https://github.com/iovisor/bcc/) for writing and
loading our eBPF monitoring program. BCC makes it easier to write eBPF programs
by providing clean abstractions for kernel instrumentation, and Python and Lua
libraries for writing user-space front-end programs that communicate with the
kernel-space eBPF program. We will write a Python front-end since many of our
existing services already use Python, and we have convenient infrastructure for
exporting Prometheus metrics from Python services.

We will create a new Python service called `kernsnoopd` (not `netsnoopd` as
this service can later become a home for more observability through eBPF). Upon
service start, `kernsnoopd` will compile the eBPF kernel instrumentation
program and load it. It will read `sampling_period` from its configuration.
Every `sampling_period` seconds, `kernsnoopd` will read the counters from the
eBPF program into Prometheus Gauges and clear the eBPF counters. Every magma
service running on the orchestrator will get a separate Prometheus Gauge by the
name `{service}_bytes_sent` and `{service}_packets_sent`. For each Gauge,
labels will indicate `networkID`, `gatewayID`, and `dstIP:dstPort`.

Once Prometheus Gauge values have been set, they will follow the existing Magma
metrics path: the `magmad` service will read the Gauges from `kernsnoopd` and
upload them to `metricsd` at the orchestrator every `sync_interval` seconds.
The NMS UI will then display these metrics.

[insert schematic]

This design achieves [goals](#goals) #1 and #2. The choice of eBPF minimizes
performance penalty (discussed [here](#performance)) by putting compiled code
into the kernel and avoiding raw packet captures. We also utilize the existing
metrics plumbing for Python services.

Now we show a prototype of the eBPF program `kernsnoopd` will use:

```c
#include <bcc/proto.h>
#include <linux/sched.h>
#include <net/inet_sock.h>

struct counters_t {
    u64 bytes;
    u64 packets;
};

struct key_t {
    u32 pid;
    u32 daddr;
    u16 dport;
};

// Create a hash map with `key_t` as key type and `counters_t` as value type
BPF_HASH(dest_counters, struct key_t, struct counters_t, 1000);

// Attach hook for the `net_dev_start_xmit` kernel trace event
TRACEPOINT_PROBE(net, net_dev_start_xmit)
{
    struct sk_buff* skb = (struct sk_buff*) args->skbaddr;
    struct sock* sk = skb->sk;

    // read destination IP address, port and current pid
    struct key_t key = {};
    bpf_probe_read(&key.daddr, sizeof(sk->sk_daddr), &sk->sk_daddr);
    bpf_probe_read(&key.dport, sizeof(sk->sk_dport), &sk->sk_dport);
    key.pid = bpf_get_current_pid_tgid() >> 32;

    // lookup or initialize item in dest_counters
    struct counters_t empty;
    __builtin_memset(&empty, 0, sizeof(empty));
    struct counters_t *data = dest_counters.lookup_or_try_init(&key, &empty);

    // increment the counters
    if (data) {
        data->bytes += skb->len;
        data->packets++;
    }
    return 0;
}
```

This program is written in restricted C (prevents direct memory access, for
example) that is suitable for LLVM compilation into BPF bytecode. It uses
macros from BCC to create a hash map data structure (`dest_counters`), and
attach a callback function to kernel trace event
[`net_dev_start_xmit`](https://patchwork.ozlabs.org/project/netdev/patch/1389392223.2025.125.camel@bwh-desktop.uk.level5networks.com/).
The kernel fires this event just before it starts a packet transmission. Our
callback function reads the appropriate context and increments relevant
counters in the hash map.

Below is an example of a front-end Python program that retrieves and displays
the counters aggregated in the kernel:

```python
import time

from bcc import BPF
from socket import inet_ntop, ntohs, AF_INET
from struct import pack

INTERVAL = 3
TASK_COMM_LEN = 16 # linux/sched.h

def print_table(table):
    print("----------------------------")
    for k, v in table.items():
        daddr = inet_ntop(AF_INET, pack("I", k.daddr))
        dport = ntohs(k.dport)
        print(f"{k.pid} {daddr}:{dport} {v.bytes} {v.packets}")
    print("----------------------------")

if __name__ == "__main__":
    # compile and load eBPF program from source file
    b = BPF(src_file="transmit_trace.c")

    # print and clear table every INTERVAL seconds
    while True:
        time.sleep(INTERVAL)
        table = b["dest_counters"]
        print_table(b["dest_counters"])
        table.clear()
```

## Alternatives considered

Here, we enumerate alternatives to the above design that we considered:

1. [**`libpcap`**](https://www.tcpdump.org/): There are several existing
monitoring tools based on `libpcap` such as
[Nethogs](https://github.com/raboof/nethogs) and
[iftop](http://www.ex-parrot.com/~pdw/iftop/). While these tools do not collect
the exact metrics required, it should be straightforward to modify them or
write a new tool based on `libpcap`. The larger issue is that `libpcap` will
make one or more copies of the packet, which incurs significant overhead
[citation needed]. Moreover, we do not see any advantages of `libpcap` over
eBPF.

2. [**`nghttpx`**](https://nghttp2.org): We use `control_proxy` to proxy TLS
connections from individual services to the orchestrator. However, `nghttpx`,
the proxy implementation we use does not collect per-process or per-destination
byte and packet counters. `nghttpx` could probably be modified to collect these
statistics, but that would probably be higher development effort and may yield
worse performance. Also, it will not work if `proxy_cloud_connections` is set
to `False` and services are connecting to the cloud directly .

3. [**`cilium/epbf`**](https://github.com/cilium/ebpf): This is a pure Go eBPF
library alternative to BCC. `cilium/ebpf` has minimal external dependencies and
delivers a single binary that will contain the eBPF program, its loader and the
front-end to communicate with it. It provides a `bpf2go` tool which compiles an
eBPF program written in C and generates a Go file containing the compiled
bytecode. The API is less convenient than that of BCC. `cilium/ebpf` API is
also explicitly unstable. They state that programs will have to be updated for
future versions of the library. Moreover, we do not have a Prometheus metrics
export mechanism for Go services so that work will have to be duplicated from
Python. In short, we did not pick `cilium/ebpf` because BCC is easier to work
with, more mature, and more popular (hence more support available), and also
allows us to use existing metrics support in Python.

4. [**`iovisor/gobpf`**](https://github.com/iovisor/gobpf): These are Go
bindings for BCC. `gobpf` would be a good choice if our metrics export system
was in Go and not in Python.

5. [**`cloudflare/ebpf_exporter`**](https://github.com/cloudflare/ebpf_exporter):
This uses `gobpf` behind the scenes. It handles compiling and loading the
provided eBPF C program just like Python tools for BCC do. It allows the
frontend program to be generated from a YAML specification. `ebpf_exporter` may
have been a good choice if we were using standard Prometheus metrics collection
and export methods, but we do not. Hence, we prefer an explicit Python frontend
program.

## Cross-cutting concerns

There are several concerns that this proposal attempts to address:

### Performance

Performance is a major concern for this proposal as we are putting code into
the kernel's network path on the access gateway. Although we have not
benchmarked the prototype above, there is a similar tool included in the BCC
distribution called `netqtop`. The authors of `netqtop` have benchmarked their
tool and observe 1.17 usec increase in packet "pingpong" latency [2]. When they
benchmark bandwidth on the loopback interface, they observe a drop of around
27% in packets per second (PPS). Since `netqtop` instruments both transmit and
receive network paths, and does more processing than we need, we expect the
performance penalty of `kernsnoopd` to be less than half of `netqtop`. We think
the performance penalty will be closer to the `getpid()` benchmark from
`ebpf_exporter` [8]: a few hundred nanoseconds per call. However, even that on
the network path may be significant and would warrant concern if network
utilization on our gateways is very high and if they are CPU-bottlenecked (see
[Open Issues](#open-issues).

One way to slash the performance penalty is to take the instrumentation out of
the network path. We could, for example, use the TCP tracepoint
`sock:inet_sock_set_state` instead of `net_dev_start_xmit`.
`sock:inet_sock_set_state` is fired when the kernel changes the state of a
socket [3]. We could use this event to collect metrics before a socket closes.
However, this means that we will not be able to observe non-TCP traffic and
there might be precision issues for long-running TCP streams.

A minor performance concern relates to compiling the kernel instrumentation
code from C to BPF bytecode and loading it onto the kernel. We have benchmarked
the prototype program above on `magma` AGW VM and across 50 runs, it takes an
average of 1.75 sec for BCC to read the program from a separate source file,
compile it and load it into the kernel. Since this will only happen when the
`kernsnoopd` service starts, it is not cause for concern.

### Deployment

This design will require BCC tools and Linux kernel headers to be installed on
the access gateway. Kernel headers are already installed. BCC is delivered
through apt packages `bpfcc-tools`, `libbpfcc`, and `python3-bpfcc`. Combined,
these packages have a download size of 15.5 MB and take up 63 MB of disk space
after installation (see [Open Issues](#open-issues)).

### Compatibility

eBPF was originally introduced to the Linux kernel in 2014 with kernel version
3.15 [5]. It has been evolving since then, and the latest TCP tracepoints we
talk about were introduced in kernel version 4.16 [3]. This is not a concern
though as our access gateways are on Ubuntu Focal with kernel version 5.8+.

### Observability and Debug

The Prometheus metrics pipeline from `magmad` gateway service to the NMS UI is
mature and well-tested. Appropriate `nosetests` will be included in the
implementation of this proposal to support validation and debugging for
collected network metrics.

### Security & privacy

The compiled instrumentation code from this proposal will be loaded into the
kernel, but there should not be security concerns because eBPF statically
verifies programs before loading and provides security guarantees. Unchecked
memory access is not allowed, neither are unbounded loops [6]. The total number
of instructions allowed is also limited [7].

The in-kernel program will be able to observe coarse network statistics for
processes running on the access gateway. Individual packets will not be
captured or inspected. Command line arguments of running processes will be
read, but they will not be stored anywhere.

## Open issues

- Is 63 MB of additional disk space required to install `bpfcc-tools` on the
access gateway really not a problem?
- Is network utilization on our access gateway usually close to 100%? Are our
gateways starving for CPU?
- We do not want to instrument the data path. Should we observe all interfaces
on the gateway? If not, what subset should we observe?

## References

[1]: Jay Schulist, Daniel Borkmann, Alexei Starovoitov. 2018. Linux Socket
Filtering aka Berkeley Packet Filter (BPF).
https://www.kernel.org/doc/Documentation/networking/filter.txt

[2]: yonghong-song. 2020. Netqtop 3037.
https://github.com/iovisor/bcc/pull/3048

[3]: Brendan Gregg. 2018. TCP Tracepoints.
https://www.brendangregg.com/blog/2018-03-22/tcp-tracepoints.html

[4]: pflua-bench. 2016. https://github.com/Igalia/pflua-bench

[5]: Alexei Starovoitov. 2014. net: filter: rework/optimize internal BPF
interpreter's instruction set.
https://www.kernel.org/doc/Documentation/networking/filter.txt

[6]: Alexei Starovoitov. 2019. bpf: introduce bounded loops.
https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next.git/commit/?id=2589726d12a1b12eaaa93c7f1ea64287e383c7a5

[7]: Quentin Monnet. 2021. eBPF Updates #4: In-Memory Loads Detection,
Debugging QUIC, Local CI Runs, MTU Checks, but No Pancakes.
https://ebpf.io/blog/ebpf-updates-2021-02

[8]: Ivan Babrou. 2018. eBPF overhead benchmark.
https://github.com/cloudflare/ebpf_exporter/tree/master/benchmark
