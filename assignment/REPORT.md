# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **5879**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

To reproduce the forwarding loop, I used the original tunnel_v1.p4 program, which forwards tunnel packets based on myTunnel.dst_id without checking any hop limit.

I modified the forwarding table on switch s2 so that packets with tunnel destination ID 3 would be sent back to s1 through port 1 instead of going to h3.

```
sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"
```

After that, I sent a packet from h1 with dst_id=3. The packet kept circulating between s1 and s2. From evidence/loop-before.txt, the packet-processing log counts were:

```
mininet> sh grep -c "Processing packet received" logs/s1.txt logs/s2.txt
logs/s1.txt:906
logs/s2.txt:905
mininet> sh sleep 5; grep -c "Processing packet received" logs/s1.txt logs/s2.txt
logs/s1.txt:1143
logs/s2.txt:1143
```

The counts continued increasing even though I only sent one packet. This shows that the original tunnel forwarding has no mechanism to stop a packet from circulating in a loop.

For normal IPv4 forwarding, the ipv4_forward action does decrement the IPv4 TTL. However, the program does not explicitly check whether the TTL has reached zero before forwarding and proceed to drop or any action. Since TTL is an 8-bit unsigned field, decrementing zero can cause it to wrap around to 255. Therefore, simply decrementing TTL is not sufficient to guarantee loop prevention in this P4 program.

```
    action ipv4_forward(egressSpec_t port) {
        standard_metadata.egress_spec = port;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }
```

Also, tunnel forwarding uses a separate action that does not decrement IPv4 TTL at all. This is why a dedicated tunnel hop limit is needed.

## 2. Design: the hop limit

I added a new hop_limit field to the custom tunnel header. Its purpose is to limit how many switches a tunnel packet can pass through before being dropped.

```
header myTunnel_t {
    bit<16> proto_id;
    bit<16> dst_id;
    bit<8>  hop_limit;
}
```
I chose an 8-bit field, allowing values from 0 to 255. This is enough for the small lab network and adds only one byte to each tunnel packet.

I placed hop_limit after dst_id so that the two existing fields keep their original positions. This also makes it easier for older switches to still recognise the tunnel destination, although it does not guarantee full compatibility.

In send.py, the default starting value is 8, with the option to change it using --hops. Since the lab only has two switches, 8 is enough for normal delivery while still keeping looped packets from circulating too many times.

The hop limit is checked in the ingress pipeline. If the value is already zero, the packet is dropped. Otherwise, it is decreased by one before the tunnel forwarding table is applied.

```
if (hdr.myTunnel.hop_limit == 0) {
    drop();
} else {
    hdr.myTunnel.hop_limit = hdr.myTunnel.hop_limit - 1;
    myTunnel_exact.apply();
}
```

Checking before decrementing is important because it prevents the value from wrapping around when it reaches zero.

The parser and deparser continue to extract and emit the complete tunnel header, including the new field. I also updated the sender and receiver so both use the same five-byte tunnel format.


## 3. Implementation

tunnel.p4
I added the 8-bit hop_limit field and updated the ingress logic to check and decrement it before tunnel forwarding. Packets with a zero hop limit are dropped, while normal IPv4 packets still use the existing IPv4 forwarding table.

send.py
I added the hop-limit field to the Scapy MyTunnel header and introduced the --hops option. The default is 8, but I can send packets with other values, such as 0 or 1, to test the expiration behaviour.

receive.py
I updated the Scapy tunnel definition to include hop_limit and modified the output so the receiver displays the remaining hop limit when a tunnelled packet arrives.


## 4. Evidence: it works

Normal tunnel delivery

I first tested delivery with both switches running the upgraded program. For a packet sent with --tunnel 2 --hops 8, h2 received the message with a remaining hop limit of 6. The hop limit decreased from 8 to 6 because the packet passed through s1 and s2. The same forwarding behaviour was tested toward h3, with the results recorded in evidence/delivery.txt.

```
mininet> h1 python3 send.py 10.79.0.3 "delivery-h2" --tunnel 2 --hops 8
mininet> h1 python3 send.py 10.79.0.3 "delivery-h3" --tunnel 3 --hops 8
mininet> sh cat logs/h2.txt logs/h3.txt
Listening on h2-eth0...
Received (tunnelled, dst_id=2, hop_limit=6) from 10.79.0.1 to 10.79.0.3, TTL 64: 'delivery-h2'
Listening on h3-eth0...
Received (tunnelled, dst_id=3, hop_limit=6) from 10.79.0.1 to 10.79.0.3, TTL 64: 'delivery-h3'
```

Normal IPv4 forwarding was not affected. The ping from h1 to h3 succeeded with 0% packet loss and TTL 62.

```
mininet> h1 ping -c 2 h3
PING 10.79.0.3 (10.79.0.3) 56(84) bytes of data.
64 bytes from 10.79.0.3: icmp_seq=1 ttl=62 time=8.15 ms
64 bytes from 10.79.0.3: icmp_seq=2 ttl=62 time=4.40 ms

--- 10.79.0.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 4.400/6.277/8.154/1.877 ms
```

I also tested boundary values. With --hops 1, the packet was forwarded by s1 but dropped at s2. With --hops 0, it was dropped immediately at s1. Neither packet reached the receivers.

Loop termination

I then recreated the same forwarding loop from Section 1, this time using the upgraded P4 program and an initial hop limit of 8.

