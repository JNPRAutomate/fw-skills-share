# Intake questionnaire (ask every block on every run)

Discovered facts are only suggested defaults; the user confirms or changes each block. At most 3 questions per
round (tappable options where the client supports it). Never ask for the pre-shared key.

## Round 1 - goal (tappable)
1. What matters most: least traffic loss on planned failover / fastest BGP-BFD recovery / quiet logs?
   Show the Step 1 table (relative %), including that unplanned failures are flag-independent (about 75-120 % of
   the BFD detection time). If the flag will be on, state the replay-log notice and get an acknowledgement.

## Round 2 - devices and prerequisites
2. Hub node names (the MNHA pair, exact `get_router_list` names), the spoke, and the upstream router if any.
3. Has the user set the IKE proposal/policy **and the same pre-shared key** on every device? (yes/no)
4. Is `junos-ike` installed (hub required, spoke recommended) and has the node rebooted since? Is a maintenance
   window available (adding `managed-services ipsec` restarts the SRG)?

## Round 3 - hub anchor
5. Floating IKE address (/32) and lo0 unit, identical on both nodes.
6. The physical interface that receives IKE/ESP and its zone. The anchor loopback goes in the **same zone**.
7. SRG number (SRG1 or higher, never SRG0) and the name of the existing MNHA eBGP export policy.

## Round 4 - tunnel and routing
8. st0 unit, tunnel /30 (hub and spoke addresses) and its zone.
9. Hub AS (already on the pair), spoke AS, tunnel BFD timers (default 300 ms x 3), DPD (default 3 s x 3).
10. Hub subnets to advertise, spoke subnets to advertise.

## Round 5 - spoke underlay and cutover
11. Spoke IKE source address, external interface and zone, and the next hop toward the anchor.
12. Pre-tunnel (bypass) static routes to the hub subnets that must be removed at cutover.

## Round 6 - policies and tests
13. Security policies needed between the tunnel zone and the LAN zones on each side.
    `ALLOW-IKE-ESP` / `ALLOW-IKE-ESP-RETURN` are automatic, mandatory, and scoped to
    the spoke IKE address and the anchor. `any`/`any` needs acknowledgment.
14. Test host(s) behind each side (iperf3, or nping and tcpdump; a traffic-generator MCP server is optional) and who approves failover and interface-down tests. The spoke
    file adds the filtered `tunnel-ev` syslog (see testing.md).

After the last round show the complete sheet and obtain one "confirmed" before rendering.
