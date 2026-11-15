# MNHA + IPsec hub-and-spoke: requirements, patterns and measured effects

Validated on virtual SRX (Junos 24.4, flat MNHA model): an MNHA pair as hub, one spoke, an upstream router.
Aligns with the HPE/Juniper TechPost "MNHA, IPSec and Multiple Routing Instances" and Juniper's MNHA IPsec VPN documentation.

> **How to read the numbers:** convergence times depend on platform, timers, hypervisor, upstream and traffic, so
> they do not transfer between labs or production. This reference gives **percentages relative to a baseline you
> measure yourself on the same path** (flag off, same timers). Treat them as indicative proportions, not guarantees,
> and measure your own baseline first (see `testing.md`).

## Hard requirements (a tunnel breaks if one is missing)
1. **Floating anchor loopback.** The IKE gateway `external-interface` and `local-address` are a /32 on `lo0.<unit>`,
   configured identically on both hub nodes, listed in a prefix list attached to SRG1+ with `managed-services ipsec`.
   Only the active node advertises it, so the tunnel follows SRG1.
2. **Same zone.** The anchor loopback and the physical interface that receives IKE/ESP must be in the same security
   zone, otherwise packets are dropped ("re-route failed").
3. **Same routing instance.** The ICL, the floating loopback and that physical interface must share one routing
   instance; IKE gateway lookup is resolved in the ICL's instance and route leaking does not help ("Gateway lookup failed").
4. **`junos-ike` process.** Required on the MNHA hub (`request system software add optional://junos-ike.tgz`); it takes
   effect only after a reboot. A non-MNHA spoke can run the legacy IKE process, so on the spoke it is recommended;
   confirm an IKE process is actually running. A node with the package installed but not rebooted shows empty
   `show security ike|ipsec ...` output and never initiates.
5. **SRG1 or higher**, never SRG0, for `managed-services ipsec`.

## The intra-zone `ALLOW-IKE-ESP` policy is REQUIRED (even at 0 hits)
The anchor loopback and the IKE/ESP-facing interface share a zone, so IKE (UDP 500/4500) and ESP (protocol 50) that
build the tunnel are **intra-zone** traffic and need an explicit permit. The policy can show **0 hits** in
`show security policies hit-count` because tunnel traffic terminating on the device's own loopback is largely
host-inbound and is not consistently counted. Never judge it by hit count and never remove it because it reads zero.
```
set security address-book global address IKE-SPOKE <spoke-ike>/32
set security address-book global address IKE-ANCHOR <anchor>/32
set applications application VPN-ESP protocol esp
set security policies from-zone <z> to-zone <z> policy ALLOW-IKE-ESP match source-address IKE-SPOKE
set security policies from-zone <z> to-zone <z> policy ALLOW-IKE-ESP match destination-address IKE-ANCHOR
set security policies from-zone <z> to-zone <z> policy ALLOW-IKE-ESP match application [ junos-ike junos-ike-nat VPN-ESP ]
set security policies from-zone <z> to-zone <z> policy ALLOW-IKE-ESP then permit
set security policies from-zone <z> to-zone <z> policy ALLOW-IKE-ESP-RETURN match source-address IKE-ANCHOR
set security policies from-zone <z> to-zone <z> policy ALLOW-IKE-ESP-RETURN match destination-address IKE-SPOKE
set security policies from-zone <z> to-zone <z> policy ALLOW-IKE-ESP-RETURN match application [ junos-ike junos-ike-nat VPN-ESP ]
set security policies from-zone <z> to-zone <z> policy ALLOW-IKE-ESP-RETURN then permit
```
Write both directions by default. `any`/`any` is a sheet override (`vpn.ike.allow_any`)
that needs acknowledgment, because it also permits IKE and ESP transit in that zone.

## Routing over the tunnel: three patterns
- **Documented by Juniper for dynamic routing: node-local tunnels** (one tunnel from the peer to each hub node, not
  tied to an SRG and not synced; routes stay local to each node). Not tested here.
- **Traffic selectors + Auto Route Insertion** (synced tunnel, no routing protocol): the remote prefix is installed on
  both nodes, so nothing re-converges on failover. Not tested here.
- **eBGP over a synced tunnel + floating statics** (what the skill builds; field-built, outside Juniper's documented
  pattern for dynamic routing, treat as experimental). The synced IPsec SA moves with SRG1, but the tunnel BGP/BFD
  session is node-local and must re-establish on the new active node. A **floating static** for the far-end prefix over
  `st0` at preference 250 on **both** ends carries traffic meanwhile: planned-failover loss dropped by about **90 %**
  compared with BGP alone. Do **not** add BFD to that static: BFD is node-local too and would flap with the failover.

## `process-packet-on-backup` (SRG option)
**Documented purpose** (Juniper, "IPsec VPN Support in Multinode High Availability"): the backup node's packet
forwarding engine also processes the SRG's VPN packets while not active, removing the delay when it becomes active
after a failover. IPsec-specific; sessions are synchronised with or without it.

