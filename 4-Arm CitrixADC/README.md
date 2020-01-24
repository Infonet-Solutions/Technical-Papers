# 4-Arm CitrixADC Deployment

As per CitrixADC Best Practics you should deploy an ADC with only 1 network interface, for security reasons
That is, the appliance should be connected to only 1 security zone (A Demilitarized one would be better)<br/>
This is not really applicable in a complex network environment, because this would require a DMZ host to bypass a lot of security rules in order to do it's job<br/>
So here I'm going to discuss the most common implementation I see in my daily job

As a side note: HA, FIS, LACP, VMAC, VLAN, and other network stuffs will not be discussed here

## 4-Arm Deploy
This is the deploy I came up once asked to split data traffic from management traffic and have very stricted security zones in places.

Some clarifications and considerations before we start:

**NSIP**<br/>
Every CitrixADC has one and only one NSIP (CitrixADC IP)<br/>
This is the physical address of your CitrixADC<br/>
It is also the only IP you'll see at Operative System level with the `ifconfig` command

**SNIP**<br/>
The CitrixADC must have a SNIP in every subnet where it needs to operate<br/>

**PBF (PBR)**<br/>
Policy Based Forwarding (or Policy Based Routing or Advanced Routing, pick up the one you prefer) is the art of granularly configure our routing table<br/>
The main difference (but not the only one) between Classic and Advanced routing is that with PBR you can operate on "Sources" as well as on "Destinations"

**MBF, Layer2, Layer3 and Use SourceIP Modes**<br/>
I hope you understand all of these 4 modes and what does it mean to enable any of it<br/>
Remember that in most cases, using any of these modes indicates you're in a very bad designed network (and thus you're not my friend!)

## Deploy Logic

We'll came up with a CitrixADC connected to 4 different Security Zones:

**Management:** will be used from the IT Team to manage appliance. This zone will also be used for what I call "Administrative Queries", like LDAP/LDAPS, DNS, NTP, etc...<br/>
**Backend:** will be used by the CitrixADC to contact backend production servers such as Citrix Store Front, Citrix Delivery Controller, Web Servers, etc...<br/>
**Private-Frontend:** will be used by the CitrixADC to expose services for internal users<br/>
**Public-Frontend:** will be used by the CitrixADC to expose services for external users, usually behind a firewall NAT

**Note:** **Management** zone is used for Administrative Queries mainly to preserve nominal access even in case of a fault of the Backend Network Interfaces. If you have multiple domains for Management and Production, than you can contact management domain from **Management** zone, and Production domain from **Production** zone

Ok, after all the considerations...

**Let's Start!!!**

## Configuration
Our goal is to deploy a 4-arm CitrixADC<br/>
For our lab we'll use the following 4 security zones and addresses:

| Zone | Network | Gateway | Addresses | VLAN ID |
|---|---|---|---|---|
| Management | 10.1.1.0/24 | 10.1.1.254 | 10.1.1.8 - NSIP01<br/>10.1.1.9 - NSIP02<br/>10.1.1.10 - SNIP | 10 |
| Backend | 192.168.1.0/24 | 192.168.1.254 | 192.168.1.10 - SNIP | 20 |
| Private-Frontend | 172.16.1.0/24 | 172.16.1.254 | 172.16.1.10 - SNIP | 30 |
| Public-Frontned | 172.25.1.0/24 | 172.25.1.254 | 172.25.1.10 - SNIP | 40 |

This split between Private and Public frontend can be useful if you want to implement different kind of authentication based on user location<br/>
Also keep in mind that right now there's a 1:1 constraint while linking Citrix Gateway Virtual Servers into Content Switching Virtual Servers (Unified Gateway anyone?!)

On the first boot you need to provide the following stuffs (for management): 

- n° 1 NSIP
- n° 1 SNIP
- n° 1 Default Gateway (according to the above NSIP)
- n° 1+ DNS
- n° 1 NTP

After that, It's time to start splitting our traffic<br/>
Be sure to bind interfaces to their respective VLANs

    > bind vlan 10 -ifnum 0/1
    > bind vlan 10 -IPAddress 10.1.1.10 255.255.255.0
    > bind vlan 20 -ifnum 10/1
    > bind vlan 20 -IPAddress 192.168.1.10
    > bind vlan 30 -ifnum 1/1
    > bind vlan 30 -IPAddress 172.16.1.10
    > bind vlan 40 -ifnum 1/2
    > bind vlan 40 -IPAddress 172.25.1.10

We now need a PBR rule to orchestrate management traffic, the rule should be something like:

`Source` -> Every IP owned by the CitrixADC on the Management Network<br/>
`Destination` -> Every IP not owned by the CitrixADC on the Management Network<br/>
`Gateway` -> Specify the address that should be the actual Default Gateway

    > add ns pbr pbr_management_network ALLOW -srcIP = 10.1.1.8-10.1.1.10 -destIP "!=" 10.1.1.8-10.1.1.10 -nextHop 10.1.1.254 -priority 10 -kernelstate SFAPPLIED61
    > apply ns pbrs