The results from evidence/loop-after.txt were:
```
mininet> sh grep -c "Processing packet received" logs/s1.txt logs/s2.txt
logs/s1.txt:661
logs/s2.txt:659

mininet> sh sleep 3; grep -c "Processing packet received" logs/s1.txt logs/s2.txt
logs/s1.txt:666
logs/s2.txt:663

[16:08:07.764] [bmv2] [T] [thread 38088] [11.0] [cxt 0] Action MyIngress.drop
[16:08:07.764] [bmv2] [T] [thread 38088] [11.0] [cxt 0] tunnel.p4(97) Primitive mark_to_drop(standard_metadata)

```
Unlike the original loop, the counts stopped increasing. The nine packet-processing events correspond to eight successful forwarding operations and one final drop at s1 when the hop limit reached zero. The s1 log confirms execution of `MyIngress.drop` and `mark_to_drop(standard_metadata)`.

The s1 logs confirmed that MyIngress.drop was executed.

This matched my design: a packet starting with hop limit 8 can be forwarded eight times, but it will be dropped if it reaches another switch with a remaining value of zero.


## 5. Evaluation

a) The mixed rollout (mixed.txt):
Before the experiment, I predicted that the old s2 switch might have problems processing packets from the upgraded s1 because the new tunnel header contains an extra byte.

**Mixed version delivery **

The old switch still recognised the tunnel destination because proto_id and dst_id remained in the same positions. However, its parser expected the IPv4 header to begin immediately after the original four-byte tunnel header. The new format adds one byte before IPv4.

The actual bytes captured at h2 included:


```
17:11:35.204102 00:00:00:00:00:01 > ff:ff:ff:ff:ff:ff, ethertype Unknown (0x1212), length 52:
        0x0000:  ffff ffff ffff 0000 0000 0001 1212 0800
        0x0010:  0002 0745 0000 2100 0100 0040 0d66 2a0a
        0x0020:  4f00 010a 4f00 0310 e110 e100 0d7f a16d
        0x0030:  6978 6564

1 packet captured
1 packet received by filter
0 packets dropped by kernel

Listening on h2-eth0...
Listening on h3-eth0...
```

The old switch treated 07 as the first byte of IPv4 instead of recognising it as the hop limit. Surprisingly, it still matched the tunnel forwarding table and transmitted the packet through port 2.

My first tcpdump attempt captured no packets. After repeating the test, tcpdump successfully captured one 52-byte Ethernet frame at h2. The capture showed that the extra hop-limit byte and original payload were still present.

This means the old switch could forward the packet even though it interpreted the IPv4 header incorrectly. However, on application-level delivery was not success. The important lesson is that successful forwarding does not necessarily mean the packet was processed correctly.

**Mixed-version loop**

I also recreated the forwarding loop with s1 running the new program and s2 running the old program.

Before injection 673 670
After injection  682 678

The old s2 switch kept forwarding packets back to s1 without decrementing the hop limit. However, s1 still recognised and decreased the hop-limit byte whenever the packet returned.

After eight visits to s1 that allowed forwarding, the ninth visit triggered the zero-hop check.

The log confirmed:
```
Condition "hdr.myTunnel.hop_limit == 0" is true
Action MyIngress.drop
Dropping packet at the end of ingress
```
Therefore, the mixed-version loop still terminated, but it required 16 forwarding operations across both switches instead of eight.

This protection worked because the loop passed through an upgraded switch. If a loop only involved old switches, the new hop-limit mechanism would not protect it.

Full evidence is recorded in evidence/mixed.txt.

b) Safe rollout strategy

Option 1: Use a new EtherType
One way to solve this problem is to use a new EtherType instead of using the old 0x1212. This helps switches know whether a packet is using the old or new tunnel format.

The good thing about this is that older switches will not mistake the new packet format for the old one. However, older switches may not know how to handle the new EtherType, so some changes may be needed to make sure the packets can still pass through the network.

Option 2: Add a version field
Another way is to add a version field to the tunnel header. This field tells the switch which version of the tunnel format is being used.

Newer switches can check this field and handle both old and new packet formats. However, older switches may not understand the new field. So, adding a version field alone is not enough. The switches would also need to be updated to support it.

c) Data-plane versus control-plane loop protection

Week 3's check_paths() works in the control plane. It can analyse forwarding paths and detect possible loops before or when forwarding rules are installed. The control plane can help prevent incorrect paths from being configured in the first place, but it may not immediately detect every temporary problem, unexpected rule change or inconsistency between switches.

The data-plane hop limit can still stop an affected packet even when the forwarding configuration is wrong. However, it only limits how long the packet circulates and does not fix the incorrect forwarding rules. Real networks use both because the control plane tries to prevent loops, while the data plane provides a final layer of protection if a loop still occurs.


d) Hardware cost at line rate

The additional hop-limit field increases the tunnel header by one byte per packet.

A hardware switch would also need to extract the field, compare it with zero, decrement it when appropriate, and drop expired packets. These operations are relatively simple, but they still consume parser and processing resources.

Overall, the overhead should be fairly small compared with the benefit of preventing packets from circulating indefinitely. However, actual line-rate performance would need to be checked against the target hardware rather than assumed from the BMv2 software-switch tests.

## AI use

I used ChatGPT during this assignment to help me understand the P4 pipeline, plan the hop-limit implementation, interpret BMv2 logs, and organise the experimental evidence.

ChatGPT also suggested and provide guidance of initial code changes for tunnel.p4, send.py and receive.py, and helped draft explanations for each section.



## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