> **NOTICE - always show this to the user whenever the flag is recommended or enabled:** during a failover or
> failback (and occasionally outside failovers, around a child-SA rekey or in steady state) the peer may log
> `RT_IPSEC: RT_IPSEC_REPLAY: Replay packet detected on IPSec tunnel ... From <hub anchor> to <peer>, ESP, SPI ..., SEQ ...`.
> In our flag-on runs with anti-replay left on, such messages appeared at most once per switchover event (none with the
> flag off). They were reproducible and, in our tests, harmless (no loss attributable to them). They can be silenced with
> `ike no-anti-replay` on the peer (new SA required), but that removes replay protection on that tunnel, so it is
> not recommended just to quiet the log. Tell operators to expect them and not to treat them as a fault.

### Relative effect of the flag (ON vs OFF), planned switchover, floating statics present
Baseline = flag OFF in the same lab; both directions loaded simultaneously.
| Metric | Change with the flag ON |
|---|---|
| Data loss, spoke to hub | about **-100 %** (none measured in any flag-on run) |
| Data loss, hub to spoke | about **-99.9 %** (a few datagrams versus thousands) |
| Tunnel BFD/BGP recovery time | about **+70 %** (range +65 % to +100 % across runs) |
| Replay messages on the peer per switchover | from 0 to roughly 0.7 |
| With the flag OFF: hub-to-spoke loss versus spoke-to-hub loss | about **4 x** higher (the one-way tests hid it) |
| Disabling anti-replay on the peer (flag ON) | replay messages to 0; recovery time unchanged (**0 %**) |
| BGP `passive` on the hub (flag ON) | no measurable benefit (**0 %**); slower session re-establishment |

Without floating statics, flag ON with BGP alone is **worse** than flag OFF because data waits for the slower BGP recovery.

### Unplanned failures (the active node's uplink goes down)
| Comparison | Change |
|---|---|
| Flag ON vs OFF, spoke to hub | about -19 % (one run each, within run-to-run noise) |
| Flag ON vs OFF, hub to spoke | about +2 % |
| Flag ON vs OFF, tunnel BFD/BGP downtime | 0 % |
| Unplanned outage vs the configured upstream BFD detection time (interval x multiplier) | roughly **75 % to 120 %** |

The flag does not materially change an unplanned outage; detection time bounds it (shortening detection time by ~40 %
shortened the outage by a similar ~40 % in our MNHA tests). The flag's benefit is for planned switchovers.

### Other observations
- With the flag ON the **backup** node still originates a trickle of packets into the tunnel (check `show interfaces st0.<n>
  statistics` twice on the backup) while its tunnel BGP sits in Connect.
- Occasional single lost datagrams hub-to-spoke coincide with replay messages and with route swaps (static to BGP);
  expect an occasional lost datagram, not literally zero forever.
- Idle BFD/BGP flaps were seen with the flag ON when no test was running, once together with a child-SA rekey (new SPI).
  **Test next:** force a rekey under load (`clear security ipsec security-associations` on the spoke mid-stream) with the
  flag on and off, and watch both directions.

## Operational caveats
- **Adding `managed-services ipsec` to a running SRG restarts it:** the VIP is withdrawn for a period measured in
  minutes on the active node. Use a maintenance window or do it during the initial MNHA build; backup node first.
- **Return-path asymmetry during cutover:** when replacing a pre-tunnel path (a bypass static on the spoke), the hub starts
  returning traffic through the tunnel as soon as the overlay route is up while the spoke may still use the old path.
  Cut over immediately after the overlay route is up, with commit confirmed.
- **The hub anchor does not answer ping** (intra-zone policy is IKE/ESP only); verify with IKE/IPsec SAs.
- A change to `no-anti-replay` applies to **new** SAs only: clear the SAs on the peer and confirm a fresh SA.

## Goal-based configuration guide
| If the user wants... | Configure | Relative result | Cost / caveat |
|---|---|---|---|
| Least traffic loss on planned failover | flag ON + floating statics on both ends | loss about -100 % both directions | BFD/BGP recovery about +70 %; **replay log messages may appear on the peer** |
| Fastest BGP/BFD recovery, simplest | flag OFF + floating statics | baseline recovery | hub-to-spoke planned loss about 4 x the spoke-to-hub loss |
| Quiet logs with the flag ON | add `ike no-anti-replay` on the peer (+ clear SAs) | replay messages to 0, timing 0 % change | removes replay protection on that tunnel |
| Dynamic routing per Juniper docs | node-local tunnels | not tested | see routing patterns |
| No routing protocol over the tunnel | traffic selectors + Auto Route Insertion | not tested | nothing re-converges |

Unplanned uplink failure: flag-independent, roughly 75-120 % of the BFD detection time.
Not covered: child-SA rekey under load, more than one spoke, node-local tunnels, ARI.
