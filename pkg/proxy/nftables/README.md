# NFTables kube-proxy

This is an implementation of service proxying via the nftables API of
the kernel netfilter subsystem.

## General theory of netfilter

Packet flow through netfilter looks something like:

```text
             +================+      +=====================+
             | hostNetwork IP |      | hostNetwork process |
             +================+      +=====================+
                         ^                |
  -  -  -  -  -  -  -  - | -  -  -  -  - [*] -  -  -  -  -  -  -  -  -
                         |                v
                     +-------+        +--------+
                     | input |        | output |
                     +-------+        +--------+
                         ^                |
      +------------+     |   +---------+  v      +-------------+
      | prerouting |-[*]-+-->| forward |--+-[*]->| postrouting |
      +------------+         +---------+         +-------------+
            ^                                           |
 -  -  -  - | -  -  -  -  -  -  -  -  -  -  -  -  -  -  |  -  -  -  -
            |                                           v
       +---------+                                  +--------+
   --->| ingress |                                  | egress |--->
       +---------+                                  +--------+
```

where the `[*]` represents a routing decision, and all of the boxes except in the top row
represent netfilter hooks. More detailed versions of this diagram can be seen at
https://en.wikipedia.org/wiki/Netfilter#/media/File:Netfilter-packet-flow.svg and
https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks but note that in the the
standard version of this diagram, the top two boxes are squished together into "local
process" which (a) fails to make a few important distinctions, and (b) makes it look like
a single packet can go `input` -> "local process" -> `output`, which it cannot. Note also
that the `ingress` and `egress` hooks are special and mostly not available to us;
kube-proxy lives in the middle section of diagram, with the five main netfilter hooks.

There are three paths through the diagram, called the "input", "forward", and "output"
paths, depending on which of those hooks it passes through. Packets coming from host
network namespace processes always take the output path, while packets coming in from
outside the host network namespace (whether that's from an external host or from a pod
network namespace) arrive via `ingress` and take the input or forward path, depending on
the routing decision made after `prerouting`; packets destined for an IP which is assigned
to a network interface in the host network namespace get routed along the input path;
anything else (including, in particular, packets destined for a pod IP) gets routed along
the forward path.

## kube-proxy's use of nftables hooks

Kube-proxy uses nftables for four things:

  - Using DNAT to rewrite traffic from service IPs (cluster IPs, external IPs, load balancer
    IP, and NodePorts on node IPs) to the corresponding endpoint IPs.

  - Using SNAT to masquerade traffic as needed to ensure that replies to it will come back
    to this node/namespace (so that they can be un-DNAT-ed).

  - Dropping packets that are filtered out by the `LoadBalancerSourceRanges` feature.

  - Dropping packets for services with `Local` traffic policy but no local endpoints.

  - Rejecting packets for services with no local or remote endpoints.

This is implemented as follows:

  - We do the DNAT for inbound traffic in `prerouting`: this covers traffic coming from
    off-node to all types of service IPs, and traffic coming from pods to all types of
    service IPs. (We *must* do this in `prerouting`, because the choice of endpoint IP may
    affect whether the packet then gets routed along the input path or the forward path.)

  - We do the DNAT for outbound traffic in `output`: this covers traffic coming from
    host-network processes to all types of service IPs. Regardless of the final
    destination, the traffic will take the "output path". (In the case where a
    host-network process connects to a service IP that DNATs it to a host-network endpoint
    IP, the traffic will still initially take the "output path", but then reappear on the
    "input path".)

  - `LoadBalancerSourceRanges` firewalling has to happen before service DNAT, so we do
    that on `prerouting` and `output` as well, with a lower (i.e. more urgent) priority
    than the DNAT chains.

  - The `drop` and `reject` rules for services with no endpoints don't need to happen
    explicitly before or after any other rules (since they match packets that wouldn't be
    matched by any other rules). But with kernels before 5.9, `reject` is not allowed in
    `prerouting`, so we can't just do them in the same place as the source ranges
    firewall. So we do these checks from `input`, `forward`, and `output`, to cover all
    three paths. (In fact, we only need to check `@no-endpoint-nodeports` on the `input`
    hook, but it's easier to just check them both in one place, and this code is likely to
    be rewritten later anyway. Note that the converse statement "we only need to check
    `@no-endpoint-services` on the `forward` and `output` hooks" is *not* true, because
    `@no-endpoint-services` may include externalIPs/LB IPs that are assigned to local
    interfaces.)

  - Masquerading has to happen in the `postrouting` hook, because "masquerade" means "SNAT
    to the IP of the interface the packet is going out on", so it has to happen after the
    final routing decision. (We don't need to masquerade packets that are going to a host
    network IP, because masquerading is about ensuring that the packet eventually gets
    routed back to the host network namespace on this node, so if it's never getting
    routed away from there, there's nothing to do.)
