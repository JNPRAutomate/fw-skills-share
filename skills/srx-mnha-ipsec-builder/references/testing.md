# Testing method (planned and unplanned failover through the tunnel)

Commands for generating traffic (iperf3, a no-dependency UDP probe, ping, hping3, tcpdump) and for triggering
failures are in `traffic-generation.md`. A traffic-generator MCP server is optional; every step works by hand.

## Principle: measure your own baseline, report percentages
Absolute convergence times depend on platform, timers, hypervisor, upstream and traffic. Run the **baseline first**
(flag OFF, your timers), then each variant, and report the **% change versus that baseline** per direction
(how to read the log: `traffic-generation.md`, section 1). Never quote another lab's seconds as a target.

## Before any test
1. State: SRG ACTIVE on the intended node, both HEALTHY; tunnel BGP Established, BFD Up; the flag state is the one
   under test (`show chassis high-availability services-redundancy-group 1` -> "Process Packet In Backup State").
2. The spoke file already adds the filtered `tunnel-ev` syslog (see `config-blocks.md`).
   If it is missing, add it with approval and an undo line (`mcp-server-notes.md`).
   Check device clocks.
3. After changing `no-anti-replay`, run `clear security ipsec security-associations` on the peer and confirm a FRESH
   SA (remaining lifetime near the full lifetime): replay settings apply to new SAs only.

## Run
- One 150 s bidirectional UDP stream per variant (see `traffic-generation.md`, section 1 or 2).
- Planned: failover at about 20 % of the run, failback at about 65 %, each on the node that is ACTIVE at that moment.
- Unplanned: on the ACTIVE hub node disable the tunnel's underlay uplink at about 20 % of the run with
  `commit confirmed 2`; check the commit time (`show system commit`) lies inside the stream and that the automatic
  rollback falls after it. Before failing back, wait until the recovered node's BGP is Established upstream.
- Repeat each variant at least twice; one run is not a result.

## Read
- Data plane: loss windows per direction (`traffic-generation.md`, sections 1 and 2), as % of the baseline.
- Control plane: filtered log on the spoke - `BFDD_STATE_UP_TO_DOWN` to `BFDD_TRAP_SHOP_STATE_UP` is the tunnel BFD/BGP
  downtime; `RT_IPSEC_REPLAY` lines are replay messages. Cross-check event times with the times of your commands.
- Report UDP loss, not the TCP stall (TCP's stall is quantised by retransmit backoff).
- A zero-loss result is only valid if the event timestamp is inside the stream window.

## What to expect (relative, from a small virtual lab; yours will differ)
| Variant vs flag-OFF baseline (planned) | Data loss | Tunnel BFD/BGP recovery |
|---|---|---|
| Flag ON + floating statics | about -100 % both directions | about +70 % |
| Flag ON, spoke anti-replay off | about -100 % | unchanged; replay messages to 0 |
| Flag ON, hub BGP passive | about -100 % | unchanged; slower session re-establishment |
| Floating statics vs BGP-only | about -90 % | unchanged |
| Unplanned (flag ON vs OFF) | within about +/-20 % (noise) | unchanged |
Unplanned outage is roughly 75-120 % of the configured upstream BFD detection time (interval x multiplier). With the flag
OFF, expect hub-to-spoke planned loss several times the spoke-to-hub loss (about 4 x in our lab).