Note that only IP owned by the CitrixADC are considered in the policy, not the whole `/24` network

After that, you can ensure PBR is working by looking at policy hits<br/>
After confirm the policy is working, you can change the default gateway of the appliance in `Configuration -> System -> Network -> Route`

    > add route 0.0.0.0 0.0.0.0 172.25.1.254
    > rm route 0.0.0.0 0.0.0.0 10.1.1.254

If everything goes right, you now have a working CitrixADC deploy with Advanced Routing in plance!

`Kudos to you!`

We need to reach Backend servers, you can accomplish this with both Classic and Advanced routing because of the above VLAN binding

    > add route 192.168.100.0 255.255.255.0 192.168.1.254

And we need to create a PBR for Administrative Queries (assuming 192.168.100.50 is our DomainController)

    > add ns pbr pbr_administrative_queries ALLOW -dstIP = 192.168.100.50 -nextHop 10.1.1.254 -priority 20

## Monitors

Once you'll start create services and vservers, you'll notice that monitors fails to contact backends<br/>
This is because for our CitrixADC monitor traffic and data traffic are 2 different kind of traffic<br/>
To ensure monitor traffic follow the correct path, you need to create a `Network Profile`<br/>
The `Network Profile` can then be used on Monitors, Services and vServers

    > add netProfile net_prof_backend -srcIP 192.168.1.10 -MBF DISABLED

## Troubleshooting

The only utility you need while troubleshooting PBR problems is `nstcpdump.sh`, a fork of `tcpdump` available in the CitrixADC shell<br/>
This combined with flags `-n` and `-e` will allow you understand on which interface traffic is going

    from tcpdump manual:
        -n     Don't convert addresses (i.e., host addresses, port numbers, etc.) to names
        -e     Print the link-level header on each dump line.  This can be used, for example, to print MAC layer addresses for protocols such as Ethernet and IEEE 802.11

    root@NS01# nstcpdump.sh -n -e 'host 192.168.100.214'
    reading from file -, link-type EN10MB (Ethernet)
    14:27:44.814810 00:e0:ed:7f:fe:ac > 00:50:56:bb:47:2d, ethertype IPv4 (0x0800), length 98: 192.168.1.10 > 192.168.100.214: ICMP echo request, id 34818, se q 0, length 64
    14:27:45.815150 00:e0:ed:7f:fe:ac > 00:50:56:bb:47:2d, ethertype IPv4 (0x0800), length 98: 192.168.1.10 > 192.168.100.214: ICMP echo request, id 34818, se q 1, length 64
    14:27:46.816222 00:e0:ed:7f:fe:ac > 00:50:56:bb:47:2d, ethertype IPv4 (0x0800), length 98: 192.168.1.10 > 192.168.100.214: ICMP echo request, id 34818, se q 2, length 64
    14:27:47.817277 00:e0:ed:7f:fe:ac > 00:50:56:bb:47:2d, ethertype IPv4 (0x0800), length 98: 192.168.1.10 > 192.168.100.214: ICMP echo request, id 34818, se q 3, length 64
    14:27:48.818377 00:e0:ed:7f:fe:ac > 00:50:56:bb:47:2d, ethertype IPv4 (0x0800), length 98: 192.168.1.10 > 192.168.100.214: ICMP echo request, id 34818, se q 4, length 64

Remember that you need to compare nstcpdump.sh MAC Address with the output of the command `show interfaces` from the Netscaler CLI (Not the Netscaler OS shell)<br/>
This can be different from the MAC Address in the GUI under `Configuration -> System -> Network -> Interfaces`

    > show interfaces

    8)      Interface 10/1 (10G Ethernet, unsupported passive DAC, 10 Gbit) #7
            flags=0x400c020 <ENABLED, UP, UP, autoneg, 802.1q>
            LACP <Active, Long timeout, key 1, priority 32768>
            MTU=1500, MAC=00:e0:ed:7f:fe:ac, uptime 747h32m37s
            Requested: media AUTO, speed AUTO, duplex AUTO, fctl OFF,
                    throughput 0
            Actual: media UTP, speed 10000, duplex FULL, fctl OFF, throughput 10000
            LLDP Mode: NONE,                 LR Priority: 1024

            RX: Pkts(1625438826) Bytes(148201814813) Errs(0) Drops(3296117) Stalls(0)
            TX: Pkts(2107243035) Bytes(165511449955) Errs(0) Drops(0) Stalls(0)
            NIC: InDisc(0) OutDisc(0) Fctls(0) Stalls(0) Hangs(0) Muted(0)
            Bandwidth thresholds are not set.
