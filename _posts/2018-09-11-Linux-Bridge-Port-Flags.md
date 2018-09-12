---
layout: post
title: Linux Bridge Port Flags
tags: [Linux Bridge, Network, Linux Code Walkthrough]
---

# Overview
Linux bridge can set flags on a port. The defined flags (until Linux 4.0) are:

```c
#define BR_HAIRPIN_MODE     BIT(0)
#define BR_BPDU_GUARD       BIT(1)
#define BR_ROOT_BLOCK       BIT(2)
#define BR_MULTICAST_FAST_LEAVE BIT(3)
#define BR_ADMIN_COST       BIT(4)
#define BR_LEARNING         BIT(5)
#define BR_FLOOD            BIT(6)
#define BR_AUTO_MASK        (BR_FLOOD | BR_LEARNING)
#define BR_PROMISC          BIT(7)
#define BR_PROXYARP         BIT(8)
#define BR_LEARNING_SYNC    BIT(9)
```

Unfortunately the code doesn't have comments explaining what does these flags
mean. Thus, we have to figure it out from the code. The bridge [man
page](http://man7.org/linux/man-pages/man8/bridge.8.html) is also very helpful.

The code we are going to walk through is [Linux
4.0](https://elixir.bootlin.com/linux/v4.0/source).

# BR_HAIRPIN_MODE
This flag controls whether traffic may be sent back out of the port on which it
was received. By default, this flag is turned off and the bridge will not
forward traffic back out of the receiving port. [2]

## Code

In file [br_forward.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_forward.c#L33)

```c
static inline int should_deliver(const struct net_bridge_port *p,
                 const struct sk_buff *skb)
{
    return ((p->flags & BR_HAIRPIN_MODE) || skb->dev != p->dev) &&
        br_allowed_egress(p->br, nbp_get_vlan_info(p), skb) &&
        p->state == BR_STATE_FORWARDING;
}
```
The function returns true if the packet should be delivered to the port. The
first condition it checks is whether `BR_HAIRPIN_MODE` flag is set on the given
port. If `BR_HAIRPIN_MODE` is not set, then the packet will be delivered to the
port only if it is not the port on which the packet was received.

## Purpose
Hairpin mode was added to support basic VEPA (Virtual Ethernet Port Aggregator)
capabilities. For more information, see [this
commit](https://patchwork.ozlabs.org/patch/31341/)

# BR_BPDU_GUARD
This flag Controls whether STP BPDUs will be processed by the bridge port. By
default, the flag is turned off allowed BPDU processing. Turning this flag on
will cause the port to stop processing STP BPDUs. [2]

## Code

In file [br_stp_bpdu.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_stp_bpdu.c#L175)
```c
void br_stp_rcv(const struct stp_proto *proto, struct sk_buff *skb,
		struct net_device *dev)
{
    ...

	if (p->flags & BR_BPDU_GUARD) {
		br_notice(br, "BPDU received on blocked port %u(%s)\n",
			  (unsigned int) p->port_no, p->dev->name);
		br_stp_disable_port(p);
		goto out;
	}

    ...
}
```
This function is called when a BPDU is received by the bridge. It first checks
whether this BPDU needs to be processed. One of the condition is to check
whether `BR_BPDU_GUARD` flag is set on the port. If it is set, then stop
processing and exit.

## Purpose
If running Spanning Tree on bridge, hostile devices on the network may send BPDU
and cause network failure. Enabling BPDU block will detect and stop this. For
more information, see [this commit](https://patchwork.ozlabs.org/patch/198752/).

# BR_ROOT_BLOCK
This flag controls whether a given port is allowed to become root port or not.
Only used when STP is enabled on the bridge. By default the flag is off. [2]

## Code

In file [br_stp.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_stp.c#L140)
```c
static void br_root_selection(struct net_bridge *br)
{
	struct net_bridge_port *p;
	u16 root_port = 0;

	list_for_each_entry(p, &br->port_list, list) {
		if (!br_should_become_root_port(p, root_port))
			continue;

		if (p->flags & BR_ROOT_BLOCK)
			br_root_port_block(br, p);
		else
			root_port = p->port_no;
	}
    ...
}
```
As we mentioned [here](/2018/08/08/The-Spanning-Tree-Protocol/), in STP, each
bridge needs to select a root port. As the name implies, this function is used
for root port selection. And the flag `BR_ROOT_BLOCK` can be set by the admin to
prevent a port being selected as the root port.

## Purpose
If using STP on a bridge and the downstream bridges are not fully trusted; this
prevents a hostile guest for rerouting traffic. For more information, see [this
commit](https://patchwork.ozlabs.org/patch/198751/)

# BR_MULTICAST_FAST_LEAVE
This flag allows the bridge to immediately stop multicast traffic on a port that
receives IGMP Leave message. It is only used with IGMP snooping is enabled on
the bridge. By default the flag is off. [2]

## Code
In file
[br_multicast.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_multicast.c#L1454)
```c
static void
br_multicast_leave_group(struct net_bridge *br,
			 struct net_bridge_port *port,
			 struct br_ip *group,
			 struct bridge_mcast_other_query *other_query,
			 struct bridge_mcast_own_query *own_query)
{
    ...
    mdb = mlock_dereference(br->mdb, br);
	mp = br_mdb_ip_get(mdb, group);
    ...
    if (port && (port->flags & BR_MULTICAST_FAST_LEAVE)) {
		struct net_bridge_port_group __rcu **pp;

		for (pp = &mp->ports;
		     (p = mlock_dereference(*pp, br)) != NULL;
		     pp = &p->next) {
			if (p->port != port)
				continue;

			rcu_assign_pointer(*pp, p->next);
			hlist_del_init(&p->mglist);
			del_timer(&p->timer);
			call_rcu_bh(&p->rcu, br_multicast_free_pg);
			br_mdb_notify(br->dev, port, group, RTM_DELMDB);

			if (!mp->ports && !mp->mglist &&
			    netif_running(br->dev))
				mod_timer(&mp->timer, jiffies);
		}
		goto out;
	}
    ...
}
```
This function is called when a port receives IGMP Leave message. If
`BR_MULTICAST_FAST_LEAVE` flag is set, it immediately removes the port from the
multicast group by deleting the entry from MDB (multicast database). Otherwise
the entry will be deleted after a timeout.

## Purpose
Fast leave allows bridge to immediately stops the multicast traffic on the port
receives IGMP Leave when IGMP snooping is enabled, no timeouts are observed. See
[this commit](https://patchwork.ozlabs.org/patch/203588/).

# BR_ADMIN_COST
When STP is enabled on a bridge, if this flag is set, it means that the admin
has set the path cost of the port. Otherwise a default value is used.

## Code
In file
[br_stp_if.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_stp_if.c#L301)
```c
int br_stp_set_path_cost(struct net_bridge_port *p, unsigned long path_cost)
{
	if (path_cost < BR_MIN_PATH_COST ||
	    path_cost > BR_MAX_PATH_COST)
		return -ERANGE;

	p->flags |= BR_ADMIN_COST;
	p->path_cost = path_cost;
	br_configuration_update(p->br);
	br_port_state_selection(p->br);
	return 0;
}
```
This function is called when the admin sets the port's path cost.
`BR_ADMIN_COST` is set so that the given cost is used rather than the default
value.

In file [br_if.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_if.c#L70)
```c
void br_port_carrier_check(struct net_bridge_port *p)
{
	struct net_device *dev = p->dev;
	struct net_bridge *br = p->br;

	if (!(p->flags & BR_ADMIN_COST) &&
	    netif_running(dev) && netif_oper_up(dev))
		p->path_cost = port_cost(dev);
    ...
}
```
Above code shows that when `BR_ADMIN_COST` is not set, the path cost is gotten
from `port_cost()` function which returns a default value based on link speed.

# BR_LEARNING
Controls whether a given port will learn MAC addresses from received traffic or
not. If learning if off, the bridge will end up flooding any traffic for which
it has no FDB entry. By default this flag is on. [2]

## Code
In file
[br_input.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_input.c#L136)

```c
int br_handle_frame_finish(struct sk_buff *skb)
{
    struct net_bridge_port *p = br_port_get_rcu(skb->dev);
    ...
    if (p->flags & BR_LEARNING)
		br_fdb_update(br, p, eth_hdr(skb)->h_source, vid, false);
    ...
}
```

This function is called when a frame is received by the bridge. The bridge will
learn the source address and update FDB (forwarding database) only if
`BR_LEARNING` flag is set on the port.

## Purpose
Allow user to control whether mac learning is enabled on the port. See [this
commit](https://patchwork.ozlabs.org/patch/249074/).

# BR_FLOOD
Controls whether a given port will flood unicast traffic for which there is no
FDB entry. By default this flag is on.

## Code
In file
[br_forward.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_forward.c#L185)

```c
static void br_flood(struct net_bridge *br, struct sk_buff *skb,
		     struct sk_buff *skb0,
		     void (*__packet_hook)(const struct net_bridge_port *p,
					   struct sk_buff *skb),
		     bool unicast)
{
	struct net_bridge_port *p;
	struct net_bridge_port *prev;

	prev = NULL;

	list_for_each_entry_rcu(p, &br->port_list, list) {
		/* Do not flood unicast traffic to ports that turn it off */
		if (unicast && !(p->flags & BR_FLOOD))
			continue;

		/* Do not flood to ports that enable proxy ARP */
		if (p->flags & BR_PROXYARP)
			continue;

		prev = maybe_deliver(prev, p, skb, __packet_hook);
		if (IS_ERR(prev))
			goto out;
	}
    ...
}
```
This function is called when the bridge receives a BUM (broadcast, unknown
unicast, multicast) frame. If the destination address is an unknown unicast and
`BR_FLOOD` flag is not set on a port, then the frame won't be delivered to the
port.

## Purpose
Add a flag to control flood of unicast traffic. See [this
commit](https://patchwork.ozlabs.org/patch/249073/).

# BR_AUTO_MASK
`BR_AUTO_MASK` is defined as `BR_FLOOD | BR_LEARNING`. It is added to keep track
of ports capable of automatic discovery. Automatic discovery means discovering
the nodes located behind the port and it is accomplished via flooding of unknown
traffic (`BR_FLOOD`) and learning the mac addresses from these packets
(`BR_LEARNING`).

## Code
In file
[br_if.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_if.c#L584)
```c
void br_port_flags_change(struct net_bridge_port *p, unsigned long mask)
{
	struct net_bridge *br = p->br;

	if (mask & BR_AUTO_MASK)
		nbp_update_port_count(br);
}
```
This function is called when the flags on a port are changed. If the port has
automatic discovery capability, then it calls `nbp_update_port_count` to update
the count of these type of ports.

# Purpose
This patch adds functionality to keep track of all ports capable of automatic
discovery. This will later be used to control promiscuity settings. See [this
commit](https://patchwork.ozlabs.org/patch/349608/).

# BR_PROMISC
This flag is set when a port is set to promiscuous mode.

## Code
In file
[br_fdb.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_fdb.c#L101)

```c
static void fdb_add_hw_addr(struct net_bridge *br, const unsigned char *addr)
{
	int err;
	struct net_bridge_port *p;

	ASSERT_RTNL();

	list_for_each_entry(p, &br->port_list, list) {
		if (!br_promisc_port(p)) {
			err = dev_uc_add(p->dev, addr);
			if (err)
				goto undo;
		}
	}

	return;
undo:
	list_for_each_entry_continue_reverse(p, &br->port_list, list) {
		if (!br_promisc_port(p))
			dev_uc_del(p->dev, addr);
	}
}
```
This function is called to add a static FDB entry. All non-promiscuous ports are
also updated with the new information.

## Purpose
When a static fdb entry is created, add the mac address from this fdb entry to
any ports that are currently running in non-promiscuous mode.  These ports need
this data so that they can receive traffic destined to these addresses. See [this
commit](https://patchwork.ozlabs.org/patch/349605/).

# BR_PROXYARP
[Proxy ARP](https://en.wikipedia.org/wiki/Proxy_ARP) is enabled on the port when
this flag is set.

## Code
In file
[br_input.c](https://elixir.bootlin.com/linux/v4.0/source/net/bridge/br_input.c#L158)
```c
int br_handle_frame_finish(struct sk_buff *skb)
{
    ...
    if (is_broadcast_ether_addr(dest)) {
		if (IS_ENABLED(CONFIG_INET) &&
		    p->flags & BR_PROXYARP &&
		    skb->protocol == htons(ETH_P_ARP))
			br_do_proxy_arp(skb, br, vid);

		skb2 = skb;
		unicast = false;
	}
    ...
}
```
This function is called when a frame is received by the bridge. If the
destination is a broadcast address and `BR_PROXYARP` flag is set and the
protocol is ARP, then it does [proxy
arp](https://en.wikipedia.org/wiki/Proxy_ARP), meaning that it replies the ARP
request on behalf of the owner of the target node.

```c
static void br_do_proxy_arp(struct sk_buff *skb, struct net_bridge *br,
			    u16 vid)
{
	struct net_device *dev = br->dev;
	struct neighbour *n;
	struct arphdr *parp;
	u8 *arpptr, *sha;
	__be32 sip, tip;

	if (dev->flags & IFF_NOARP)
		return;

	if (!pskb_may_pull(skb, arp_hdr_len(dev))) {
		dev->stats.tx_dropped++;
		return;
	}
	parp = arp_hdr(skb);

	if (parp->ar_pro != htons(ETH_P_IP) ||
	    parp->ar_op != htons(ARPOP_REQUEST) ||
	    parp->ar_hln != dev->addr_len ||
	    parp->ar_pln != 4)
		return;

	arpptr = (u8 *)parp + sizeof(struct arphdr);
	sha = arpptr;
	arpptr += dev->addr_len;	/* sha */
	memcpy(&sip, arpptr, sizeof(sip));
	arpptr += sizeof(sip);
	arpptr += dev->addr_len;	/* tha */
	memcpy(&tip, arpptr, sizeof(tip));

	if (ipv4_is_loopback(tip) ||
	    ipv4_is_multicast(tip))
		return;

	n = neigh_lookup(&arp_tbl, &tip, dev);
	if (n) {
		struct net_bridge_fdb_entry *f;

		if (!(n->nud_state & NUD_VALID)) {
			neigh_release(n);
			return;
		}

		f = __br_fdb_get(br, n->ha, vid);
		if (f)
			arp_send(ARPOP_REPLY, ETH_P_ARP, sip, skb->dev, tip,
				 sha, n->ha, sha);

		neigh_release(n);
	}
}
```

## Purpose
This feature is defined in IEEE Std 802.11-2012, 10.23.13. It allows the AP
devices to keep track of the hardware-address-to-IP-address mapping of the
mobile devices within the WLAN network. See [this
commit](https://patchwork.ozlabs.org/patch/402697/).

# BR_LEARNING_SYNC
Controls whether a given port will sync MAC addresses learned on device port to
bridge FDB. [2]

## Code
Following is from Rocker switch device driver
[rocker.c](https://elixir.bootlin.com/linux/v4.0/source/drivers/net/ethernet/rocker/rocker.c)

```c
static int rocker_port_fdb_learn(struct rocker_port *rocker_port,
				 int flags, const u8 *addr, __be16 vlan_id)
{
	struct rocker_fdb_learn_work *lw;
	enum rocker_of_dpa_table_id goto_tbl =
		ROCKER_OF_DPA_TABLE_ID_ACL_POLICY;
	u32 out_lport = rocker_port->lport;
	u32 tunnel_id = 0;
	u32 group_id = ROCKER_GROUP_NONE;
	bool syncing = !!(rocker_port->brport_flags & BR_LEARNING_SYNC);

    ...

	if (!syncing)
		return 0;

	if (!rocker_port_is_bridged(rocker_port))
		return 0;

	lw = kmalloc(sizeof(*lw), rocker_op_flags_gfp(flags));
	if (!lw)
		return -ENOMEM;

	INIT_WORK(&lw->work, rocker_port_fdb_learn_work);

    ...
}
```
This function is called when Rocker switch device learns a FDB entry. If
`BR_LEARNING_SYNC` is set on the port, then it will sync the new information
with Linux bridge.

`rocker_port_fdb_learn_work` is shown below. It synchronizes the information
with Linux bridge by generating `NETDEV_SWITCH_FDB_DEL` or
`NETDEV_SWITCH_FDB_ADD` events.

```c
static void rocker_port_fdb_learn_work(struct work_struct *work)
{
	struct rocker_fdb_learn_work *lw =
		container_of(work, struct rocker_fdb_learn_work, work);
	bool removing = (lw->flags & ROCKER_OP_FLAG_REMOVE);
	bool learned = (lw->flags & ROCKER_OP_FLAG_LEARNED);
	struct netdev_switch_notifier_fdb_info info;

	info.addr = lw->addr;
	info.vid = lw->vid;

	if (learned && removing)
		call_netdev_switch_notifiers(NETDEV_SWITCH_FDB_DEL,
					     lw->dev, &info.info);
	else if (learned && !removing)
		call_netdev_switch_notifiers(NETDEV_SWITCH_FDB_ADD,
					     lw->dev, &info.info);

	kfree(work);
}
```

## Purpose
This policy flag controls syncing of learned FDB entries to bridge's FDB.  If
on, FDB entries learned on bridge port device will be synced. If off, device
may still learn new FDB entries but they will not be synced with bridge's FDB.
See [this commit](https://patchwork.ozlabs.org/patch/415862/).

# Reference
[1] [if_bridge.h](https://elixir.bootlin.com/linux/v4.0/source/include/linux/if_bridge.h)<br>
[2] [BRIDGE 8](http://man7.org/linux/man-pages/man8/bridge.8.html)<br>
