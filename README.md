# node-protection-checker
Tool to check de facto node protection coverage for link-protected prefixes in TI-LFA

# Introduction

Unlike other local repair technologies, TI-LFA calculates backup paths to be on post-convergence path, so that traffic is forwarded over the optimal path immediately after failure. Using a non-post-convergence path can lead to performance degradation for a few seconds after failure (before global repair) due to traffic being forwarded via high-delay or low-bandwidth links.

Post-convergence path for link failure and node failure is not always the same. This tool checks SID with link protection enabled on your PLR and verifies whether they are protected from nexthop node failure. 

Based on the output, you can decide whether it makes sense to enable node protection in the network, or link protection is sufficient.

# Sample topologies

## Same post-convergence path for link and node failures

![TI-LFA-zero-segment.png](https://github.com/routingcraft/node-protection-checker/blob/master/images/TI-LFA-zero-segment.png)

In this topology, it doesn't matter whether link protection or node protection is enabled. Some TI-LFA implementations in this case will show that node protection is active even if it's not configured.

This will be the case for most Ring and Clos topologies with equal link metrics.

I refer to this case as **De Facto Protected SID type 1**

## Different post-convergence paths for link and node failures

![TI-LFA_node_protection_different_pc.png](https://github.com/routingcraft/node-protection-checker/blob/master/images/TI-LFA_node_protection_different_pc.png)

In this example, if R1-R2 link fails, R2 will be on the post-convergence path for traffic towards R3. So link protection will not protect against node failure.

But!

### De Facto node protection 

![TI-LFA_node_protection_defacto.png](https://github.com/routingcraft/node-protection-checker/blob/master/images/TI-LFA_node_protection_defacto.png)

Node failure is the failure of all links connected to the given node. It will be detected simultaneously by all directly connected routers (provided that BFD is configured properly)

If link protection is enabled on both R1 and R4 in this case, node protection will be provided by cascading effect of multiple link protections. This effect has been known for quite a while, since IP LFA. See [RFC6571#section-2](https://tools.ietf.org/html/rfc6571#section-2)

I refer to this case as **De Facto Protected SID type 2**

### De Facto node protection not possible

![TI-LFA_node_protection_defacto_loop.png](https://github.com/routingcraft/node-protection-checker/blob/master/images/TI-LFA_node_protection_defacto_loop.png)

In this example, R4's backup path for link protection points back to R1. While both R1-R2 and R4-R2 link are protected by link protection, there is no node protection against R2 failure. In order to protect traffic from R1 to R3 from node failure, you must configure node protection on R1. 

SID matching this scenario are printed as **SID that require node protection**

### No node protection possible

![TI-LFA_link_protection_no_node.png](https://github.com/routingcraft/node-protection-checker/blob/master/images/TI-LFA_link_protection_no_node.png)

If destination becomes completely isolated in case of node failure, no node protection is possible. Most TI-LFA implementation fallback to link protection in this case. 

SID matching this scenario are printed as **Link-protected SID with no node protection possible** with reason **SID isolated in case of node failure** and are not counted in statistics.


# Usage

You can run the script using the file with nodes to check as argument, or without any argument. In the latter case it will prompt to enter IP, username and password of the router to check.

Examples:

```
./np-checker npc_nodes.yml
```
npc_nodes.yml:

```
R1:
    username: "admin"
    password: "admin"
    nodeIp: "10.83.13.98"
    type: "eos"
```

It can contain many nodes in this format, np-checker will check all of them.

Run without arguments:

```
./np-checker
No argument specified
Enter credentials manually
Router IP: 10.83.13.98
Username: admin
Password:
```

## Sample output

```
./np-checker npc_nodes.yml

Connecting to node 10.83.13.98 type eos

IS-IS instance 1 on R1
*Note: only SID with active link protection are checked




Level: 2
Address Family: ipv4

De Facto Protected SID type 1
Prefix          SID        System ID
3.3.3.3/32      3          R3

De Facto Protected SID type 2
Prefix          SID        System ID
5.5.5.5/32      5          R5
4.4.4.4/32      4          R4

SID that require node protection
Prefix          SID        System ID
6.6.6.6/32      6          R6

De Facto type 1 coverage: 25.0%
De Facto total coverage: 75.0%

Link-protected SID with no node protection possible
Prefix          SID        System ID  Reason unprotected
2.2.2.2/32      2          R2         Directly connected to PLR



Level: 2
Address Family: ipv6

De Facto Protected SID type 1
Prefix          SID        System ID
2001::3/128     1003       R3

De Facto Protected SID type 2
Prefix          SID        System ID
2001::5/128     1005       R5
2001::4/128     1004       R4

SID that require node protection
Prefix          SID        System ID
2001::6/128     1006       R6

De Facto type 1 coverage: 25.0%
De Facto total coverage: 75.0%

Link-protected SID with no node protection possible
Prefix          SID        System ID  Reason unprotected
2001::2/128     1002       R2         Directly connected to PLR



Summary statistics for all address families and levels:

De Facto type 1 coverage: 25.0%
De Facto total coverage: 75.0%
```

The meaning of **De Facto Protected SID type 1** and **De Facto Protected SID type 2** is explained above. It is safe to assume those are protected against node failure as long as TI-LFA link protection is enabled on all routers in the network. **SID that require node protection** will not be protected against node failure, unless node protection is explicitly enabled. 

These 3 types of SID are represented in statistics which show de facto node protection coverage. Summary statistics show average for all address families and levels.

**Link-protected SID with no node protection possible** are not counted in statistics. In this example, no node protection is possible for SIDs originated by R2 since it is directly connected. Note that even if the router is directly connected but reachable via another router (because of better metrics), its SID can be protected against node failure and it will be counted.


# Support status


This is a prototype/proof of concept implentation, hence not much is supported. If there is any interest in this tool, I can add support for different platforms and segment types.


## Supported platforms

Currently only Arista EOS is supported. EAPI with HTTP transport must be enabled on the PLR that must be checked. 
```
management api http-commands
   protocol http
   no shutdown
```

## Supported protocols and segments

Only IS-IS is supported. Both IPv4 and IPv6, as well as L1 and L2 are supported. IPv6 multi-topology is not supported (nor is it supported in TI-LFA on EOS).

Only prefix SID with active link protection are checked. This means, if you don't have link protection enabled for some SID, or it is not available for whatever reason, np-checker will ignore that SID.

Multihomed (anycast or leaked from another level) SID are not supported. 

Global adjacency SID are not supported (yet). Local adjacency SID cannot get node protection for obvious reasons.

Also see caveats below for behaviour in possible corner cases.


# Caveats

## ECMP

If ECMP links have different nexthops, protection is possible (it will be **De Facto Protected SID type 1**, like in the first example). 

In the below topology, if the PLR calculates backup paths for ECMP segments, it the backup path will point to the same nexthop router as the primary path. Unless SRLG protection is used, there will be no node protection for this path. Since those links don't necessarily belong to the same SRLG, the assumption here is **no De Facto node protection.**

![TI-LFA-zero-segment_ECMP.png](https://github.com/routingcraft/node-protection-checker/blob/master/images/TI-LFA-zero-segment_ECMP.png)

This includes all scenarions when there are more than 2 ECMP paths and at least 2 of them are via the same nexthop node.

Note that if the PLR doesn't support TI-LFA for ECMP segments (e.g. Arista EOS as of 4.23.0), those segments will not be even checked.


### ECMP on nodes in the cascade

If SID is reachable via ECMP not from PLR, but from one of the nodes in the cascade (**De Facto Protected SID type 2**), all of the above applies to it as well. But since np-checker doesn't know what TI-LFA implementation runs on that router, it assumes it supports protection of ECMP SID. I hope this is not a big deal since all implementations should support it eventually.

### ECMP post-convergence paths

![TI-LFA_node_protection_defacto_loop_ECMP.png](https://github.com/routingcraft/node-protection-checker/blob/master/images/TI-LFA_node_protection_defacto_loop_ECMP.png)

In this example, when R4 will calculate post-convergence path for R4-R2 link failure, it will discover 2 ECMP routes, one of them via R1. What it will use depends on implementation - either ECMP backup route, or apply a tie-breaker to choose one and install it as backup. If it happens to choose the route via R5, De Facto node protection might work for traffic from R1 to R3, since R4 will not loop traffic back to R1.

I think its better to be conservative in this case, so np-checker will count R3's SID as unprotected (**SID that require node protection**) 


## Pseudonodes

No popular TI-LFA implemetation supports protection on multiaccess links. If one of ECMP links is a multiaccess link, or one of the nodes in the cascade (**De Facto Protected SID type 2**) sees protected SID via multiaccess link, there will be no protection. 

On the other hand, backup path may include multiaccess links. 

## Protected SID type

When checking for cascading link-protection effect (**De Facto Protected SID type 2**), we don't know what TI-LFA implementation run on other routers and what SID it can protect. There are certain assumptions I have to make, which might be wrong in certain corner cases.

If the PLR that is checked supports TI-LFA for a certain SID type, all other routers must also support it.

## IS-IS Configuration caveats

There are a couple of minor issues with the code, I will fix them when I have time.

* Secondary IP on interfaces are not supported
* All IS-IS routers must advertise hostname TLV 
* Only one IS-IS process, and only in the default VRF is supported

